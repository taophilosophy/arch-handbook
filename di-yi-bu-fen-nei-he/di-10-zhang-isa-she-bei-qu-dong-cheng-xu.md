# 第 10 章 ISA 设备驱动程序

## 10.1. 概要

本章介绍了编写 ISA 设备驱动程序相关的问题。这里呈现的伪代码非常详细，类似于真实代码，但仍然只是伪代码。它避免了与讨论主题无关的细节。真实驱动程序的源代码中可以找到真实示例。特别是驱动程序 ep 和 aha 是很好的信息来源。

## 10.2. 基本信息

一个典型的 ISA 驱动程序需要以下包含文件：

```
#include <sys/module.h>
#include <sys/bus.h>
#include <machine/bus.h>
#include <machine/resource.h>
#include <sys/rman.h>

#include <isa/isavar.h>
#include <isa/pnpvar.h>
```

它们描述了 ISA 和通用总线子系统的特定内容。

总线子系统以面向对象的方式实现，其主要结构通过关联的方法函数访问。

ISA 驱动程序实现的总线方法列表与任何其他总线的方法类似。对于一个名为"xxx"的假设驱动程序，它们将是：

* static void xxx_isa_identify (driver_t *, device_t); 通常用于总线驱动程序，而不是设备驱动程序。但对于 ISA 设备，此方法可能具有特殊用途：如果设备提供一些特定于设备的（非即插即用）方式来自动检测设备，则此例程可能会实现它。
* static int xxx_isa_probe (device_t dev); 探测已知（或即插即用）位置的设备。此例程还可以适应于部分配置设备的设备特定参数的自动检测。
* static int xxx_isa_attach (device_t dev); 附加并初始化设备。
* static int xxx_isa_detach (device_t dev); 在卸载驱动程序模块之前分离设备。
* static int xxx_isa_shutdown (device_t dev); 在系统关机之前执行设备关机。
* static int xxx_isa_suspend (device_t dev); 在系统进入省电状态之前暂停设备。也可能中止进入省电状态的转换。
* static int xxx_isa_resume (device_t dev); 在从省电状态返回后恢复设备活动。

xxx_isa_probe() 和 xxx_isa_attach() 是必需的，其余的例程根据设备的需求而定。

驱动程序与系统连接，具有以下一组描述。

```
    /* table of supported bus methods */
    static device_method_t xxx_isa_methods[] = {
        /* list all the bus method functions supported by the driver */
        /* omit the unsupported methods */
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

这里的 struct xxx_softc 是一个特定于设备的结构，包含私有驱动程序数据和驱动程序资源的描述符。总线代码会根据需要为每个设备自动分配一个 softc 描述符。

如果驱动程序实现为可加载模块，则会调用 load_function() 来执行驱动程序特定的初始化或清理工作，当加载或卸载驱动程序时会将 load_argument 作为其参数之一传递。如果驱动程序不支持动态加载（换句话说，它必须始终链接到内核中），那么这些值应设置为 0，最后一个定义将如下所示：

```
 DRIVER_MODULE(xxx, isa, xxx_isa_driver,
       xxx_devclass, 0, 0);
```

如果驱动程序用于支持 PnP 的设备，则必须定义支持的 PnP ID 表。该表包括此驱动程序支持的一系列 PnP ID，以及具有这些 ID 的硬件类型和型号的人类可读描述。它看起来像：

```
    static struct isa_pnp_id xxx_pnp_ids[] = {
        /* a line for each supported PnP ID */
        { 0x12345678,   "Our device model 1234A" },
        { 0x12345679,   "Our device model 1234B" },
        { 0,        NULL }, /* end of table */
    };
```

如果驱动程序不支持 PnP 设备，则仍然需要一个空的 PnP ID 表，如下所示：

```
    static struct isa_pnp_id xxx_pnp_ids[] = {
        { 0,        NULL }, /* end of table */
    };
```

## 10.3. device_t 指针

device_t 是设备结构的指针类型。在这里，我们只考虑从设备驱动程序编写者的角度来看感兴趣的方法。用于操作设备结构中的值的方法有：

* device_t device_get_parent(dev) 获取设备的父总线。
* driver_t device_get_driver(dev) 获取指向其驱动程序结构的指针。
* char *device_get_name(dev) 获取驱动程序名称，例如我们的示例中的 "xxx" 。
* int device_get_unit(dev) 获取单元号（从 0 开始对每个驱动程序关联的设备进行编号）。
* char *device_get_nameunit(dev) 获取包括单元号在内的设备名称，如 "xxx0"，"xxx1" 等等。
* char *device_get_desc(dev) 获取设备描述。通常，它以人类可读的形式描述设备的确切型号。
* device_set_desc(dev, desc) 设置描述。这将使设备描述指向字符串 desc，之后可能不会被释放或更改。
* device_set_desc_copy(dev, desc) 设置描述。描述被复制到内部动态分配的缓冲区中，因此字符串 desc 可以在之后更改而不会产生不良影响。
* void *device_get_softc(dev) 获取与此设备关联的设备描述符指针（结构 xxx_softc ）。
* u_int32_t device_get_flags(dev) 获取配置文件中指定的设备标志。

便捷功能 device_printf(dev, fmt, …) 可用于打印设备驱动程序中的消息。它会自动将单元名称和冒号前置到消息之前。

device_t 方法在文件 kern/bus_subr.c 中实现。

## 10.4. 配置文件和自动配置期间识别和探测顺序

ISA 设备在内核配置文件中描述如下：

```
device xxx0 at isa? port 0x300 irq 10 drq 5
       iomem 0xd0000 flags 0x1 sensitive
