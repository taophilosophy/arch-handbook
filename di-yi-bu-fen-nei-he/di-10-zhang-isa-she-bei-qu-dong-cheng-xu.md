# 第 10 章 ISA 设备驱动程序

## 10.1. 概述

本章介绍了编写 ISA 设备驱动程序时需要考虑的问题。这里呈现的伪代码相当详细，并且接近实际代码，但仍然只是伪代码。它避免了与讨论主题无关的细节。实际的示例可以在真实驱动程序的源代码中找到，特别是 `ep` 和 `aha` 驱动程序是很好的信息来源。

## 10.2. 基本信息

一个典型的 ISA 驱动程序需要以下包含文件：

```c
#include <sys/module.h>
#include <sys/bus.h>
#include <machine/bus.h>
#include <machine/resource.h>
#include <sys/rman.h>

#include <isa/isavar.h>
#include <isa/pnpvar.h>
```

这些文件描述了特定于 ISA 和通用总线子系统的内容。

总线子系统采用面向对象的方式实现，主要结构通过关联的方法函数进行访问。

ISA 驱动程序实现的总线方法列表与其他总线驱动程序类似。假设有一个名为 "xxx" 的驱动程序，其方法如下：

* `static void xxx_isa_identify (driver_t *, device_t);` 通常用于总线驱动程序，而非设备驱动程序。但是对于 ISA 设备，这个方法可能有特殊用途：如果设备提供某种设备特定的（非 PnP）方式来自动检测设备，则该例程可以实现这一功能。
* `static int xxx_isa_probe (device_t dev);` 在已知的（或 PnP）位置探测设备。该例程还可以适应设备特定的参数自动检测，适用于部分配置的设备。
* `static int xxx_isa_attach (device_t dev);` 附加并初始化设备。
* `static int xxx_isa_detach (device_t dev);` 在卸载驱动程序模块之前卸载设备。
* `static int xxx_isa_shutdown (device_t dev);` 在系统关机之前执行设备的关闭操作。
* `static int xxx_isa_suspend (device_t dev);` 在系统进入省电模式之前挂起设备。也可能中止进入省电模式的操作。
* `static int xxx_isa_resume (device_t dev);` 在从省电状态恢复后恢复设备活动。

`xxx_isa_probe()` 和 `xxx_isa_attach()` 是必须实现的，其余方法根据设备的需求可以选择性实现。

驱动程序通过以下描述集与系统链接。

```c
/* 支持的总线方法表 */
    static device_method_t xxx_isa_methods[] = {
        /* 列出驱动程序支持的所有总线方法函数 */
        /* 忽略不支持的方法 */
        DEVMETHOD(device_identify,  xxx_isa_identify),
        DEVMETHOD(device_probe,     xxx_isa_probe),
        DEVMETHOD(device_attach,    xxx_isa_attach),
        DEVMETHOD(device_detach,    xxx_isa_detach),
        DEVMETHOD(device_shutdown,  xxx_isa_shutdown),
        DEVMETHOD(device_suspend,   xxx_isa_suspend),
        DEVMETHOD(device_resume,    xxx_isa_resume),

	DEVMETHOD_END
    };

    static driver_t xxx_isa_driver = {
        "xxx",
        xxx_isa_methods,
        sizeof(struct xxx_softc),
    };

    static devclass_t xxx_devclass;

    DRIVER_MODULE(xxx, isa, xxx_isa_driver, xxx_devclass,
        load_function, load_argument);
```

这里 `struct xxx_softc` 是一个设备特定的结构体，包含了私有的驱动程序数据和驱动程序资源的描述符。总线代码会根据需要自动分配每个设备的一个 softc 描述符。

如果驱动程序是作为可加载模块实现的，那么在加载或卸载驱动程序时会调用 `load_function()` 来进行驱动程序的初始化或清理操作，`load_argument` 会作为其参数之一传递。如果驱动程序不支持动态加载（即必须始终链接到内核中），则这些值应设置为 0，最后的定义应如下所示：

```c
DRIVER_MODULE(xxx, isa, xxx_isa_driver,
       xxx_devclass, 0, 0);
```

如果驱动程序是针对支持 PnP 的设备，则必须定义一个支持的 PnP ID 表。该表包含了此驱动程序支持的 PnP ID 列表以及带有这些 ID 的硬件类型和型号的可读描述。示例如下：

```c
static struct isa_pnp_id xxx_pnp_ids[] = {
        /* 每个支持的 PnP ID 一行 */
        { 0x12345678,   "Our device model 1234A" },
        { 0x12345679,   "Our device model 1234B" },
        { 0,        NULL }, /* 表格结束 */
    };
```

如果驱动程序不支持 PnP 设备，它仍然需要一个空的 PnP ID 表，如下所示：

```c
static struct isa_pnp_id xxx_pnp_ids[] = {
        { 0,        NULL }, /* 表格结束 */
    };
```

## 10.3. `device_t` 指针

`device_t` 是设备结构的指针类型。在这里，我们只考虑从设备驱动程序编写者角度来看有用的方法。用于操作设备结构中值的方法如下：

* `device_t device_get_parent(dev)` 获取设备的父总线。
* `driver_t device_get_driver(dev)` 获取指向设备驱动程序结构的指针。
* `char *device_get_name(dev)` 获取驱动程序名称，例如我们示例中的 `"xxx"`。
* `int device_get_unit(dev)` 获取设备的单元号（设备的单元号从 0 开始，为每个驱动程序关联的设备编号）。
* `char *device_get_nameunit(dev)` 获取包括单元号的设备名称，例如 "xxx0"、"xxx1" 等。
* `char *device_get_desc(dev)` 获取设备描述。通常，它描述设备的确切型号，以人类可读的形式表示。
* `device_set_desc(dev, desc)` 设置设备描述。此操作使设备描述指向字符串 `desc`，并且该字符串在之后不能被释放或修改。
* `device_set_desc_copy(dev, desc)` 设置设备描述。描述被复制到一个内部动态分配的缓冲区，因此字符串 `desc` 之后可以进行修改，而不会产生不良影响。
* `void *device_get_softc(dev)` 获取与此设备关联的设备描述符（结构 `xxx_softc`）的指针。
* `u_int32_t device_get_flags(dev)` 获取设备在配置文件中指定的标志。

可以使用便利函数 `device_printf(dev, fmt, …)` 来打印设备驱动程序的消息。它会自动在消息前面加上单元名称和冒号。

`device_t` 方法在文件 **kern/bus\_subr.c** 中实现。

## 10.4. 配置文件和自动配置过程中识别与探测的顺序

ISA 设备在内核配置文件中描述如下：

```c
device xxx0 at isa? port 0x300 irq 10 drq 5
       iomem 0xd0000 flags 0x1 sensitive
```

端口、IRQ 等的值会被转换为与设备关联的资源值。它们是可选的，具体取决于设备的需求以及自动配置的能力。例如，某些设备根本不需要 DRQ，而一些设备允许驱动程序从设备的配置端口读取 IRQ 设置。如果机器有多个 ISA 总线，可以在配置行中指定精确的总线，如 `isa0` 或 `isa1`，否则设备将在所有 ISA 总线上进行搜索。

`sensitive` 是一个资源标记，表示该设备必须在所有非敏感设备之前进行探测。它已被支持，但在当前的驱动程序中似乎没有使用。

对于传统的 ISA 设备，驱动程序在很多情况下仍能检测到配置参数。但系统中每个设备都必须有一个配置行。如果系统中安装了两台相同类型的设备，但只有一个对应驱动程序的配置行，例如：

```c
device xxx0 at isa?
```

```
那么只有一个设备会被配置。
```

但是对于支持通过 Plug-n-Play 或某些专有协议自动识别的设备，一行配置足以配置系统中的所有设备，如上所示，或者只需简单地写：

```c
device xxx at isa?
```

如果驱动程序同时支持自动识别和传统设备，并且两种设备都在同一台机器中安装，那么只需要在配置文件中描述传统设备即可。自动识别的设备会自动添加。

当一个 ISA 总线进行自动配置时，事件发生的顺序如下：

