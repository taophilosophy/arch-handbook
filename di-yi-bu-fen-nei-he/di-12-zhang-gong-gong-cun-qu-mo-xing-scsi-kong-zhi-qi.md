# 第 12 章 公共存取模型 SCSI 控制器

## 12.1. 概要

本文假定读者对 FreeBSD 中的设备驱动程序和 SCSI 协议有一般的了解。本文中的许多信息都是从驱动程序中提取的：

* ncr (/sys/pci/ncr.c) 由 Wolfgang Stanglmeier 和 Stefan Esser 编写
* sym (/sys/dev/sym/sym_hipd.c) 由 Gerard Roudier 编写
* aic7xxx (/sys/dev/aic7xxx/aic7xxx.c) 由 Justin T. Gibbs 编写

以及来自 CAM 代码本身（由 Justin T. Gibbs 编写，参见/sys/cam/*）。当某些解决方案看起来最合乎逻辑，并且基本上是 Justin T. Gibbs 从代码中直接提取出来时，我将其标记为“推荐”。

该文档以伪代码示例进行说明。虽然有时示例具有许多细节并且看起来像真实代码，但它仍然是伪代码。它是为了以一种易于理解的方式演示概念而编写的。对于真实的驱动程序，其他方法可能更模块化和高效。它还抽象出硬件细节，以及可能会混淆演示概念或应该在开发人员手册的其他章节中描述的问题。这些细节通常显示为调用具有描述性名称的函数、注释或伪语句。幸运的是，真实生活中带有所有细节的全尺寸示例可以在真实的驱动程序中找到。

## 12.2. 通用架构

CAM 代表通用访问方法。这是一种以类似 SCSI 的方式寻址 I/O 总线的通用方式。这允许将通用设备驱动程序与控制 I/O 总线的驱动程序分开：例如，磁盘驱动程序能够控制 SCSI、IDE 和/或任何其他总线上的磁盘，因此磁盘驱动程序部分不必为每个新的 I/O 总线重写（或复制和修改）。因此，最重要的两个活动实体是：

* 外围模块 - 外围设备（磁盘、磁带、CD-ROM 等）的驱动程序。
* SCSI 接口模块（SIM）- 用于连接到 I/O 总线（如 SCSI 或 IDE）的主机总线适配器驱动程序。

外围驱动程序接收来自操作系统的请求，将其转换为一系列 SCSI 命令，并将这些 SCSI 命令传递给 SCSI 接口模块。SCSI 接口模块负责将这些命令传递给实际的硬件（或者如果实际的硬件不是 SCSI 而是 IDE，还需要将 SCSI 命令转换为硬件的本机命令）。

由于我们对编写 SCSI 适配器驱动程序感兴趣，在这一点上，我们将从 SIM 的角度考虑一切。

## 12.3. 全局变量和样板文件

典型的 SIM 驱动程序需要包含以下与 CAM 相关的头文件：

```
#include <cam/cam.h>
#include <cam/cam_ccb.h>
#include <cam/cam_sim.h>
#include <cam/cam_xpt_sim.h>
#include <cam/cam_debug.h>
#include <cam/scsi/scsi_all.h>
```

## 12.4. 设备配置: xxx_attach

每个 SIM 驱动程序必须做的第一件事是向 CAM 子系统注册自己。这是在驱动程序的 xxx_attach() 函数中完成的（这里和后续使用 xxx_ 来表示唯一的驱动程序名称前缀）。 xxx_attach() 函数本身是由系统总线自动配置代码调用的，我们在这里不进行描述。

这是通过多个步骤实现的: 首先需要为与此 SIM 相关的请求队列分配空间:

```
    struct cam_devq *devq;

    if ((devq = cam_simq_alloc(SIZE)) == NULL) {
        error; /* some code to handle the error */
    }
```

这里 SIZE 是要分配的队列大小，它可以包含的最大请求数。这是 SIM 驱动程序在一个 SCSI 卡上可以并行处理的请求数。通常可以计算为：

```
SIZE = NUMBER_OF_SUPPORTED_TARGETS * MAX_SIMULTANEOUS_COMMANDS_PER_TARGET
```

接下来，我们创建 SIM 的描述符：

```
    struct cam_sim *sim;

    if ((sim = cam_sim_alloc(action_func, poll_func, driver_name,
            softc, unit, mtx, max_dev_transactions,
            max_tagged_dev_transactions, devq)) == NULL) {
        cam_simq_free(devq);
        error; /* some code to handle the error */
    }
```

请注意，如果我们无法创建 SIM 描述符，我们也会释放 devq ，因为我们无法做其他任何事情，我们希望保留内存。

如果 SCSI 卡上有多个 SCSI 总线，则每个总线都需要自己的 cam_sim 结构。

有趣的问题是，如果 SCSI 卡有多个 SCSI 总线，我们是需要每张卡一个 devq 结构还是每个 SCSI 总线一个？CAM 代码中的评论给出的答案是：无论哪种方式，都取决于驱动程序的作者偏好。

 参数为：

* action_func - 指向驱动程序的 xxx_action 函数。

```
static void xxx_action(struct cam_sim *, union ccb *);
```

* poll_func - 指向驱动程序的 xxx_poll()

  ```
  static void xxx_poll(struct cam_sim *);
  ```
* driver_name - 实际驱动程序的名称，如"ncr"或"wds"。
* softc - 指向此 SCSI 卡的驱动程序内部描述符的指针。驱动程序将来会使用此指针获取私有数据。
* unit - 控制器单元号，例如对于控制器“mps0”，此编号为 0。
* mtx - 与此 SIM 关联的锁。对于不了解锁定的 SIM，请传递 Giant。对于了解的 SIM，请传递用于保护此 SIM 数据结构的锁。当调用 xxx_action 和 xxx_poll 时，将持有此锁。
* max_dev_transactions - 非标记模式下每个 SCSI 目标的最大同时事务数。这个值几乎普遍等于 1，只有非 SCSI 卡可能有例外。希望在执行一个事务时准备另一个事务的驱动程序可能将其设置为 2，但这似乎不值得复杂化。
* max_tagged_dev_transactions - 相同的概念，但在标记模式下。标记是 SCSI 启动设备上多个事务的方式：每个事务被分配一个唯一的标记，然后将事务发送到设备。当设备完成某个事务时，它会将结果与标记一起发送回来，以便 SCSI 适配器（和驱动程序）可以知道哪个事务已完成。这个参数也被称为最大标记深度。它取决于 SCSI 适配器的能力。

最后，我们注册与我们的 SCSI 适配器关联的 SCSI 总线：

```
    if (xpt_bus_register(sim, softc, bus_number) != CAM_SUCCESS) {
        cam_sim_free(sim, /*free_devq*/ TRUE);
        error; /* some code to handle the error */
    }
```

如果每个 SCSI 总线上只有一个 devq 结构（即，我们将具有多个总线的卡视为具有单个总线的多个卡），那么总线编号将始终为 0，否则 SCSI 卡上的每个总线都应该获得一个不同的编号。每个总线都需要有自己单独的结构 cam_sim。

之后，我们的控制器完全连接到 CAM 系统。现在可以丢弃 devq 的值：sim 将在以后所有的 CAM 调用中作为参数传递，并且 devq 可以从中派生出来。

CAM 为这种异步事件提供框架。一些事件源自较低级别（SIM 驱动程序），一些事件源自外围驱动程序，一些事件源自 CAM 子系统本身。任何驱动程序都可以为某些类型的异步事件注册回调，以便在发生这些事件时通知它。

例如此类事件的典型示例是设备重置。每个事务和事件通过“路径”来标识适用于哪些设备。目标特定事件通常发生在与该设备的事务过程中。因此，该事务的路径可能被重复使用以报告此事件（这是安全的，因为事件路径在事件报告例程中被复制但未被释放或传递给其他地方）。动态分配路径在任何时候都是安全的，包括中断例程，尽管这会带来一定的开销，而采用这种方法可能存在的问题是在那时可能没有空闲内存。对于总线重置事件，我们需要定义一个包含总线上所有设备的通配符路径。因此，我们可以提前为未来的总线重置事件创建路径，避免未来的内存短缺问题：

```
    struct cam_path *path;

    if (xpt_create_path(&path, /*periph*/NULL,
                cam_sim_path(sim), CAM_TARGET_WILDCARD,
                CAM_LUN_WILDCARD) != CAM_REQ_CMP) {
        xpt_bus_deregister(cam_sim_path(sim));
        cam_sim_free(sim, /*free_devq*/TRUE);
        error; /* some code to handle the error */
    }

    softc->wpath = path;
    softc->sim = sim;