```

port的值，IRQ 等等，都会转换为与设备关联的资源值。这些是可选的，取决于设备对自动配置的需求和能力。例如，一些设备根本不需要 DRQ，而一些允许驱动程序从设备配置中读取 IRQ 设置ports。如果一台机器有多个 ISA 总线，则可以在配置行中指定确切的总线，比如 isa0 或 isa1 ，否则设备将在所有 ISA 总线上搜索。

sensitive 是一个资源请求，要求在所有非敏感设备之前对此设备进行探测。它得到支持，但似乎在任何当前驱动程序中都没有使用。

对于许多情况下的传统 ISA 设备，驱动程序仍然能够检测到配置参数。但系统中要配置的每个设备都必须有一个配置行。如果系统中安装了两个相同类型的设备，但相应驱动程序只有一条配置行，即：

```
device xxx0 at isa?
```

```
then only one device will be configured.
```

但对于通过即插即用或某种专有协议支持自动识别的设备，只需一行配置就足以配置系统中的所有设备，就像上面那个或只是简单地：

```
device xxx at isa?
```

如果驱动程序同时支持自动识别和传统设备，并且同时在一台机器上安装了这两种设备，则只需要在配置文件中描述传统设备即可。自动识别设备将自动添加。

当 ISA 总线自动配置时，事件发生如下：

所有驱动程序的识别例程（包括识别所有 PnP 设备的 PnP 识别例程）都是随机调用的。当它们识别设备时，它们将设备添加到 ISA 总线的列表中。通常，驱动程序的识别例程会将其驱动程序与新设备关联起来。PnP 识别例程还不知道其他驱动程序，因此它不会将任何驱动程序与它添加的新设备关联起来。

使用 PnP 协议将 PnP 设备置于睡眠状态，以防止它们被探测为传统设备。

标记为 sensitive 的非 PnP 设备的探测例程被调用。如果设备探测成功，则为其调用附加例程。

所有非 PNP 设备的探测和附加例程同样被调用。

PnP 设备从睡眠状态恢复并分配它们请求的资源：I/O 和内存地址范围，IRQ 和 DRQ，所有这些资源都不会与已连接的传统设备发生冲突。

然后，对于每个 PnP 设备，调用所有当前 ISA 驱动程序的探测例程。第一个声称拥有该设备的驱动程序将被附加。可能会有多个驱动程序声称拥有具有不同优先级的设备；在这种情况下，优先级最高的驱动程序获胜。探测例程必须调用 ISA_PNP_PROBE() 来将实际 PnP ID 与驱动程序支持的 ID 列表进行比较，如果 ID 不在表中，则返回失败。这意味着绝对每个驱动程序，即使不支持任何 PnP 设备的驱动程序，也必须调用 ISA_PNP_PROBE() ，至少使用空的 PnP ID 表来在未知的 PnP 设备上返回失败。

探针例程在出错时返回正值（错误代码），成功时返回零或负值。

当 PnP 设备支持多个接口时，使用负的返回值。例如，支持不同驱动程序的旧兼容接口和新高级接口。那么两个驱动程序都会检测到该设备。在探针例程中返回更高值的驱动程序优先（换句话说，返回 0 的驱动程序具有最高优先级，返回 -1 的次之，返回 -2 的再次之，依此类推）。结果是仅支持旧接口的设备将由旧驱动程序处理（该驱动程序应从探针例程中返回 -1），而还支持新接口的设备将由新驱动程序处理（该驱动程序应从探针例程中返回 0）。如果多个驱动程序返回相同值，则先调用哪个获胜。因此，如果驱动程序返回值 0，则可以确定它获胜了优先级仲裁。

设备特定的识别例程还可以为设备分配驱动程序的类别。然后，为该设备探查该类别中的所有驱动程序，就像 PnP 一样。此功能尚未在任何现有驱动程序中实现，并且在本文档中不再考虑。

当探测传统设备时禁用即插即用设备，它们不会被附加两次（一次作为传统设备，一次作为即插即用设备）。但是，在设备相关的识别例程中，驱动程序有责任确保同一设备不会被驱动程序附加两次：一次作为传统用户配置，一次作为自动识别。

对于自动识别的设备（即即插即用和特定设备），另一个实际后果是无法从内核配置文件传递标志给它们。因此，它们要么根本不使用标志，要么对所有自动识别的设备使用设备单元 0 的标志，要么使用 sysctl 接口而不是标志。

其他不寻常的配置可能通过直接访问功能族 resource_query_ *() 和 resource_* _value() 的配置资源来实现。它们的实现位于 kern/subr_bus.c 中。旧的 IDE 硬盘驱动程序 i386/isa/wd.c 包含了这种用法的示例。但是，始终应优先使用标准配置手段。将配置资源的解析留给总线配置代码。

## 10.5. 资源

用户输入到内核配置文件中的信息将被处理并作为配置资源传递给内核。这些信息将被总线配置代码解析，并转换为结构设备 device_t 的值以及与其关联的总线资源。驱动程序可以直接访问配置资源，对于更复杂的配置情况使用函数 resource_* 。然而，一般情况下这既不需要也不推荐，因此本问题在此不再讨论。

总线资源与每个设备相关联。它们由类型和类型内编号确定。对于 ISA 总线，定义了以下类型：

* SYS_RES_IRQ - 中断号
* SYS_RES_DRQ - ISA DMA 通道号
* SYS_RES_MEMORY - 设备内存范围映射到系统内存空间
* SYS_RES_IOPORT - 设备 I/O 寄存器范围

类型内的枚举从 0 开始，因此，如果设备有两个内存区域，则类型为 SYS_RES_MEMORY 的资源将被编号为 0 和 1。资源类型与 C 语言类型无关，所有资源值都具有 C 语言类型 unsigned long ，必要时必须进行转换。资源编号不必连续，尽管对于 ISA 而言，它们通常是连续的。ISA 设备允许的资源编号为：

```
          IRQ: 0-1
          DRQ: 0-1
          MEMORY: 0-3
          IOPORT: 0-7