所有驱动程序的识别例程（包括用于识别所有 PnP 设备的 PnP 识别例程）会被随机顺序调用。当它们识别设备时，会将设备添加到 ISA 总线的设备列表中。通常，驱动程序的识别例程会将它们的驱动程序与新设备关联。PnP 识别例程还不知道其他驱动程序，因此它不会将任何驱动程序与它所添加的新设备关联。

PnP 设备会使用 PnP 协议进入睡眠状态，以防止它们作为传统设备被探测。

标记为 `sensitive` 的非 PnP 设备的探测例程会被调用。如果设备探测成功，将会调用该设备的附加例程。

所有非 PnP 设备的探测和附加例程会按同样的方式调用。

PnP 设备会从睡眠状态中恢复，并分配它们请求的资源：I/O 和内存地址范围、IRQ 和 DRQ，所有这些都与已附加的传统设备没有冲突。

然后，对于每个 PnP 设备，所有现有 ISA 驱动程序的探测例程都会被调用。第一个声明该设备的驱动程序会被附加。可能会有多个驱动程序以不同优先级声明同一个设备；在这种情况下，优先级最高的驱动程序胜出。探测例程必须调用 `ISA_PNP_PROBE()` 来比较实际的 PnP ID 和驱动程序支持的 ID 列表，如果该 ID 不在表中，则返回失败。这意味着每个驱动程序，甚至是那些不支持任何 PnP 设备的驱动程序，也必须调用 `ISA_PNP_PROBE()`，至少用一个空的 PnP ID 表来返回不支持的 PnP 设备的失败。

探测例程在出错时返回正值（错误码），在成功时返回零或负值。

负返回值用于支持多个接口的 PnP 设备。例如，支持较旧兼容接口和新接口的设备，这两个接口由不同的驱动程序支持。在这种情况下，两个驱动程序都会探测到该设备。返回值较高的驱动程序优先（换句话说，返回 0 的驱动程序优先，返回 -1 的次之，返回 -2 的次之，以此类推）。结果是，仅支持旧接口的设备会由旧驱动程序（探测例程应返回 -1）处理，而支持新接口的设备会由新驱动程序（探测例程应返回 0）处理。如果多个驱动程序返回相同的值，那么先被调用的驱动程序获胜。因此，如果驱动程序返回值为 0，则可以确信它赢得了优先级竞争。

设备特定的识别例程还可以将设备与一类驱动程序而不是单个驱动程序关联。然后，所有该类中的驱动程序都会对该设备进行探测，就像 PnP 设备一样。但这种特性在现有的驱动程序中没有实现，本文件不再进一步讨论。

由于在探测传统设备时 PnP 设备被禁用，因此它们不会被附加两次（一次作为传统设备，一次作为 PnP 设备）。但是，对于设备特定的识别例程，确保同一设备不会被驱动程序附加两次是驱动程序的责任：一次作为传统的用户配置设备，另一次作为自动识别设备。

自动识别设备（无论是 PnP 设备还是设备特定的设备）的另一个实际影响是，内核配置文件中的标志不能传递给它们。因此，它们必须要么根本不使用标志，要么使用设备单元 0 的标志来处理所有自动识别的设备，或者使用 sysctl 接口代替标志。

其他不寻常的配置可以通过直接访问配置资源来处理，使用 `resource_query_*()` 和 `resource_*_value()` 函数族。这些函数的实现位于 **kern/subr\_bus.c** 中。旧的 IDE 硬盘驱动程序 **i386/isa/wd.c** 包含了此类用法的示例。但始终应优先使用标准的配置方式，将配置资源的解析留给总线配置代码。

## 10.5. 资源

用户在内核配置文件中输入的信息将被处理并传递给内核作为配置资源。这些信息由总线配置代码解析，并转换为 `device_t` 结构及与之关联的总线资源。驱动程序可以使用 `resource_*` 函数直接访问配置资源，但通常情况下这并不必要，也不推荐这么做，因此此问题在此文中不再讨论。

总线资源与每个设备相关联。它们通过类型和类型内的编号进行标识。对于 ISA 总线，定义了以下几种类型：

* `SYS_RES_IRQ` - 中断号
* `SYS_RES_DRQ` - ISA DMA 通道号
* `SYS_RES_MEMORY` - 映射到系统内存空间的设备内存范围
* `SYS_RES_IOPORT` - 设备 I/O 寄存器的范围

在类型内的枚举从 0 开始，因此，如果一个设备有两个内存区域，它将拥有编号为 0 和 1 的 `SYS_RES_MEMORY` 类型资源。资源类型与 C 语言类型无关，所有资源值的 C 语言类型都是 `unsigned long`，并且在需要时必须进行类型转换。资源编号不必是连续的，尽管在 ISA 中通常是连续的。ISA 设备的资源编号范围如下：

```
IRQ: 0-1
DRQ: 0-1
MEMORY: 0-3
IOPORT: 0-7
```

所有资源都表示为一个范围，包含开始值和计数。对于 IRQ 和 DRQ 资源，计数通常为 1。内存的值指的是物理地址。

可以对资源执行三种类型的操作：

* 设置/获取
* 分配/释放
* 激活/停用

设置操作为资源设置范围。分配操作保留请求的范围，确保没有其他驱动程序能够占用该范围（并检查是否已经有其他驱动程序占用该范围）。激活操作通过做必要的事情使得资源对驱动程序可用（例如，对于内存，它会映射到内核虚拟地址空间）。

用于操作资源的函数包括：

* `int bus_set_resource(device_t dev, int type, int rid, u_long start, u_long count)`
  设置资源的范围。如果成功返回 0，失败时返回错误码。通常，当 `type`、`rid`、`start` 或 `count` 的值超出允许的范围时，该函数才会返回错误。

  * dev - 驱动程序的设备
  * type - 资源类型，`SYS_RES_*`
  * rid - 类型内的资源编号（ID）
  * start, count - 资源范围
* `int bus_get_resource(device_t dev, int type, int rid, u_long *startp, u_long *countp)`
  获取资源的范围。如果成功返回 0，若资源尚未定义则返回错误码。
* `u_long bus_get_resource_start(device_t dev, int type, int rid)`
  `u_long bus_get_resource_count(device_t dev, int type, int rid)`
  方便的函数，仅获取开始位置或计数。如果出错，返回 0。因此，如果资源开始位置为 0 并且这是合法值之一，就无法分辨是值为 0 还是发生了错误。幸运的是，ISA 资源的附加驱动程序的起始值不可能为 0。
* `void bus_delete_resource(device_t dev, int type, int rid)`
  删除资源，使其变为未定义状态。
* `struct resource * bus_alloc_resource(device_t dev, int type, int *rid, u_long start, u_long end, u_long count, u_int flags)`
  分配一个资源范围，范围内的计数值未被其他人占用，且范围介于 `start` 和 `end` 之间。遗憾的是，这里不支持对齐。如果资源尚未设置，则会自动创建该资源。特殊值 `start = 0` 和 `end = ~0`（所有位为 1）意味着之前通过 `bus_set_resource()` 设置的固定值必须被使用：`start` 和 `count` 作为它们自己，`end = (start + count)`，如果资源之前没有定义，则会返回错误。尽管 `rid` 通过引用传递，但在 ISA 总线的资源分配代码中不会对其进行修改（其他总线可能使用不同的方式并修改它）。

标志是一个位图，调用者感兴趣的标志有：

* `RF_ACTIVE` - 在分配资源后，自动激活该资源。
* `RF_SHAREABLE` - 资源可以同时由多个驱动程序共享。
* `RF_TIMESHARE` - 资源可以被多个驱动程序共享时间段，即许多驱动程序可以同时分配资源，但每次只能由一个驱动程序激活。

返回 0 表示错误。分配的值可以通过返回的句柄使用 `rhand_*()` 方法获取。

* `int bus_release_resource(device_t dev, int type, int rid, struct resource *r)`
  释放资源，`r` 是由 `bus_alloc_resource()` 返回的句柄。成功时返回 0，失败时返回错误码。

* `int bus_activate_resource(device_t dev, int type, int rid, struct resource *r)`
  `int bus_deactivate_resource(device_t dev, int type, int rid, struct resource *r)`
  激活或停用资源。成功时返回 0，失败时返回错误码。如果资源是时间共享的，并且当前由另一个驱动程序激活，则返回 `EBUSY`。