```

如你所见路径包括：

* 外设驱动程序的 ID（这里为空，因为我们没有）
* SIM 驱动程序的 ID ( cam_sim_path(sim) )
* 设备的 SCSI 目标编号 (CAM_TARGET_WILDCARD 表示 "所有设备")
* 子设备的 SCSI LUN 编号 (CAM_LUN_WILDCARD 表示 "所有 LUNs")

如果驱动程序无法分配此路径，则将无法正常工作，因此在这种情况下，我们拆卸那个 SCSI 总线。

然后我们将路径指针保存在 softc 结构中以供将来使用。之后，我们保存 sim 的值（或者如果希望，在 xxx_probe() 退出时也可以将其丢弃）。

这就是最小化初始化的全部内容。要做正确的事情，还有一个问题尚未解决。

对于 SIM 驱动程序，有一个特别有趣的事件：当目标设备被视为丢失时。在这种情况下，重置与该设备的 SCSI 协商可能是一个好主意。因此，我们使用 CAM 为此事件注册回调。请求通过请求 CAM 在此类型请求的 CAM 控制块上执行 CAM 操作来传递：

```
    struct ccb_setasync csa;

    xpt_setup_ccb(&csa.ccb_h, path, /*priority*/5);
    csa.ccb_h.func_code = XPT_SASYNC_CB;
    csa.event_enable = AC_LOST_DEVICE;
    csa.callback = xxx_async;
    csa.callback_arg = sim;
    xpt_action((union ccb *)&csa);
```

## 12.5. 处理 CAM 消息：xxx_action

```
static void xxx_action(struct cam_sim *sim, union ccb *ccb);
```

在 CAM 子系统请求时执行一些操作。Sim 描述了该请求的 SIM，CCB 是请求本身。CCB 代表“CAM 控制块”。它是许多特定实例的联合体，每个实例描述某种类型交易的参数。所有这些实例共享 CCB 头，其中存储了参数的公共部分。

CAM 支持 SCSI 控制器以发起方（“正常”）模式和目标（模拟 SCSI 设备）模式工作。在这里，我们只考虑与发起方模式相关的部分。

有一些函数和宏（换句话说，方法）被定义用来访问结构体 sim 中的公共数据：

* cam_sim_path(sim) - 路径标识（参见上文）
* cam_sim_name(sim) - sim 的名称
* cam_sim_softc(sim) - softc（驱动程序私有数据）结构的指针
* cam_sim_unit(sim) - 单元编号
* cam_sim_bus(sim) - 公交车 ID

要识别设备， xxx_action() 可以使用这些函数获取单元编号和指向其结构 softc 的指针。

请求类型存储在 ccb→ccb_h.func_code 中。因此，通常 xxx_action() 包含一个大开关：

```
    struct xxx_softc *softc = (struct xxx_softc *) cam_sim_softc(sim);
    struct ccb_hdr *ccb_h = &ccb->ccb_h;
    int unit = cam_sim_unit(sim);
    int bus = cam_sim_bus(sim);

    switch (ccb_h->func_code) {
    case ...:
        ...
    default:
        ccb_h->status = CAM_REQ_INVALID;
        xpt_done(ccb);
        break;
    }
```

从默认情况可以看出（如果收到未知命令），命令的返回代码被设置为 ccb→ccb_h.status ，并通过调用 xpt_done(ccb) 将完成的 CCB 返回给 CAM。

xpt_done() 不必从 xxx_action() 调用：例如，I/O 请求可以在 SIM 驱动程序内排队，和/或其 SCSI 控制器内。然后，当设备发出中断信号表明此请求的处理已完成时， xpt_done() 可能会从中断处理例程中调用。

实际上，CCB 状态不仅被分配为返回代码，而且 CCB 始终具有一些状态。在将 CCB 传递给 xxx_action() 例程之前，它会获得状态 CCB_REQ_INPROG，表示它正在进行中。在/sys/cam/cam.h 中定义了令人惊讶的状态值数量，应该能够详细表示请求的状态。更有趣的是，状态实际上是一个枚举状态值（低 6 位）和可能的附加类似标志位（高位）的“按位或”。枚举值将在稍后更详细地讨论。它们的摘要可以在错误摘要部分找到。可能的状态标志包括：

* 如果 SIM 驱动程序在处理 CCB 时发生严重错误（例如，设备不响应选择或破坏 SCSI 协议），则应通过调用 xpt_freeze_simq() 冻结请求队列，将此设备的其他尚未处理的但已入列的 CCB 返回到 CAM 队列，然后设置此问题 CCB 的标志并调用 xpt_done() 。此标志导致 CAM 子系统在处理错误后解冻队列。
* 如果设备返回错误条件并且在 CCB 中未设置标志 CAM_DIS_AUTOSENSE，则 SIM 驱动程序必须自动执行 REQUEST SENSE 命令以从设备中提取感觉（扩展错误信息）数据。如果此尝试成功，则应将感觉数据保存在 CCB 中，并设置此标志。
* 类似于 CAM_DEV_QFRZN，但在 SCSI 控制器本身出现问题（或资源短缺）时使用。然后，所有对控制器的其他请求都应通过 xpt_freeze_simq() 停止。一旦 SIM 驱动程序克服短缺并通过设置此标志返回某个 CCB 通知 CAM 后，控制器队列将重新启动。
* CAM_SIM_QUEUED - 当 SIM 将 CCB 放入其请求队列时，应设置此标志（在此 CCB 被返回给 CAM 之前被移除）。此标志目前在 CAM 代码中未被使用，所以其目的纯粹是诊断性的。
* CAM_QOS_VALID - QOS 数据现在有效。

函数 xxx_action() 不允许休眠，因此所有资源访问的同步都必须使用 SIM 或设备队列冻结进行。除了前述标志外，CAM 子系统还提供函数 xpt_release_simq() 和 xpt_release_devq() 来直接解冻队列，而无需将 CCB 传递给 CAM。

CCB 头部包含以下字段：

* 路径 - 请求的路径 ID
* target_id - 请求的目标设备 ID
* 目标 LUN - 目标设备的 LUN ID
* 超时 - 此命令的超时间隔，以毫秒为单位
* 超时句柄 - 用于 SIM 驱动程序存储超时句柄的便利位置（CAM 子系统本身不对此做任何假设）
* 标志 - 有关请求 spriv_ptr0 和 spriv_ptr1 的各种信息-这些字段由 SIM 驱动程序的私有使用保留（例如，链接到 SIM 队列或 SIM 私有控制块）；实际上，它们作为联合存在:spriv_ptr0 和 spriv_ptr1 具有类型(void *)，spriv_field0 和 spriv_field1 的类型为 unsigned long，sim_priv.entries[0].bytes 和 sim_priv.entries[1].bytes 是具有与联合的其他 incarnations 一致大小的字节数组，sim_priv.bytes 是一个数组，大小是两倍大。

使用 CCB 的 SIM 私有字段的推荐方法是为其定义一些有意义的名称，并在驱动程序中使用这些有意义的名称，如下：

```
#define ccb_some_meaningful_name    sim_priv.entries[0].bytes
#define ccb_hcb spriv_ptr1 /* for hardware control block */
```

最常见的发起者模式请求是：

### 12.5.1. XPT_SCSI_IO - 执行 I/O 事务

联合 ccb 的实例 "struct ccb_scsiio csio" 用于传递参数。它们是:

* cdb_io - 指向 SCSI 命令缓冲区或缓冲区本身
* cdb_len - SCSI 命令长度
* data_ptr - 数据缓冲区的指针（如果使用 scatter/gather 会有点复杂）
* dxfer_len - 传输数据的长度
* sglist_cnt - 散射/聚集段的计数器
* scsi_status - 返回 SCSI 状态的位置
* sense_data - 如果命令返回错误，则用于存储 SCSI 感知信息的缓冲区（在这种情况下，如果 CCB 标志 CAM_DIS_AUTOSENSE 未设置，SIM 驱动程序应自动运行请求感知命令）
* sense_len - 缓冲区的长度（如果恰巧高于 sense_data 的大小，SIM 驱动程序必须悄悄地假设较小的值）
* resid, sense_resid - 如果数据传输或 SCSI sense 返回错误，则这些是残余（未传输）数据的返回计数器。它们似乎并不特别有意义，因此在难以计算的情况下（例如，在 SCSI 控制器的 FIFO 缓冲区中计算字节），大约值也足够。对于成功完成的传输，它们必须设置为零。
* tag_action - 要使用的标记类型：

  * CAM_TAG_ACTION_NONE - 不使用标签进行此事务
  * MSG_SIMPLE_Q_TAG, MSG_HEAD_OF_Q_TAG, MSG_ORDERED_Q_TAG - 值等于适当的标签消息（请参阅/sys/cam/scsi/scsi_message.h）；这只提供标签类型，SIM 驱动程序必须自行分配标签值

处理此请求的一般逻辑如下：

首先要做的是检查可能的竞争条件，确保命令在队列中等待时没有被中止：

```
    struct ccb_scsiio *csio = &ccb->csio;

    if ((ccb_h->status & CAM_STATUS_MASK) != CAM_REQ_INPROG) {
        xpt_done(ccb);
        return;
    }
