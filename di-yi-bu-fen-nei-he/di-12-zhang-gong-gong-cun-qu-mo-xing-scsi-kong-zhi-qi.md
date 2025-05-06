# 第 12 章 公共存取模型 SCSI 控制器

## 12.1. 概述

本文件假设读者对 FreeBSD 中的设备驱动程序和 SCSI 协议有一定的理解。本文中的大部分信息来源于以下驱动程序：

* ncr (**/sys/pci/ncr.c**) 作者：Wolfgang Stanglmeier 和 Stefan Esser
* sym (**/sys/dev/sym/sym\_hipd.c**) 作者：Gerard Roudier
* aic7xxx (**/sys/dev/aic7xxx/aic7xxx.c**) 作者：Justin T. Gibbs

以及来自 CAM 代码本身（由 Justin T. Gibbs 编写，见 **/sys/cam/**）。当某个解决方案看起来最为合理，并且基本上是直接从 Justin T. Gibbs 的代码中提取时，我将其标记为“推荐”。

本文使用伪代码的示例进行说明。尽管有时这些示例包含许多细节并看起来像实际代码，但它们仍然是伪代码。它们的编写目的是以易于理解的方式展示概念。对于实际的驱动程序，其他方法可能在模块化和效率上更好。它还抽象了硬件细节，以及可能会掩盖演示概念或应该在其他章节中描述的内容。这些细节通常以描述性名称的函数调用、注释或伪语句的形式呈现。幸运的是，现实中的完整示例包含了所有细节，可以在实际的驱动程序中找到。

## 12.2. 一般架构

CAM 代表通用访问方法（Common Access Method）。它是一种以类似 SCSI 的方式访问 I/O 总线的通用方法。这使得通用设备驱动程序与控制 I/O 总线的驱动程序可以分离：例如，磁盘驱动程序可以控制 SCSI、IDE 以及任何其他总线上的磁盘，因此磁盘驱动程序部分不必为每个新的 I/O 总线重写（或复制并修改）。因此，最重要的两个活跃实体是：

* *外设模块* - 外设设备的驱动程序（如磁盘、磁带、CD-ROM 等）
* *SCSI 接口模块*（SIM） - 连接到 I/O 总线（如 SCSI 或 IDE）的主机总线适配器驱动程序

外设驱动程序从操作系统接收请求，将它们转换为一系列 SCSI 命令，并将这些命令传递给 SCSI 接口模块。SCSI 接口模块负责将这些命令传递给实际的硬件（如果实际硬件不是 SCSI，而是例如 IDE，那么它还会将 SCSI 命令转换为硬件的本地命令）。

由于我们在这里关注的是编写 SCSI 适配器驱动程序，因此从这一点开始，我们将从 SIM 的角度来看待所有内容。

### 12.3. 全局变量和模板代码

一个典型的 SIM 驱动程序需要包含以下与 CAM 相关的头文件：

```c
#include <cam/cam.h>
#include <cam/cam_ccb.h>
#include <cam/cam_sim.h>
#include <cam/cam_xpt_sim.h>
#include <cam/cam_debug.h>
#include <cam/scsi/scsi_all.h>
```

### 12.4. 设备配置：xxx\_attach

每个 SIM 驱动程序首先需要将自己注册到 CAM 子系统中。这是在驱动程序的 `xxx_attach()` 函数中完成的（此处以及之后的所有 `xxx_` 用来表示唯一的驱动程序名称前缀）。`xxx_attach()` 函数本身是由系统总线自动配置代码调用的，我们在此不做详细描述。

这一过程分为多个步骤：首先需要为与此 SIM 相关的请求队列分配内存：

```c
struct cam_devq *devq;

if ((devq = cam_simq_alloc(SIZE)) == NULL) {
    error; /* 处理错误的代码 */
}
```

这里的 `SIZE` 是要分配的队列大小，即队列中可以容纳的最大请求数。它是 SIM 驱动程序在一个 SCSI 卡上可以并行处理的请求数。通常可以通过以下方式计算：

```c
SIZE = NUMBER_OF_SUPPORTED_TARGETS * MAX_SIMULTANEOUS_COMMANDS_PER_TARGET
```

接下来我们创建 SIM 的描述符：

```c
struct cam_sim *sim;

if ((sim = cam_sim_alloc(action_func, poll_func, driver_name,
        softc, unit, mtx, max_dev_transactions,
        max_tagged_dev_transactions, devq)) == NULL) {
    cam_simq_free(devq);
    error; /* 处理错误的代码 */
}
```

注意，如果无法创建 SIM 描述符，我们也会释放 `devq`，因为如果无法创建 SIM 描述符，我们就无法继续使用它，同时也希望释放内存。

如果一张 SCSI 卡上有多个 SCSI 总线，那么每条总线需要自己的 `cam_sim` 结构体。

一个有趣的问题是，如果一张 SCSI 卡有多个 SCSI 总线，是否需要每个卡一个 `devq` 结构，还是每个 SCSI 总线一个？根据 CAM 代码中的注释，答案是：两者都可以，根据驱动程序的作者的选择。

以下是相关的参数说明：

* **action\_func**：指向驱动程序的 `xxx_action` 函数的指针。

```c
static void xxx_action(struct cam_sim *, union ccb *);
```

* **poll\_func**：指向驱动程序的 `xxx_poll` 函数的指针。

```c
static void xxx_poll(struct cam_sim *);
```

* **driver\_name**：实际驱动程序的名称，例如 "ncr" 或 "wds"。

* **softc**：指向驱动程序的内部描述符的指针，代表该 SCSI 卡。该指针将用于驱动程序在未来访问私有数据。

* **unit**：控制器单元号，例如对于 "mps0" 控制器，该号为 0。

* **mtx**：与该 SIM 相关的锁。对于不涉及锁定的 SIM，可以传入 `Giant`。对于涉及锁定的 SIM，传入用于保护该 SIM 数据结构的锁。在调用 `xxx_action` 和 `xxx_poll` 时会持有该锁。

* **max\_dev\_transactions**：每个 SCSI 目标在非标记模式下允许的最大并发事务数。此值通常为 1，只有少数非 SCSI 卡可能有不同的设置。支持预处理一个事务而另一个事务正在执行的驱动程序可能将其设置为 2，但这通常不值得为此增加复杂性。

* **max\_tagged\_dev\_transactions**：同样的参数，但在标记模式下。标记是 SCSI 用来发起多事务的方式：每个事务都分配一个唯一的标记，并将事务发送到设备。当设备完成某个事务时，会将结果与标记一起发送回，以便 SCSI 适配器（和驱动程序）能够识别哪个事务已完成。这个参数也被称为最大标记深度，取决于 SCSI 适配器的能力。

最终，我们注册与 SCSI 适配器相关联的 SCSI 总线：

```c
if (xpt_bus_register(sim, softc, bus_number) != CAM_SUCCESS) {
    cam_sim_free(sim, /*free_devq*/ TRUE);
    error; /* 处理错误的代码 */
}
```

如果每个 SCSI 总线有一个 `devq` 结构（即我们认为一张卡上有多个总线就像有多个每个只有一个总线的卡），那么总线编号将始终为 0；否则，每个 SCSI 卡上的每个总线应当分配一个不同的编号。每个总线需要独立的 `cam_sim` 结构。

完成这些步骤后，我们的控制器就完全连接到了 CAM 系统。`devq` 的值可以被丢弃，因为从现在开始，`sim` 会作为所有后续 CAM 调用的参数传递，而 `devq` 可以从中派生出来。

CAM 提供了框架来处理这样的异步事件。有些事件来自低层（SIM 驱动程序），有些事件来自外设驱动程序，还有一些事件来自 CAM 子系统本身。任何驱动程序都可以注册回调函数来处理某些类型的异步事件，从而在这些事件发生时得到通知。

一个典型的事件例子是设备重置。每个事务和事件通过“路径”来标识其应用的设备。特定目标的事件通常发生在与该设备的事务期间。因此，可以重用来自事务的路径来报告该事件（这是安全的，因为事件路径在事件报告过程中被复制，但不会被释放或传递到其他地方）。同样，在任何时候，包括中断例程中，动态分配路径也是安全的，尽管这会带来一定的开销，而这个方法的潜在问题是可能在此时没有足够的内存。对于总线重置事件，我们需要定义一个包含总线上所有设备的通配符路径。因此，我们可以提前创建路径，以便为未来的总线重置事件使用，从而避免内存短缺的问题：

```c
struct cam_path *path;

