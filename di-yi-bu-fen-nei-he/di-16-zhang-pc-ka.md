# 第 16 章 PC 卡

本章将讨论如何为 PC Card 或 CardBus 设备编写 FreeBSD 设备驱动程序的机制。然而，目前它仅记录了如何向现有的 pccard 驱动程序中添加新设备。

## 16.1. 添加设备

设备驱动程序知道它们支持哪些设备。内核中有一个支持设备的表格，驱动程序使用它来附加到设备。

### 16.1.1. 概述

PC 卡通过两种方式之一进行识别，这两种方式都基于存储在卡上的 *Card Information Structure*（CIS）。第一种方法是使用数字制造商和产品编号。第二种方法是使用同样包含在 CIS 中的可读字符串。PC 卡总线使用一个集中式数据库和一些宏来促进设计模式，帮助驱动程序编写者将设备与其驱动程序匹配。

原始设备制造商（OEM）通常会开发一个 PC 卡产品的参考设计，然后将这个设计出售给其他公司进行市场营销。这些公司会改进设计、将产品推广到他们的目标受众或地区，并在卡上加上他们自己的品牌标签。通常这些卡的物理改进非常小，或者根本没有改动。为了增强品牌，这些厂商会将公司名称放入 CIS 空间中的可读字符串中，但保持制造商和产品 ID 不变。

由于这种做法，FreeBSD 驱动程序通常依赖数字 ID 来识别设备。使用数字 ID 和集中式数据库增加 ID 和卡支持到系统中变得复杂。必须仔细检查卡的真正制造商，特别是在发现卡的制造商可能已经在集中数据库中列出了不同的制造商 ID 时。Linksys、D-Link 和 NetGear 是一些美国制造商，通常会出售相同的设计。这些相同的设计可能会以 Buffalo 和 Corega 等名称在日本销售。这些设备通常会有相同的制造商和产品 ID。

PC 卡总线代码保持着一个卡信息的中央数据库，但不包含与之关联的驱动程序，位于 **/sys/dev/pccard/pccarddevs**。它还提供了一组宏，允许我们轻松地在驱动程序用于声明设备的表格中构建简单的条目。

最后，一些非常低端的设备根本没有包含制造商标识。这些设备必须通过匹配可读的 CIS 字符串来检测。尽管我们希望不必将此方法作为后备方案，但对于一些非常低端的 CD-ROM 播放器和以太网卡来说，这是必要的。这种方法通常应该避免，但有一些设备在这个章节中列出，因为它们是在意识到 PC 卡业务的 OEM 特性之前添加的。在添加新设备时，建议优先使用数字方法。

### 16.1.2. **pccarddevs** 文件格式

**pccarddevs** 文件分为四个部分。第一部分列出了使用制造商编号的供应商。此部分按数字顺序排序。接下来的部分列出了这些供应商使用的所有产品，以及它们的产品 ID 和描述字符串。描述字符串通常不使用（即使我们根据数字版本匹配设备，仍然基于人类可读的 CIS 设置设备的描述）。这两个部分随后会为使用字符串匹配方法的设备重复一次。最后，C 风格的注释被允许出现在文件中的任何位置，注释以 `/*` 和 `*/` 符号括起来。

文件的第一部分包含了供应商 ID。请保持这个列表按数字顺序排序。同时，请协调对此文件的更改，因为我们与 NetBSD 共享该文件，以帮助促进这一信息的公共清算。例如，以下是前几个供应商 ID：

```c
vendor FUJITSU			0x0004  Fujitsu Corporation
vendor NETGEAR_2		0x000b  Netgear
vendor PANASONIC		0x0032	Matsushita Electric Industrial Co.
vendor SANDISK			0x0045	Sandisk Corporation
```