```

所有资源都表示为范围，具有起始值和计数。对于 IRQ 和 DRQ 资源，计数通常等于 1。内存的值是指物理地址。

可以对资源执行三种类型的活动：

* 设置/获取
* 分配/释放
* 激活/停用

设置定义了资源使用的范围。分配会保留请求的范围，以防止其他驱动程序能够保留它（并检查没有其他驱动程序已经保留了这个范围）。激活会使资源对驱动程序可访问，通过执行必要的操作（例如，对于内存来说，会映射到内核虚拟地址空间）。

用于操作资源的函数有：

* int bus_set_resource(device_t dev, int type, int rid, u_long start, u_long count) 为资源设置范围。如果成功返回 0，否则返回错误代码。通常，只有当 type 、 rid 、 start 或 count 中有一个值超出允许范围时，此函数才会返回错误。

  * dev - 驱动程序的设备
  * type - 资源类型，SYS_RES_*
  * rid - 资源类型内的资源编号（ID）
  * start, count - 资源范围
* int bus_get_resource(device_t dev, int type, int rid, u_long *startp, u_long *countp) 获取资源范围。如果成功返回 0，如果资源尚未定义则返回错误代码。
* u_long bus_get_resource_start(device_t dev, int type, int rid) u_long bus_get_resource_count (device_t dev, int type, int rid) 仅获取开始或计数的便利函数。在发生错误时返回 0，因此，如果资源的开始值在合法值中包含 0，则无法确定该值是 0 还是发生了错误。幸运的是，对于附加驱动程序的 ISA 资源，可能不会有一个开始值等于 0。
* void bus_delete_resource(device_t dev, int type, int rid) 删除资源，使其未定义。
* struct resource * bus_alloc_resource(device_t dev, int type, int *rid, u_long start, u_long end, u_long count, u_int flags) 将资源分配为在开始和结束之间的计数值范围，这些值未被其他任何人分配。遗憾的是，不支持对齐。如果资源尚未设置，则会自动创建。开始值为 0 和结束值为~0（全为 1）的特殊值意味着先前由 bus_set_resource() 设置的固定值必须改为使用：开始和计数本身，以及结束=（开始+计数），在这种情况下，如果资源在之前未定义，则会返回错误。尽管 rid 是通过引用传递的，但 ISA 总线的资源分配代码不会在任何地方设置它。（其他总线可能会使用不同的方法并对其进行修改）。

标志是一个位图，对调用者感兴趣的标志有：

* RF_ACTIVE - 导致资源在分配后自动激活。
* RF_SHAREABLE - 资源可以同时被多个驱动程序共享。
* RF_TIMESHARE - 资源可以被多个驱动程序共享，即可以同时被多个驱动程序分配，但在任何给定时刻只能由一个驱动程序激活。
* 返回错误时返回 0。分配的值可以通过使用方法 rhand_*() 从返回的句柄中获取。
* `int bus_release_resource(device_t dev, int type, int rid, struct resource *r)`
* 释放资源，r 是 bus_alloc_resource() 返回的句柄。成功返回 0，否则返回错误代码。
* `int bus_activate_resource(device_t dev, int type, int rid, struct resource *r) int bus_deactivate_resource(device_t dev, int type, int rid, struct resource *r)`
* 激活或停用资源。成功时返回 0，否则返回错误码。如果资源是共享时间的，并且当前由另一个驱动程序激活，则返回 EBUSY 。
* `int bus_setup_intr(device_t dev, struct resource *r, int flags, driver_intr_t *handler, void *arg, void **cookiep) int bus_teardown_intr(device_t dev, struct resource *r, void *cookie)`
* 将中断处理程序与设备关联或取消关联。成功时返回 0，否则返回错误码。
* r - 已激活的资源处理程序，描述 IRQ 标志 - 中断优先级级别之一：

  * INTR_TYPE_TTY - 终端和其他类似字符型设备。要对它们进行掩码，请使用 spltty() 。
  * (INTR_TYPE_TTY | INTR_TYPE_FAST) - 具有小输入缓冲区的终端类型设备，在输入时对数据丢失至关重要（例如老式串行ports）。要对它们进行掩码，请使用 spltty() 。
  * INTR_TYPE_BIO - 块设备，除了那些在 CAM 控制器上的设备。要对它们进行掩码，请使用 splbio() 。
  * INTR_TYPE_CAM - CAM（Common Access Method）总线控制器。要屏蔽它们，请使用 splcam() 。
  * INTR_TYPE_NET - 网络接口控制器。要屏蔽它们，请使用 splimp() 。
  * INTR_TYPE_MISC - 杂项设备。除了通过屏蔽所有中断的 splhigh() 外，没有其他屏蔽它们的方法。

当中断处理程序执行时，所有其他与其优先级相匹配的中断将被屏蔽。唯一的例外是 MISC 级别，其他中断都不会被屏蔽，也不会被任何其他中断屏蔽。

* 处理程序 - 指向处理程序函数的指针，类型 driver_intr_t 被定义为 void driver_intr_t(void *)
* arg - 传递给处理程序以识别特定设备的参数。在处理程序中，它从 void*强制转换为任何真实类型。对 ISA 中断处理程序的旧惯例是使用单元号作为参数，新（推荐）惯例是使用指向设备 softc 结构的指针。
* cookie[p] - 从 setup() 接收到的值用于标识在传递给 teardown() 时要使用的处理程序

定义了许多方法来操作资源处理程序（struct resource *）。 设备驱动程序编写人员感兴趣的是：

* u_long rman_get_start(r) u_long rman_get_end(r) 获取分配的资源范围的起始和结束。
* 获取激活内存资源的虚拟地址。

## 10.6. 总线内存映射

在许多情况下，数据通过内存在驱动程序和设备之间交换。有两种可能的变体：

(a) 存储器位于设备卡上

(b) 存储器是计算机的主存储器

在情况（a）中，驱动程序始终根据需要在设备卡上的存储器和主存储器之间来回复制数据。要将设备卡上的存储器映射到内核虚拟地址空间，必须定义设备卡上的物理地址和长度作为 SYS_RES_MEMORY 资源。然后可以分配并激活该资源，并使用 rman_get_virtual() 获得其虚拟地址。较旧的驱动程序曾经使用函数 pmap_mapdev() 来实现此目的，不应再直接使用。现在这是资源激活的内部步骤之一。

大多数 ISA 卡的内存配置为物理位置范围在 640KB-1MB 之间。一些 ISA 卡需要更大的内存范围，应放置在 16MB 以下（因为 ISA 总线上存在 24 位地址限制）。在这种情况下，如果计算机的内存多于设备内存的起始地址（换句话说，它们重叠），必须在设备使用的地址范围配置内存空洞。许多 BIOS 允许在 14MB 或 15MB 处启动 1MB 的内存空洞。如果 BIOS 正确报告它们，FreeBSD 可以正确处理内存空洞（此功能可能在旧 BIOS 上失效）。

在情况（b）中，只发送数据的地址给设备，设备使用 DMA 实际访问主内存中的数据。存在两个限制：首先，ISA 卡只能访问 16MB 以下的内存。其次，虚拟地址空间中的连续页面在物理地址空间中可能不连续，因此设备可能需要执行分散/聚集操作。总线子系统为其中一些问题提供了现成的解决方案，其余问题必须由驱动程序自行解决。

用于 DMA 内存分配的两种结构， bus_dma_tag_t 和 bus_dmamap_t 。标签描述了 DMA 内存所需的属性。映射表示根据这些属性分配的内存块。可以将多个映射与同一标签关联。

标签以树状层次结构组织，属性可以继承。子标签继承其父标签的所有要求，并且可以使它们更严格，但绝不更松散。

通常为每个设备单元创建一个顶层标签（没有父级）。如果每个设备需要具有不同要求的多个内存区域，则可以为每个内存区域创建一个父标签的子标签。

标签可以通过两种方式创建地图。

首先，可以分配符合标记要求的一块连续内存（稍后可以释放）。这通常用于为与设备通信的相对长寿命的内存区域分配空间。将这样的内存加载到映射中非常简单：它始终被视为适当物理内存范围内的一个块。

其次，可以将任意虚拟内存区域加载到映射中。将检查该内存的每个页面是否符合映射要求。如果符合，则保留在原始位置。如果不符合，则分配一个新的符合要求的“反弹页面”并用作中间存储。在写入来自非符合原始页面的数据时，它们将首先被复制到其反弹页面，然后从反弹页面传输到设备。在读取数据时，数据将从设备传输到反弹页面，然后复制到其非符合原始页面。在原始页面和反弹页面之间复制的过程称为同步。通常，这是基于每次传输的基础进行的：每次传输都会加载缓冲区，完成传输后卸载缓冲区。

在 DMA 内存上运行的函数有：

* int bus_dma_tag_create(bus_dma_tag_t parent, bus_size_t alignment, bus_size_t boundary, bus_addr_t lowaddr, bus_addr_t highaddr, bus_dma_filter_t *filter, void *filterarg, bus_size_t maxsize, int nsegments, bus_size_t maxsegsz, int flags, bus_dma_tag_t *dmat) 创建一个新标签。成功返回 0，否则返回错误代码。

  * parent - 父标签，或者为 NULL 以创建顶级标签。
  * alignment - 为分配给此标签的内存区域所需的物理对齐。对于未来的 bus_dmamem_alloc() 而不是 bus_dmamap_create() 调用，请使用值 1 表示“无特定对齐”。
  * 边界 - 分配内存时不能越过的物理地址边界。对于“无边界”，请使用值 0。仅适用于未来的 bus_dmamem_alloc() 而不是 bus_dmamap_create() 调用。必须是 2 的幂。如果计划在非级联 DMA 模式下使用内存（即，DMA 地址将不是由设备本身提供，而是由 ISA DMA 控制器提供），则边界不能大于 64KB（64*1024），因为 DMA 硬件的限制。
  * lowaddr，highaddr - 这些名称有点误导；这些值用于限制用于分配内存的物理地址范围。具体含义取决于计划的未来用途：

    * 对于 bus_dmamem_alloc() ，所有从 0 到 lowaddr-1 的地址被视为允许的，更高的地址被禁止。
    * 对于 bus_dmamap_create() ，在包含范围 [lowaddr; highaddr] 之外的所有地址被视为可访问。范围内的页面地址传递给过滤器函数，该函数决定它们是否可访问。如果未提供过滤器函数，则认为整个范围都是不可访问的。
    * 对于 ISA 设备，没有过滤器函数的正常值为：lowaddr = BUS_SPACE_MAXADDR_24BIT
      highaddr = BUS_SPACE_MAXADDR
  * filter，filterarg - 过滤函数及其参数。如果对于 filter 传递了 NULL，则在进行 bus_dmamap_create() 时将认为整个范围[lowaddr，highaddr]是不可访问的。否则，将范围[lowaddr; highaddr]中每个尝试页面的物理地址传递给过滤函数，该函数决定它是否可以访问。过滤函数的原型是： int filterfunc(void *arg, bus_addr_t paddr) 。如果页面可访问则必须返回 0，否则返回非零值。
  * maxsize - 通过此标签可以分配的内存的最大大小（以字节为单位）。如果难以估计或可能是任意大的情况，ISA 设备的值将为 BUS_SPACE_MAXSIZE_24BIT 。
  * nsegments - 设备支持的最大分散-聚集段数。如果不受限制，则应使用值 BUS_SPACE_UNRESTRICTED 。此值建议用于父标签，然后可以为子标签指定实际限制。 nsegments 等于 BUS_SPACE_UNRESTRICTED 的标签不得用于实际加载映射，只能用作父标签。 nsegments 的实际限制似乎约为 250-300，更高的值将导致内核堆栈溢出（硬件通常无法支持那么多分散-聚集缓冲区）。
  * maxsegsz - 设备支持的分散-聚集段的最大大小。ISA 设备的最大值为 BUS_SPACE_MAXSIZE_24BIT 。
  * flags - 一组标志位。唯一有趣的标志位是:

    * BUS_DMA_ALLOCNOW - 在创建标记时请求分配所有潜在需要的反弹页面。
  * dmat - 指向要返回的新标签存储。
* int bus_dma_tag_destroy(bus_dma_tag_t dmat) 销毁一个标签。成功返回 0，否则返回错误代码。
  dmat - 要销毁的标签。
* int bus_dmamem_alloc(bus_dma_tag_t dmat, void** vaddr, int flags, bus_dmamap_t *mapp) 分配由标签描述的连续内存区域。要分配的内存大小为标签的最大尺寸。成功时返回 0，否则返回错误代码。在使用物理地址获取内存之前，仍需由 bus_dmamap_load() 加载结果。

  * dmat - 标签
  * vaddr - 指向用于返回分配区域的内核虚拟地址的存储指针。
  * 标志 - 一组标志位图。唯一有趣的标志是：

    * BUS_DMA_NOWAIT - 如果内存不可立即使用，则返回错误。如果未设置此标志，则允许例程休眠，直到内存可用为止。
  * mapp - 指向将返回的新映射的存储指针。
* void bus_dmamem_free(bus_dma_tag_t dmat, void *vaddr, bus_dmamap_t map) 释放由 bus_dmamem_alloc() 分配的内存。目前，尚未实现对具有 ISA 限制的内存进行释放。因此，建议的使用模式是尽可能保留并重复使用已分配的区域。不要轻易释放某些区域，然后不久再次分配它。这并不意味着完全不应使用 bus_dmamem_free() ：希望它很快能够得到正确实现。

  * dmat - 标签
  * vaddr - 内存的内核虚拟地址
  * 地图 - 内存地图（从 bus_dmamem_alloc() 返回）
* int bus_dmamap_create(bus_dma_tag_t dmat, int flags, bus_dmamap_t *mapp) 为标签创建一个映射，稍后在 bus_dmamap_load() 中使用。成功时返回 0，否则返回错误代码。

  * dmat - 标签
  * 标志 - 理论上，表示标志的位图。但是目前没有定义任何标志，因此目前它将始终为 0。
  * mapp - 指向要返回的新地图的存储指针
* int bus_dmamap_destroy(bus_dma_tag_t dmat, bus_dmamap_t map) 销毁地图。成功时返回 0，否则返回错误代码。

  * dmat - 地图关联的标签
  * map - 要销毁的地图
* int bus_dmamap_load(bus_dma_tag_t dmat, bus_dmamap_t map, void *buf, bus_size_t buflen, bus_dmamap_callback_t *callback, void *callback_arg, int flags) 将缓冲区加载到地图中（地图必须由 bus_dmamap_create() 或 bus_dmamem_alloc() 预先创建）。缓冲区的所有页面都会检查是否符合标签要求，对于不符合要求的页面，将分配跳跃页面。一个物理段描述符数组将被构建并传递给回调例程。然后期望该回调例程以某种方式处理它。系统中的跳跃缓冲区数量有限，因此如果需要但未立即可用时，请求将排队，当跳跃缓冲区可用时将调用回调。立即执行回调返回 0，若请求排队等待执行则返回 EINPROGRESS 。在后者情况下，与排队回调例程的同步是驱动程序的责任。

  * dmat - 标签
  * map - 地图
  * buf - 缓冲区的内核虚拟地址
  * 缓冲区长度
  * 回调， callback_arg - 回调函数及其参数 回调函数的原型是: void callback(void *arg, bus_dma_segment_t *seg, int nseg, int error)
  * 参数 - 与传递给 bus_dmamap_load() 的回调参数相同
  * seg - 段描述符数组
  * nseg - 数组中的描述符数量
  * error - 表示段号溢出：如果设置为 EFBIG ，则缓冲区不适合标签允许的最大段数。在这种情况下，数组中将只包含允许的描述符数量。处理此情况取决于驱动程序：根据所需的语义，它可以将其视为错误或拆分缓冲区并单独处理第二部分。段数组中的每个条目包含以下字段：
  * ds_addr - 段的物理总线地址
  * ds_len - 段的长度
* void bus_dmamap_unload(bus_dma_tag_t dmat, bus_dmamap_t map) 卸载地图。

  * dmat - 标签
  * 地图 - 已加载地图
* void bus_dmamap_sync (bus_dma_tag_t dmat, bus_dmamap_t map, bus_dmasync_op_t op) 将加载的缓冲区与其弹跳页面在物理传输到或从设备之前和之后进行同步。这是执行原始缓冲区和其映射版本之间所有必要数据复制的功能。必须在执行传输之前和之后同步缓冲区。

  * dmat - 标签
  * map - 载入的地图
  * op - 要执行的同步操作类型:
  * BUS_DMASYNC_PREREAD - 从设备读取到缓冲区之前
  * BUS_DMASYNC_POSTREAD - 从设备读取到缓冲区之后
  * BUS_DMASYNC_PREWRITE - 将缓冲区写入设备之前
  * BUS_DMASYNC_POSTWRITE - 写入缓冲区到设备后

目前 PREREAD 和 POSTWRITE 是空操作，但将来可能会发生变化，因此驱动程序不得忽略它们。不需要对从 bus_dmamem_alloc() 获取的内存进行同步。

在从 bus_dmamap_load() 调用回调函数之前，段数组存储在堆栈中。并且为标签允许的最大段数进行了预分配。由此导致在 i386 架构上段数的实际限制约为 250-300（内核堆栈为 4KB 减去用户结构的大小，段数组条目的大小为 8 字节，必须留有一些空间）。由于数组是基于最大数目分配的，因此此值不得设置得比实际需要的更高。幸运的是，对于大多数硬件，支持的最大段数要低得多。但是，如果驱动程序希望处理具有非常大数量的分散-聚集段的缓冲区，则应该分批处理：加载缓冲区的一部分，将其传输到设备，加载缓冲区的下一部分，依此类推。

另一个实际后果是，段的数量可能限制缓冲区的大小。如果缓冲区中的所有页面在物理上不连续，则该碎片化情况下支持的最大缓冲区大小将为（nsegments * page_size）。例如，如果支持最大数量的 10 个段，则在 i386 上最大保证支持的缓冲区大小将为 40K。如果需要更大的大小，则驱动程序应使用特殊技巧。

如果硬件根本不支持散射-聚集，或者驱动程序希望支持即使在非常碎片化的情况下也支持某些缓冲区大小，则解决方案是在驱动程序中分配一个连续的缓冲区，并在原始缓冲区不适合时将其用作中间存储。

在使用映射时，典型的调用序列取决于映射的用途。字符→用于显示时间流。

在设备连接和断开期间几乎保持不变的缓冲区：

bus_dmamem_alloc → bus_dmamap_load → …使用缓冲区… → → bus_dmamap_unload → bus_dmamem_free

对于频繁更改并从驱动程序外部传递的缓冲区：

```
          bus_dmamap_create ->
          -> bus_dmamap_load -> bus_dmamap_sync(PRE...) -> do transfer ->
          -> bus_dmamap_sync(POST...) -> bus_dmamap_unload ->
          ...
          -> bus_dmamap_load -> bus_dmamap_sync(PRE...) -> do transfer ->
          -> bus_dmamap_sync(POST...) -> bus_dmamap_unload ->
          -> bus_dmamap_destroy