if (xpt_create_path(&path, /*periph*/NULL,
            cam_sim_path(sim), CAM_TARGET_WILDCARD,
            CAM_LUN_WILDCARD) != CAM_REQ_CMP) {
    xpt_bus_deregister(cam_sim_path(sim));
    cam_sim_free(sim, /*free_devq*/TRUE);
    error; /* 处理错误的代码 */
}

softc->wpath = path;
softc->sim = sim;
```

如你所见，路径包括：

* 外设驱动程序的 ID（这里是 NULL，因为我们没有外设）
* SIM 驱动程序的 ID（`cam_sim_path(sim)`）
* 设备的 SCSI 目标号（`CAM_TARGET_WILDCARD` 表示“所有设备”）
* 子设备的 SCSI LUN 号（`CAM_LUN_WILDCARD` 表示“所有 LUN”）

如果驱动程序无法分配此路径，则无法正常工作，因此在这种情况下，我们会拆卸该 SCSI 总线。

然后，我们将路径指针保存在 `softc` 结构中以供未来使用。之后，我们保存 `sim` 的值（或者在 `xxx_probe()` 退出时也可以丢弃它，如果我们愿意的话）。

这就是最简初始化的全部内容。但为了做好这些工作，还有一个问题需要解决。

对于 SIM 驱动程序来说，有一个特别有趣的事件：当目标设备被认为丢失时。此时，重置与该设备的 SCSI 协商可能是一个好主意。因此，我们需要为此事件在 CAM 中注册一个回调。请求通过在 CAM 控制块上请求 CAM 操作来传递给 CAM，该控制块处理此类型的请求：

```c
struct ccb_setasync csa;

xpt_setup_ccb(&csa.ccb_h, path, /*priority*/5);
csa.ccb_h.func_code = XPT_SASYNC_CB;
csa.event_enable = AC_LOST_DEVICE;
csa.callback = xxx_async;
csa.callback_arg = sim;
xpt_action((union ccb *)&csa);
```

## 12.5. 处理 CAM 消息：xxx\_action

```c
static void xxx_action(struct cam_sim *sim, union ccb *ccb);
```

根据 CAM 子系统的请求执行某些操作。`sim` 描述了请求的 SIM，`ccb` 是请求本身。CCB 代表“CAM 控制块”（CAM Control Block）。它是一个联合体，包含多种具体实例，每个实例描述某种类型的事务的参数。所有这些实例共享一个 CCB 头部，其中存储了公共参数的部分。

CAM 支持工作在发起者（“正常”）模式和目标（模拟 SCSI 设备）模式下的 SCSI 控制器。这里我们只讨论与发起者模式相关的部分。

有一些函数和宏（换句话说，是方法）被定义用来访问 `sim` 结构中的公共数据：

* `cam_sim_path(sim)` - 路径 ID（如上所述）
* `cam_sim_name(sim)` - SIM 的名称
* `cam_sim_softc(sim)` - 指向 softc（驱动程序私有数据）结构的指针
* `cam_sim_unit(sim)` - 单元号
* `cam_sim_bus(sim)` - 总线 ID

为了识别设备，`xxx_action()` 可以使用这些函数获取单元号和指向其 softc 结构的指针。

请求的类型存储在 `ccb→ccb_h.func_code` 中。因此，通常 `xxx_action()` 包含一个大的 `switch` 语句：

```c
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

如 `default` 案例所示（如果收到未知命令），命令的返回代码会被设置到 `ccb→ccb_h.status` 中，处理完的 CCB 会通过调用 `xpt_done(ccb)` 返回给 CAM。

`xpt_done()` 不一定要在 `xxx_action()` 中调用：例如，一个 I/O 请求可能会被放入 SIM 驱动程序和/或其 SCSI 控制器的队列中。然后，当设备通过中断信号表明请求处理完成时，`xpt_done()` 可能会在中断处理程序中被调用。

实际上，CCB 状态不仅仅在返回代码中分配，CCB 始终具有一些状态。在 CCB 被传递给 `xxx_action()` 例程之前，它的状态会被设置为 `CCB_REQ_INPROG`，表示它正在处理中。**/sys/cam/cam.h** 中定义了大量状态值，可以详细表示请求的状态。更有趣的是，状态实际上是“按位或”了一个枚举状态值（低 6 位）和可能的附加标志位（高位）。枚举值会在后面详细讨论。它们的汇总可以在错误总结部分找到。可能的状态标志包括：

* **CAM\_DEV\_QFRZN** - 如果 SIM 驱动程序在处理 CCB 时遇到严重错误（例如，设备没有响应选择或违反了 SCSI 协议），它应该通过调用 `xpt_freeze_simq()` 冻结请求队列，将其他尚未处理的 CCB 返回 CAM 队列，然后为该问题 CCB 设置此标志，并调用 `xpt_done()`。该标志会导致 CAM 子系统在处理完错误后解冻队列。
* **CAM\_AUTOSNS\_VALID** - 如果设备返回了错误状态且 `CAM_DIS_AUTOSENSE` 标志未设置，SIM 驱动程序必须自动执行 `REQUEST SENSE` 命令，以从设备中提取感知（扩展错误信息）数据。如果此操作成功，感知数据应保存到 CCB 中，并设置此标志。
* **CAM\_RELEASE\_SIMQ** - 类似于 **CAM\_DEV\_QFRZN**，但用于在 SCSI 控制器本身出现问题（或资源短缺）时。此时，所有未来对该控制器的请求应通过 `xpt_freeze_simq()` 停止。控制器队列将在 SIM 驱动程序解决短缺并通过返回带有该标志的 CCB 通知 CAM 后恢复。
* **CAM\_SIM\_QUEUED** - 当 SIM 将 CCB 放入其请求队列时，应设置此标志（当该 CCB 在被返回 CAM 之前出队时移除此标志）。目前，CAM 代码中并未使用此标志，因此其目的是纯粹的诊断用途。
* **CAM\_QOS\_VALID** - QOS 数据现在有效。