* `int bus_setup_intr(device_t dev, struct resource *r, int flags, driver_intr_t *handler, void *arg, void **cookiep)`
  `int bus_teardown_intr(device_t dev, struct resource *r, void *cookie)`
  将中断处理程序与设备关联或取消关联。成功时返回 0，失败时返回错误码。

* `r` - 激活资源句柄，描述 IRQ。

* `flags` - 中断优先级级别，以下之一：

  * `INTR_TYPE_TTY` - 终端和其他类似字符类型的设备。使用 `spltty()` 来屏蔽它们。
  * `(INTR_TYPE_TTY | INTR_TYPE_FAST)` - 具有小输入缓冲区的终端设备，输入时数据丢失非常关键（例如旧式串口）。使用 `spltty()` 来屏蔽它们。
  * `INTR_TYPE_BIO` - 块类型设备，除了 CAM 控制器上的设备。使用 `splbio()` 来屏蔽它们。
  * `INTR_TYPE_CAM` - CAM（通用访问方法）总线控制器。使用 `splcam()` 来屏蔽它们。
  * `INTR_TYPE_NET` - 网络接口控制器。使用 `splimp()` 来屏蔽它们。
  * `INTR_TYPE_MISC` - 杂项设备。没有其他方式来屏蔽它们，只有通过 `splhigh()`，它会屏蔽所有中断。

当一个中断处理程序执行时，所有与其优先级匹配的中断将被屏蔽。唯一的例外是 MISC 级别的中断，它不会屏蔽任何其他中断，并且不会被任何其他中断屏蔽。

* `handler` - 指向处理程序函数的指针，`driver_intr_t` 类型定义为 `void driver_intr_t(void *)`。
* `arg` - 传递给处理程序的参数，用于标识该特定设备。处理程序会将其从 `void*` 强制转换为实际类型。旧的 ISA 中断处理程序约定是使用单元号作为参数，新的（推荐的）约定是使用指向设备 `softc` 结构的指针。
* `cookie[p]` - 从 `setup()` 返回的值，用于在 `teardown()` 时标识处理程序。

有许多方法可以操作资源处理程序（`struct resource *`）。对设备驱动程序作者感兴趣的方法包括：

* `u_long rman_get_start(r) u_long rman_get_end(r)`
  获取已分配资源范围的起始和结束位置。
* `void *rman_get_virtual(r)`
  获取激活的内存资源的虚拟地址。

## 10.6. 总线内存映射

在许多情况下，数据通过内存在驱动程序和设备之间交换。有两种可能的变体：

(a) 内存位于设备卡上

(b) 内存是计算机的主内存

在情况 (a) 中，驱动程序会根据需要将数据来回复制到设备卡上的内存和主内存之间。为了将设备卡上的内存映射到内核虚拟地址空间中，必须将设备卡上内存的物理地址和长度定义为 `SYS_RES_MEMORY` 资源。然后可以分配并激活该资源，通过 `rman_get_virtual()` 获取其虚拟地址。旧版驱动程序使用 `pmap_mapdev()` 函数来实现这一目的，但现在应该避免直接使用该函数。现在，它是资源激活的内部步骤之一。

大多数 ISA 卡的内存会被配置在物理地址范围 640KB 到 1MB 之间。某些 ISA 卡需要更大的内存范围，这些内存应放置在 16MB 以下（因为 ISA 总线的 24 位地址限制）。在这种情况下，如果计算机的内存大于设备内存的起始地址（换句话说，它们重叠了），则必须在设备使用的地址范围内配置内存孔。许多 BIOS 允许配置一个从 14MB 或 15MB 开始的 1MB 的内存孔。如果 BIOS 正确报告这些信息，FreeBSD 可以正确处理内存孔（旧版 BIOS 上可能存在问题）。

在情况 (b) 中，仅将数据的地址发送到设备，设备通过 DMA 来实际访问主内存中的数据。此情况下有两个限制：首先，ISA 卡只能访问 16MB 以下的内存。其次，虚拟地址空间中的连续页面在物理地址空间中可能不是连续的，因此设备可能需要进行散布/聚集操作。总线子系统为其中一些问题提供了现成的解决方案，其余的问题则需要驱动程序自行处理。

DMA 内存分配使用两个结构：`bus_dma_tag_t` 和 `bus_dmamap_t`。`bus_dma_tag_t` 描述了 DMA 内存所需的属性，`bus_dmamap_t` 代表一个根据这些属性分配的内存块。多个映射可以与同一个标签相关联。

标签按树状层次结构组织，支持属性的继承。子标签继承父标签的所有要求，并且可以使这些要求更严格，但不能使其更宽松。

通常为每个设备单元创建一个顶级标签（没有父标签）。如果每个设备需要多个具有不同要求的内存区域，则可以为每个内存区域创建一个作为父标签子标签的标签。

标签可以通过两种方式创建映射：

第一种方法是分配符合标签要求的连续内存块（稍后可以释放）。通常用于为与设备通信分配相对长生命周期的内存区域。将这种内存加载到映射中是简单的：它始终被视为适当物理内存范围内的一个块。

第二种方法是将任意虚拟内存区域加载到映射中。此内存的每一页将检查是否符合映射要求。如果符合要求，则保持在其原始位置。如果不符合要求，则分配一个新的符合要求的“跃点页面”并用作中间存储。在将数据从不符合要求的原始页面写入时，它们会先被复制到跃点页面，然后从跃点页面传输到设备。当读取数据时，数据会从设备传输到跃点页面，然后再复制到不符合要求的原始页面。这种在原始页面和跃点页面之间复制的过程称为同步。通常这是按每次传输进行的：每次传输的缓冲区都会被加载、传输完成后缓冲区被卸载。

DMA 内存操作相关的函数如下：

* `int bus_dma_tag_create(bus_dma_tag_t parent, bus_size_t alignment, bus_size_t boundary, bus_addr_t lowaddr, bus_addr_t highaddr, bus_dma_filter_t *filter, void *filterarg, bus_size_t maxsize, int nsegments, bus_size_t maxsegsz, int flags, bus_dma_tag_t *dmat)`
  创建一个新的标签。成功返回 0，失败返回错误代码。

  * *parent* - 父标签，如果为 NULL，则创建顶级标签。
  * *alignment* - 为此标签分配的内存区域要求的物理对齐方式。若无特定对齐要求，则使用值 1。仅适用于未来的 `bus_dmamem_alloc()`，而不适用于 `bus_dmamap_create()` 调用。
  * *boundary* - 分配内存时不能跨越的物理地址边界。若没有边界要求，则使用值 0。仅适用于未来的 `bus_dmamem_alloc()`，而不适用于 `bus_dmamap_create()` 调用。该值必须是 2 的幂。如果内存计划在非级联 DMA 模式下使用（即 DMA 地址不会由设备本身提供，而是由 ISA DMA 控制器提供），则由于 DMA 硬件的限制，边界不能大于 64KB（64 \* 1024）。
  * *lowaddr, highaddr* - 这两个名称有点误导；这些值用于限制分配内存时允许使用的物理地址范围。具体含义取决于未来的使用方式：

    * 对于 `bus_dmamem_alloc()`，所有从 0 到 lowaddr-1 的地址都被视为允许的，较高的地址则被禁止。
    * 对于 `bus_dmamap_create()`，所有在区间 \[lowaddr, highaddr] 之外的地址都被视为可访问。区间内的页面地址会传递给过滤函数，由它决定是否可访问。如果没有提供过滤函数，则整个范围被视为不可访问。
    * 对于 ISA 设备，正常的值（无过滤函数时）为：
      lowaddr = `BUS_SPACE_MAXADDR_24BIT`
      highaddr = `BUS_SPACE_MAXADDR`
  * *filter, filterarg* - 过滤函数及其参数。如果传递 NULL，则 `bus_dmamap_create()` 时整个范围 \[lowaddr, highaddr] 会被视为不可访问。否则，将会传递每个在 \[lowaddr, highaddr] 范围内的页面物理地址给过滤函数，由它判断该页面是否可访问。过滤函数的原型为：`int filterfunc(void *arg, bus_addr_t paddr)`。如果页面可访问，则返回 0，否则返回非 0 值。
  * *maxsize* - 通过此标签分配的内存的最大大小（以字节为单位）。如果很难估算或可能是任意大的，则 ISA 设备的值应为 `BUS_SPACE_MAXSIZE_24BIT`。
  * *nsegments* - 设备支持的最大散布/聚集段数。如果没有限制，则应使用 `BUS_SPACE_UNRESTRICTED` 值。推荐为父标签使用此值，实际的限制会在子标签中指定。nsegments 等于 `BUS_SPACE_UNRESTRICTED` 的标签不能用于实际加载映射，它们只能作为父标签使用。nsegments 的实际限制似乎约为 250-300，较高的值会导致内核栈溢出（而硬件通常也无法支持这么多的散布/聚集缓冲区）。
  * *maxsegsz* - 设备支持的最大散布/聚集段大小。ISA 设备的最大值为 `BUS_SPACE_MAXSIZE_24BIT`。
  * *flags* - 标志的位图。唯一感兴趣的标志是：

    * *BUS\_DMA\_ALLOCNOW* - 请求在创建标签时分配所有可能需要的跃点页面。
  * *dmat* - 存储新标签的指针。