```

加载由 bus_dmamem_alloc() 创建的地图时，传递的地址和缓冲区大小必须与 bus_dmamem_alloc() 中使用的相同。在这种情况下，保证整个缓冲区将被映射为一个段（因此回调可以基于此假设），并且请求将立即执行（永远不会返回 EINPROGRESS）。在这种情况下，回调所需做的就是保存物理地址。

典型示例如下：

```
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
          struct somedata *vsomedata; /* virtual address */
          bus_addr_t psomedata; /* physical bus-relative address */
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

看起来有点长且复杂，但这就是操作方法。实际结果是：如果多个内存区域总是一起分配，则将它们全部组合成一个结构并作为一个单元分配将是一个非常好的主意（如果对齐和边界限制允许的话）。

当将任意缓冲区加载到 bus_dmamap_create() 创建的地图中时，必须采取特殊措施来同步回调，以防延迟。 代码看起来像：

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
                * Do whatever is needed to ensure synchronization
                * with callback. Callback is guaranteed not to be started
                * until we do splx() or tsleep().
                */
              }
           splx(s);
          }
```

请求处理的两种可能方法是：

1. 如果通过显式标记完成请求（例如 CAM 请求）来完成请求，则将所有进一步处理放入回调驱动程序将更简单，该驱动程序在请求完成时标记请求。 然后，就不需要太多额外的同步。出于流量控制的原因，将请求队列冻结直到该请求完成可能是一个好主意。
2. 如果请求在函数返回时完成（例如在字符设备上的经典读取或写入请求），则应在缓冲区描述符中设置同步标志并调用 tsleep() 。稍后当回调被调用时，它将进行处理并检查此同步标志。如果已设置，则回调应发出唤醒。在这种方法中，回调函数可以执行所有所需的处理（就像前一种情况一样），也可以简单地将段数组保存在缓冲区描述符中。然后在回调完成后，调用函数可以使用此保存的段数组并执行所有处理。

## 10.7. DMA

直接内存访问（DMA）通过 DMA 控制器在 ISA 总线上实现（实际上，有两个控制器，但这是一个无关紧要的细节）。为使早期 ISA 设备简单且廉价，总线控制和地址生成的逻辑集中在 DMA 控制器中。幸运的是，FreeBSD 提供了一组函数，大部分隐藏了 DMA 控制器的烦人细节，使设备驱动程序更易于编写。

最简单的情况是对于相当智能的设备。就像 PCI 上的总线主设备一样，它们可以自己生成总线周期和内存地址。它们真正需要的唯一一件事情是来自 DMA 控制器的总线仲裁。因此，为此目的，它们假装是级联的从 DMA 控制器。系统 DMA 控制器唯一需要的是在附加驱动程序时通过调用以下函数来在 DMA 通道上启用级联模式：

`void isa_dmacascade(int channel_number)`

所有进一步的活动都是通过对设备进行编程来完成的。在卸载驱动程序时，不需要调用任何与 DMA 相关的函数。

对于更简单的设备，情况变得更加复杂。使用的函数包括：

* int isa_dma_acquire(int chanel_number) 保留 DMA 通道。成功时返回 0，如果通道已被此驱动程序或其他驱动程序预留，则返回 EBUSY。大多数 ISA 设备无法共享 DMA 通道，因此通常在连接设备时调用此函数。尽管现代总线资源接口使此保留变得多余，但仍必须与后者一起使用。如果不使用，稍后其他 DMA 例程将会崩溃。
* int isa_dma_release(int chanel_number) 释放先前保留的 DMA 通道。在释放通道时不得有传输正在进行（此外，设备在通道释放后不得尝试启动传输）。
* void isa_dmainit(int chan, u_int bouncebufsize) 为指定通道分配一个弹跳缓冲区。缓冲区的请求大小不能超过 64KB。如果传输缓冲区不是物理上连续的，或者超出 ISA 总线可访问的内存范围，或者跨越 64KB 边界，则稍后将自动使用此弹跳缓冲区。如果传输始终来自符合这些条件的缓冲区（例如由 bus_dmamem_alloc() 分配的具有适当限制的缓冲区），则不必调用 isa_dmainit() 。但是使用 DMA 控制器传输任意数据非常方便。弹跳缓冲区将自动处理分散-聚集问题。

  * 通道号
  * 弹跳缓冲区大小 - 弹跳缓冲区的大小，以字节为单位
* void isa_dmastart(int flags, caddr_t addr, u_int nbytes, int chan) 准备开始 DMA 传输。在实际启动设备上的传输之前，必须调用此函数设置 DMA 控制器。它检查缓冲区是否连续并是否在 ISA 内存范围内，如果不是，则自动使用弹跳缓冲区。如果需要弹跳缓冲区但未通过 isa_dmainit() 设置或请求传输大小过大，则系统将发生紧急情况。在使用弹跳缓冲区的写入请求时，数据将自动复制到弹跳缓冲区中。
* 标志 - 一个位掩码，确定要执行的操作类型。方向位 B_READ 和 B_WRITE 是互斥的。

  * B_READ - 从 ISA 总线读取到内存中
  * B_WRITE - 从内存写入到 ISA 总线
  * 如果设置为 B_RAW，则 DMA 控制器将记住缓冲区，并在传输结束后自动重新初始化以再次重复传输相同的缓冲区（当然，在设备中启动另一个传输之前，驱动程序可能会更改缓冲区中的数据）。 如果未设置，则参数仅适用于一个传输，并且在启动下一个传输之前必须再次调用 isa_dmastart() 。仅当不使用跳变缓冲区时，使用 B_RAW 才有意义。
* 地址 - 缓冲区的虚拟地址
* nbytes - 缓冲区的长度。必须小于或等于 64KB。不允许长度为 0：DMA 控制器将其理解为 64KB，而内核代码将其理解为 0，这将导致不可预测的效果。对于通道号在 4 及以上的通道，长度必须是偶数，因为这些通道每次传输 2 字节。在长度为奇数时，最后一个字节将不会被传输。
* 通道号
* void isa_dmadone(int flags, caddr_t addr, int nbytes, int chan) 在设备报告传输完成后同步内存。如果这是一个带有弹簧缓冲区的读操作，则数据将从弹簧缓冲区复制到原始缓冲区。参数与 isa_dmastart() 相同。标记 B_RAW 被允许但不会以任何方式影响 isa_dmadone() 。
* int isa_dmastatus(int channel_number) 返回当前传输中剩余的字节数。如果在 isa_dmastart() 中设置了标志 B_READ，则返回的数字永远不会等于零。在传输结束时，它将自动重置为缓冲区的长度。正常用法是在设备信号传输完成后检查剩余字节数。如果字节数不为 0，则该传输可能出现问题。
* int isa_dmastop(int channel_number) 中止当前传输并返回未传输的字节数。

## 10.8. xxx_isa_probe

此函数探测设备是否存在。如果驱动程序支持某些设备配置的自动检测（例如中断向量或内存地址），则此自动检测必须在此例程中完成。

对于任何其他总线，如果设备无法被检测到或者被检测到但是自检未通过或发生其他问题，则返回一个正值的错误。如果设备不存在，则必须返回值 ENXIO 。其他错误值可能表示其他条件。零或负值表示成功。大多数驱动程序返回零表示成功。

负返回值用于 PnP 设备支持多个接口的情况。例如，一个旧的兼容接口和一个支持不同驱动程序的新的高级接口。然后两个驱动程序都会检测到设备。在探测例程中返回更高值的驱动程序优先（换句话说，返回 0 的驱动程序拥有最高优先级，返回-1 的次之，返回-2 的在其后，依此类推）。结果，只支持旧接口的设备将由旧驱动程序处理（该驱动程序应在探测例程中返回-1），而同时支持新接口的设备将由新驱动程序处理（该驱动程序应在探测例程中返回 0）。

系统在调用探测程序之前通过分配结构 xxx_softc 为设备分配描述符。如果探测程序返回错误，系统将自动释放描述符。因此，如果发生探测错误，驱动程序必须确保在探测过程中使用的所有资源被释放，没有任何东西阻止描述符安全释放。如果探测成功完成，系统将保留描述符并稍后传递给例程 xxx_isa_attach() 。如果驱动程序返回负值，则不能确定它将具有最高优先级并且将调用其附加程序。因此，在这种情况下，在返回之前还必须释放所有资源，如果需要，在附加程序中重新分配资源。当 xxx_isa_probe() 返回 0 时，在返回之前释放资源也是一个好主意，行为良好的驱动程序应该这样做。但是，在释放资源方面存在问题的情况下，允许驱动程序在从探测程序返回 0 到执行附加程序之间保留资源。

典型的探测程序以获取设备描述符和单元开始：

```
         struct xxx_softc *sc = device_get_softc(dev);
          int unit = device_get_unit(dev);
          int pnperror;
          int error = 0;

          sc->dev = dev; /* link it back */
          sc->unit = unit;