```

同样，我们要检查设备是否被我们的控制器支持：

```
    if (ccb_h->target_id > OUR_MAX_SUPPORTED_TARGET_ID
    || cch_h->target_id == OUR_SCSI_CONTROLLERS_OWN_ID) {
        ccb_h->status = CAM_TID_INVALID;
        xpt_done(ccb);
        return;
    }
    if (ccb_h->target_lun > OUR_MAX_SUPPORTED_LUN) {
        ccb_h->status = CAM_LUN_INVALID;
        xpt_done(ccb);
        return;
    }
```

然后分配我们需要处理此请求的任何数据结构（例如依赖于卡的硬件控制块）。如果我们无法分配，则冻结 SIM 队列并记住我们有一个待处理操作，将 CCB 返回并要求 CAM 重新排队。稍后当资源变得可用时，SIM 队列必须通过返回一个状态中设置了 CAM_SIMQ_RELEASE 位的 ccb 来解冻。否则，如果一切顺利，将 CCB 与硬件控制块（HCB）链接并标记为已排队。

```
    struct xxx_hcb *hcb = allocate_hcb(softc, unit, bus);

    if (hcb == NULL) {
        softc->flags |= RESOURCE_SHORTAGE;
        xpt_freeze_simq(sim, /*count*/1);
        ccb_h->status = CAM_REQUEUE_REQ;
        xpt_done(ccb);
        return;
    }

    hcb->ccb = ccb; ccb_h->ccb_hcb = (void *)hcb;
    ccb_h->status |= CAM_SIM_QUEUED;
```

从 CCB 中提取目标数据到硬件控制块。检查是否要求分配标签，如果是，则生成唯一标签并构建 SCSI 标签消息。SIM 驱动程序还负责与设备协商，以设置最大互相支持的总线宽度、同步速率和偏移量。

```
    hcb->target = ccb_h->target_id; hcb->lun = ccb_h->target_lun;
    generate_identify_message(hcb);
    if (ccb_h->tag_action != CAM_TAG_ACTION_NONE)
        generate_unique_tag_message(hcb, ccb_h->tag_action);
    if (!target_negotiated(hcb))
        generate_negotiation_messages(hcb);
```

然后设置 SCSI 命令。命令存储可以以许多有趣的方式在 CCB 中指定，由 CCB 标志指定。命令缓冲区可以包含在 CCB 中或指向，在后一种情况下，指针可以是物理的或虚拟的。由于硬件通常需要物理地址，我们始终将地址转换为物理地址，通常使用 busdma API。

如果请求物理地址，则可以返回带有状态 CAM_REQ_INVALID 的 CCB，当前的驱动程序就是这样做的。如果需要，物理地址也可以转换或映射回虚拟地址，但非常痛苦，所以我们不这样做。

```
    if (ccb_h->flags & CAM_CDB_POINTER) {
        /* CDB is a pointer */
        if (!(ccb_h->flags & CAM_CDB_PHYS)) {
            /* CDB pointer is virtual */
            hcb->cmd = vtobus(csio->cdb_io.cdb_ptr);
        } else {
            /* CDB pointer is physical */
            hcb->cmd = csio->cdb_io.cdb_ptr ;
        }
    } else {
        /* CDB is in the ccb (buffer) */
        hcb->cmd = vtobus(csio->cdb_io.cdb_bytes);
    }
    hcb->cmdlen = csio->cdb_len;
```

现在是设置数据的时候了。同样，数据存储可以通过 CCB 中的许多有趣方式指定，由 CCB 标志指定。首先，我们得到数据传输的方向。最简单的情况是如果没有数据需要传输：

```
    int dir = (ccb_h->flags & CAM_DIR_MASK);

    if (dir == CAM_DIR_NONE)
        goto end_data;
```

然后我们检查数据是在一个块中还是在一个分散-聚集列表中，地址是物理还是虚拟的。SCSI 控制器可能只能处理有限数量的有限长度的块。如果请求达到了这个限制，我们会返回一个错误。我们使用一个特殊的函数来返回 CCB，以便在一个地方处理 HCB 资源短缺。添加块的函数取决于驱动程序，这里我们不对其进行详细实现。请参阅 SCSI 命令（CDB）处理的描述以获取有关地址转换问题的详细信息。如果某些变化对于使用特定卡实现来说太困难或不可能，将状态 CAM_REQ_INVALID 返回是可以的。实际上，看起来分散-聚集能力现在在 CAM 代码中没有被使用。但至少要实现非分散虚拟缓冲区的情况，它被 CAM 积极使用。

```
    int rv;

    initialize_hcb_for_data(hcb);

    if ((!(ccb_h->flags & CAM_SCATTER_VALID)) {
        /* single buffer */
        if (!(ccb_h->flags & CAM_DATA_PHYS)) {
            rv = add_virtual_chunk(hcb, csio->data_ptr, csio->dxfer_len, dir);
            }
        } else {
            rv = add_physical_chunk(hcb, csio->data_ptr, csio->dxfer_len, dir);
        }
    } else {
        int i;
        struct bus_dma_segment *segs;
        segs = (struct bus_dma_segment *)csio->data_ptr;

        if ((ccb_h->flags & CAM_SG_LIST_PHYS) != 0) {
            /* The SG list pointer is physical */
            rv = setup_hcb_for_physical_sg_list(hcb, segs, csio->sglist_cnt);
        } else if (!(ccb_h->flags & CAM_DATA_PHYS)) {
            /* SG buffer pointers are virtual */
            for (i = 0; i < csio->sglist_cnt; i++) {
                rv = add_virtual_chunk(hcb, segs[i].ds_addr,
                    segs[i].ds_len, dir);
                if (rv != CAM_REQ_CMP)
                    break;
            }
        } else {
            /* SG buffer pointers are physical */
            for (i = 0; i < csio->sglist_cnt; i++) {
                rv = add_physical_chunk(hcb, segs[i].ds_addr,
                    segs[i].ds_len, dir);
                if (rv != CAM_REQ_CMP)
                    break;
            }
        }
    }
    if (rv != CAM_REQ_CMP) {
        /* we expect that add_*_chunk() functions return CAM_REQ_CMP
         * if they added a chunk successfully, CAM_REQ_TOO_BIG if
         * the request is too big (too many bytes or too many chunks),
         * CAM_REQ_INVALID in case of other troubles
         */
        free_hcb_and_ccb_done(hcb, ccb, rv);
        return;
    }
    end_data:
```

如果为此 CCB 禁用了断开连接，我们将这些信息传递给 hcb:

```
    if (ccb_h->flags & CAM_DIS_DISCONNECT)
        hcb_disable_disconnect(hcb);