* `int bus_dma_tag_destroy(bus_dma_tag_t dmat)`
  销毁标签。成功返回 0，失败返回错误代码。

  *dmat* - 要销毁的标签。

* `int bus_dmamem_alloc(bus_dma_tag_t dmat, void** vaddr, int flags, bus_dmamap_t *mapp)`
  分配标签描述的连续内存区域。要分配的内存大小为标签的 `maxsize`。成功返回 0，失败返回错误代码。结果仍需通过 `bus_dmamap_load()` 加载，以便获取内存的物理地址。

  * *dmat* - 标签
  * *vaddr* - 存储分配的内核虚拟地址的指针。
  * *flags* - 标志的位图。唯一感兴趣的标志是：

    * *BUS\_DMA\_NOWAIT* - 如果内存不可用立即返回错误。如果未设置此标志，则该例程可以在内存可用时休眠。
  * *mapp* - 存储新映射的指针。

继续 DMA 内存操作的函数说明：

* `void bus_dmamem_free(bus_dma_tag_t dmat, void *vaddr, bus_dmamap_t map)`
  释放通过 `bus_dmamem_alloc()` 分配的内存。目前，对于具有 ISA 限制的内存分配，释放操作尚未实现。因此，建议的使用模式是尽可能长时间地保留并重用分配的内存区域。避免轻率地释放某个区域，然后很快重新分配该区域。这并不意味着 `bus_dmamem_free()` 应该完全不使用：希望它会尽快正确实现。

  * *dmat* - 标签
  * *vaddr* - 内存的内核虚拟地址
  * *map* - 内存的映射（由 `bus_dmamem_alloc()` 返回）

* `int bus_dmamap_create(bus_dma_tag_t dmat, int flags, bus_dmamap_t *mapp)`
  为标签创建一个映射，以便以后在 `bus_dmamap_load()` 中使用。成功返回 0，失败返回错误代码。

  * *dmat* - 标签
  * *flags* - 理论上是标志的位图，但目前没有定义任何标志，因此现在始终为 0。
  * *mapp* - 用于返回新映射的存储指针

* `int bus_dmamap_destroy(bus_dma_tag_t dmat, bus_dmamap_t map)`
  销毁一个映射。成功返回 0，失败返回错误代码。

  * *dmat* - 与映射关联的标签
  * *map* - 要销毁的映射

* `int bus_dmamap_load(bus_dma_tag_t dmat, bus_dmamap_t map, void *buf, bus_size_t buflen, bus_dmamap_callback_t *callback, void *callback_arg, int flags)`
  将缓冲区加载到映射中（映射必须是通过 `bus_dmamap_create()` 或 `bus_dmamem_alloc()` 先前创建的）。缓冲区的所有页面都会检查是否符合标签要求，对于不符合要求的页面，会分配跃点页面。一个物理段描述符数组将被构建并传递给回调函数。回调函数应以某种方式处理它。如果跃点缓冲区需要但暂时不可用，请求将被排队，并且当跃点缓冲区变为可用时，回调函数会被调用。如果回调函数立即执行，返回 0；如果请求已排队，返回 `EINPROGRESS`，此时驱动程序负责与排队的回调函数进行同步。

  * *dmat* - 标签
  * *map* - 映射
  * *buf* - 缓冲区的内核虚拟地址
  * *buflen* - 缓冲区的长度
  * *callback*, `callback_arg` - 回调函数及其参数
    回调函数的原型为：`void callback(void *arg, bus_dma_segment_t *seg, int nseg, int error)`
  * *arg* - 与 `callback_arg` 相同
  * *seg* - 段描述符的数组
  * *nseg* - 数组中的描述符数量
  * *error* - 段号溢出的指示：如果设置为 `EFBIG`，则说明缓冲区没有适应标签允许的最大段数。在这种情况下，数组中将仅包含允许的描述符数。如何处理这种情况取决于驱动程序：根据期望的语义，驱动程序可以将其视为错误，或者将缓冲区分成两部分并分别处理第二部分。

  每个段描述符数组项包含以下字段：

  * *ds\_addr* - 段的物理总线地址
  * *ds\_len* - 段的长度

* `void bus_dmamap_unload(bus_dma_tag_t dmat, bus_dmamap_t map)`
  卸载映射。

  * *dmat* - 标签
  * *map* - 已加载的映射

* `void bus_dmamap_sync(bus_dma_tag_t dmat, bus_dmamap_t map, bus_dmasync_op_t op)`
  在物理数据传输前后，同步已加载的缓冲区与其跃点页面。这是执行原始缓冲区和映射版本之间数据复制的函数。缓冲区在传输前后都必须同步。

  * *dmat* - 标签
  * *map* - 已加载的映射
  * *op* - 要执行的同步操作类型：

    * `BUS_DMASYNC_PREREAD` - 在从设备读取到缓冲区之前
    * `BUS_DMASYNC_POSTREAD` - 在从设备读取到缓冲区之后
    * `BUS_DMASYNC_PREWRITE` - 在将缓冲区写入设备之前
    * `BUS_DMASYNC_POSTWRITE` - 在将缓冲区写入设备之后

关于 DMA 映射和缓冲区的管理，以下是一些重要细节和实践注意事项：

当前 `BUS_DMASYNC_PREREAD` 和 `BUS_DMASYNC_POSTWRITE` 是空操作，但将来可能会改变。因此，驱动程序中不应忽略这些操作，即使它们现在不执行任何操作。特别地，内存通过 `bus_dmamem_alloc()` 获取的情况下，不需要同步操作。

在调用 `bus_dmamap_load()` 的回调函数之前，段数组会被存储在栈上，并为标签允许的最大段数进行预分配。因此，i386 架构上支持的最大段数大约是 250-300（内核栈为 4KB，减去用户结构的大小，每个段数组项的大小为 8 字节，还需要留出一些空间）。因此，实际允许的段数应根据需求设置，避免过高。由于这个数组是基于最大段数进行分配的，因此这个最大值不能设置得过大。幸运的是，对于大多数硬件而言，最大支持的段数要低得多。如果驱动程序需要处理大量散射-聚集段的缓冲区，应分批加载缓冲区：加载缓冲区的一部分，传输到设备，然后加载下一部分，以此类推。

如果缓冲区的所有页面都是物理上不连续的，那么该缓冲区的最大支持大小将受到段数的限制。例如，如果最大支持 10 个段，则 i386 上最大保证支持的缓冲区大小为 40K。如果需要更大的缓冲区大小，驱动程序应该使用一些特殊的技巧来处理。如果硬件不支持散射-聚集（scatter-gather），或者驱动程序想要支持一个即使是高度碎片化的缓冲区大小，解决方案是驱动程序分配一个连续的缓冲区，并在原始缓冲区不适配时作为中间存储使用。

对于在设备附加和分离之间几乎保持不变的缓冲区，典型的调用序列如下：