`NETGEAR_2` 条目的出现很可能是 Netgear 从某个 OEM 采购的卡片，而当时支持这些卡的作者并未意识到 Netgear 使用了他人 ID。这些条目非常简单。`vendor` 关键字表示这一行的类型，后面跟随供应商的名称。这个名称将在 **pccarddevs** 中重复，并在驱动程序的匹配表中使用，因此保持简短，并确保它是有效的 C 标识符。接下来是以十六进制表示的供应商的数字 ID。不要添加 `0xffffffff` 或 `0xffff` 形式的 ID，因为这些是保留的 ID（前者表示“未设置 ID”，后者有时出现在极差质量的卡中，试图表示“无”）。最后是一个描述制造商的字符串。此字符串在 FreeBSD 中仅用于注释目的，其他地方不会使用。

文件的第二部分包含了产品。在这个例子中，格式与供应商行类似：

```c
/* Allied Telesis K.K. */
product ALLIEDTELESIS LA_PCM	0x0002 Allied Telesis LA-PCM

/* Archos */
product	ARCHOS ARC_ATAPI	0x0043 MiniCD
```

`product` 关键字后跟供应商名称，与前面重复。接着是产品名称，驱动程序将使用此名称，它应为有效的 C 标识符，但也可以以数字开头。与供应商一样，该卡的十六进制产品 ID 遵循相同的 `0xffffffff` 和 `0xffff` 规则。最后是设备本身的描述字符串。该字符串通常不会在 FreeBSD 中使用，因为 FreeBSD 的 pccard 总线驱动程序会根据人类可读的 CIS 条目构造一个字符串，但在某些特殊情况下，如果这个方法不足够，还可以使用它。产品按制造商的字母顺序排列，然后按产品 ID 的数字顺序排列。在每个制造商的条目之前有一个 C 注释，并且条目之间有一个空行。

### 16.1.3. 示例探测例程

要理解如何将设备添加到受支持设备的列表中，必须了解许多驱动程序具有的探测和/或匹配例程。由于 FreeBSD 5.x 中也存在 OLDCARD 的兼容层，这一点稍微复杂一些。由于只是外观不同，本文将提供一个理想化版本。

```c
static const struct pccard_product wi_pccard_products[] = {
	PCMCIA_CARD(3COM, 3CRWE737A, 0),
	PCMCIA_CARD(BUFFALO, WLI_PCM_S11, 0),
	PCMCIA_CARD(BUFFALO, WLI_CF_S11G, 0),
	PCMCIA_CARD(TDK, LAK_CD011WL, 0),
	{ NULL }
};

static int
wi_pccard_probe(dev)
	device_t	dev;
{
	const struct pccard_product *pp;

	if ((pp = pccard_product_lookup(dev, wi_pccard_products,
	    sizeof(wi_pccard_products[0]), NULL)) != NULL) {
		if (pp->pp_name != NULL)
			device_set_desc(dev, pp->pp_name);
		return (0);
	}
	return (ENXIO);
}
```

这里有一个简单的 pccard 探测例程，它匹配几个设备。如上所述，函数名可能不同（如果不是 `foo_pccard_probe()`，它将是 `foo_pccard_match()`）。`pccard_product_lookup()` 函数是一个通用函数，它遍历表格并返回第一个匹配的条目的指针。有些驱动程序可能使用此机制向驱动程序的其余部分传递有关某些卡的附加信息，因此表格中可能会有所不同。唯一的要求是，表格的每一行必须具有 `struct pccard_product` 作为第一个元素。

查看表格 `wi_pccard_products`，可以看到所有条目都采用 `PCMCIA_CARD(foo, <span> </span>bar, <span> </span>baz)` 形式。*foo* 部分是来自 **pccarddevs** 的供应商 ID。*bar* 部分是产品 ID。*baz* 是此卡的预期功能编号。许多 pccard 可能具有多个功能，因此需要某种方式来区分功能 1 和功能 0。你可能会看到 `PCMCIA_CARD_D`，它包含来自 **pccarddevs** 的设备描述。你还可能会看到 `PCMCIA_CARD2` 和 `PCMCIA_CARD2_D`，它们在需要同时匹配 CIS 字符串和制造商编号时使用，分别表示“使用默认描述”和“从 pccarddevs 获取描述”。