```

如果控制器能够独自运行 REQUEST SENSE 命令，那么 CAM_DIS_AUTOSENSE 标志的值也应该传递给它，以防止 CAM 子系统不需要时自动执行 REQUEST SENSE。

唯一剩下的事情是设置超时，将我们的 hcb 传递给硬件并返回，其余的将由中断处理程序（或超时处理程序）完成。

```
    ccb_h->timeout_ch = timeout(xxx_timeout, (caddr_t) hcb,
        (ccb_h->timeout * hz) / 1000); /* convert milliseconds to ticks */
    put_hcb_into_hardware_queue(hcb);
    return;
```

这里是一个返回 CCB 的函数的可能实现。

```
    static void
    free_hcb_and_ccb_done(struct xxx_hcb *hcb, union ccb *ccb, u_int32_t status)
    {
        struct xxx_softc *softc = hcb->softc;

        ccb->ccb_h.ccb_hcb = 0;
        if (hcb != NULL) {
            untimeout(xxx_timeout, (caddr_t) hcb, ccb->ccb_h.timeout_ch);
            /* we're about to free a hcb, so the shortage has ended */
            if (softc->flags & RESOURCE_SHORTAGE)  {
                softc->flags &= ~RESOURCE_SHORTAGE;
                status |= CAM_RELEASE_SIMQ;
            }
            free_hcb(hcb); /* also removes hcb from any internal lists */
        }
        ccb->ccb_h.status = status |
            (ccb->ccb_h.status & ~(CAM_STATUS_MASK|CAM_SIM_QUEUED));
        xpt_done(ccb);
    }
```

### 12.5.2. XPT_RESET_DEV - 向设备发送 SCSI“总线设备复位”消息

在 CCB 中没有传输任何数据，除了头部之外，其中最有趣的参数是目标 ID。根据控制器硬件的不同，可能会构建类似于 XPT_SCSI_IO 请求的硬件控制块（请参阅 XPT_SCSI_IO 请求描述）并发送到控制器，或者 SCSI 控制器可能会立即被编程以向设备发送此复位消息，或者此请求可能根本不受支持（并返回状态 CAM_REQ_INVALID ）。此外，在请求完成时，必须中止针对该目标的所有断开的事务（可能在中断例程中）。

此外，重置时所有针对目标的当前协商都会丢失，因此它们可能也会被清除。或者清除可能会被推迟，因为无论如何，目标都会在下一个事务上请求重新协商。

### 12.5.3. XPT_RESET_BUS - 将复位信号发送到 SCSI 总线

在 CCB 中不传递参数，唯一感兴趣的参数是通过结构 sim 指针指示的 SCSI 总线。

一个简化的实现会忘记总线上所有设备的 SCSI 协商，并返回状态 CAM_REQ_CMP。

实际上，正确的实现将重置 SCSI 总线（可能还会重置 SCSI 控制器），并将所有正在处理的 CCB，包括硬件队列中的和断开连接的，都标记为状态 CAM_SCSI_BUS_RESET，如：

```
    int targ, lun;
    struct xxx_hcb *h, *hh;
    struct ccb_trans_settings neg;
    struct cam_path *path;

    /* The SCSI bus reset may take a long time, in this case its completion
     * should be checked by interrupt or timeout. But for simplicity
     * we assume here that it is really fast.
     */
    reset_scsi_bus(softc);

    /* drop all enqueued CCBs */
    for (h = softc->first_queued_hcb; h != NULL; h = hh) {
        hh = h->next;
        free_hcb_and_ccb_done(h, h->ccb, CAM_SCSI_BUS_RESET);
    }

    /* the clean values of negotiations to report */
    neg.bus_width = 8;
    neg.sync_period = neg.sync_offset = 0;
    neg.valid = (CCB_TRANS_BUS_WIDTH_VALID
        | CCB_TRANS_SYNC_RATE_VALID | CCB_TRANS_SYNC_OFFSET_VALID);

    /* drop all disconnected CCBs and clean negotiations  */
    for (targ=0; targ <= OUR_MAX_SUPPORTED_TARGET; targ++) {
        clean_negotiations(softc, targ);

        /* report the event if possible */
        if (xpt_create_path(&path, /*periph*/NULL,
                cam_sim_path(sim), targ,
                CAM_LUN_WILDCARD) == CAM_REQ_CMP) {
            xpt_async(AC_TRANSFER_NEG, path, &neg);
            xpt_free_path(path);
        }

        for (lun=0; lun <= OUR_MAX_SUPPORTED_LUN; lun++)
            for (h = softc->first_discon_hcb[targ][lun]; h != NULL; h = hh) {
                hh=h->next;
                free_hcb_and_ccb_done(h, h->ccb, CAM_SCSI_BUS_RESET);
            }
    }

    ccb->ccb_h.status = CAM_REQ_CMP;
    xpt_done(ccb);

    /* report the event */
    xpt_async(AC_BUS_RESET, softc->wpath, NULL);
    return;
```

将 SCSI 总线复位实现为一个函数可能是个好主意，因为如果事情出错，它将被超时函数作为最后手段重复使用。

### 12.5.4. XPT_ABORT - 中止指定的 CCB

参数被传递给联合 ccb 的实例"struct ccb_abort cab"中。它中唯一的参数字段是：

* abort_ccb - 指向要中止的 CCB 的指针

如果中止不受支持，只需返回状态 CAM_UA_ABORT。这也是最简单实现此调用的方法，在任何情况下返回 CAM_UA_ABORT。

用诚实的方式实现这个请求的困难方式。首先检查中止是否适用于 SCSI 事务：

```
    struct ccb *abort_ccb;
    abort_ccb = ccb->cab.abort_ccb;

    if (abort_ccb->ccb_h.func_code != XPT_SCSI_IO) {
        ccb->ccb_h.status = CAM_UA_ABORT;
        xpt_done(ccb);
        return;
    }
```

然后需要在我们的队列中找到这个 CCB。这可以通过遍历所有与此 CCB 关联的硬件控制块列表来完成：

```
    struct xxx_hcb *hcb, *h;

    hcb = NULL;

    /* We assume that softc->first_hcb is the head of the list of all
     * HCBs associated with this bus, including those enqueued for
     * processing, being processed by hardware and disconnected ones.
     */
    for (h = softc->first_hcb; h != NULL; h = h->next) {
        if (h->ccb == abort_ccb) {
            hcb = h;
            break;
        }
    }

    if (hcb == NULL) {
        /* no such CCB in our queue */
        ccb->ccb_h.status = CAM_PATH_INVALID;
        xpt_done(ccb);
        return;
    }

    hcb=found_hcb;
```

现在我们看一下 HCB 的当前处理状态。它可能是坐在队列中等待发送到 SCSI 总线，正在传输中，或者已经断开连接并等待命令的结果，或者实际上已经被硬件完成但尚未被软件标记为完成。为了确保我们不会与硬件发生任何竞争，我们将 HCB 标记为已中止，这样如果这个 HCB 即将被发送到 SCSI 总线，SCSI 控制器将看到这个标志并跳过它。

```
    int hstatus;

    /* shown as a function, in case special action is needed to make
     * this flag visible to hardware
     */
    set_hcb_flags(hcb, HCB_BEING_ABORTED);

    abort_again:

    hstatus = get_hcb_status(hcb);
    switch (hstatus) {
    case HCB_SITTING_IN_QUEUE:
        remove_hcb_from_hardware_queue(hcb);
        /* FALLTHROUGH */
    case HCB_COMPLETED:
        /* this is an easy case */
        free_hcb_and_ccb_done(hcb, abort_ccb, CAM_REQ_ABORTED);
        break;
```

如果 CCB 正在传输中，我们希望以某种硬件相关的方式向 SCSI 控制器发出我们要中止当前传输的信号。SCSI 控制器将设置 SCSI 注意信号，当目标响应时发送中止消息。我们还重置超时以确保目标不会永远沉睡。如果命令在合理的时间内（如 10 秒）内没有被中止，超时程序将继续重置整个 SCSI 总线。由于在合理的时间内命令将被中止，我们现在可以将中止请求返回为成功完成，并将中止的 CCB 标记为已中止（但尚未标记为已完成）。

```
    case HCB_BEING_TRANSFERRED:
        untimeout(xxx_timeout, (caddr_t) hcb, abort_ccb->ccb_h.timeout_ch);
        abort_ccb->ccb_h.timeout_ch =
            timeout(xxx_timeout, (caddr_t) hcb, 10 * hz);
        abort_ccb->ccb_h.status = CAM_REQ_ABORTED;
        /* ask the controller to abort that HCB, then generate
         * an interrupt and stop
         */
        if (signal_hardware_to_abort_hcb_and_stop(hcb) < 0) {
            /* oops, we missed the race with hardware, this transaction
             * got off the bus before we aborted it, try again */
            goto abort_again;
        }

        break;
```

如果 CCB 在已断开连接列表中，则将其设置为中止请求，并将其重新排队放在硬件队列的最前面。重置超时并报告中止请求已完成。

```
    case HCB_DISCONNECTED:
        untimeout(xxx_timeout, (caddr_t) hcb, abort_ccb->ccb_h.timeout_ch);
        abort_ccb->ccb_h.timeout_ch =
            timeout(xxx_timeout, (caddr_t) hcb, 10 * hz);
        put_abort_message_into_hcb(hcb);
        put_hcb_at_the_front_of_hardware_queue(hcb);
        break;
    }
    ccb->ccb_h.status = CAM_REQ_CMP;
    xpt_done(ccb);
    return;