```c
bus_dmamem_alloc → bus_dmamap_load → …​use buffer…​ → → bus_dmamap_unload → bus_dmamem_free
```

对于频繁变化并且从外部传入的缓冲区，调用序列如下：

```c
bus_dmamap_create →
          → bus_dmamap_load → bus_dmamap_sync(PRE...) → do transfer →
          → bus_dmamap_sync(POST...) → bus_dmamap_unload →
          ...
          → bus_dmamap_load → bus_dmamap_sync(PRE...) → do transfer →
          → bus_dmamap_sync(POST...) → bus_dmamap_unload →
          → bus_dmamap_destroy
```

当使用 `bus_dmamem_alloc()` 创建的映射时，传递的缓冲区地址和大小必须与 `bus_dmamem_alloc()` 中使用的完全相同。在这种情况下，可以保证整个缓冲区将作为一个单一的段进行映射（因此回调可以基于此假设进行），并且请求将立即执行（不会返回 `EINPROGRESS`）。在这种情况下，回调函数只需要保存物理地址即可。

典型示例如下：

```c
static void
        alloc_callback(void *arg, bus_dma_segment_t *seg, int nseg, int error)
        {
          *(bus_addr_t *)arg = seg[0].ds_addr;
        }

          ...
          int error;
          struct somedata {
            ....
          };
          struct somedata *vsomedata; /* 虚拟地址 */
          bus_addr_t psomedata; /* 物理总线相对地址 */
          bus_dma_tag_t tag_somedata;
          bus_dmamap_t map_somedata;
          ...

          error=bus_dma_tag_create(parent_tag, alignment,
           boundary, lowaddr, highaddr, /*filter*/ NULL, /*filterarg*/ NULL,
           /*maxsize*/ sizeof(struct somedata), /*nsegments*/ 1,
           /*maxsegsz*/ sizeof(struct somedata), /*flags*/ 0,
           &tag_somedata);
          if(error)
          return error;

          error = bus_dmamem_alloc(tag_somedata, &vsomedata, /* flags*/ 0,
             &map_somedata);
          if(error)
             return error;

          bus_dmamap_load(tag_somedata, map_somedata, (void *)vsomedata,
             sizeof (struct somedata), alloc_callback,
             (void *) &psomedata, /*flags*/0);
```

看起来有点长且复杂，但这是正确的方法。实际的后果是：如果多个内存区域总是一起分配，最好将它们都合并成一个结构体并作为一个整体进行分配（前提是对齐和边界的限制允许这样做）。

当将一个任意的缓冲区加载到通过 `bus_dmamap_create()` 创建的映射时，如果回调可能会延迟，必须采取特殊措施进行同步。代码看起来如下：

```
{
           int s;
           int error;

           s = splsoftvm();
           error = bus_dmamap_load(
               dmat,
               dmamap,
               buffer_ptr,
               buffer_len,
               callback,
               /*callback_arg*/ buffer_descriptor,
               /*flags*/0);
           if (error == EINPROGRESS) {
               /*
                * 做一些必要的同步工作
                * 确保回调不会在我们调用 splx() 或 tsleep() 之前开始。
                */
              }
           splx(s);
          }
```

请求处理的两种可能方法：

1. 如果通过显式标记请求已完成（例如 CAM 请求），那么将所有进一步的处理放入回调驱动程序中会更简单，回调完成时会标记请求为已完成。然后不需要太多额外的同步工作。出于流控的考虑，最好在请求完成之前冻结请求队列。
2. 如果请求在函数返回时完成（例如字符设备的经典读写请求），则应在缓冲区描述符中设置同步标志，并调用 `tsleep()`。稍后当回调被调用时，它将执行处理并检查此同步标志。如果标志已设置，则回调函数应发出唤醒信号。在这种方法中，回调函数可以执行所有必要的处理（就像前一种情况一样），也可以仅仅将段数组保存到缓冲区描述符中。然后在回调完成后，调用函数可以使用这个保存的段数组并完成所有的处理。

## 10.7. DMA

直接内存访问（DMA）通过 DMA 控制器在 ISA 总线上实现（实际上有两个 DMA 控制器，但这是一个无关的细节）。为了使早期的 ISA 设备简单且便宜，公交控制和地址生成的逻辑集中在 DMA 控制器中。幸运的是，FreeBSD 提供了一组函数，基本上隐藏了设备驱动程序需要处理的 DMA 控制器的细节。

最简单的情况是对于相当智能的设备。例如，像 PCI 上的总线主设备，它们能够自己生成总线周期和内存地址。它们实际上只需要 DMA 控制器提供总线仲裁。因此，它们假装是级联的从 DMA 控制器。系统 DMA 控制器所需的唯一操作是在 DMA 通道上启用级联模式，可以通过在附加驱动时调用以下函数来实现：

`void isa_dmacascade(int channel_number)`

所有进一步的活动都由设备编程完成。卸载驱动时，不需要调用任何与 DMA 相关的函数。

对于较简单的设备，事情变得更复杂。使用的函数包括：

* `int isa_dma_acquire(int channel_number)`
  保留一个 DMA 通道。如果通道已被当前驱动程序或其他驱动程序保留，则返回 EBUSY，成功时返回 0。大多数 ISA 设备无法共享 DMA 通道，因此通常在附加设备时调用此函数。尽管现代总线资源接口使得这个保留操作变得冗余，但它仍然必须在后者之前使用。如果未使用此函数，后续的其他 DMA 函数将导致系统 panic。
* `int isa_dma_release(int channel_number)`
  释放先前保留的 DMA 通道。在释放通道时，不能有正在进行的传输（此外，设备在释放通道后也不能尝试发起传输）。
* `void isa_dmainit(int chan, u_int bouncebufsize)`
  为指定的通道分配一个跳跃缓冲区。请求的缓冲区大小不能超过 64KB。如果传输缓冲区不连续，或者不在 ISA 总线可访问的内存范围内，或者跨越 64KB 边界时，此跳跃缓冲区将在稍后自动使用。如果传输总是从符合这些条件的缓冲区（如通过 `bus_dmamem_alloc()` 分配的缓冲区，并且具有适当的限制）进行，则不必调用 `isa_dmainit()`。但对于通过 DMA 控制器传输任意数据而言，调用此函数是非常方便的。跳跃缓冲区将自动处理散布-聚集问题。

  * *chan* - 通道号
  * *bouncebufsize* - 跳跃缓冲区的大小（以字节为单位）
* `void isa_dmastart(int flags, caddr_t addr, u_int nbytes, int chan)`
  准备开始 DMA 传输。在实际开始设备上的传输之前，必须调用此函数来设置 DMA 控制器。它检查缓冲区是否连续，并且是否位于 ISA 内存范围内，如果不是，则自动使用跳跃缓冲区。如果需要跳跃缓冲区，但未通过 `isa_dmainit()` 设置，或者跳跃缓冲区太小以适应请求的传输大小，系统将 panic。在写请求的情况下，数据将自动复制到跳跃缓冲区。
* flags - 用于确定要执行的操作类型的位掩码。方向位 B\_READ 和 B\_WRITE 是互斥的。

  * B\_READ - 从 ISA 总线读取到内存
  * B\_WRITE - 从内存写入到 ISA 总线
  * B\_RAW - 如果设置，DMA 控制器将在传输结束后记住缓冲区，并自动重新初始化自己，以便再次传输相同的缓冲区（当然，驱动程序可以在开始设备传输之前更改缓冲区中的数据）。如果未设置，则这些参数仅适用于一次传输，`isa_dmastart()` 必须在启动下一次传输之前再次调用。使用 B\_RAW 仅在没有使用跳跃缓冲区时才有意义。

* addr - 缓冲区的虚拟地址
* nbytes - 缓冲区的长度。必须小于或等于 64KB。不允许长度为 0：DMA 控制器会将其理解为 64KB，而内核代码则会将其理解为 0，这将导致不可预测的效果。对于编号为 4 及更高的通道，长度必须是偶数，因为这些通道每次传输 2 个字节。如果长度为奇数，最后一个字节将不会被传输。
* chan - 通道号
* `void isa_dmadone(int flags, caddr_t addr, int nbytes, int chan)`
  在设备报告传输完成后，同步内存。如果这是一次读取操作并且使用了跳跃缓冲区，那么数据将从跳跃缓冲区复制到原始缓冲区。参数与 `isa_dmastart()` 相同。B\_RAW 标志是允许的，但它不会以任何方式影响 `isa_dmadone()`。
