# 第 16 章 PC 卡


本章将讨论 FreeBSD 为 PC 卡或 CardBus 设备编写设备驱动程序的机制。然而，目前它只是记录了如何向现有的 pccard 驱动程序添加新设备。

## 16.1. 添加设备

设备驱动程序知道它们支持哪些设备。内核中有一个支持设备的表，驱动程序使用它来连接到设备。

### 16.1.1. 概述

PC 卡可以通过两种方式进行识别，都基于存储在卡上的卡信息结构（CIS）。第一种方法是使用数字制造商和产品编号。第二种方法是使用包含在 CIS 中的可读字符串。PC 卡总线使用集中式数据库和一些宏来促进一种设计模式，以帮助驱动程序编写者将设备与其驱动程序匹配。

原始设备制造商（OEM）经常为 PC 卡产品开发参考设计，然后将该设计出售给其他公司进行营销。这些公司改进设计，将产品推向目标受众或地理区域，并在卡上放上自己的商标。对物理卡的改进通常非常微小，如果有任何更改的话。为了加强其品牌，这些供应商在 CIS 空间的可读字符串中放置其公司名称，但保持制造商和产品 ID 不变。

由于这种做法，FreeBSD 驱动程序通常依赖于设备识别的数字 ID。使用数字 ID 和一个集中的数据库会使系统添加 ID 和对卡的支持变得复杂。必须仔细检查看看到底谁真正制造了这张卡片，特别是当制造这张卡片的厂商在中央数据库中可能已经有不同的制造商 ID 的情况下。Linksys、D-Link 和 NetGear 是一些美国的 LAN 硬件制造商，它们经常销售相同的设计。这些相同的设计在日本可能以 Buffalo 和 Corega 等名称销售。通常，这些设备都会拥有相同的制造商和产品 ID。

PC Card 总线代码保留了卡信息的中央数据库，但未保留与之关联的驱动程序在/sys/dev/pccard/pccarddevs 中。它还提供了一组宏，允许用户轻松构建驱动程序用来索赔设备的表中的简单条目。

最后，一些真正低端的设备根本没有制造商标识。这些设备必须通过匹配可读的 CIS 字符串来检测。虽然如果我们不需要这种方法作为后备是很好的，但对一些非常低端的 CD-ROM 播放器和以太网卡来说是必要的。这种方法通常应该避免使用，但由于在识别 PC Card 业务的 OEM 性质之前添加了许多设备，这里列出了一些设备。在添加新设备时，请优先使用数字方法。

### 16.1.2. pccarddevs 的格式

pccarddevs 文件中有四个部分。第一部分列出了使用厂商编号的供应商。这一部分按数字顺序排序。接下来的部分包含所有这些供应商使用的产品，以及它们的产品 ID 号和描述字符串。描述字符串通常不使用（而是基于可读的 CIS 设置设备的描述，即使我们匹配了数字版本）。然后为使用字符串匹配方法的设备重复这两个部分。最后，文件中允许在任何位置使用 / ** 和 ** / 字符括起的 C 样式注释。

文件的第一部分包含供应商 ID。请将此列表按数字顺序排序。此外，请协调对此文件的更改，因为我们与 NetBSD 共享此信息以帮助促进这一信息的共同清理。例如，这里有前几个供应商 ID:

```
vendor FUJITSU			0x0004  Fujitsu Corporation
vendor NETGEAR_2		0x000b  Netgear
vendor PANASONIC		0x0032	Matsushita Electric Industrial Co.
vendor SANDISK			0x0045	Sandisk Corporation
```

很有可能， NETGEAR_2 这个条目实际上是 NETGEAR 从 OEM 那里购买的卡，支持这些卡的作者当时可能不知道 Netgear 正在使用别人的 ID。这些条目相当直观。厂商关键字表示这是什么类型的行，接着是厂商的名称。这个名称将在后面的 pccarddevs 中重复出现，并且被用在驱动程序的匹配表中，所以保持它简短且是有效的 C 标识符。十六进制的数字 ID 标识制造商。不要添加类似 0xffffffff 或 0xffff 这样的 ID，因为这些是保留的 ID（前者是 "没有设置 ID"，而后者有时出现在质量极差的卡中，试图表示 "无"）。最后是制作这张卡的公司的描述字符串。这个字符串在 FreeBSD 中除了用于评论目的外并没有其他用途。