```

关于中止请求就是这些，尽管还有一个问题。由于中止消息清除了 LUN 上的所有正在进行的事务，我们必须将此 LUN 上的所有其他活动事务标记为已中止。这应该在事务被中止后的中断程序中完成。

实现 CCB 中止作为一个函数可能是一个很好的主意，如果 I/O 事务超时，这个函数可以被重复使用。唯一的区别是超时的事务会返回状态 CAM_CMD_TIMEOUT 用于请求超时。然后 XPT_ABORT 情况会很小，就像这样：

```
    case XPT_ABORT:
        struct ccb *abort_ccb;
        abort_ccb = ccb->cab.abort_ccb;

        if (abort_ccb->ccb_h.func_code != XPT_SCSI_IO) {
            ccb->ccb_h.status = CAM_UA_ABORT;
            xpt_done(ccb);
            return;
        }
        if (xxx_abort_ccb(abort_ccb, CAM_REQ_ABORTED) < 0)
            /* no such CCB in our queue */
            ccb->ccb_h.status = CAM_PATH_INVALID;
        else
            ccb->ccb_h.status = CAM_REQ_CMP;
        xpt_done(ccb);
        return;
```

### 12.5.5. XPT_SET_TRAN_SETTINGS - 明确设置 SCSI 传输设置的值

参数传递到联合 ccb 的实例“struct ccb_trans_setting cts”中。

* 有效 - 一个位掩码，显示应更新哪些设置：

  * CCB_TRANS_SYNC_RATE_VALID - 同步传输速率
  * CCB_TRANS_SYNC_OFFSET_VALID - 同步偏移
  * CCB_TRANS_BUS_WIDTH_VALID - 总线宽度
  * CCB_TRANS_DISC_VALID - 设置启用/禁用断开连接
  * CCB_TRANS_TQ_VALID - 设置启用/禁用标记排队
* 标志 - 由两部分组成，二进制参数和子操作的标识。 二进制参数是：

  * CCB_TRANS_DISC_ENB - 启用断开连接
  * CCB_TRANS_TAG_ENB - 启用标记排队
* 子操作如下：

  * CCB_TRANS_CURRENT_SETTINGS - 更改当前协商
  * CCB_TRANS_USER_SETTINGS - 记住所需的用户值同步周期，同步偏移 - 不言自明，如果 sync_offset==0，则请求异步模式 bus_width - 总线宽度，以位为单位（不是字节）

支持两组协商参数，用户设置和当前设置。用户设置在 SIM 驱动程序中并没有被广泛使用，这主要只是一个内存空间，上层可以在其中存储（以及稍后调用）有关参数的想法。设置用户参数不会导致传输速率重新协商。但是，当 SCSI 控制器进行协商时，绝不能将值设置得高于用户参数，因此它实质上是顶部边界。

当前设置是，顾名思义，当前的设置。更改它们意味着参数必须在下一次传输时重新协商。同样，这些“新的当前设置”不应该被强加给设备，它们只是协商的初始步骤。此外，它们必须受到 SCSI 控制器实际能力的限制：例如，如果 SCSI 控制器具有 8 位总线，并且请求要求设置 16 位宽传输，则在将其发送到设备之前，此参数必须被静默截断为 8 位传输。

一个注意事项是总线宽度和同步参数是每个目标的，而断开连接和标记启用参数是每个逻辑单元的。

推荐的实现方法是保留 3 组协商的参数（总线宽度和同步传输）：

* 用户 - 与上述相同的用户集
* 当前 - 实际生效的那些
* 目标 - 由"current"参数设置请求的那些

代码看起来像：

```
    struct ccb_trans_settings *cts;
    int targ, lun;
    int flags;

    cts = &ccb->cts;
    targ = ccb_h->target_id;
    lun = ccb_h->target_lun;
    flags = cts->flags;
    if (flags & CCB_TRANS_USER_SETTINGS) {
        if (flags & CCB_TRANS_SYNC_RATE_VALID)
            softc->user_sync_period[targ] = cts->sync_period;
        if (flags & CCB_TRANS_SYNC_OFFSET_VALID)
            softc->user_sync_offset[targ] = cts->sync_offset;
        if (flags & CCB_TRANS_BUS_WIDTH_VALID)
            softc->user_bus_width[targ] = cts->bus_width;

        if (flags & CCB_TRANS_DISC_VALID) {
            softc->user_tflags[targ][lun] &= ~CCB_TRANS_DISC_ENB;
            softc->user_tflags[targ][lun] |= flags & CCB_TRANS_DISC_ENB;
        }
        if (flags & CCB_TRANS_TQ_VALID) {
            softc->user_tflags[targ][lun] &= ~CCB_TRANS_TQ_ENB;
            softc->user_tflags[targ][lun] |= flags & CCB_TRANS_TQ_ENB;
        }
    }
    if (flags & CCB_TRANS_CURRENT_SETTINGS) {
        if (flags & CCB_TRANS_SYNC_RATE_VALID)
            softc->goal_sync_period[targ] =
                max(cts->sync_period, OUR_MIN_SUPPORTED_PERIOD);
        if (flags & CCB_TRANS_SYNC_OFFSET_VALID)
            softc->goal_sync_offset[targ] =
                min(cts->sync_offset, OUR_MAX_SUPPORTED_OFFSET);
        if (flags & CCB_TRANS_BUS_WIDTH_VALID)
            softc->goal_bus_width[targ] = min(cts->bus_width, OUR_BUS_WIDTH);

        if (flags & CCB_TRANS_DISC_VALID) {
            softc->current_tflags[targ][lun] &= ~CCB_TRANS_DISC_ENB;
            softc->current_tflags[targ][lun] |= flags & CCB_TRANS_DISC_ENB;
        }
        if (flags & CCB_TRANS_TQ_VALID) {
            softc->current_tflags[targ][lun] &= ~CCB_TRANS_TQ_ENB;
            softc->current_tflags[targ][lun] |= flags & CCB_TRANS_TQ_ENB;
        }
    }
    ccb->ccb_h.status = CAM_REQ_CMP;
    xpt_done(ccb);
    return;
```

然后，当下一个 I/O 请求被处理时，它将检查是否需要重新协商，例如通过调用函数 target_negotiated(hcb)。可以这样实现：

```
    int
    target_negotiated(struct xxx_hcb *hcb)
    {
        struct softc *softc = hcb->softc;
        int targ = hcb->targ;

        if (softc->current_sync_period[targ] != softc->goal_sync_period[targ]
        || softc->current_sync_offset[targ] != softc->goal_sync_offset[targ]
        || softc->current_bus_width[targ] != softc->goal_bus_width[targ])
            return 0; /* FALSE */
        else
            return 1; /* TRUE */
    }