`xxx_action()` 不允许睡眠，因此所有资源访问的同步必须通过冻结 SIM 或设备队列来完成。除了上述标志，CAM 子系统还提供了 `xpt_release_simq()` 和 `xpt_release_devq()` 函数，用于直接解冻队列，而无需传递 CCB 给 CAM。

CCB 头部包含以下字段：

* **path** - 请求的路径 ID
* **target\_id** - 请求的目标设备 ID
* **target\_lun** - 目标设备的 LUN ID
* **timeout** - 此命令的超时间隔（毫秒）
* **timeout\_ch** - SIM 驱动程序用来存储超时句柄的便利字段（CAM 子系统本身不会对其做出任何假设）
* **flags** - 请求的各种标志位
* **spriv\_ptr0, spriv\_ptr1** - 供 SIM 驱动程序私用的字段（如链接到 SIM 队列或 SIM 私有控制块）；实际上，这些字段是联合体：`spriv_ptr0` 和 `spriv_ptr1` 的类型是 `void *`，`spriv_field0` 和 `spriv_field1` 的类型是 `unsigned long`，`sim_priv.entries[0].bytes` 和 `sim_priv.entries[1].bytes` 是与联合体的其他版本大小一致的字节数组，`sim_priv.bytes` 是一个更大的字节数组。

推荐的使用 SIM 私有字段的方式是为它们定义一些有意义的名称，并在驱动程序中使用这些名称，例如：

```c
#define ccb_some_meaningful_name    sim_priv.entries[0].bytes
#define ccb_hcb spriv_ptr1 /* 用于硬件控制块 */
```

最常见的发起者模式请求有：

### 12.5.1. *XPT\_SCSI\_IO* - 执行 I/O 事务

联合体 `ccb` 中的实例 `struct ccb_scsiio csio` 用于传递参数。它们包括：

* **cdb\_io** - 指向 SCSI 命令缓冲区的指针，或者是缓冲区本身
* **cdb\_len** - SCSI 命令的长度
* **data\_ptr** - 指向数据缓冲区的指针（如果使用了 scatter/gather，情况会更复杂）
* **dxfer\_len** - 要传输的数据长度
* **sglist\_cnt** - scatter/gather 段的计数器
* **scsi\_status** - 返回 SCSI 状态的地方
* **sense\_data** - 如果命令返回错误，SCSI 感知信息的缓冲区（如果 CCB 标志 `CAM_DIS_AUTOSENSE` 没有设置，SIM 驱动程序应自动执行 REQUEST SENSE 命令）
* **sense\_len** - 该缓冲区的长度（如果其大小大于 `sense_data` 的大小，SIM 驱动程序必须默认为较小的值）
* **resid**, **sense\_resid** - 如果数据或 SCSI 感知的传输出现错误，这些是返回的残余（未传输的）数据计数器。它们似乎没有特别的意义，因此在计算困难时（例如，计算 SCSI 控制器 FIFO 缓冲区中的字节数）可以使用一个近似值。对于成功完成的传输，它们必须设置为零。
* **tag\_action** - 要使用的标签类型：

  * `CAM_TAG_ACTION_NONE` - 不为此事务使用标签
  * `MSG_SIMPLE_Q_TAG`, `MSG_HEAD_OF_Q_TAG`, `MSG_ORDERED_Q_TAG` - 与适当标签消息相等的值（参见 `/sys/cam/scsi/scsi_message.h`）；这仅给出了标签类型，SIM 驱动程序必须自己分配标签值

处理此请求的一般逻辑如下：

首先要做的是检查可能的竞态，确保命令在排队时没有被中止：

```c
struct ccb_scsiio *csio = &ccb->csio;

if ((ccb_h->status & CAM_STATUS_MASK) != CAM_REQ_INPROG) {
    xpt_done(ccb);
    return;
}
```

接着检查设备是否被我们的控制器支持：

```c
if (ccb_h->target_id > OUR_MAX_SUPPORTED_TARGET_ID
    || ccb_h->target_id == OUR_SCSI_CONTROLLERS_OWN_ID) {
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

然后分配我们需要的任何数据结构（例如，卡依赖的硬件控制块）来处理此请求。如果无法分配，则冻结 SIM 队列并记住我们有一个挂起的操作，将 CCB 返回并请求 CAM 重新排队它。稍后，当资源可用时，SIM 队列必须通过返回一个带有 `CAM_SIMQ_RELEASE` 位设置的 CCB 来解冻。否则，如果一切顺利，将 CCB 与硬件控制块（HCB）关联并标记为已排队。

```c
struct xxx_hcb *hcb = allocate_hcb(softc, unit, bus);

if (hcb == NULL) {
    softc->flags |= RESOURCE_SHORTAGE;
    xpt_freeze_simq(sim, /*count*/1);
    ccb_h->status = CAM_REQUEUE_REQ;
    xpt_done(ccb);
    return;
}

hcb->ccb = ccb;
ccb_h->ccb_hcb = (void *)hcb;
ccb_h->status |= CAM_SIM_QUEUED;
```

从 CCB 提取目标数据到硬件控制块中。检查是否要求分配标签，如果是，则生成一个唯一标签并构建 SCSI 标签消息。SIM 驱动程序还负责与设备进行协商，以设置最大互相支持的总线宽度、同步速率和偏移量。

```c
hcb->target = ccb_h->target_id;
hcb->lun = ccb_h->target_lun;
generate_identify_message(hcb);
if (ccb_h->tag_action != CAM_TAG_ACTION_NONE)
    generate_unique_tag_message(hcb, ccb_h->tag_action);
if (!target_negotiated(hcb))
    generate_negotiation_messages(hcb);
```

接下来设置 SCSI 命令。命令存储可能通过 CCB 中的各种方式指定，这些方式由 CCB 标志指定。命令缓冲区可以包含在 CCB 中或通过指针指向，在后一种情况下，指针可以是物理或虚拟的。由于硬件通常需要物理地址，因此我们总是将地址转换为物理地址，通常使用 busdma API。

### 12.5.1. *XPT\_SCSI\_IO* - 执行 I/O 事务 (继续)

如果请求的是物理地址，返回带有状态 `CAM_REQ_INVALID` 的 CCB 是可以接受的，当前的驱动程序就是这么做的。如果需要，也可以将物理地址转换回虚拟地址，但这样做相当麻烦，因此我们通常不这么做。

```c
if (ccb_h->flags & CAM_CDB_POINTER) {
    /* CDB 是指针 */
    if (!(ccb_h->flags & CAM_CDB_PHYS)) {
        /* CDB 指针是虚拟的 */
        hcb->cmd = vtobus(csio->cdb_io.cdb_ptr);
    } else {
        /* CDB 指针是物理的 */
        hcb->cmd = csio->cdb_io.cdb_ptr;
    }
} else {
    /* CDB 在 ccb 中（缓冲区） */
    hcb->cmd = vtobus(csio->cdb_io.cdb_bytes);
}
hcb->cmdlen = csio->cdb_len;
```

现在是时候设置数据了。同样，数据存储可以通过 CCB 中的各种方式指定，这些方式由 CCB 标志指定。首先获取数据传输的方向。最简单的情况是，如果没有数据要传输：

```c
int dir = (ccb_h->flags & CAM_DIR_MASK);