* `int isa_dmastatus(int channel_number)`
  返回当前传输中剩余的字节数。如果在 `isa_dmastart()` 中设置了 B\_READ 标志，返回的数字将永远不会等于 0。传输结束时，它将自动重置为缓冲区的长度。正常使用是检查在设备信号表明传输已完成后，剩余的字节数。如果剩余字节数不为 0，则可能是传输出了问题。
* `int isa_dmastop(int channel_number)`
  中止当前传输并返回剩余未传输的字节数。

## 10.8. xxx\_isa\_probe

此函数探测设备是否存在。如果驱动程序支持自动检测设备配置的某些部分（如中断向量或内存地址），则必须在此例程中进行自动检测。

与任何其他总线一样，如果设备无法检测到，或者检测到设备但自检失败或发生其他问题，则返回正值错误。如果设备不存在，则必须返回 `ENXIO`。其他错误值可能表示其他情况。零或负值表示成功。大多数驱动程序返回零表示成功。

负值返回用于支持多个接口的 PnP 设备。例如，支持较旧的兼容接口和较新的高级接口的设备，分别由不同的驱动程序支持。然后，两个驱动程序都将检测设备。返回探测例程中较高值的驱动程序优先（换句话说，返回 0 的驱动程序优先，其次是返回 -1 的，之后是返回 -2 的，依此类推）。因此，仅支持旧接口的设备将由旧驱动程序处理（该驱动程序应该从探测例程返回 -1），而同时支持新接口的设备将由新驱动程序处理（该驱动程序应该从探测例程返回 0）。

设备描述符结构 `xxx_softc` 在调用探测例程之前由系统分配。如果探测例程返回错误，系统会自动释放该描述符。因此，如果发生探测错误，驱动程序必须确保在探测期间使用的所有资源都已释放，并且没有任何东西阻止描述符被安全地释放。如果探测成功完成，描述符将由系统保留，并在稍后传递给 `xxx_isa_attach()` 例程。如果驱动程序返回负值，它不能确保自己具有最高的优先级，其附加例程将被调用。因此，在这种情况下，它还必须在返回之前释放所有资源，并在必要时在附加例程中重新分配它们。当 `xxx_isa_probe()` 返回 0 时，返回之前释放资源也是一个好主意，表现良好的驱动程序应该这么做。但在某些释放资源时存在问题的情况下，驱动程序可以在从探测例程返回 0 和执行附加例程之间保持资源。

一个典型的探测例程开始时获取设备描述符和单元：

```c
struct xxx_softc *sc = device_get_softc(dev);
int unit = device_get_unit(dev);
int pnperror;
int error = 0;

sc->dev = dev; /* 绑定回设备 */
sc->unit = unit;
```

接着检查是否为 PnP 设备。检查通过一个表格进行，表格包含该驱动程序支持的 PnP ID 列表，以及与这些 ID 对应的设备模型的人类可读描述。

```c
pnperror = ISA_PNP_PROBE(device_get_parent(dev), dev, xxx_pnp_ids);
if (pnperror == ENXIO) return ENXIO;
```

`ISA_PNP_PROBE` 的逻辑如下：如果该卡（设备单元）没有被检测为 PnP，则返回 ENOENT。如果它被检测为 PnP，但其检测到的 ID 与表格中的任何 ID 都不匹配，则返回 ENXIO。最后，如果它具有 PnP 支持并且与表格中的某个 ID 匹配，则返回 0，并且通过 `device_set_desc()` 设置表格中的相应描述。

如果驱动程序仅支持 PnP 设备，则条件如下：

```c
if (pnperror != 0)
    return pnperror;
```

对于不支持 PnP 的驱动程序，则不需要特殊处理，因为它们传递一个空的 PnP ID 表格，并且在对 PnP 卡调用时总是会返回 ENXIO。

探测例程通常至少需要一些基本的资源集，如 I/O 端口号，用于找到并探测卡。根据硬件的不同，驱动程序可能能够自动发现其他必要的资源。PnP 设备的所有资源都由 PnP 子系统预设，因此驱动程序无需自行发现它们。

通常，访问设备所需的最小信息是 I/O 端口号。然后，一些设备允许从设备的配置寄存器中获取其余的信息（尽管并非所有设备都这样做）。因此，首先我们尝试获取端口起始值：

```c
sc->port0 = bus_get_resource_start(dev, SYS_RES_IOPORT, 0 /*rid*/);
if (sc->port0 == 0) return ENXIO;
```

基址端口地址保存在结构 `softc` 中供以后使用。如果该端口地址会被频繁使用，那么每次调用资源函数会非常慢。我们如果没有获取到端口地址，就返回错误。一些设备驱动程序可以更聪明地尝试探测所有可能的端口，像这样：

```c
/* 所有可能的基地址 I/O 端口地址表 */
static struct xxx_allports {
    u_short port; /* 端口地址 */
    short used; /* 标志：如果此端口已被某个单元使用 */
} xxx_allports = {
    { 0x300, 0 },
    { 0x320, 0 },
    { 0x340, 0 },
    { 0, 0 } /* 表格结束 */
};

...
int port, i;
...

port = bus_get_resource_start(dev, SYS_RES_IOPORT, 0 /*rid*/);
if (port != 0) {
    for (i = 0; xxx_allports[i].port != 0; i++) {
        if (xxx_allports[i].used || xxx_allports[i].port != port)
            continue;

        /* 找到端口 */
        xxx_allports[i].used = 1;
        /* 在已知端口上进行探测 */
        return xxx_really_probe(dev, port);
    }
    return ENXIO; /* 端口未知或已被使用 */
}

/* 只有在需要猜测端口时才会到这里 */
for (i = 0; xxx_allports[i].port != 0; i++) {
    if (xxx_allports[i].used)
        continue;

    /* 标记为已使用 - 即使我们在此端口上什么也没找到
     * 至少我们以后不会再探测此端口
     */
    xxx_allports[i].used = 1;

    error = xxx_really_probe(dev, xxx_allports[i].port);
    if (error == 0) /* 在该端口找到设备 */
        return 0;
}

/* 探测了所有可能的地址，都没有成功 */
return ENXIO;
```

当然，通常应该使用驱动程序的 `identify()` 函数来处理此类事情。但有一个有效的原因说明为什么在 `probe()` 中处理可能更好：如果这个探测过程会使其他敏感设备出现问题。探测例程是按 `sensitive` 标志排序的：敏感设备首先被探测，其他设备随后被探测。但是，`identify()` 例程在任何探测之前就会被调用，因此它们不会考虑敏感设备，并且可能会扰乱它们。

现在，在我们获取到起始端口后，我们需要设置端口计数（非 PnP 设备除外），因为内核在配置文件中没有这个信息。

```c
if (pnperror /* 仅对于非 PnP 设备 */
    && bus_set_resource(dev, SYS_RES_IOPORT, 0, sc->port0,
    XXX_PORT_COUNT) < 0)
        return ENXIO;
```

最后，分配并激活一块端口地址空间（起始和结束的特殊值意味着“使用我们通过 `bus_set_resource()` 设置的值”）：

```c
sc->port0_rid = 0;
sc->port0_r = bus_alloc_resource(dev, SYS_RES_IOPORT,
    &sc->port0_rid,
    /*start*/ 0, /*end*/ ~0, /*count*/ 0, RF_ACTIVE);

if (sc->port0_r == NULL)
    return ENXIO;
```

现在，拥有访问端口映射寄存器的权限后，我们可以以某种方式与设备进行交互，并检查它是否按预期作出反应。如果没有响应，则可能是此地址处有其他设备，或者根本没有设备。

通常，驱动程序不会在 `attach` 例程之前设置中断处理程序。相反，它们会使用 `DELAY()` 函数在轮询模式下进行探测并设置超时。探测例程永远不能无限期挂起，所有对设备的等待必须使用超时机制。如果设备在超时时间内没有响应，那么它可能已经损坏或配置错误，驱动程序必须返回错误。在确定超时间隔时，给设备一些额外的时间以确保安全：尽管 `DELAY()` 在任何机器上都应该延迟相同的时间，但它有一定的误差范围，具体取决于 CPU 的不同。