```

在重新协商值之后，必须将得到的值分配给当前参数和目标参数，因此对于将来的 I/O 事务，当前参数和目标参数将是相同的，并且 target_negotiated() 将返回 TRUE。当卡初始化时（在 xxx_attach() 中），当前协商值必须初始化为窄异步模式，目标和当前值必须初始化为控制器支持的最大值。

### 12.5.6. XPT_GET_TRAN_SETTINGS - 获取 SCSI 传输设置的值

这些操作是 XPT_SET_TRAN_SETTINGS 的反向操作。使用 CCB_TRANS_CURRENT_SETTINGS 或 CCB_TRANS_USER_SETTINGS 标志根据请求的数据填充 CCB 实例"struct ccb_trans_setting cts"（如果两者都设置，则现有驱动程序将返回当前设置）。将 valid 字段中的所有位设置。

### 12.5.7. XPT_CALC_GEOMETRY - 计算磁盘的逻辑（BIOS）几何结构

参数传递在联合 ccb 的实例“struct ccb_calc_geometry ccg”中进行：

* block_size - 输入，以字节为单位的块（又名扇区）大小
* 输入, 以字节为单位的卷大小
* 输出, 逻辑柱面
* 输出, 逻辑磁头
* *secs_per_track* - output, logical sectors per track

If the returned geometry differs much enough from what the SCSI controller BIOS thinks and a disk on this SCSI controller is used as bootable the system may not be able to boot. The typical calculation example taken from the aic7xxx driver is:

```
    struct    ccb_calc_geometry *ccg;
    u_int32_t size_mb;
    u_int32_t secs_per_cylinder;
    int   extended;

    ccg = &ccb->ccg;
    size_mb = ccg->volume_size
        / ((1024L * 1024L) / ccg->block_size);
    extended = check_cards_EEPROM_for_extended_geometry(softc);

    if (size_mb > 1024 && extended) {
        ccg->heads = 255;
        ccg->secs_per_track = 63;
    } else {
        ccg->heads = 64;
        ccg->secs_per_track = 32;
    }
    secs_per_cylinder = ccg->heads * ccg->secs_per_track;
    ccg->cylinders = ccg->volume_size / secs_per_cylinder;
    ccb->ccb_h.status = CAM_REQ_CMP;
    xpt_done(ccb);
    return;
```

This gives the general idea, the exact calculation depends on the quirks of the particular BIOS. If BIOS provides no way set the "extended translation" flag in EEPROM this flag should normally be assumed equal to 1. Other popular geometries are:

```
    128 heads, 63 sectors - Symbios controllers
    16 heads, 63 sectors - old controllers
```

一些系统 BIOS 和 SCSI BIOS 彼此争斗，成功的程度不同，例如 Symbios 875/895 SCSI 和 Phoenix BIOS 的组合在上电后可能会给出几何 128/63，在硬重置或软重启后可能会给出几何 255/63。

### 12.5.8. XPT_PATH_INQ - 路径查询，换句话说，获取 SIM 驱动程序和 SCSI 控制器（也称为 HBA - 主机总线适配器）属性

这些属性以联合 ccb 的实例"struct ccb_pathinq cpi"的形式返回：

* 版本号 - SIM 驱动程序版本号，现在所有驱动程序使用 1
* hba_inquiry - 控制器支持的特性位掩码：

  * PI_MDP_ABLE - 支持 MDP 消息（来自 SCSI3 的某些内容？）
  * 支持 32 位宽 SCSI
  * 支持 16 位宽 SCSI
  * 可以协商同步传输速率
  * PI_LINKED_CDB - 支持链接命令
  * PI_TAG_ABLE - 支持标记命令
  * PI_SOFT_RST - 支持软复位替代方案（在 SCSI 总线内，硬复位和软复位是互斥的）
* target_sprt - 用于目标模式支持的标志，如果不支持则为 0
* hba_misc - 杂项控制器功能：

  * PIM_SCANHILO - 从高 ID 到低 ID 进行总线扫描
  * 不可删除 - 可移动设备不包含在扫描中
  * PIM_NOINITIATOR - 发起者角色不支持
  * PIM_NOBUSRESET - 用户已禁用初始总线复位
* hba_eng_cnt - 神秘的 HBA 引擎计数，与压缩有关，现在始终设置为 0
* vuhba_flags - 供应商特有标志，现在未使用
* max_target - 最大支持的目标 ID（8 位总线为 7，16 位总线为 15，光纤通道为 127）
* max_lun - 最大支持的 LUN ID（对于较旧的 SCSI 控制器为 7，对于较新的为 63）
* async_flags - 安装的异步处理程序的位掩码，目前未使用
* hpath_id - 子系统中最高的路径 ID，目前未使用
* 单元编号 - 控制器单元编号，cam_sim_unit（模拟器）
* 总线编号 - 总线编号，cam_sim_bus（模拟器）
* 发起者编号 - 控制器本身的 SCSI ID
* 基础传输速度 - 异步窄带传输的标称传输速度，对于 SCSI 等于 3300
* sim_vid - SIM 驱动程序的供应商 ID，最大长度为 SIM_IDLEN 的以零结尾的字符串，包括终止零
* hba_vid - SCSI 控制器的供应商 ID，最大长度为 HBA_IDLEN 的以零结尾的字符串，包括终止零
* dev_name - 设备驱动程序名称，长度最大为 DEV_IDLEN（包括终止零），与 cam_sim_name(sim)相等

设置字符串字段的推荐方法是使用 strncpy，例如：

```
    strncpy(cpi->dev_name, cam_sim_name(sim), DEV_IDLEN);
```

在设置值后，将状态设置为 CAM_REQ_CMP，并将 CCB 标记为完成。

## 12.6. 轮询 xxx_poll

```
static void xxx_poll(struct cam_sim *);
```

当中断子系统不起作用时（例如，系统崩溃并正在创建系统转储时），使用轮询功能来模拟中断。CAM 子系统在调用轮询例程之前设置适当的中断级别。因此，它所需做的就是调用中断例程（或者反过来，轮询例程可能正在执行实际操作，而中断例程只会调用轮询例程）。那么为什么要费心使用单独的函数呢？这与不同的调用约定有关。 xxx_poll 例程将结构 cam_sim 指针作为其参数，而根据通用约定，PCI 中断例程会获得指向结构 xxx_softc 的指针，ISA 中断例程只会获得设备单元号。因此，轮询例程通常会如下所示：

```
static void
xxx_poll(struct cam_sim *sim)
{
    xxx_intr((struct xxx_softc *)cam_sim_softc(sim)); /* for PCI device */
}
```

 或

```
static void
xxx_poll(struct cam_sim *sim)
{
    xxx_intr(cam_sim_unit(sim)); /* for ISA device */
}
```

## 12.7. 异步事件

如果已设置异步事件回调，则应定义回调函数。

```
static void
ahc_async(void *callback_arg, u_int32_t code, struct cam_path *path, void *arg)
```

* callback_arg - 在注册回调时提供的值。
* 代码 - 识别事件类型
* 路径 - 标识事件适用的设备
* 参数 - 事件特定参数

一个类型的事件的实现，AC_LOST_DEVICE，看起来像：

```
    struct xxx_softc *softc;
    struct cam_sim *sim;
    int targ;
    struct ccb_trans_settings neg;

    sim = (struct cam_sim *)callback_arg;
    softc = (struct xxx_softc *)cam_sim_softc(sim);
    switch (code) {
    case AC_LOST_DEVICE:
        targ = xpt_path_target_id(path);
        if (targ <= OUR_MAX_SUPPORTED_TARGET) {
            clean_negotiations(softc, targ);
            /* send indication to CAM */
            neg.bus_width = 8;
            neg.sync_period = neg.sync_offset = 0;
            neg.valid = (CCB_TRANS_BUS_WIDTH_VALID
                | CCB_TRANS_SYNC_RATE_VALID | CCB_TRANS_SYNC_OFFSET_VALID);
            xpt_async(AC_TRANSFER_NEG, path, &neg);
        }
        break;
    default:
        break;
    }
```

## 12.8. 中断

中断例程的确切类型取决于 SCSI 控制器连接的外围总线的类型（PCI、ISA 等）。

SIM 驱动程序的中断例程在中断级别 splcam 上运行。因此，在驱动程序中应该使用 splcam() 来同步中断例程和驱动程序的其他部分（对于一个多处理器感知的驱动程序，情况会更加有趣，但我们在这里忽略这种情况）。本文档中的伪代码快乐地忽略了同步的问题。真实的代码不应该忽略它们。一个简单的方法是在进入其他例程时设置 splcam() ，并在返回时将其重置，从而通过一个大的临界段来保护它们。为了确保中断级别总是会被恢复，可以定义一个包装函数，如：

```
    static void
    xxx_action(struct cam_sim *sim, union ccb *ccb)
    {
        int s;
        s = splcam();
        xxx_action1(sim, ccb);
        splx(s);
    }

    static void
    xxx_action1(struct cam_sim *sim, union ccb *ccb)
    {
        ... process the request ...
    }