if (dir == CAM_DIR_NONE)
    goto end_data;
```

接下来检查数据是一个块还是 scatter-gather 列表，地址是物理的还是虚拟的。SCSI 控制器可能只能处理有限数量的块，且块的长度也有限。如果请求超过了这一限制，我们就返回一个错误。我们使用一个特殊的函数来返回 CCB，在一个地方处理 HCB 资源短缺。添加块的函数是驱动程序相关的，这里我们不提供详细的实现。有关地址转换问题的更多信息，请参考 SCSI 命令（CDB）处理部分。如果某些变体在特定卡上实现困难或不可能实现，返回 `CAM_REQ_INVALID` 状态是可以接受的。实际上，现在似乎在 CAM 代码中没有使用 scatter-gather 功能。但至少必须实现针对单个非散布虚拟缓冲区的情况，因为它在 CAM 中被广泛使用。

```c
int rv;

initialize_hcb_for_data(hcb);

if (!(ccb_h->flags & CAM_SCATTER_VALID)) {
    /* 单一缓冲区 */
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
        /* SG 列表指针是物理的 */
        rv = setup_hcb_for_physical_sg_list(hcb, segs, csio->sglist_cnt);
    } else if (!(ccb_h->flags & CAM_DATA_PHYS)) {
        /* SG 缓冲区指针是虚拟的 */
        for (i = 0; i < csio->sglist_cnt; i++) {
            rv = add_virtual_chunk(hcb, segs[i].ds_addr,
                segs[i].ds_len, dir);
            if (rv != CAM_REQ_CMP)
                break;
        }
    } else {
        /* SG 缓冲区指针是物理的 */
        for (i = 0; i < csio->sglist_cnt; i++) {
            rv = add_physical_chunk(hcb, segs[i].ds_addr,
                segs[i].ds_len, dir);
            if (rv != CAM_REQ_CMP)
                break;
        }
    }
}

if (rv != CAM_REQ_CMP) {
    /* 我们期望 add_*_chunk() 函数如果成功添加块则返回 CAM_REQ_CMP，
     * 如果请求太大（太多字节或太多块），则返回 CAM_REQ_TOO_BIG，
     * 如果发生其他问题，则返回 CAM_REQ_INVALID
     */
    free_hcb_and_ccb_done(hcb, ccb, rv);
    return;
}

end_data:
```

这段代码描述了如何根据 CCB 标志设置数据的传输模式，并处理各种情况，包括单一数据块、scatter-gather 列表以及虚拟/物理地址的处理。如果处理过程中出现错误，则返回相应的状态，并通过 `free_hcb_and_ccb_done` 函数释放资源。

### 12.5.1. *XPT\_SCSI\_IO* - 执行 I/O 事务 (继续)

如果此 CCB 禁用了断开连接，我们将此信息传递给 HCB：

```c
if (ccb_h->flags & CAM_DIS_DISCONNECT)
    hcb_disable_disconnect(hcb);
```

如果控制器能够独立运行 `REQUEST SENSE` 命令，则应该将 `CAM_DIS_AUTOSENSE` 标志的值传递给它，以防 CAM 子系统不希望自动执行 `REQUEST SENSE`。

最后，我们需要设置超时，将 HCB 传递给硬件，并返回，剩下的工作将在中断处理程序（或超时处理程序）中完成：

```c
ccb_h->timeout_ch = timeout(xxx_timeout, (caddr_t) hcb,
    (ccb_h->timeout * hz) / 1000); /* 将毫秒转换为时钟滴答 */
put_hcb_into_hardware_queue(hcb);
return;
```

下面是返回 CCB 的可能实现：

```c
static void
free_hcb_and_ccb_done(struct xxx_hcb *hcb, union ccb *ccb, u_int32_t status)
{
    struct xxx_softc *softc = hcb->softc;

    ccb->ccb_h.ccb_hcb = 0;
    if (hcb != NULL) {
        untimeout(xxx_timeout, (caddr_t) hcb, ccb->ccb_h.timeout_ch);
        /* 我们即将释放一个 HCB，因此资源短缺已经结束 */
        if (softc->flags & RESOURCE_SHORTAGE)  {
            softc->flags &= ~RESOURCE_SHORTAGE;
            status |= CAM_RELEASE_SIMQ;
        }
        free_hcb(hcb); /* 也会将 HCB 从任何内部列表中移除 */
    }
    ccb->ccb_h.status = status |
        (ccb->ccb_h.status & ~(CAM_STATUS_MASK|CAM_SIM_QUEUED));
    xpt_done(ccb);
}
```

### 12.5.2. *XPT\_RESET\_DEV* - 向设备发送 SCSI "BUS DEVICE RESET" 消息

此 CCB 中没有数据传输，只有头部，最有趣的参数是 `target_id`。根据控制器硬件的不同，可能会像 `XPT_SCSI_IO` 请求一样构建一个硬件控制块（见 `XPT_SCSI_IO` 请求描述）并发送到控制器，或者 SCSI 控制器可能会立即编程以向设备发送此 `RESET` 消息，或者此请求可能根本不被支持（并返回状态 `CAM_REQ_INVALID`）。此外，在请求完成时，必须终止该目标的所有断开连接事务（可能在中断例程中完成）。

另外，所有当前的协商都将丢失，因此它们可能也需要被清除。或者清除过程可以延迟，因为无论如何目标将在下一次事务中请求重新协商。

### 12.5.3. *XPT\_RESET\_BUS* - 向 SCSI 总线发送 RESET 信号

此 CCB 中没有传递任何参数，唯一有趣的参数是由 `struct sim` 指针指示的 SCSI 总线。

最简化的实现会忽略该总线所有设备的 SCSI 协商，并返回状态 `CAM_REQ_CMP`。

正确的实现应该除了实际重置 SCSI 总线（可能还需要重置 SCSI 控制器）外，还标记所有正在处理的 CCB，无论是硬件队列中的还是断开连接的，都标记为完成并设置状态 `CAM_SCSI_BUS_RESET`。示例如下：

```c
void
reset_scsibus(struct sim *sim)
{
    struct ccb *ccb;
    int i;

    /* 重置 SCSI 总线（可能重置 SCSI 控制器） */
    reset_scsibus_hardware(sim);

    /* 标记所有与该总线相关的 CCB 完成 */
    for (i = 0; i < sim->ccb_count; i++) {
        ccb = sim->ccb_queue[i];
        if (ccb) {
            ccb->ccb_h.status = CAM_SCSI_BUS_RESET;
            xpt_done(ccb);
        }
    }
}
```

在这个实现中，`reset_scsibus_hardware` 负责执行硬件层面的重置操作，`xpt_done` 则标记每个 CCB 为完成，并传递状态 `CAM_SCSI_BUS_RESET`。

### 12.5.4. *XPT\_ABORT* - 中止指定的 CCB

在实现 SCSI 总线重置时，将其作为函数实现可能是个好主意，因为如果事情出现问题，超时处理程序可以作为最后的手段重新使用该函数。

首先，参数被传输到 `union ccb` 中的实例 `struct ccb_abort cab`。其中唯一的参数字段是：

* `abort_ccb` - 指向要中止的 CCB 的指针。

如果不支持中止，直接返回状态 `CAM_UA_ABORT`。这也是最简单的方式，在任何情况下都返回 `CAM_UA_ABORT`。

更复杂的方法是诚实地实现此请求。首先需要检查中止是否适用于 SCSI 事务：

```c
struct ccb *abort_ccb;
abort_ccb = ccb->cab.abort_ccb;