```

然后检查 PnP 设备。检查通过包含此驱动程序支持的 PnP ID 列表及这些 ID 对应的设备型号的人类可读描述的表格进行。

```
        pnperror=ISA_PNP_PROBE(device_get_parent(dev), dev,
        xxx_pnp_ids); if(pnperror == ENXIO) return ENXIO;
```

ISA_PNP_PROBE 的逻辑如下：如果此卡（设备单元）未被检测为 PnP，则将返回 ENOENT。如果它被检测为 PnP，但其检测到的 ID 与表中的任何 ID 都不匹配，则返回 ENXIO。最后，如果它具有 PnP 支持并且与表中的 ID 之一匹配，则返回 0，并由 device_set_desc() 设置表中的适当描述。

如果驱动程序仅支持 PnP 设备，则条件将如下所示：

```
          if(pnperror != 0)
              return pnperror;
```

不需要对不支持 PnP 的驱动程序进行特殊处理，因为它们传递空的 PnP ID 表，如果在 PnP 卡上调用，则始终会收到 ENXIO。

探测例行程序通常至少需要一些最小的资源集，比如输入/输出port号，以便找到卡并对其进行探测。根据硬件，驱动程序可能能够自动发现其他必要的资源。即插即用设备已经由即插即用子系统预设了所有资源，因此驱动程序不需要自己去发现它们。

通常，要访问设备所需的最少信息是输入/输出port号。然后，一些设备允许从设备配置寄存器中获取其余信息（虽然并非所有设备都这样做）。因此，首先我们尝试获取port起始值：

```
 sc->port0 = bus_get_resource_start(dev,
        SYS_RES_IOPORT, 0 /*rid*/); if(sc->port0 == 0) return ENXIO;