```

这种方法简单而稳健，但它的问题在于中断可能会被阻塞相对长的时间，这会对系统的性能产生负面影响。另一方面， spl() 系列的功能开销相当高，因此大量微小的临界段可能也不是一个好主意。

中断例程处理的条件和详细信息非常依赖于硬件。我们考虑“典型”条件集。

首先，我们检查总线上是否遇到了 SCSI 复位（可能是由另一个 SCSI 控制器引起的）。如果是这样，我们将放弃所有已排队和断开连接的请求，报告事件并重新初始化我们的 SCSI 控制器。在此期间很重要的一点是，在此初始化过程中，控制器不会发出另一个复位信号，否则同一条 SCSI 总线上的两个控制器可能会永远地发出复位信号。致命控制器错误/挂起的情况可能也可以在同一位置处理，但可能还需要向 SCSI 总线发送复位信号，以重置与 SCSI 设备的连接状态。

```
    int fatal=0;
    struct ccb_trans_settings neg;
    struct cam_path *path;

    if (detected_scsi_reset(softc)
    || (fatal = detected_fatal_controller_error(softc))) {
        int targ, lun;
        struct xxx_hcb *h, *hh;

        /* drop all enqueued CCBs */
        for(h = softc->first_queued_hcb; h != NULL; h = hh) {
            hh = h->next;
            free_hcb_and_ccb_done(h, h->ccb, CAM_SCSI_BUS_RESET);
        }

        /* the clean values of negotiations to report */
        neg.bus_width = 8;
        neg.sync_period = neg.sync_offset = 0;
        neg.valid = (CCB_TRANS_BUS_WIDTH_VALID
            | CCB_TRANS_SYNC_RATE_VALID | CCB_TRANS_SYNC_OFFSET_VALID);

        /* drop all disconnected CCBs and clean negotiations  */
        for (targ=0; targ <= OUR_MAX_SUPPORTED_TARGET; targ++) {
            clean_negotiations(softc, targ);

            /* report the event if possible */
            if (xpt_create_path(&path, /*periph*/NULL,
                    cam_sim_path(sim), targ,
                    CAM_LUN_WILDCARD) == CAM_REQ_CMP) {
                xpt_async(AC_TRANSFER_NEG, path, &neg);
                xpt_free_path(path);
            }

            for (lun=0; lun <= OUR_MAX_SUPPORTED_LUN; lun++)
                for (h = softc->first_discon_hcb[targ][lun]; h != NULL; h = hh) {
                    hh=h->next;
                    if (fatal)
                        free_hcb_and_ccb_done(h, h->ccb, CAM_UNREC_HBA_ERROR);
                    else
                        free_hcb_and_ccb_done(h, h->ccb, CAM_SCSI_BUS_RESET);
                }
        }

        /* report the event */
        xpt_async(AC_BUS_RESET, softc->wpath, NULL);

        /* re-initialization may take a lot of time, in such case
         * its completion should be signaled by another interrupt or
         * checked on timeout - but for simplicity we assume here that
         * it is really fast
         */
        if (!fatal) {
            reinitialize_controller_without_scsi_reset(softc);
        } else {
            reinitialize_controller_with_scsi_reset(softc);
        }
        schedule_next_hcb(softc);
        return;
    }
```

如果中断不是由控制器范围内的条件引起的，那么当前硬件控制块可能发生了一些问题。根据硬件的不同，可能还会发生其他与 HCB 无关的事件，我们在这里不考虑它们。然后，我们分析此 HCB 发生了什么：

```
    struct xxx_hcb *hcb, *h, *hh;
    int hcb_status, scsi_status;
    int ccb_status;
    int targ;
    int lun_to_freeze;

    hcb = get_current_hcb(softc);
    if (hcb == NULL) {
        /* either stray interrupt or something went very wrong
         * or this is something hardware-dependent
         */
        handle as necessary;
        return;
    }

    targ = hcb->target;
    hcb_status = get_status_of_current_hcb(softc);
```

首先，我们检查 HCB 是否已完成，如果是，则检查返回的 SCSI 状态。

```
    if (hcb_status == COMPLETED) {
        scsi_status = get_completion_status(hcb);
```

然后查看此状态是否与 REQUEST SENSE 命令相关，如果是，请以简单方式处理。

```
        if (hcb->flags & DOING_AUTOSENSE) {
            if (scsi_status == GOOD) { /* autosense was successful */
                hcb->ccb->ccb_h.status |= CAM_AUTOSNS_VALID;
                free_hcb_and_ccb_done(hcb, hcb->ccb, CAM_SCSI_STATUS_ERROR);
            } else {
        autosense_failed:
                free_hcb_and_ccb_done(hcb, hcb->ccb, CAM_AUTOSENSE_FAIL);
            }
            schedule_next_hcb(softc);
            return;
        }
```

否则，命令本身已完成，请更加注意细节。如果未为此 CCB 禁用自动感知，并且命令因感知数据而失败，则运行 REQUEST SENSE 命令以接收该数据。

```
        hcb->ccb->csio.scsi_status = scsi_status;
        calculate_residue(hcb);

        if ((hcb->ccb->ccb_h.flags & CAM_DIS_AUTOSENSE)==0
        && (scsi_status == CHECK_CONDITION
                || scsi_status == COMMAND_TERMINATED)) {
            /* start auto-SENSE */
            hcb->flags |= DOING_AUTOSENSE;
            setup_autosense_command_in_hcb(hcb);
            restart_current_hcb(softc);
            return;
        }
        if (scsi_status == GOOD)
            free_hcb_and_ccb_done(hcb, hcb->ccb, CAM_REQ_CMP);
        else
            free_hcb_and_ccb_done(hcb, hcb->ccb, CAM_SCSI_STATUS_ERROR);
        schedule_next_hcb(softc);
        return;
    }
```

一个典型的情况可能是协商事件：从 SCSI 目标接收的协商消息（作为回应我们的协商尝试或目标主动发起）或目标无法协商（拒绝我们的协商消息或不回应它们）。

```
    switch (hcb_status) {
    case TARGET_REJECTED_WIDE_NEG:
        /* revert to 8-bit bus */
        softc->current_bus_width[targ] = softc->goal_bus_width[targ] = 8;
        /* report the event */
        neg.bus_width = 8;
        neg.valid = CCB_TRANS_BUS_WIDTH_VALID;
        xpt_async(AC_TRANSFER_NEG, hcb->ccb.ccb_h.path_id, &neg);
        continue_current_hcb(softc);
        return;
    case TARGET_ANSWERED_WIDE_NEG:
        {
            int wd;

            wd = get_target_bus_width_request(softc);
            if (wd <= softc->goal_bus_width[targ]) {
                /* answer is acceptable */
                softc->current_bus_width[targ] =
                softc->goal_bus_width[targ] = neg.bus_width = wd;

                /* report the event */
                neg.valid = CCB_TRANS_BUS_WIDTH_VALID;
                xpt_async(AC_TRANSFER_NEG, hcb->ccb.ccb_h.path_id, &neg);
            } else {
                prepare_reject_message(hcb);
            }
        }
        continue_current_hcb(softc);
        return;
    case TARGET_REQUESTED_WIDE_NEG:
        {
            int wd;

            wd = get_target_bus_width_request(softc);
            wd = min (wd, OUR_BUS_WIDTH);
            wd = min (wd, softc->user_bus_width[targ]);

            if (wd != softc->current_bus_width[targ]) {
                /* the bus width has changed */
                softc->current_bus_width[targ] =
                softc->goal_bus_width[targ] = neg.bus_width = wd;

                /* report the event */
                neg.valid = CCB_TRANS_BUS_WIDTH_VALID;
                xpt_async(AC_TRANSFER_NEG, hcb->ccb.ccb_h.path_id, &neg);
            }
            prepare_width_nego_rsponse(hcb, wd);
        }
        continue_current_hcb(softc);
        return;
    }