文件的第二部分包含了产品。正如本例所示，格式类似于厂商行：

```
/* Allied Telesis K.K. */
product ALLIEDTELESIS LA_PCM	0x0002 Allied Telesis LA-PCM

/* Archos */
product	ARCHOS ARC_ATAPI	0x0043 MiniCD
```

第 product 个关键字后面跟着上面重复的厂商名称。然后是产品名称，驱动程序使用，应该是一个有效的 C 标识符，但也可以以数字开头。与供应商一样，该卡的十六进制产品 ID 遵循相同的约定 0xffffffff 和 0xffff 。最后是设备本身的字符串描述。这个字符串通常在 FreeBSD 中不被使用，因为 FreeBSD 的 pccard 总线驱动程序将从可人类阅读的 CIS 条目构建一个字符串，但在某些极其不足的情况下可以使用。产品按照制造商的字母顺序，然后按产品 ID 的数字顺序排列。在每个制造商的条目之前有一个 C 注释，并在条目之间有一个空行。

第三部分类似于前一个供应商部分，但所有制造商的数字 ID 都设置为 -1 ，意思是在 FreeBSD pccard 总线代码中"匹配任何找到的"。由于这些是 C 标识符，它们的名称必须是唯一的。否则，格式与文件的第一部分相同。

最后一部分包含必须通过字符串条目标识的卡的条目。此部分的格式与通用部分略有不同：

```
product ADDTRON AWP100		{ "Addtron", "AWP-100&spWireless&spPCMCIA", "Version&sp01.02", NULL }
product ALLIEDTELESIS WR211PCM	{ "Allied&spTelesis&spK.K.", "WR211PCM", NULL, NULL } Allied Telesis WR211PCM
```

熟悉的 product 关键字后面跟着供应商名称和卡名称，就像文件的第二部分一样。在这里，格式与先前使用的格式有所偏差。出现了{}分组，后面跟着一些字符串。这些字符串对应于在 CIS_INFO 元组中定义的供应商、产品和附加信息。这些字符串由生成 pccarddevs.h 的程序过滤，以将&sp 替换为真实空格。NULL 字符串意味着应忽略条目的相应部分。此处显示的示例包含错误条目。它不应包含版本号，除非对卡的操作至关重要。有时供应商将在字段中具有许多不同版本的卡，所有这些都有效，在这种情况下，该信息只会使使用类似卡的人更难以在 FreeBSD 上使用它。有时候，当供应商希望在相同品牌下销售许多不同的零件时，由于市场考虑（供货能力、价格等），对于将卡消除歧义则是至关重要的。现在不提供正则表达式匹配。

### 16.1.3. 示例探测例程

要了解如何将设备添加到受支持设备列表中，必须了解许多驱动程序具有的探测和/或匹配例程。在 FreeBSD 5.x 中，由于还存在与 OLDCARD 的兼容性层，这变得有点复杂。由于只是装饰有所不同，这里将呈现一个理想化的版本。