```c
/* 如果探测例程确实想检查中断是否真正工作，它也可以配置和探测中断，但这并不推荐。 */

/* 以某种非常设备特定的方式实现 */
if (error = xxx_probe_ports(sc))
    goto bad; /* 在返回之前会释放资源 */

```

函数 `xxx_probe_ports()` 还可能根据其发现的具体设备型号设置设备描述。但如果只有一个支持的设备型号，可以硬编码来实现。当然，对于 PnP 设备，PnP 支持会自动从表中设置描述。

```c
if (pnperror)
    device_set_desc(dev, "我们的设备型号 1234");
```

然后，探测例程应该通过读取设备配置寄存器来发现所有资源的范围，或者确保它们已经由用户明确设置。我们将通过一个板载内存的例子来考虑它。探测例程应该尽可能不具侵入性，因此资源的其余部分（除了端口）的分配和功能检查最好留给 `attach` 例程来处理。

内存地址可以在内核配置文件中指定，或者在某些设备上它可能已经预先配置在非易失性配置寄存器中。如果两种来源都可用且不同，应该使用哪一种？可能如果用户在内核配置文件中明确设置了地址，他们知道自己在做什么，因此应该优先使用此地址。实现的示例可能是：

```c
/* 尝试首先找出配置地址 */
sc->mem0_p = bus_get_resource_start(dev, SYS_RES_MEMORY, 0 /*rid*/);
if (sc->mem0_p == 0) { /* 没有，用户未指定 */
    sc->mem0_p = xxx_read_mem0_from_device_config(sc);

    if (sc->mem0_p == 0)
        /* 也无法从设备配置寄存器中获取 */
        goto bad;
} else {
    if (xxx_set_mem0_address_on_device(sc) < 0)
        goto bad; /* 设备不支持该地址 */
}

/* 就像端口一样，设置内存大小，
 * 对于某些设备，内存大小不是固定的
 * 而是应该从设备配置寄存器中读取，
 * 以适应不同型号的设备。另一种选择是让用户将内存大小设置为“msize”配置
 * 资源，这将由 ISA 总线自动处理。
 */
if (pnperror) { /* 仅对于非 PnP 设备 */
    sc->mem0_size = bus_get_resource_count(dev, SYS_RES_MEMORY, 0 /*rid*/);
    if (sc->mem0_size == 0) /* 用户未指定 */
        sc->mem0_size = xxx_read_mem0_size_from_device_config(sc);

    if (sc->mem0_size == 0) {
        /* 假设这是一个非常老旧的设备模型，没有
         * 自动配置功能，并且用户未给出首选项，
         * 因此假设最小的情况
         * （当然，实际值会随着驱动程序变化）
         */
        sc->mem0_size = 8 * 1024;
    }

    if (xxx_set_mem0_size_on_device(sc) < 0)
        goto bad; /* 设备不支持该大小 */

    if (bus_set_resource(dev, SYS_RES_MEMORY, /*rid*/0,
            sc->mem0_p, sc->mem0_size) < 0)
        goto bad;
} else {
    sc->mem0_size = bus_get_resource_count(dev, SYS_RES_MEMORY, 0 /*rid*/);
}
```

IRQ 和 DRQ 的资源检查可以通过类比来进行。

```c
/* 如果一切顺利，则释放所有资源并返回成功 */
xxx_free_resources(sc);
return 0;
```

最后，处理麻烦的情况。在返回之前，所有资源应该被释放。我们利用了这样的事实：在结构 `softc` 传递给我们之前，它会被清零，所以我们可以发现是否分配了某些资源：如果有分配的资源，其描述符不为零。

```c
bad:

xxx_free_resources(sc);
if (error)
    return error;
else /* 精确的错误未知 */
    return ENXIO;
```

这就是探测例程的所有内容。资源的释放在多个地方进行，因此它被移到一个函数中，函数代码可能如下所示：

```c
static void
xxx_free_resources(sc)
    struct xxx_softc *sc;
{
    /* 检查每个资源，如果不为零则释放 */

    /* 中断处理程序 */
    if (sc->intr_r) {
        bus_teardown_intr(sc->dev, sc->intr_r, sc->intr_cookie);
        bus_release_resource(sc->dev, SYS_RES_IRQ, sc->intr_rid, sc->intr_r);
        sc->intr_r = 0;
    }

    /* 所有我们可能分配的内存映射 */
    if (sc->data_p) {
        bus_dmamap_unload(sc->data_tag, sc->data_map);
        sc->data_p = 0;
    }
    if (sc->data) { /* sc->data_map 可能合法地等于 0 */
        /* map 也将被释放 */
        bus_dmamem_free(sc->data_tag, sc->data, sc->data_map);
        sc->data = 0;
    }
    if (sc->data_tag) {
        bus_dma_tag_destroy(sc->data_tag);
        sc->data_tag = 0;
    }

    /* ... 释放其他映射和标签（如果有的话）... */

    if (sc->parent_tag) {
        bus_dma_tag_destroy(sc->parent_tag);
        sc->parent_tag = 0;
    }

    /* 释放所有总线资源 */
    if (sc->mem0_r) {
        bus_release_resource(sc->dev, SYS_RES_MEMORY, sc->mem0_rid, sc->mem0_r);
        sc->mem0_r = 0;
    }
    ...
    if (sc->port0_r) {
        bus_release_resource(sc->dev, SYS_RES_IOPORT, sc->port0_rid, sc->port0_r);
        sc->port0_r = 0;
    }
}
```

## 10.9. xxx\_isa\_attach

`attach` 例程实际上将驱动程序连接到系统，如果探测例程返回成功且系统选择附加该驱动程序。如果探测例程返回 0，那么 `attach` 例程可以期望接收到设备结构 `softc`，并且它的内容应当与探测例程设置的一致。此外，如果探测例程返回 0，它还可以期望该设备的 `attach` 例程将在未来某个时刻被调用。如果探测例程返回负值，则驱动程序不应作出这些假设。

`attach` 例程返回 0 表示成功，否则返回错误代码。

`attach` 例程开始的方式与探测例程类似，首先将一些常用的数据存储到更易访问的变量中。

```c
struct xxx_softc *sc = device_get_softc(dev);
int unit = device_get_unit(dev);
int error = 0;
```

然后分配并激活所有必要的资源。由于通常端口范围会在返回探测例程时被释放，因此必须再次分配它。我们期望探测例程已经正确设置了所有资源范围，并将它们保存在结构 `softc` 中。如果探测例程已经分配了某些资源，则不需要再次分配（如果重新分配，则视为错误）。

```c
sc->port0_rid = 0;
sc->port0_r = bus_alloc_resource(dev, SYS_RES_IOPORT, &sc->port0_rid,
    /*start*/ 0, /*end*/ ~0, /*count*/ 0, RF_ACTIVE);

if(sc->port0_r == NULL)
    return ENXIO;

/* 板载内存 */
sc->mem0_rid = 0;
sc->mem0_r = bus_alloc_resource(dev, SYS_RES_MEMORY, &sc->mem0_rid,
    /*start*/ 0, /*end*/ ~0, /*count*/ 0, RF_ACTIVE);

if(sc->mem0_r == NULL)
    goto bad;

/* 获取其虚拟地址 */
sc->mem0_v = rman_get_virtual(sc->mem0_r);
```

DMA 请求通道（DRQ）的分配方法类似。要初始化它，可以使用 `isa_dma*()` 系列函数。例如：

```c
isa_dmacascade(sc→drq0);
```

中断请求行（IRQ）稍微特殊一些。除了分配外，驱动程序的中断处理程序应该与其关联。历史上，在旧的 ISA 驱动程序中，系统传递给中断处理程序的参数是设备的单元号。但在现代驱动程序中，约定是传递指向 `softc` 结构的指针。这样做的一个重要原因是，当 `softc` 结构动态分配时，从 `softc` 获取单元号很容易，而从单元号获取 `softc` 却很困难。此外，这种约定使得不同总线的驱动程序看起来更加统一，并且能够共享代码：每个总线有自己的探测、附加、分离和其他总线特定的例程，而大部分驱动程序代码可以在它们之间共享。