if (abort_ccb->ccb_h.func_code != XPT_SCSI_IO) {
    ccb->ccb_h.status = CAM_UA_ABORT;
    xpt_done(ccb);
    return;
}
```

然后，必须在队列中找到该 CCB。这可以通过遍历我们所有硬件控制块的列表来实现，以查找与此 CCB 关联的一个：

```c
struct xxx_hcb *hcb, *h;

hcb = NULL;

/* 假设 softc->first_hcb 是与该总线关联的所有 HCB 的头，包括那些正在排队等待处理的、正在硬件处理中以及已断开连接的 */
for (h = softc->first_hcb; h != NULL; h = h->next) {
    if (h->ccb == abort_ccb) {
        hcb = h;
        break;
    }
}

if (hcb == NULL) {
    /* 队列中没有此 CCB */
    ccb->ccb_h.status = CAM_PATH_INVALID;
    xpt_done(ccb);
    return;
}

hcb = found_hcb;
```

现在我们查看 HCB 当前的处理状态。它可能处于以下几种状态之一：

* 正在排队等待发送到 SCSI 总线，
* 当前正在传输，
* 断开连接并等待命令的结果，
* 或者已经由硬件完成，但尚未由软件标记为完成。

为了确保我们不会与硬件发生竞争条件，我们将 HCB 标记为正在中止，这样如果该 HCB 即将发送到 SCSI 总线，SCSI 控制器就会看到这个标志并跳过它：

```c
/* 标记 HCB 为正在中止 */
hcb->status |= HCB_ABORTED;
```

接下来，取消当前的事务，确保中止请求的正确处理。如果 HCB 处于处理中状态，我们需要相应地通知硬件进行处理。

```c
int hstatus;

/* 以函数形式展示，以防需要特殊操作使得此标志对硬件可见 */
set_hcb_flags(hcb, HCB_BEING_ABORTED);

abort_again:

