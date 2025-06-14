# 第 7 章 虚拟内存系统

## 7.1. 物理内存管理 `vm_page_t`

物理内存是通过 `vm_page_t` 结构逐页管理的。页面的物理内存通过将各自的 `vm_page_t` 结构放置到多个分页队列中的一个来分类。

页面可以处于有线、活动、非活动、缓存或空闲状态。除了有线状态外，页面通常被放置在代表其当前状态的双向链表队列中。有线页面不会被放置到任何队列中。

FreeBSD 实现了一个更复杂的缓存和空闲页面分页队列，用于实现页面着色。每个状态都涉及多个队列，队列按处理器的 L1 和 L2 缓存的大小排列。当需要分配新页面时，FreeBSD 会尝试从内存中获取一个与 L1 和 L2 缓存相对齐的页面，以便与正在分配的 VM 对象对齐。

此外，页面可以通过引用计数来持有或通过忙碌计数来锁定。VM 系统还通过页面的标志中的 PG\_BUSY 位实现了一个“最终锁定”状态。

通常，每个分页队列按 LRU（最近最少使用）方式操作。页面最初通常会放置在有线或活动状态下。当页面被标记为有线时，通常会与某个页面表相关联。VM 系统通过扫描活动分页队列（LRU）中的页面并将其移到不那么活跃的分页队列来“老化”页面。被移动到缓存中的页面仍然与 VM 对象关联，但可以立即重新使用。空闲队列中的页面是完全空闲的。FreeBSD 尽量减少空闲队列中的页面数量，但必须保持一定数量的真正空闲页面，以便在中断时进行页面分配。

如果进程尝试访问其页面表中不存在的页面，但该页面在某个分页队列中存在（例如，非活动队列或缓存队列中），则会发生一个相对便宜的页面重新激活错误，导致页面被重新激活。如果页面根本不存在于系统内存中，进程必须阻塞，直到页面从磁盘中加载进来。

FreeBSD 动态地调整其分页队列，并尝试保持队列中页面的合理比例，同时努力维持干净页面与脏页面的合理分布。发生的重平衡量取决于系统的内存负载。此重平衡由页面输出守护进程（pageout daemon）实现，并涉及将脏页面同步到其备份存储、监视页面是否被活动引用（通过重置其在 LRU 队列中的位置或在队列之间移动它们）、当队列失衡时在队列之间迁移页面等。FreeBSD 的 VM 系统愿意接受一定数量的重新激活页面错误，以确定页面实际是活跃还是空闲。这使得在何时洗净或交换页面时做出更好的决策。

## 7.2. 统一缓冲区缓存 `vm_object_t`

FreeBSD 实现了一个通用的“VM 对象”概念。VM 对象可以与不同类型的备份存储相关联：无备份、交换备份、物理设备备份或文件备份存储。由于文件系统使用相同的 VM 对象来管理与文件相关的内存中的数据，因此得出了一个统一的缓冲区缓存。

VM 对象可以被**影射（shadowed）**，即可以堆叠在一起。例如，你可以将一个交换备份的 VM 对象堆叠在一个文件备份的 VM 对象上，以实现 MAP\_PRIVATE 的 mmap() 映射。这种堆叠也用于实现各种共享属性，包括对地址空间的复制时写（copy-on-write）。

需要注意的是，`vm_page_t` 只能与一个 VM 对象关联一次。VM 对象的影射实现了在多个实例之间共享同一页面的效果。

## 7.3. 文件系统 I/O `struct buf`

由 vnode 支持的 VM 对象，如文件支持的对象，通常需要维护独立于 VM 系统的干净/脏信息。例如，当 VM 系统决定将物理页面同步到其备份存储时，VM 系统需要在页面实际写入备份存储之前将其标记为干净。此外，文件系统还需要能够将文件或文件元数据的部分映射到 KVM 中，以便对其进行操作。

用于管理这些操作的实体被称为文件系统缓冲区，`struct buf` 或 `bp`。当文件系统需要操作 VM 对象的一部分时，它通常会将对象的一部分映射到一个 `struct buf` 中，然后将 `struct buf` 中的页面映射到 KVM。同样，磁盘 I/O 通常通过将对象的部分映射到缓冲区结构中，然后对缓冲区结构发起 I/O 操作来实现。在 I/O 的整个过程中，底层的 `vm_page_t` 通常会被忙碌标记。文件系统缓冲区也有其自己的忙碌概念，这对于文件系统驱动代码非常有用，因为它们更愿意操作文件系统缓冲区而不是硬 VM 页面。

FreeBSD 保留了一定量的 KVM 来存放 `struct buf` 的映射，但需要明确的是，这部分 KVM 仅用于存放映射，并不限制数据缓存的能力。物理数据缓存严格来说是 `vm_page_t` 的功能，而非文件系统缓冲区。然而，由于文件系统缓冲区用于占位 I/O 操作，它们本质上限制了可以并发执行的 I/O 数量。不过，由于通常有几千个文件系统缓冲区可用，这通常不是问题。