```

基本的port地址保存在结构 softc 中以供将来使用。如果它会经常被使用，那么每次调用资源函数将会非常慢。如果我们没有得到port，我们只返回一个错误。一些设备驱动程序可以更聪明，尝试探测所有可能的ports，就像这样：

```
          /* table of all possible base I/O port addresses for this device */
          static struct xxx_allports {
              u_short port; /* port address */
              short used; /* flag: if this port is already used by some unit */
          } xxx_allports = {
              { 0x300, 0 },
              { 0x320, 0 },
              { 0x340, 0 },
              { 0, 0 } /* end of table */
          };

          ...
          int port, i;
          ...

          port =  bus_get_resource_start(dev, SYS_RES_IOPORT, 0 /*rid*/);
          if(port !=0 ) {
              for(i=0; xxx_allports[i].port!=0; i++) {
                  if(xxx_allports[i].used || xxx_allports[i].port != port)
                      continue;

                  /* found it */
                  xxx_allports[i].used = 1;
                  /* do probe on a known port */
                  return xxx_really_probe(dev, port);
              }
              return ENXIO; /* port is unknown or already used */
          }

          /* we get here only if we need to guess the port */
          for(i=0; xxx_allports[i].port!=0; i++) {
              if(xxx_allports[i].used)
                  continue;

              /* mark as used - even if we find nothing at this port
               * at least we won't probe it in future
               */
               xxx_allports[i].used = 1;

              error = xxx_really_probe(dev, xxx_allports[i].port);
              if(error == 0) /* found a device at that port */
                  return 0;
          }
          /* probed all possible addresses, none worked */
          return ENXIO;
```

当然，通常应该使用驱动程序的 identify() 例程来执行这样的操作。但有一个有效的理由可能更好的在 probe() 中完成：如果此探针可能会让其他敏感设备发疯。探针例程考虑到 sensitive 标志的顺序：首先对敏感设备进行探测，然后是其余设备。但 identify() 例程是在任何探测之前调用的，因此它们对敏感设备没有尊重，可能会使它们受到影响。

现在，当我们获得起始port后，我们需要设置port计数（除了 PnP 设备），因为内核在配置文件中没有此信息。

```
         if(pnperror /* only for non-PnP devices */
         && bus_set_resource(dev, SYS_RES_IOPORT, 0, sc->port0,
         XXX_PORT_COUNT)<0)
             return ENXIO;
```

最后分配并激活一块port地址空间（开始值和结束值的特殊值表示“使用我们通过 bus_set_resource() 设置的值”）:

```
          sc->port0_rid = 0;
          sc->port0_r = bus_alloc_resource(dev, SYS_RES_IOPORT,
          &sc->port0_rid,
              /*start*/ 0, /*end*/ ~0, /*count*/ 0, RF_ACTIVE);

          if(sc->port0_r == NULL)
              return ENXIO;
```

现在访问port-映射寄存器后，我们可以以某种方式探测设备并检查其反应是否如预期。如果没有反应，那么可能在这个地址上有其他设备或根本没有设备。

通常驱动程序不会在连接例程之前设置中断处理程序。相反，他们使用 DELAY() 函数在轮询模式下进行探测。探测例程绝不能永远挂起，对设备的所有等待都必须设置超时。如果设备在规定时间内没有响应，可能是损坏或配置错误，驱动程序必须返回错误。在确定超时间隔时，给设备一些额外的时间以确保安全：虽然 DELAY() 应该在任何机器上延迟相同的时间，但它有一些误差范围，具体取决于 CPU。

如果探测例程真的想检查中断是否真的有效，它也可以配置和探测中断。但这不推荐。

```
          /* implemented in some very device-specific way */
          if(error = xxx_probe_ports(sc))
              goto bad; /* will deallocate the resources before returning */
```

函数 xxx_probe_ports() 还可以根据发现的设备的确切型号设置设备描述。但是，如果只有一个受支持的设备型号，这也可以以硬编码的方式完成。当然，对于即插即用设备，即插即用支持会自动从表中设置描述。

```
          if(pnperror)
              device_set_desc(dev, "Our device model 1234");