hstatus = get_hcb_status(hcb);
switch (hstatus) {
case HCB_SITTING_IN_QUEUE:
    remove_hcb_from_hardware_queue(hcb);
    /* 继续执行 */
case HCB_COMPLETED:
    /* 这是一个简单的情况 */
    free_hcb_and_ccb_done(hcb, abort_ccb, CAM_REQ_ABORTED);
    break;
```

如果 CCB 当前正在传输中，我们希望通过某种硬件相关的方式向 SCSI 控制器发出中止当前传输的信号。SCSI 控制器会设置 SCSI ATTENTION 信号，并在目标响应时发送 ABORT 消息。同时，我们会重置超时，以确保目标不会一直处于休眠状态。如果命令在合理的时间内（例如 10 秒）没有被中止，超时例程会重置整个 SCSI 总线。由于命令会在合理的时间内被中止，我们可以立即返回中止请求，并将被中止的 CCB 标记为已中止（但不标记为完成）。

```c
case HCB_BEING_TRANSFERRED:
    untimeout(xxx_timeout, (caddr_t) hcb, abort_ccb->ccb_h.timeout_ch);
    abort_ccb->ccb_h.timeout_ch =
        timeout(xxx_timeout, (caddr_t) hcb, 10 * hz);
    abort_ccb->ccb_h.status = CAM_REQ_ABORTED;
    /* 请求控制器中止该 HCB，然后生成中断并停止 */
    if (signal_hardware_to_abort_hcb_and_stop(hcb) < 0) {
        /* 哎呀，我们错过了与硬件的竞争，这个事务已经离开了总线 */
        /* 在我们中止它之前，尝试再次操作 */
        goto abort_again;
    }

    break;
```

如果 CCB 在断开连接的列表中，则将其设置为中止请求，并将其重新排队到硬件队列的前端。重置超时并报告中止请求已完成。

```c
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

这就是 ABORT 请求的全部内容，尽管还有一个问题。由于 ABORT 消息会清除 LUN 上所有正在进行的事务，我们需要将该 LUN 上的其他所有活动事务标记为已中止。这个操作应该在中断例程中进行，在事务被中止之后。

将 CCB 中止实现为一个函数是一个很好的主意，因为在 I/O 事务超时时可以重复使用此函数。唯一的区别是，超时的事务会返回状态 CAM\_CMD\_TIMEOUT，而不是 CAM\_REQ\_ABORTED。然后，XPT\_ABORT 的处理就像这样：

```c
case XPT_ABORT:
    struct ccb *abort_ccb;
    abort_ccb = ccb->cab.abort_ccb;

    if (abort_ccb->ccb_h.func_code != XPT_SCSI_IO) {
        ccb->ccb_h.status = CAM_UA_ABORT;
        xpt_done(ccb);
        return;
    }
    if (xxx_abort_ccb(abort_ccb, CAM_REQ_ABORTED) < 0)
        /* 没有找到该 CCB 在我们的队列中 */
        ccb->ccb_h.status = CAM_PATH_INVALID;
    else
        ccb->ccb_h.status = CAM_REQ_CMP;
    xpt_done(ccb);
    return;
```

### 12.5.5. *XPT\_SET\_TRAN\_SETTINGS* - 显式设置 SCSI 传输设置的值

参数通过联合体 `ccb` 的实例 `struct ccb_trans_setting cts` 传递：

* **valid** - 一个位掩码，显示哪些设置应该更新：

  * **CCB\_TRANS\_SYNC\_RATE\_VALID** - 同步传输速率
  * **CCB\_TRANS\_SYNC\_OFFSET\_VALID** - 同步偏移
  * **CCB\_TRANS\_BUS\_WIDTH\_VALID** - 总线宽度
  * **CCB\_TRANS\_DISC\_VALID** - 设置启用/禁用断开连接
  * **CCB\_TRANS\_TQ\_VALID** - 设置启用/禁用标记排队
* **flags** - 由两部分组成，二进制参数和子操作的标识。二进制参数包括：

  * **CCB\_TRANS\_DISC\_ENB** - 启用断开连接
  * **CCB\_TRANS\_TAG\_ENB** - 启用标记排队
* 子操作包括：

  * **CCB\_TRANS\_CURRENT\_SETTINGS** - 更改当前的协商设置
  * **CCB\_TRANS\_USER\_SETTINGS** - 记住所需的用户值，如 `sync_period`, `sync_offset` - 不言自明，如果 `sync_offset == 0` 则请求异步模式，`bus_width` - 总线宽度，以位为单位（不是字节）

支持两组协商的参数，分别是用户设置和当前设置。用户设置在 SIM 驱动程序中并未真正使用，通常只是一个内存区域，上层可以在其中存储（并在后续调用时回忆）其对参数的看法。设置用户参数不会导致重新协商传输速率。但是，当 SCSI 控制器进行协商时，必须始终将值设置为不大于用户设置的值，因此它本质上是上限。

当前设置正如其名称所示，是实际生效的设置。更改它们意味着下次传输时必须重新协商这些参数。同样，这些“新的当前设置”不应强制应用于设备，它们只是协商的初始步骤。它们还必须受到 SCSI 控制器实际能力的限制：例如，如果 SCSI 控制器具有 8 位总线，而请求要求设置 16 位宽的传输，则在将其发送给设备之前，此参数必须静默地截断为 8 位传输。

需要注意的是，总线宽度和同步参数是按目标 (target) 设置的，而断开连接和标记启用参数是按 LUN 设置的。

推荐的实现方法是保留三组协商的（总线宽度和同步传输）参数：

* **user** - 用户设置，如上所述
* **current** - 当前实际生效的设置
* **goal** - 通过设置“当前”参数请求的设置

代码示例：

```c
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

然后，在处理下一个 I/O 请求时，它将检查是否需要重新协商，例如通过调用函数 `target_negotiated(hcb)`。它可以这样实现：

```c
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

在重新协商值之后，结果值必须分配给当前和目标参数，因此对于未来的 I/O 事务，当前和目标参数将相同，并且 `target_negotiated()` 将返回 TRUE。当卡片初始化时（在 `xxx_attach()` 中），当前协商值必须初始化为狭窄的异步模式，目标和当前值必须初始化为控制器支持的最大值。

### 12.5.6. *XPT\_GET\_TRAN\_SETTINGS* - 获取 SCSI 传输设置的值

此操作是 XPT\_SET\_TRAN\_SETTINGS 的反向操作。根据 CCB\_TRANS\_CURRENT\_SETTINGS 或 CCB\_TRANS\_USER\_SETTINGS 中的标志，填充 CCB 实例 "struct ccb\_trans\_setting cts"（如果两个标志都设置，则现有的驱动程序返回当前设置）。设置有效字段中的所有位。

### 12.5.7. *XPT\_CALC\_GEOMETRY* - 计算磁盘的逻辑（BIOS）几何结构

参数通过联合体 `ccb` 的实例 "struct ccb\_calc\_geometry ccg" 传递：

* **block\_size** - 输入，块（即扇区）大小，单位字节
* **volume\_size** - 输入，卷大小，单位字节
* **cylinders** - 输出，逻辑柱面数
* **heads** - 输出，逻辑磁头数
* **secs\_per\_track** - 输出，每磁道逻辑扇区数

如果返回的几何结构与 SCSI 控制器 BIOS 认为的几何结构差异过大，且此 SCSI 控制器上的磁盘被用作可启动盘，系统可能无法启动。以下是来自 aic7xxx 驱动程序的典型计算示例：

```c
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

这给出了大致的思路，确切的计算取决于特定 BIOS 的细节。如果 BIOS 无法设置 EEPROM 中的“扩展翻译”标志，则此标志通常应假定为 1。其他流行的几何结构包括：

```
128 磁头，63 扇区 - Symbios 控制器
16 磁头，63 扇区 - 旧控制器
```

一些系统 BIOS 和 SCSI BIOS 彼此之间有时会发生冲突，例如，Symbios 875/895 SCSI 与 Phoenix BIOS 的组合可能会在上电后给出 128/63 的几何结构，而在硬重启或软重启后给出 255/63 的几何结构。

### 12.5.8. *XPT\_PATH\_INQ* - 路径查询，换句话说，获取 SIM 驱动程序和 SCSI 控制器（也称为 HBA - 主机总线适配器）的属性

这些属性通过联合体 `ccb` 中的实例 "struct ccb\_pathinq cpi" 返回：

* **version\_num** - SIM 驱动程序的版本号，目前所有驱动程序使用 1
* **hba\_inquiry** - 控制器支持的特性位掩码：

  * PI\_MDP\_ABLE - 支持 MDP 消息（来自 SCSI3？）
  * PI\_WIDE\_32 - 支持 32 位宽 SCSI
  * PI\_WIDE\_16 - 支持 16 位宽 SCSI
  * PI\_SDTR\_ABLE - 可以协商同步传输速率
  * PI\_LINKED\_CDB - 支持链式命令
  * PI\_TAG\_ABLE - 支持标记命令
  * PI\_SOFT\_RST - 支持软复位替代方式（硬复位和软复位在 SCSI 总线上是互斥的）
* **target\_sprt** - 目标模式支持的标志，如果不支持则为 0
* **hba\_misc** - 控制器的其他特性：

  * PIM\_SCANHILO - 总线从高 ID 扫描到低 ID
  * PIM\_NOREMOVE - 扫描中不包括可拆卸设备
  * PIM\_NOINITIATOR - 不支持启动器角色
  * PIM\_NOBUSRESET - 用户已禁用初始总线复位
* **hba\_eng\_cnt** - 神秘的 HBA 引擎计数，可能与压缩相关，目前始终设置为 0
* **vuhba\_flags** - 唯一厂商标志，目前未使用
* **max\_target** - 最大支持的目标 ID（对于 8 位总线为 7，对于 16 位总线为 15，对于光纤通道为 127）
* **max\_lun** - 最大支持的 LUN ID（对于较旧的 SCSI 控制器为 7，对于较新的为 63）
* **async\_flags** - 已安装的异步处理程序的位掩码，目前未使用
* **hpath\_id** - 子系统中的最高路径 ID，目前未使用
* **unit\_number** - 控制器单元编号，cam\_sim\_unit(sim)
* **bus\_id** - 总线编号，cam\_sim\_bus(sim)
* **initiator\_id** - 控制器本身的 SCSI ID
* **base\_transfer\_speed** - 异步窄传输的名义传输速率，单位 KB/s，对于 SCSI 等于 3300
* **sim\_vid** - SIM 驱动程序的厂商 ID，最大长度为 SIM\_IDLEN 的零终止字符串，包括终止符
* **hba\_vid** - SCSI 控制器的厂商 ID，最大长度为 HBA\_IDLEN 的零终止字符串，包括终止符
* **dev\_name** - 设备驱动程序名称，最大长度为 DEV\_IDLEN 的零终止字符串，包括终止符，等于 cam\_sim\_name(sim)

设置字符串字段的推荐方法是使用 `strncpy`，例如：

```c
strncpy(cpi->dev_name, cam_sim_name(sim), DEV_IDLEN);
```

设置完值后，将状态设置为 `CAM_REQ_CMP`，并标记 CCB 为完成。

## 12.6. Polling xxx\_poll

```c
static void xxx_poll(struct cam_sim *);
```

`poll` 函数用于在中断子系统不可用时模拟中断（例如，当系统崩溃并创建系统转储时）。CAM 子系统在调用 `poll` 例程之前会设置适当的中断级别。因此，它只需要做的是调用中断例程（或者相反，`poll` 例程可能执行实际操作，而中断例程只会调用 `poll` 例程）。那么为什么要使用一个单独的函数呢？这是因为调用约定不同。`xxx_poll` 例程获取 `struct cam_sim` 指针作为其参数，而 PCI 中断例程根据惯例获取 `struct xxx_softc` 的指针，而 ISA 中断例程仅获取设备单元编号。因此，`poll` 例程通常看起来像这样：

```c
static void
xxx_poll(struct cam_sim *sim)
{
    xxx_intr((struct xxx_softc *)cam_sim_softc(sim)); /* 对于 PCI 设备 */
}
```

或者

```c
static void
xxx_poll(struct cam_sim *sim)
{
    xxx_intr(cam_sim_unit(sim)); /* 对于 ISA 设备 */
}
```

## 12.7. 异步事件

如果已设置异步事件回调，则应定义回调函数。

```c
static void
ahc_async(void *callback_arg, u_int32_t code, struct cam_path *path, void *arg)
```

* **callback\_arg** - 在注册回调时提供的值
* **code** - 标识事件的类型
* **path** - 标识事件适用的设备
* **arg** - 与事件相关的参数

单个事件类型 `AC_LOST_DEVICE` 的实现如下所示：

```c
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
            /* 向 CAM 发送指示 */
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

中断例程的确切类型取决于连接到 SCSI 控制器的外设总线类型（PCI、ISA 等）。

SIM 驱动程序的中断例程在中断级别 `splcam` 上运行。因此，驱动程序中应使用 `splcam()` 来同步中断例程与驱动程序其他部分的活动（对于支持多处理器的驱动程序，情况会更复杂，但在此我们忽略这种情况）。本文档中的伪代码忽略了同步问题。实际代码必须不忽略这些问题。一个简单的做法是在进入其他例程时设置 `splcam()`，并在返回时重置它，从而通过一个大的临界区来保护它们。为了确保中断级别始终恢复，可以定义一个包装函数，例如：

```c
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
        ... 处理请求 ...
    }
```

这种方法简单且健壮，但它的问题是中断可能会被阻塞较长时间，这会对系统性能产生负面影响。另一方面，`spl()` 家族的函数具有相当高的开销，因此大量的小临界区也可能不太合适。

中断例程处理的条件和细节在很大程度上取决于硬件。我们考虑“典型”条件的集合。

首先，我们检查总线上是否遇到了 SCSI 复位（可能是由同一 SCSI 总线上的另一个 SCSI 控制器引起的）。如果是这样，我们会丢弃所有排队的和断开的请求，报告事件，并重新初始化我们的 SCSI 控制器。重要的是，在此初始化过程中，控制器不能发出另一个复位，否则同一 SCSI 总线上的两个控制器可能会相互“反复”复位。致命的控制器错误/挂起的情况可以在同一位置处理，但它可能还需要向 SCSI 总线发送 RESET 信号，以重置与 SCSI 设备连接的状态。

```c
int fatal=0;
    struct ccb_trans_settings neg;
    struct cam_path *path;

    if (detected_scsi_reset(softc)
    || (fatal = detected_fatal_controller_error(softc))) {
        int targ, lun;
        struct xxx_hcb *h, *hh;

        /* 丢弃所有排队的 CCB */
        for(h = softc->first_queued_hcb; h != NULL; h = hh) {
            hh = h->next;
            free_hcb_and_ccb_done(h, h->ccb, CAM_SCSI_BUS_RESET);
        }

        /* 报告的干净的协商值 */
        neg.bus_width = 8;
        neg.sync_period = neg.sync_offset = 0;
        neg.valid = (CCB_TRANS_BUS_WIDTH_VALID
            | CCB_TRANS_SYNC_RATE_VALID | CCB_TRANS_SYNC_OFFSET_VALID);

        /* 丢弃所有断开连接的 CCB，并清理协商 */
        for (targ=0; targ <= OUR_MAX_SUPPORTED_TARGET; targ++) {
            clean_negotiations(softc, targ);

            /* 如果可能，报告事件 */
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

        /* 报告事件 */
        xpt_async(AC_BUS_RESET, softc->wpath, NULL);

        /* 重新初始化可能需要较长时间，在这种情况下
         * 其完成应通过另一个中断信号或
         * 超时检查 - 但为了简单起见，我们假设它是非常快的
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

如果中断不是由控制器范围的条件引起的，那么可能是当前硬件控制块发生了问题。根据硬件的不同，可能还有其他与 HCB 无关的事件，这里我们不予考虑。然后我们分析发生了什么：

```c
struct xxx_hcb *hcb, *h, *hh;
    int hcb_status, scsi_status;
    int ccb_status;
    int targ;
    int lun_to_freeze;

    hcb = get_current_hcb(softc);
    if (hcb == NULL) {
        /* 可能是无关的中断或发生了严重错误
         * 或者这是某些与硬件相关的情况
         */
        根据需要处理;
        return;
    }

    targ = hcb->target;
    hcb_status = get_status_of_current_hcb(softc);
```

首先检查 HCB 是否已经完成，如果是，我们检查返回的 SCSI 状态。

```c
if (hcb_status == COMPLETED) {
        scsi_status = get_completion_status(hcb);
```

然后查看这个状态是否与 `REQUEST SENSE` 命令相关，如果是，则以简单的方式处理它。

```c
if (hcb->flags & DOING_AUTOSENSE) {
            if (scsi_status == GOOD) { /* autosense 成功 */
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

否则，命令本身已完成，请更关注细节。如果此 CCB 未禁用自动感知，并且命令因带有感知数据而失败，则运行 REQUEST SENSE 命令以接收该数据。

```c
hcb->ccb->csio.scsi_status = scsi_status;
        calculate_residue(hcb);

        if ((hcb->ccb->ccb_h.flags & CAM_DIS_AUTOSENSE)==0
        && (scsi_status == CHECK_CONDITION
                || scsi_status == COMMAND_TERMINATED)) {
            /* 启动自动感知 */
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

一个典型的情况是协商事件：从 SCSI 目标收到的协商消息（响应我们的协商尝试或由目标主动发起）或目标无法协商（拒绝我们的协商消息或不回应）。

```c
switch (hcb_status) {
    case TARGET_REJECTED_WIDE_NEG:
        /* 恢复为 8 位总线 */
        softc->current_bus_width[targ] = softc->goal_bus_width[targ] = 8;
        /* 报告事件 */
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
                /* 答复是可接受的 */
                softc->current_bus_width[targ] =
                softc->goal_bus_width[targ] = neg.bus_width = wd;

                /* 报告事件 */
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
                /* 总线宽度已更改 */
                softc->current_bus_width[targ] =
                softc->goal_bus_width[targ] = neg.bus_width = wd;

                /* 报告事件 */
                neg.valid = CCB_TRANS_BUS_WIDTH_VALID;
                xpt_async(AC_TRANSFER_NEG, hcb->ccb.ccb_h.path_id, &neg);
            }
            prepare_width_nego_rsponse(hcb, wd);
        }
        continue_current_hcb(softc);
        return;
    }
```

然后我们像之前一样，用简单的方式处理在自动感知过程中可能发生的任何错误。否则，我们再次仔细查看细节。

```c
if (hcb->flags & DOING_AUTOSENSE)
        goto autosense_failed;

    switch (hcb_status) {
```

接下来我们考虑的是意外断开连接。对于 ABORT 或 BUS DEVICE RESET 消息后，这是正常现象，而在其他情况下则为异常。

```c
case UNEXPECTED_DISCONNECT:
        if (requested_abort(hcb)) {
            /* 中止影响该目标+LUN上的所有命令，因此
             * 将所有该目标+LUN上的断开连接的 HCB 标记为中止 */
            for (h = softc->first_discon_hcb[hcb->target][hcb->lun];
                    h != NULL; h = hh) {
                hh=h->next;
                free_hcb_and_ccb_done(h, h->ccb, CAM_REQ_ABORTED);
            }
            ccb_status = CAM_REQ_ABORTED;
        } else if (requested_bus_device_reset(hcb)) {
            int lun;

            /* 重置影响该目标上的所有命令，因此
             * 将所有该目标+LUN上的断开连接的 HCB 标记为重置 */

            for (lun=0; lun <= OUR_MAX_SUPPORTED_LUN; lun++)
                for (h = softc->first_discon_hcb[hcb->target][lun];
                        h != NULL; h = hh) {
                    hh=h->next;
                    free_hcb_and_ccb_done(h, h->ccb, CAM_SCSI_BUS_RESET);
                }

            /* 发送事件 */
            xpt_async(AC_SENT_BDR, hcb->ccb->ccb_h.path_id, NULL);

            /* 这是 CAM_RESET_DEV 请求本身，已完成 */
            ccb_status = CAM_REQ_CMP;
        } else {
            calculate_residue(hcb);
            ccb_status = CAM_UNEXP_BUSFREE;
            /* 请求进一步的代码来冻结队列 */
            hcb->ccb->ccb_h.status |= CAM_DEV_QFRZN;
            lun_to_freeze = hcb->lun;
        }
        break;
```

如果目标拒绝接受标签，我们会通知 CAM 并返回该 LUN 的所有命令：

```c
case TAGS_REJECTED:
        /* 报告事件 */
        neg.flags = 0 & ~CCB_TRANS_TAG_ENB;
        neg.valid = CCB_TRANS_TQ_VALID;
        xpt_async(AC_TRANSFER_NEG, hcb->ccb.ccb_h.path_id, &neg);

        ccb_status = CAM_MSG_REJECT_REC;
        /* 请求进一步的代码来冻结队列 */
        hcb->ccb->ccb_h.status |= CAM_DEV_QFRZN;
        lun_to_freeze = hcb->lun;
        break;
```

然后我们检查其他一些条件，处理基本上限于设置 CCB 状态：

```c
case SELECTION_TIMEOUT:
        ccb_status = CAM_SEL_TIMEOUT;
        /* 请求进一步的代码来冻结队列 */
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
        /* 其他所有错误通过通用方式处理 */
        ccb_status = CAM_REQ_CMP_ERR;
        /* 请求进一步的代码来冻结队列 */
        hcb->ccb->ccb_h.status |= CAM_DEV_QFRZN;
        lun_to_freeze = CAM_LUN_WILDCARD;
        break;
    }
```

接着我们检查错误是否严重到需要冻结输入队列，直到其被处理，如果是这样的话，我们就这么做：

```c
if (hcb->ccb->ccb_h.status & CAM_DEV_QFRZN) {
        /* 冻结队列 */
        xpt_freeze_devq(ccb->ccb_h.path, /*count*/1);

        /* 将该目标/LUN的所有命令重新排队到 CAM */

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

这结束了通用的中断处理，尽管特定的控制器可能需要一些附加处理。

## 12.9. 错误摘要

在执行 I/O 请求时，可能会发生很多问题。错误的原因可以通过 CCB 状态报告详细信息。本文档中有许多使用示例。为了完整性，以下是典型错误条件的推荐响应摘要：

* *CAM\_RESRC\_UNAVAIL* - 某些资源暂时不可用，SIM 驱动程序无法生成事件来指示其何时可用。此类资源的一个例子是某个控制器内部硬件资源，该控制器在该资源变得可用时不会生成中断。
* *CAM\_UNCOR\_PARITY* - 发生了无法恢复的奇偶校验错误
* *CAM\_DATA\_RUN\_ERR* - 数据溢出或意外的数据阶段（与 CAM\_DIR\_MASK 中指定的方向相反），或宽传输的奇数传输长度
* *CAM\_SEL\_TIMEOUT* - 选择超时发生（目标没有响应）
* *CAM\_CMD\_TIMEOUT* - 命令超时发生（超时函数已运行）
* *CAM\_SCSI\_STATUS\_ERROR* - 设备返回错误
* *CAM\_AUTOSENSE\_FAIL* - 设备返回错误且 REQUEST SENSE 命令失败
* *CAM\_MSG\_REJECT\_REC* - 收到 MESSAGE REJECT 消息
* *CAM\_SCSI\_BUS\_RESET* - 收到 SCSI 总线重置
* *CAM\_REQ\_CMP\_ERR* - 发生了“不可能的”SCSI 阶段或其他类似的异常，或者如果没有进一步细节可用，则作为通用错误
* *CAM\_UNEXP\_BUSFREE* - 发生了意外断开连接
* *CAM\_BDR\_SENT* - 向目标发送了 BUS DEVICE RESET 消息
* *CAM\_UNREC\_HBA\_ERROR* - 不可恢复的主机总线适配器错误
* *CAM\_REQ\_TOO\_BIG* - 请求对于此控制器来说过大
* *CAM\_REQUEUE\_REQ* - 该请求应重新排队以保持事务顺序。这通常发生在 SIM 识别到一个应该冻结队列的错误时，并且必须将目标的其他排队请求在 sim 层返回到 XPT 队列中。典型的此类错误情况包括选择超时、命令超时等。在这种情况下，出现问题的命令返回指示错误的状态，而那些尚未发送到总线的其他命令则被重新排队。
* *CAM\_LUN\_INVALID* - 请求中的 LUN ID 不被 SCSI 控制器支持
* *CAM\_TID\_INVALID* - 请求中的目标 ID 不被 SCSI 控制器支持

## 12.10. 超时处理

当 HCB 的超时到期时，应中止该请求，就像执行 XPT\_ABORT 请求一样。唯一的区别是，中止请求返回的状态应该是 CAM\_CMD\_TIMEOUT 而不是 CAM\_REQ\_ABORTED（这就是为什么更好地将中止操作实现为一个函数的原因）。但还有一个可能的问题：如果中止请求本身卡住了怎么办？在这种情况下，应该重置 SCSI 总线，就像执行 XPT\_RESET\_BUS 请求一样（这里也建议像之前一样将其实现为一个从两个地方调用的函数）。如果设备重置请求卡住了，我们也应该重置整个 SCSI 总线。因此，超时函数的实现如下所示：

```c
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

当我们中止一个请求时，所有与该目标/LUN 断开的其他请求也会被中止。那么问题来了，我们应该用 CAM\_REQ\_ABORTED 还是 CAM\_CMD\_TIMEOUT 状态来返回它们？当前的驱动程序使用 CAM\_CMD\_TIMEOUT。这是合理的，因为如果一个请求超时了，那么可能发生了非常严重的问题，如果它们不被干扰，它们可能会自行超时。