## 7.4. 映射页表 `vm_map_t, vm_entry_t`

FreeBSD 将物理页表拓扑与 VM 系统分开。所有硬件的每进程页表可以动态重建，通常被视为临时的。管理 KVM 的特殊页表通常会被永久预分配。这些页表不是临时的。

FreeBSD 通过 `vm_map_t` 和 `vm_entry_t` 结构将 `vm_object` 的部分与虚拟内存中的地址范围关联。页表是通过 `vm_map_t`/`vm_entry_t`/`vm_object_t` 层级直接合成的。回想一下，我提到过物理页面只直接与 `vm_object` 关联；这并不完全准确。`vm_page_t` 还会链接到它们实际关联的页表中。一个 `vm_page_t` 可以被链接到多个 *pmaps* 中，这些页表即是所谓的“页面映射”。不过，层级关联依然有效，因此同一对象中的所有对同一页面的引用都会引用相同的 `vm_page_t`，从而实现统一的缓存。

## 7.5. KVM 内存映射

FreeBSD 使用 KVM 来存储各种内核结构。KVM 中存储的单一最大实体是文件系统缓冲区缓存。即，与 `struct buf` 实体相关的映射。

与 Linux 不同，FreeBSD **不会**将所有物理内存映射到 KVM 中。这意味着 FreeBSD 在 32 位平台上可以处理最大 4G 的内存配置。实际上，如果 MMU 足够支持，FreeBSD 理论上可以在 32 位平台上处理最多 8TB 的内存配置。然而，由于大多数 32 位平台只能映射 4GB 的内存，因此这个问题在实际应用中是无关紧要的。

KVM 是通过多种机制来管理的。管理 KVM 的主要机制是 **zone 分配器**。zone 分配器将一块 KVM 内存分割成固定大小的内存块，用于分配特定类型的结构。你可以使用 `vmstat -m` 命令查看当前按 zone 分类的 KVM 使用情况。

## 7.6. 调整 FreeBSD VM 系统

FreeBSD 内核已做出集中努力，动态调整自身配置。通常情况下，你无需修改除 `maxusers` 和 `NMBCLUSTERS` 内核配置选项以外的任何内容。也就是说，内核编译选项通常在 **/usr/src/sys/i386/conf/CONFIG\_FILE** 文件中指定。所有可用的内核配置选项的描述可以在 **/usr/src/sys/i386/conf/LINT** 中找到。

在大型系统配置中，你可能希望增加 `maxusers`。其值通常范围从 10 到 128。请注意，将 `maxusers` 设置得过高可能会导致系统溢出可用的 KVM，进而导致不可预测的操作。最好将 `maxusers` 设置为合理的数值，并增加其他选项（如 `NMBCLUSTERS`）来增加特定的资源。

如果你的系统将频繁使用网络，你可能希望增加 `NMBCLUSTERS`。典型的值范围从 1024 到 4096。

`NBUF` 参数传统上用于调整系统。此参数决定了系统可用于映射文件系统缓冲区进行 I/O 的 KVA 数量。请注意，此参数与统一缓冲区缓存无关！该参数在 3.0-CURRENT 及更高版本的内核中是动态调整的，通常不应手动调整。我们建议你不要尝试指定 `NBUF` 参数。过小的值可能导致极其低效的文件系统操作，而过大的值可能会导致页面队列饥饿，因而使得太多页面被固定。

默认情况下，FreeBSD 内核并未进行优化。你可以在内核配置文件中使用 `makeoptions` 指令设置调试和优化标志。请注意，除非你能够处理生成的大型内核（通常超过 7 MB），否则不应使用 `-g`。

```ini
makeoptions      DEBUG="-g"
makeoptions      COPTFLAGS="-O -pipe"
```

Sysctl 提供了一种在运行时调整内核参数的方法。通常，你无需修改任何 sysctl 变量，尤其是与 VM 相关的变量。

运行时的 VM 和系统调整相对简单。首先，尽可能在你的 UFS/FFS 文件系统上使用 Soft Updates。**/usr/src/sys/ufs/ffs/README.softupdates** 包含了配置它的说明（和限制）。

其次，配置足够的交换空间。你应该在每个物理磁盘上配置一个交换分区，最多四个，即使是你的“工作”磁盘。交换空间的大小应至少是主内存的两倍，如果内存较小，可能需要更多的交换空间。你还应该根据你计划在机器上配置的最大内存量来调整交换分区的大小，以便以后无需重新分区。如果你希望能够容纳崩溃转储，你的第一个交换分区必须至少与主内存大小相同，并且 **/var/crash** 必须有足够的空闲空间来存储转储。

基于 NFS 的交换在 4.X 或更高版本的系统上完全可以接受，但你必须意识到，NFS 服务器将承受大部分的分页负载。