### 16.1.4. 将所有内容整合在一起

要添加新设备，首先需要获取设备的识别信息。最简单的方法是将设备插入 PC Card 或 CF 插槽，然后执行 `devinfo -v`。示例输出：

```c
cbb1 pnpinfo vendor=0x104c device=0xac51 subvendor=0x1265 subdevice=0x0300 class=0x060700 at slot=10 function=1
          cardbus1
          pccard1
            unknown pnpinfo manufacturer=0x026f product=0x030c cisvendor="BUFFALO" cisproduct="WLI2-CF-S11" function_type=6 at function=0
```

`manufacturer` 和 `product` 是此产品的数字 ID，而 `cisvendor` 和 `cisproduct` 是来自 CIS 的产品描述字符串。

由于我们首先希望优先使用数字选项，因此首先尝试基于此构建条目。上面的卡片已经稍微进行了虚构化处理。供应商是 BUFFALO，我们看到它已经有一个条目：

```c
vendor BUFFALO			0x026f	BUFFALO (Melco Corporation)
```

但是没有这个特定卡片的条目。相反，我们发现：

```c
/* BUFFALO */
product BUFFALO WLI_PCM_S11	0x0305	BUFFALO AirStation 11Mbps WLAN
product BUFFALO LPC_CF_CLT	0x0307	BUFFALO LPC-CF-CLT
product	BUFFALO	LPC3_CLT	0x030a	BUFFALO LPC3-CLT Ethernet Adapter
product BUFFALO WLI_CF_S11G	0x030b	BUFFALO AirStation 11Mbps CF WLAN
```

为了添加该设备，我们只需将此条目添加到 **pccarddevs** 中：

```
product BUFFALO WLI2_CF_S11G	0x030c	BUFFALO AirStation ultra 802.11b CF
```

完成这些步骤后，设备就可以添加到驱动程序中。这是一个简单的操作，即添加一行：

```c
static const struct pccard_product wi_pccard_products[] = {
	PCMCIA_CARD(3COM, 3CRWE737A, 0),
	PCMCIA_CARD(BUFFALO, WLI_PCM_S11, 0),
	PCMCIA_CARD(BUFFALO, WLI_CF_S11G, 0),
+	PCMCIA_CARD(BUFFALO, WLI_CF2_S11G, 0),
	PCMCIA_CARD(TDK, LAK_CD011WL, 0),
	{ NULL }
};
```

请注意，我在我添加的行之前包含了一个“+”，但那只是为了突出显示这一行。实际驱动程序中不应添加它。添加该行后，可以重新编译内核或模块并进行测试。如果设备被识别并且正常工作，请提交补丁。如果设备无法正常工作，请找出使其正常工作的必要步骤并提交补丁。如果设备完全无法识别，那么说明你在某些步骤上出错了，需要重新检查每一步。

如果你是 FreeBSD 的源代码提交者，且一切正常，则可以将更改提交到树中。然而，仍然有一些小的技巧需要注意。**pccarddevs** 必须首先提交到树中。然后，必须重新生成并提交 **pccarddevs.h**，确保后者文件中有正确的 \$FreeBSD\$ 标签。最后，提交对驱动程序的添加。

### 16.1.5. 提交新设备

请不要直接将新设备的条目发送给作者。相反，应作为 PR 提交，并将 PR 编号发送给作者以供记录。这可以确保条目不会丢失。提交 PR 时，不必将 **pccarddevs.h** 的差异包含在补丁中，因为这些差异将会重新生成。必须包括设备的描述以及对客户端驱动程序的补丁。如果你不知道设备的名称，请使用 OEM99 作为名称，作者会在调查后相应地调整 OEM99。提交者不应提交 OEM99，而应查找最高的 OEM 条目并提交其后的条目。