```

然后，我们以与之前相同的简单方式处理在自动感应过程中可能发生的任何错误。否则，我们再次仔细查看细节。

```
    if (hcb->flags & DOING_AUTOSENSE)
        goto autosense_failed;

    switch (hcb_status) {
```

我们考虑的下一个事件是意外断开连接。在 ABORT 或 BUS DEVICE RESET 消息后被视为正常，其他情况下视为异常。

```
    case UNEXPECTED_DISCONNECT:
        if (requested_abort(hcb)) {
            /* abort affects all commands on that target+LUN, so
             * mark all disconnected HCBs on that target+LUN as aborted too
             */
            for (h = softc->first_discon_hcb[hcb->target][hcb->lun];
                    h != NULL; h = hh) {
                hh=h->next;
                free_hcb_and_ccb_done(h, h->ccb, CAM_REQ_ABORTED);
            }
            ccb_status = CAM_REQ_ABORTED;
        } else if (requested_bus_device_reset(hcb)) {
            int lun;

            /* reset affects all commands on that target, so
             * mark all disconnected HCBs on that target+LUN as reset
             */

            for (lun=0; lun <= OUR_MAX_SUPPORTED_LUN; lun++)
                for (h = softc->first_discon_hcb[hcb->target][lun];
                        h != NULL; h = hh) {
                    hh=h->next;
                    free_hcb_and_ccb_done(h, h->ccb, CAM_SCSI_BUS_RESET);
                }

            /* send event */
            xpt_async(AC_SENT_BDR, hcb->ccb->ccb_h.path_id, NULL);

            /* this was the CAM_RESET_DEV request itself, it is completed */
            ccb_status = CAM_REQ_CMP;
        } else {
            calculate_residue(hcb);
            ccb_status = CAM_UNEXP_BUSFREE;
            /* request the further code to freeze the queue */
            hcb->ccb->ccb_h.status |= CAM_DEV_QFRZN;
            lun_to_freeze = hcb->lun;
        }
        break;
```

如果目标拒绝接受标签，我们通知 CAM 并将返回该 LUN 的所有命令。

```
    case TAGS_REJECTED:
        /* report the event */
        neg.flags = 0 & ~CCB_TRANS_TAG_ENB;
        neg.valid = CCB_TRANS_TQ_VALID;
        xpt_async(AC_TRANSFER_NEG, hcb->ccb.ccb_h.path_id, &neg);

        ccb_status = CAM_MSG_REJECT_REC;
        /* request the further code to freeze the queue */
        hcb->ccb->ccb_h.status |= CAM_DEV_QFRZN;
        lun_to_freeze = hcb->lun;
        break;
```

然后我们检查其他一些条件，处理基本上仅限于设置 CCB 状态：

```
    case SELECTION_TIMEOUT:
        ccb_status = CAM_SEL_TIMEOUT;
        /* request the further code to freeze the queue */
        hcb->ccb->ccb_h.status |= CAM_DEV_QFRZN;
        lun_to_freeze = CAM_LUN_WILDCARD;
        break;
    case PARITY_ERROR:
        ccb_status = CAM_UNCOR_PARITY;
        break;
    case DATA_OVERRUN:
    case ODD_WIDE_TRANSFER:
        ccb_status = CAM_DATA_RUN_ERR;
        break;
    default:
        /* all other errors are handled in a generic way */
        ccb_status = CAM_REQ_CMP_ERR;
        /* request the further code to freeze the queue */
        hcb->ccb->ccb_h.status |= CAM_DEV_QFRZN;
        lun_to_freeze = CAM_LUN_WILDCARD;
        break;
    }
```

然后我们检查错误是否严重到需要冻结输入队列直到处理完毕，如果是，则执行：

```
    if (hcb->ccb->ccb_h.status & CAM_DEV_QFRZN) {
        /* freeze the queue */
        xpt_freeze_devq(ccb->ccb_h.path, /*count*/1);

        /* re-queue all commands for this target/LUN back to CAM */

        for (h = softc->first_queued_hcb; h != NULL; h = hh) {
            hh = h->next;

            if (targ == h->targ
            && (lun_to_freeze == CAM_LUN_WILDCARD || lun_to_freeze == h->lun))
                free_hcb_and_ccb_done(h, h->ccb, CAM_REQUEUE_REQ);
        }
    }
    free_hcb_and_ccb_done(hcb, hcb->ccb, ccb_status);
    schedule_next_hcb(softc);
    return;
```

这总结了通用中断处理，尽管特定控制器可能需要一些补充。

## 12.9. 错误摘要

在执行 I/O 请求时，可能会出现许多问题。错误的原因可以在 CCB 状态中详细报告。本文档中散布了使用示例。为了完整起见，这里是典型错误条件的推荐响应摘要：

* CAM_RESRC_UNAVAIL - 某些资源暂时不可用，SIM 驱动程序无法在资源可用时生成事件。此资源的一个示例是某些控制器内部硬件资源，控制器在其可用时不生成中断。
* CAM_UNCOR_PARITY - 发生未恢复的奇偶校验错误
* CAM_DATA_RUN_ERR - 数据超限或意外数据阶段（与 CAM_DIR_MASK 中指定的方向相反）或宽传输的奇数传输长度
* CAM_SEL_TIMEOUT - 选择超时发生（目标未响应）
* 命令超时-发生了命令超时（超时函数运行）
* SCSI 状态错误-设备返回错误
* 自动感知失败-设备返回错误，请求感知命令失败
* 收到 CAM_MSG_REJECT_REC - 收到 MESSAGE REJECT 消息
* 收到 CAM_SCSI_BUS_RESET - 收到 SCSI 总线复位
* 收到 CAM_REQ_CMP_ERR - "不可能的"SCSI 阶段发生或者其他奇怪的情况或者如果没有更多细节可用，则是一般错误
* CAM_UNEXP_BUSFREE - 意外断开发生
* CAM_BDR_SENT - 总线设备复位消息已发送到目标
* CAM_UNREC_HBA_ERROR - 无法恢复的主机总线适配器错误
* CAM_REQ_TOO_BIG - 请求对于此控制器太大
* CAM_REQUEUE_REQ - 此请求应重新排队以保留事务顺序。当 SIM 识别应该冻结队列并必须将目标的其他排队请求放回 XPT 队列时，通常会发生这种情况。此类错误的典型情况包括选择超时、命令超时和其他类似情况。在这种情况下，有问题的命令返回指示错误的状态，尚未发送到总线的其他命令将被重新排队。
* CAM_LUN_INVALID - 请求中的 LUN ID 不受 SCSI 控制器支持
* CAM_TID_INVALID - 请求中的目标 ID 不受 SCSI 控制器支持

## 12.10. 超时处理

当 HCB 的超时时间到期时，该请求应该被中止，就像 XPT_ABORT 请求一样。唯一的区别是，中止请求的返回状态应该是 CAM_CMD_TIMEOUT，而不是 CAM_REQ_ABORTED（这就是为什么最好将中止的实现作为一个函数来完成）。但还有一个可能的问题：如果中止请求本身被卡住怎么办？在这种情况下，SCSI 总线应该被重置，就像 XPT_RESET_BUS 请求一样（在这两个地方都调用一个函数的想法同样适用于这里）。此外，如果设备重置请求被卡住，我们也应该重置整个 SCSI 总线。因此，超时函数最终会是这样的：

```
static void
xxx_timeout(void *arg)
{
    struct xxx_hcb *hcb = (struct xxx_hcb *)arg;
    struct xxx_softc *softc;
    struct ccb_hdr *ccb_h;

    softc = hcb->softc;
    ccb_h = &hcb->ccb->ccb_h;

    if (hcb->flags & HCB_BEING_ABORTED || ccb_h->func_code == XPT_RESET_DEV) {
        xxx_reset_bus(softc);
    } else {
        xxx_abort_ccb(hcb->ccb, CAM_CMD_TIMEOUT);
    }
}
```

当我们中止一个请求时，所有其他与同一目标/LUN 断开连接的请求也会被中止。因此，出现了一个问题，我们应该用状态 CAM_REQ_ABORTED 还是 CAM_CMD_TIMEOUT 返回它们？当前的驱动程序使用 CAM_CMD_TIMEOUT。这似乎是合乎逻辑的，因为如果一个请求超时了，那么设备可能发生了真正糟糕的事情，所以如果它们不被打扰，它们会自己超时。