```
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

这里有一个简单的 pccard 探测例程，匹配几个设备。如上所述，名称可能会有所不同（如果不是 foo_pccard_probe() ，那么将是 foo_pccard_match() ）。函数 pccard_product_lookup() 是一个通用函数，它遍历表并返回与之匹配的第一个条目的指针。一些驱动程序可能使用此机制向其余驱动程序传递有关某些卡的附加信息，因此表中可能会有一些变化。唯一的要求是表的每一行的第一个元素必须是 struct pccard_product 。

观察表格 wi_pccard_products ，人们会注意到所有条目的形式都是 PCMCIA_CARD(foo,<span> </span>bar,<span> </span>baz) 。foo 部分是来自 pccarddevs 的制造商 ID。bar 部分是产品 ID。baz 是此卡的预期功能号。许多 PCCards 可以具有多个功能，并且需要某种方法来区分功能 1 和功能 0。你可能看到 PCMCIA_CARD_D ，其中包括来自 pccarddevs 的设备描述。你还可能看到 PCMCIA_CARD2 和 PCMCIA_CARD2_D ，在需要匹配 CIS 字符串和制造商编号时使用，分别是“使用默认描述”和“从 pccarddevs 获取描述”。

### 16.1.4. 将所有内容汇总

要添加新设备，必须首先从设备获取标识信息。这样做的最简单方法是将设备插入 PC 卡或 CF 槽并发出 devinfo -v 。示例输出：

```
        cbb1 pnpinfo vendor=0x104c device=0xac51 subvendor=0x1265 subdevice=0x0300 class=0x060700 at slot=10 function=1
          cardbus1
          pccard1
            unknown pnpinfo manufacturer=0x026f product=0x030c cisvendor="BUFFALO" cisproduct="WLI2-CF-S11" function_type=6 at function=0
```

manufacturer 和 product 是该产品的数字 ID，而 cisvendor 和 cisproduct 是来自 CIS 的产品描述字符串。

由于我们首先希望优先选择数字选项，因此请首先尝试基于该选项构建条目。上面的卡片为本示例略作虚构。供应商是 BUFFALO，我们看到已经有一个条目：

```
vendor BUFFALO			0x026f	BUFFALO (Melco Corporation)
```

但没有这张特定卡片的条目。相反，我们发现：

```
/* BUFFALO */
product BUFFALO WLI_PCM_S11	0x0305	BUFFALO AirStation 11Mbps WLAN
product BUFFALO LPC_CF_CLT	0x0307	BUFFALO LPC-CF-CLT
product	BUFFALO	LPC3_CLT	0x030a	BUFFALO LPC3-CLT Ethernet Adapter
product BUFFALO WLI_CF_S11G	0x030b	BUFFALO AirStation 11Mbps CF WLAN
```

要添加设备，我们只需将此条目添加到 pccarddevs 中：

```
product BUFFALO WLI2_CF_S11G	0x030c	BUFFALO AirStation ultra 802.11b CF
```

完成这些步骤后，可以将该卡添加到驱动程序中。这只是简单地添加一行：

```
static const struct pccard_product wi_pccard_products[] = {
	PCMCIA_CARD(3COM, 3CRWE737A, 0),
	PCMCIA_CARD(BUFFALO, WLI_PCM_S11, 0),
	PCMCIA_CARD(BUFFALO, WLI_CF_S11G, 0),
+	PCMCIA_CARD(BUFFALO, WLI_CF2_S11G, 0),
	PCMCIA_CARD(TDK, LAK_CD011WL, 0),
	{ NULL }
};
```

请注意，在我添加的行之前的行中我包含了一个“+”，但那只是为了突出显示该行。不要将其添加到实际驱动程序中。添加了该行后，你可以重新编译你的内核或模块并进行测试。如果设备被识别并正常工作，请提交补丁。如果不起作用，请找出需要做什么使其正常工作并提交补丁。如果设备根本没有被识别，那么你做错了什么，应该重新检查每一步。

如果你是一个 FreeBSD src 提交者，并且一切正常，那么你可以将更改提交到树中。但是，有一些小技巧需要考虑。首先必须将 pccarddevs 提交到树中。然后，必须重新生成 pccarddevs.h 并作为第二步提交，确保后一个文件中有正确的$FreeBSD$标记。最后，提交对驱动程序的添加。

### 16.1.5. 提交新设备

请不要直接向作者发送新设备的条目。请将它们作为 PR 提交，并将 PR 编号发送给作者以备记录。这样可以确保条目不会丢失。提交 PR 时，无需在补丁中包含 pccardevs.h 的差异，因为这些将被重新生成。需要包括设备描述以及客户端驱动程序的补丁。如果你不知道名称，请使用 OEM99 作为名称，作者将在调查后相应调整 OEM99。提交者不应提交 OEM99，而应找到最高的 OEM 条目并提交比那更高一个。