```c
sc->intr_rid = 0;
sc->intr_r = bus_alloc_resource(dev, SYS_RES_MEMORY, &sc->intr_rid,
    /*start*/ 0, /*end*/ ~0, /*count*/ 0, RF_ACTIVE);

if(sc->intr_r == NULL)
    goto bad;

/*
 * XXX_INTR_TYPE 应根据驱动程序的类型来定义，
 * 例如作为 INTR_TYPE_CAM 用于 CAM 驱动程序
 */
error = bus_setup_intr(dev, sc->intr_r, XXX_INTR_TYPE,
    (driver_intr_t *) xxx_intr, (void *) sc, &sc->intr_cookie);
if(error)
    goto bad;
```

这样完成了资源分配并设置了中断处理程序。

如果设备需要对主内存进行 DMA 操作，则应像之前描述的那样分配内存：

```c
error = bus_dma_tag_create(NULL, /*alignment*/ 4,
              /*boundary*/ 0, /*lowaddr*/ BUS_SPACE_MAXADDR_24BIT,
              /*highaddr*/ BUS_SPACE_MAXADDR, /*filter*/ NULL, /*filterarg*/ NULL,
              /*maxsize*/ BUS_SPACE_MAXSIZE_24BIT,
              /*nsegments*/ BUS_SPACE_UNRESTRICTED,
              /*maxsegsz*/ BUS_SPACE_MAXSIZE_24BIT, /*flags*/ 0,
              &sc->parent_tag);
if(error)
    goto bad;

/* 很多属性会从父标签继承
 * sc->data 应该指向共享数据的结构，
 * 例如，对于一个环形缓冲区，它可以是：
 * struct {
 *   u_short rd_pos;
 *   u_short wr_pos;
 *   char    bf[XXX_RING_BUFFER_SIZE]
 * } *data;
 */
error = bus_dma_tag_create(sc->parent_tag, 1,
              0, BUS_SPACE_MAXADDR, 0, /*filter*/ NULL, /*filterarg*/ NULL,
              /*maxsize*/ sizeof(*sc->data), /*nsegments*/ 1,
              /*maxsegsz*/ sizeof(*sc->data), /*flags*/ 0,
              &sc->data_tag);
if(error)
    goto bad;

error = bus_dmamem_alloc(sc->data_tag, &sc->data, /*flags*/ 0,
              &sc->data_map);
if(error)
    goto bad;

/* xxx_alloc_callback() 只是将物理地址保存在
 * 传递给它的指针中，在这种情况下是 &sc->data_p。
 * 参见总线内存映射部分的详细信息。
 * 它可以像这样实现：
 *
 * static void
 * xxx_alloc_callback(void *arg, bus_dma_segment_t *seg,
 *     int nseg, int error)
 * {
 *    *(bus_addr_t *)arg = seg[0].ds_addr;
 * }
 */
bus_dmamap_load(sc->data_tag, sc->data_map, (void *)sc->data,
              sizeof(*sc->data), xxx_alloc_callback, (void *)&sc->data_p,
              /*flags*/ 0);
```

在分配了所有必要的资源之后，设备应进行初始化。初始化可能包括测试所有预期功能是否正常工作。

```c
if(xxx_initialize(sc) < 0)
    goto bad;
```

总线子系统会自动在控制台上打印探测过程中设置的设备描述信息。但是，如果驱动程序希望打印一些额外的设备信息，它也可以这么做，例如：

```c
device_printf(dev, "has on-card FIFO buffer of %d bytes\n", sc->fifosize);
```

如果初始化过程中遇到任何问题，建议在返回错误之前打印相关信息。

附加例程的最终步骤是将设备附加到内核中的其功能子系统。具体如何操作取决于驱动程序的类型：字符设备、块设备、网络设备、CAM SCSI 总线设备等。

如果一切顺利，则返回成功：

```c
error = xxx_attach_subsystem(sc);
if(error)
    goto bad;

return 0;
```

最后，处理那些棘手的情况。在返回错误之前，所有资源都应该被释放。我们利用结构 softc 在传递给我们之前会被清零的事实，因此可以通过检查其描述符是否为非零值来判断是否已分配某些资源。

```c
bad:

xxx_free_resources(sc);
if(error)
    return error;
else /* 精确错误未知 */
    return ENXIO;
```

这就是附加例程的全部内容。

## 10.10. xxx\_isa\_detach

如果此函数存在且驱动程序被编译为可加载模块，则该驱动程序可以被卸载。这个功能在硬件支持热插拔时非常重要，但 ISA 总线不支持热插拔，因此这个功能对于 ISA 设备来说并不是特别重要。卸载驱动程序的能力在调试时可能有用，但在许多情况下，仅在旧版本驱动程序某种方式卡住系统并且需要重新启动的情况下，才需要安装新版本的驱动程序，因此花费时间编写卸载例程可能不值得。另一个认为卸载可以在生产机器上升级驱动程序的论点似乎大多是理论性的。在生产机器上执行安装新版本驱动程序的操作是危险的操作，应当避免（并且在系统处于安全模式时是不允许的）。尽管如此，为了完整性，仍然可以提供卸载例程。

卸载例程返回 0 表示驱动程序已成功卸载，或者返回错误码。

卸载的逻辑是附加例程的镜像。首先要做的是将驱动程序从内核子系统中分离。如果设备当前正在打开，那么驱动程序有两个选择：拒绝卸载或强制关闭并继续卸载。选择哪种方式取决于特定内核子系统是否支持强制关闭以及驱动程序作者的偏好。通常情况下，强制关闭似乎是更为常见的选择。

```c
struct xxx_softc *sc = device_get_softc(dev);
int error;

error = xxx_detach_subsystem(sc);
if(error)
    return error;
```

接下来，驱动程序可能希望将硬件重置到某个一致的状态。这包括停止任何正在进行的传输，禁用 DMA 通道和中断，以避免设备对内存造成损坏。对于大多数驱动程序来说，这正是关闭例程所执行的操作，因此如果在驱动程序中包含了它，我们可以直接调用它。

```c
xxx_isa_shutdown(dev);
```

最后释放所有资源并返回成功：

```c
xxx_free_resources(sc);
return 0;
```

## 10.11. xxx\_isa\_shutdown

当系统即将关闭时，调用此例程。它期望将硬件置于某个一致的状态。对于大多数 ISA 设备来说，不需要特别的操作，因为设备会在重新启动时重新初始化。然而，某些设备需要通过特殊的程序进行关闭，以确保它们在软重启后能被正确检测（尤其是许多设备使用专有的识别协议）。无论如何，禁用 DMA 和中断并停止正在进行的传输是一个好主意。具体操作取决于硬件，因此我们在此不做详细讨论。

## 10.12. xxx\_intr

当接收到中断时，若中断来自该特定设备，则会调用中断处理程序。ISA 总线不支持中断共享（除非在一些特殊情况下），因此在实际操作中，如果中断处理程序被调用，那么中断几乎可以肯定是来自该设备。然而，中断处理程序必须轮询设备寄存器，并确保中断确实是由该设备产生的。如果不是，则应该直接返回。

旧的 ISA 驱动程序约定是将设备单元号作为参数传递。这种方式已经过时，新的驱动程序会接收在调用 `bus_setup_intr()` 时为其指定的参数。按照新的约定，应该是指向结构 softc 的指针。所以，中断处理程序通常是这样开始的：

```c
static void
xxx_intr(struct xxx_softc *sc)
{
```

它以由 `bus_setup_intr()` 中的中断类型参数指定的中断优先级运行。这意味着同一类型的所有其他中断以及所有软件中断都会被禁用。

为了避免竞态条件，中断处理程序通常会写成一个循环：

```c
while(xxx_interrupt_pending(sc)) {
    xxx_process_interrupt(sc);
    xxx_acknowledge_interrupt(sc);
}
```

中断处理程序必须仅对设备进行中断确认，而不是对中断控制器进行确认，后者由系统负责处理。