```

然后，探测例程应通过读取设备配置寄存器来发现所有资源的范围，或者确保资源范围是由用户明确设置的。我们将以板载内存的示例来考虑它。探测例程应尽可能非侵入性，因此资源的分配和功能检查（除ports之外）最好留给附加例程。

内存地址可以在内核配置文件中指定，或者在某些设备上可能预先配置在非易失性配置寄存器中。如果两个来源都可用且不同，应该使用哪一个？可能如果用户在内核配置文件中明确设置了地址，他们知道自己在做什么，这个应该优先。实现示例可能如下：

```
          /* try to find out the config address first */
          sc->mem0_p = bus_get_resource_start(dev, SYS_RES_MEMORY, 0 /*rid*/);
          if(sc->mem0_p == 0) { /* nope, not specified by user */
              sc->mem0_p = xxx_read_mem0_from_device_config(sc);

          if(sc->mem0_p == 0)
                  /* can't get it from device config registers either */
                  goto bad;
          } else {
              if(xxx_set_mem0_address_on_device(sc) < 0)
                  goto bad; /* device does not support that address */
          }

          /* just like the port, set the memory size,
           * for some devices the memory size would not be constant
           * but should be read from the device configuration registers instead
           * to accommodate different models of devices. Another option would
           * be to let the user set the memory size as "msize" configuration
           * resource which will be automatically handled by the ISA bus.
           */
           if(pnperror) { /* only for non-PnP devices */
              sc->mem0_size = bus_get_resource_count(dev, SYS_RES_MEMORY, 0 /*rid*/);
              if(sc->mem0_size == 0) /* not specified by user */
                  sc->mem0_size = xxx_read_mem0_size_from_device_config(sc);

              if(sc->mem0_size == 0) {
                  /* suppose this is a very old model of device without
                   * auto-configuration features and the user gave no preference,
                   * so assume the minimalistic case
                   * (of course, the real value will vary with the driver)
                   */
                  sc->mem0_size = 8*1024;
              }

              if(xxx_set_mem0_size_on_device(sc) < 0)
                  goto bad; /* device does not support that size */

              if(bus_set_resource(dev, SYS_RES_MEMORY, /*rid*/0,
                      sc->mem0_p, sc->mem0_size)<0)
                  goto bad;
          } else {
              sc->mem0_size = bus_get_resource_count(dev, SYS_RES_MEMORY, 0 /*rid*/);
          }
```

通过类比很容易检查 IRQ 和 DRQ 的资源。

如果一切顺利，释放所有资源并返回成功。

```
          xxx_free_resources(sc);
          return 0;
```

最后，在处理棘手的情况。在返回之前，应该释放所有资源。我们利用 softc 结构传递给我们之前被清零的事实，因此我们可以找出是否有一些资源被分配：然后其描述符则为非零。

```
          bad:

          xxx_free_resources(sc);
          if(error)
                return error;
          else /* exact error is unknown */
              return ENXIO;
```

这将是探测例程的全部内容。释放资源是从多个地方完成的，因此移至一个函数，可能看起来像：

```
static void
           xxx_free_resources(sc)
              struct xxx_softc *sc;
          {
              /* check every resource and free if not zero */

              /* interrupt handler */
              if(sc->intr_r) {
                  bus_teardown_intr(sc->dev, sc->intr_r, sc->intr_cookie);
                  bus_release_resource(sc->dev, SYS_RES_IRQ, sc->intr_rid,
                      sc->intr_r);
                  sc->intr_r = 0;
              }

              /* all kinds of memory maps we could have allocated */
              if(sc->data_p) {
                  bus_dmamap_unload(sc->data_tag, sc->data_map);
                  sc->data_p = 0;
              }
               if(sc->data) { /* sc->data_map may be legitimately equal to 0 */
                  /* the map will also be freed */
                  bus_dmamem_free(sc->data_tag, sc->data, sc->data_map);
                  sc->data = 0;
              }
              if(sc->data_tag) {
                  bus_dma_tag_destroy(sc->data_tag);
                  sc->data_tag = 0;
              }

              ... free other maps and tags if we have them ...

              if(sc->parent_tag) {
                  bus_dma_tag_destroy(sc->parent_tag);
                  sc->parent_tag = 0;
              }

              /* release all the bus resources */
              if(sc->mem0_r) {
                  bus_release_resource(sc->dev, SYS_RES_MEMORY, sc->mem0_rid,
                      sc->mem0_r);
                  sc->mem0_r = 0;
              }
              ...
              if(sc->port0_r) {
                  bus_release_resource(sc->dev, SYS_RES_IOPORT, sc->port0_rid,
                      sc->port0_r);
                  sc->port0_r = 0;
              }
          }
```

## 10.9. xxx_isa_attach

如果探测例程返回成功并且系统选择连接该驱动程序，则连接例程实际上将驱动程序连接到系统。如果探测例程返回 0，则连接例程可能期望接收设备结构 softc 完整，就像探测例程设置的那样。此外，如果探测例程返回 0，则可以期望将来某个时候为此设备调用连接例程。如果探测例程返回负值，则驱动程序可能不做出这些假设中的任何一个。

attach 例程返回 0 表示成功完成，否则返回错误代码。

attach 例程的开始方式与 probe 例程相同，将一些经常使用的数据放入更易访问的变量中。

```
          struct xxx_softc *sc = device_get_softc(dev);
          int unit = device_get_unit(dev);
          int error = 0;
```

然后分配和激活所有必要的资源。由于在返回 probe 之前通常会释放port范围，所以必须再次分配它。我们期望 probe 例程已正确设置了所有资源范围，并将它们保存在结构 softc 中。如果 probe 例程留下了一些资源已分配的情况，则不需要再次分配（这将被视为错误）。

```
          sc->port0_rid = 0;
          sc->port0_r = bus_alloc_resource(dev, SYS_RES_IOPORT,  &sc->port0_rid,
              /*start*/ 0, /*end*/ ~0, /*count*/ 0, RF_ACTIVE);

          if(sc->port0_r == NULL)
               return ENXIO;

          /* on-board memory */
          sc->mem0_rid = 0;
          sc->mem0_r = bus_alloc_resource(dev, SYS_RES_MEMORY,  &sc->mem0_rid,
              /*start*/ 0, /*end*/ ~0, /*count*/ 0, RF_ACTIVE);

          if(sc->mem0_r == NULL)
                goto bad;

          /* get its virtual address */
          sc->mem0_v = rman_get_virtual(sc->mem0_r);
```

DMA 请求通道（DRQ）也是类似分配的。要初始化它，请使用 isa_dma*() 系列的函数。例如：

`isa_dmacascade(sc→drq0);`

中断请求线（IRQ）有点特殊。除了分配外，驱动程序的中断处理程序应与之关联。在旧的 ISA 驱动程序中，系统传递给中断处理程序的参数是设备单元号。但在现代驱动程序中，惯例建议传递指向结构体 softc 的指针。重要的原因是，当动态分配结构体 softc 时，从 softc 获取单元号很容易，而从单元号获取 softc 很困难。此外，这种惯例使得不同总线的驱动程序看起来更加统一，并允许它们共享代码：每个总线都有自己的探测、附加、分离和其他特定于总线的例程，而驱动程序的大部分代码可以在它们之间共享。

```
          sc->intr_rid = 0;
          sc->intr_r = bus_alloc_resource(dev, SYS_RES_MEMORY,  &sc->intr_rid,
                /*start*/ 0, /*end*/ ~0, /*count*/ 0, RF_ACTIVE);

          if(sc->intr_r == NULL)
              goto bad;

          /*
           * XXX_INTR_TYPE is supposed to be defined depending on the type of
           * the driver, for example as INTR_TYPE_CAM for a CAM driver
           */
          error = bus_setup_intr(dev, sc->intr_r, XXX_INTR_TYPE,
              (driver_intr_t *) xxx_intr, (void *) sc, &sc->intr_cookie);
          if(error)
              goto bad;
```

如果设备需要对主内存进行 DMA，则应像之前描述的那样分配该内存：

```
          error=bus_dma_tag_create(NULL, /*alignment*/ 4,
              /*boundary*/ 0, /*lowaddr*/ BUS_SPACE_MAXADDR_24BIT,
              /*highaddr*/ BUS_SPACE_MAXADDR, /*filter*/ NULL, /*filterarg*/ NULL,
              /*maxsize*/ BUS_SPACE_MAXSIZE_24BIT,
              /*nsegments*/ BUS_SPACE_UNRESTRICTED,
              /*maxsegsz*/ BUS_SPACE_MAXSIZE_24BIT, /*flags*/ 0,
              &sc->parent_tag);
          if(error)
              goto bad;

          /* many things get inherited from the parent tag
           * sc->data is supposed to point to the structure with the shared data,
           * for example for a ring buffer it could be:
           * struct {
           *   u_short rd_pos;
           *   u_short wr_pos;
           *   char    bf[XXX_RING_BUFFER_SIZE]
           * } *data;
           */
          error=bus_dma_tag_create(sc->parent_tag, 1,
              0, BUS_SPACE_MAXADDR, 0, /*filter*/ NULL, /*filterarg*/ NULL,
              /*maxsize*/ sizeof(* sc->data), /*nsegments*/ 1,
              /*maxsegsz*/ sizeof(* sc->data), /*flags*/ 0,
              &sc->data_tag);
          if(error)
              goto bad;

          error = bus_dmamem_alloc(sc->data_tag, &sc->data, /* flags*/ 0,
              &sc->data_map);
          if(error)
               goto bad;

          /* xxx_alloc_callback() just saves the physical address at
           * the pointer passed as its argument, in this case &sc->data_p.
           * See details in the section on bus memory mapping.
           * It can be implemented like:
           *
           * static void
           * xxx_alloc_callback(void *arg, bus_dma_segment_t *seg,
           *     int nseg, int error)
           * {
           *    *(bus_addr_t *)arg = seg[0].ds_addr;
           * }
           */
          bus_dmamap_load(sc->data_tag, sc->data_map, (void *)sc->data,
              sizeof (* sc->data), xxx_alloc_callback, (void *) &sc->data_p,
              /*flags*/0);
```

分配所有必要资源后，应初始化设备。初始化可能包括测试所有预期功能是否可用。

```
          if(xxx_initialize(sc) < 0)
               goto bad;
```

总线子系统将自动在控制台上打印由探测设置的设备描述。但是，如果驱动程序想要打印有关设备的一些额外信息，例如：

```
        device_printf(dev, "has on-card FIFO buffer of %d bytes\n", sc->fifosize);
```

如果初始化过程出现任何问题，则建议在返回错误之前打印有关这些问题的消息。

附加程序的最后一步是将设备附加到内核中的功能子系统。这样做的确切方式取决于驱动程序的类型：字符设备、块设备、网络设备、CAM SCSI 总线设备等。

如果一切顺利，就返回成功。

```
          error = xxx_attach_subsystem(sc);
          if(error)
              goto bad;

          return 0;
```

最后，处理棘手的情况。在返回错误之前，应释放所有资源。我们利用这样一个事实：在结构 softc 传递给我们之前，它被清零，因此我们可以查明是否分配了某些资源：在这种情况下，其描述符是非零的。

```
          bad:

          xxx_free_resources(sc);
          if(error)
              return error;
          else /* exact error is unknown */
              return ENXIO;
```

这将是附加例程的全部内容。

## 10.10. xxx_isa_detach

如果此功能存在于驱动程序中，并且驱动程序被编译为可加载模块，则驱动程序具有卸载的能力。如果硬件支持热插拔，则这是一个重要的功能。但是 ISA 总线不支持热插拔，因此对 ISA 设备来说，这个功能并不特别重要。在调试驱动程序时，卸载驱动程序的能力可能是有用的，但在许多情况下，只有在旧版本驱动程序在某种程度上使系统崩溃并且需要重新启动时，才需要安装新版本的驱动程序，因此编写卸载例程所花费的精力可能不值得。另一个论点是卸载将允许在生产机器上升级驱动程序，这似乎大多是理论性的。在生产机器上安装新版本的驱动程序是一项危险的操作，绝不能在生产机器上执行（并且在系统运行在安全模式时是不允许的）。尽管如此，出于完整性考虑，可能会提供卸载例程。

分离例程在成功分离驱动程序时返回 0，否则返回错误代码。

分离的逻辑与附加的逻辑相反。首先要做的是将驱动程序从其内核子系统中分离。如果设备当前正在打开，则驱动程序有两种选择：拒绝分离或强制关闭并继续分离。所选择的方式取决于特定内核子系统执行强制关闭的能力以及驱动程序作者的偏好。通常，强制关闭似乎是首选的替代方案。

```
          struct xxx_softc *sc = device_get_softc(dev);
          int error;

          error = xxx_detach_subsystem(sc);
          if(error)
              return error;
```

接下来，驱动程序可能希望将硬件重置为一致的状态。这包括停止任何正在进行的传输，禁用 DMA 通道和中断，以避免设备引起内存损坏。对于大多数驱动程序来说，这正是关闭例程所做的事情，因此如果它包含在驱动程序中，我们只需调用它。

`xxx_isa_shutdown(dev);`

最后释放所有资源并返回成功。

```
          xxx_free_resources(sc);
          return 0;
```

## 10.11. xxx_isa_shutdown

当系统即将关闭时，将调用此例程。预计将硬件带到某种一致状态。对于大多数 ISA 设备，不需要特殊操作，因此该函数实际上并不是必需的，因为设备将在重新启动时重新初始化。但是，一些设备必须通过特殊过程关闭，以确保它们在软重启后能够被正确检测到（这对于许多具有专有识别协议的设备尤为重要）。无论如何，在设备寄存器中禁用 DMA 和中断，并停止任何正在进行的传输都是一个好主意。确切的操作取决于硬件，因此我们在这里不会详细考虑。

## 10.12. xxx_intr

当接收到中断时，将调用中断处理程序，该中断可能来自该特定设备。ISA 总线不支持中断共享（除非在某些特殊情况下），因此实际上，如果调用中断处理程序，则几乎可以确定中断来自其设备。但是，中断处理程序必须轮询设备寄存器，并确保中断是由其设备生成的。如果不是，则应该直接返回。

ISA 驱动程序的旧约定是将设备单元号作为参数。这已经过时，新驱动程序在调用 bus_setup_intr() 时接收指定给它们的任何参数。根据新约定，它应该是指向结构体 softc 的指针。因此，中断处理程序通常以以下方式开始：

```
          static void
          xxx_intr(struct xxx_softc *sc)
          {
```

它以 bus_setup_intr() 指定的中断类型参数中断优先级别运行。这意味着所有同一类型的其他中断以及所有软件中断都被禁用。

为了避免竞争，通常写成循环：

```
          while(xxx_interrupt_pending(sc)) {
              xxx_process_interrupt(sc);
              xxx_acknowledge_interrupt(sc);
          }
```

中断处理程序只需向设备确认中断，而不需要向中断控制器确认，系统会处理后者。
