# 第 6 章 TrustedBSD MAC 框架

## 6.1. MAC 文档版权

本文档由 Safeport Network Services 和 Network Associates Laboratories（网络关联实验室）安全研究部的 Chris Costello 为 FreeBSD 项目开发，属于 DARPA/SPAWAR 合同 N66001-01-C-8035（“CBOSS”）的一部分，作为 DARPA CHATS 研究项目的一部分。

允许在源代码（SGML DocBook）和“已编译”形式（SGML、HTML、PDF、PostScript、RTF 等）中进行再分发和使用，无论是否修改，只要满足以下条件：

1. 源代码（SGML DocBook）再分发必须保留上述版权声明、此条件列表以及以下免责声明，作为文件的前几行，并且不做修改。
2. 已编译形式的再分发（转换为其他 DTD，转换为 PDF、PostScript、RTF 和其他格式）必须在文档和/或随附的其他材料中复制上述版权声明、此条件列表和以下免责声明。

>**重要**
>
>本文档由 NETWORKS ASSOCIATES TECHNOLOGY, INC 提供，“按原样”提供，且不作任何明示或暗示的担保，包括但不限于对适销性和特定用途适用性的默示担保。在任何情况下，NETWORKS ASSOCIATES TECHNOLOGY, INC 不对因使用本文档而导致的任何直接、间接、附带、特殊、惩罚性或间接损害负责（包括但不限于：替代商品或服务的采购；使用、数据或利润的丧失；或业务中断），无论是合同、严格责任或侵权（包括疏忽或其他）理论下，是否已被告知可能发生此类损害。

## 6.2. 概述

FreeBSD 包括对几种强制访问控制（MAC）策略的实验性支持，并提供了一个内核安全扩展框架——TrustedBSD MAC 框架。MAC 框架是一个可插拔的访问控制框架，允许新的安全策略轻松地链接到内核中，可以在启动时加载，也可以在运行时动态加载。该框架提供了多种功能，使得实现新的安全策略变得更容易，包括能够轻松地将安全标签（如机密性信息）附加到系统对象。

本章介绍了 MAC 策略框架，并提供了一个示例 MAC 策略模块的文档。

## 6.3. 介绍

TrustedBSD MAC 框架提供了一种机制，允许在编译时或运行时扩展内核访问控制模型。新的系统策略可以作为内核模块实现并链接到内核；如果存在多个策略模块，它们的结果将会组合。MAC 框架为策略编写者提供了多种访问控制基础设施服务，包括对瞬时和持久的、与策略无关的对象安全标签的支持。目前，这项支持被认为是实验性的。

本章提供了适用于策略模块开发人员以及潜在的 MAC 启用环境消费者的信息，帮助他们了解 MAC 框架如何支持对内核的访问控制扩展。

## 6.4. 策略背景

强制访问控制（MAC）指的是一组由操作系统强制执行的访问控制策略。这些 MAC 策略可以与自由裁量访问控制（DAC）保护进行对比，后者允许非管理员用户（按其自由裁量）保护对象。在传统的 UNIX 系统中，DAC 保护包括文件权限和访问控制列表；而 MAC 保护包括防止用户间调试的进程控制和防火墙。操作系统设计师和安全研究人员已经提出了各种 MAC 策略，包括多级安全（MLS）机密性策略、Biba 完整性策略、基于角色的访问控制（RBAC）、域和类型强制（DTE）以及类型强制（TE）。每种模型的决策基于多种因素，包括用户身份、角色和安全许可，以及表示数据敏感性和完整性等概念的对象安全标签。

TrustedBSD MAC 框架能够支持实现所有这些策略的模块，以及一大类系统硬化策略，这些策略可以利用现有的安全属性，如用户和组 ID，以及文件的扩展属性和其他系统属性。此外，尽管名字中带有“强制访问控制”，MAC 框架也可以用来实现纯粹的自由裁量策略，因为策略模块在授权保护时具有相当大的灵活性。

## 6.5. MAC 框架内核架构

TrustedBSD MAC 框架允许内核模块扩展操作系统安全策略，同时提供许多访问控制模块所需的基础设施功能。如果多个策略同时加载，MAC 框架将有效地（在某些定义下有用地）组合这些策略的结果。

### 6.5.1. 内核元素

MAC 框架包含多个内核元素：

* 框架管理接口
* 并发和同步原语
* 策略注册
* 可扩展的内核对象安全标签
* 策略入口点组成运算符
* 标签管理原语
* 内核服务调用的入口点 API
* 策略模块的入口点 API
* 入口点实现（策略生命周期、对象生命周期/标签管理、访问控制检查）
* 与策略无关的标签管理系统调用
* `mac_syscall()` 多路复用系统调用
* 作为 MAC 策略模块实现的各种安全策略

### 6.5.2. 框架管理接口

TrustedBSD MAC 框架可以通过 sysctl、加载器调优选项和系统调用进行直接管理。

在大多数情况下，具有相同名称的 sysctl 和加载器调优选项会修改相同的参数，并控制与各种内核子系统相关的保护执行。此外，如果内核中编译了 MAC 调试支持，将维护多个计数器，用于跟踪标签分配。通常建议在生产环境中不要使用每个子系统的执行控制来控制策略行为，因为它们广泛影响所有活动策略的操作。相反，应优先使用每个策略的控制，因为它们为策略模块提供了更大的粒度和更大的操作一致性。

策略模块的加载和卸载是通过系统模块管理系统调用和其他系统接口执行的，包括启动加载器变量；策略模块将有机会影响加载和卸载事件，包括防止不希望的策略卸载。

### 6.5.3. 策略列表并发性和同步

由于活动策略集可能在运行时发生变化，并且入口点的调用是非原子的，因此需要同步以防止在入口点调用过程中加载或卸载策略，从而在此期间冻结活动策略集。这是通过框架忙碌计数来完成的：每当进入一个入口点时，忙碌计数会增加；每当退出时，忙碌计数会减少。在忙碌计数为正时，不允许修改策略列表，试图修改策略列表的线程将被挂起，直到列表不再忙碌。忙碌计数由互斥锁保护，并且使用条件变量唤醒等待策略列表修改的线程。该同步模型的一个副作用是，允许从策略模块内部递归进入 MAC 框架，尽管通常不会使用。

为了减少忙碌计数的开销，使用了各种优化，包括避免在列表为空或仅包含静态条目的情况下完全增减计数（静态条目是指在系统启动之前加载的政策，并且不能卸载）。还提供了一个编译时选项，禁止在运行时更改已加载的策略集，这消除了与支持动态加载和卸载策略相关的互斥锁锁定开销，因为不再需要同步。

由于 MAC 框架在某些入口点中不允许阻塞，因此不能使用普通的睡眠锁；因此，加载或卸载尝试可能会因等待框架变为空闲而阻塞相当长的时间。

### 6.5.4. 标签同步

由于感兴趣的内核对象通常可能被多个线程同时访问，并且允许多个线程同时进入 MAC 框架，因此 MAC 框架维护的安全属性存储必须小心同步。通常，使用现有的内核同步机制来保护内核对象上的 MAC 框架安全标签：例如，套接字上的 MAC 标签使用现有的套接字互斥锁进行保护。同样，并发访问的语义通常与容器对象的语义相同：对于凭证，标签内容采用写时复制语义，类似于凭证结构的其余部分。当使用对象引用调用 MAC 框架时，它会在对象上加上必要的锁。策略作者必须了解这些同步语义，因为它们有时会限制对标签的访问类型：例如，当通过入口点将对凭证的只读引用传递给策略时，只允许对附加到凭证的标签状态进行读取操作。

### 6.5.5. 策略同步和并发性

策略模块必须假设由于 FreeBSD 内核的并行性和抢占性，多个内核线程可能会同时进入一个或多个策略入口点。如果策略模块使用可变状态，则可能需要在策略内使用同步原语，以防止在该状态上产生不一致的视图，从而导致策略操作错误。策略通常可以利用现有的 FreeBSD 同步原语来实现这一目的，包括互斥锁、睡眠锁、条件变量和计数信号量。然而，策略应该小心使用这些原语，遵守现有的内核锁定顺序，并认识到某些入口点不允许阻塞，因此在这些入口点中使用的原语仅限于互斥锁和唤醒操作。

当策略模块调用其他内核子系统时，它们通常需要释放任何策略内的锁，以避免违反内核锁定顺序或导致锁递归。这将使策略锁保持为全局锁顺序中的叶子锁，有助于避免死锁。

### 6.5.6. 策略注册

MAC 框架维护两种活动策略列表：静态列表和动态列表。这些列表仅在锁定语义上有所不同：使用静态列表时不需要提高引用计数。当包含 MAC 框架策略的内核模块被加载时，策略模块将使用 `SYSINIT` 调用注册函数；当策略模块被卸载时，`SYSINIT` 也会调用注销函数。如果策略模块被加载多次，或者在注册过程中没有足够的资源（例如，策略可能需要标签，而可用的标签状态不足），或者其他策略先决条件没有满足（某些策略可能仅能在启动前加载），注册可能会失败。同样，如果策略被标记为不可卸载，注销也可能失败。

### 6.5.7. 入口点

内核服务通过两种方式与 MAC 框架交互：它们调用一系列 API 通知框架相关事件，并提供安全相关对象中的与策略无关的标签结构指针。标签指针由 MAC 框架通过标签管理入口点维护，允许框架通过对维护对象的内核子系统进行相对不具侵入性的修改，为策略模块提供标签服务。例如，标签指针已被添加到进程、进程凭证、套接字、管道、vnode、Mbuf、网络接口、IP 重组队列以及其他各种安全相关结构中。当内核服务执行重要的安全决策时，它们也会调用 MAC 框架，允许策略模块基于自己的标准（可能包括存储在安全标签中的数据）来增强这些决策。这些安全关键的决策大多是显式的访问控制检查；然而，某些决策影响更一般的决策函数，例如套接字的包匹配和程序执行时的标签过渡。

### 6.5.8. 策略组合

当多个策略模块同时加载到内核中时，框架将使用组合运算符组合策略模块的结果。该运算符当前是硬编码的，要求所有活动策略必须批准请求才能返回成功。由于策略可能返回多种错误条件（成功、访问被拒绝、对象不存在等），优先级运算符从策略返回的错误集选择最终的错误。通常，指示对象不存在的错误会优先于指示拒绝访问对象的错误。虽然无法保证结果组合将是有用或安全的，但我们发现许多有用的策略选择适用于此。例如，传统的受信系统通常使用两种或更多策略进行类似的组合。

### 6.5.9. 标签支持

由于许多有趣的访问控制扩展依赖于对象上的安全标签，MAC 框架提供了一组与策略无关的标签管理系统调用，涵盖了各种用户暴露的对象。常见的标签类型包括分区标识符、敏感度标签、完整性标签、隔离区、域、角色和类型。所谓与策略无关，意味着策略模块能够完全定义与对象相关的元数据的语义。策略模块参与用户应用程序提供的基于字符串的标签的内化和外化，并可以在需要时向应用程序暴露多个标签元素。

内存中的标签存储在 slab 分配的 `struct label` 中，该结构由一个固定长度的联合体数组组成，每个联合体包含一个 `void *` 指针和一个 `long`。注册标签存储的策略将被分配一个“槽”标识符，该标识符可用于解引用标签存储。存储的语义完全由策略模块决定：模块提供与内核对象生命周期相关的多种入口点，包括初始化、关联/创建和销毁。通过这些接口，策略模块可以实现引用计数和其他存储模型。通常，策略模块不需要直接访问对象结构来检索标签，因为 MAC 框架通常会将对象的指针和对象标签的直接指针传递给入口点。唯一的例外是进程凭证，必须手动解引用才能访问凭证标签。这可能会在 MAC 框架的未来版本中发生变化。

初始化入口点通常包括一个睡眠处置标志，指示是否允许初始化时进入睡眠状态；如果不允许睡眠，可能会返回失败，取消标签（及其对象）的分配。例如，在网络栈中的中断处理期间，不能进行睡眠，或者当调用者持有互斥锁时，可能会发生这种情况。由于在飞行中的网络数据包（Mbuf）上维护标签的性能开销，策略必须明确声明需要分配 Mbuf 标签。使用标签的动态加载策略必须能够处理其初始化函数尚未在对象上调用的情况，因为在加载策略时，某些对象可能已经存在。MAC 框架保证未初始化的标签槽将保持 0 或 NULL 值，策略可以使用该值来检测未初始化的标签。然而，由于 Mbuf 标签的分配是条件性的，如果 Mbuf 标签尚未分配，策略也必须能够处理 NULL 标签指针。

在文件系统标签的情况下，提供了对安全标签在扩展属性中的持久化存储的特殊支持。在可用的情况下，使用扩展属性事务来进行安全标签的复合更新，以便在 vnodes 上一致地更新安全标签——目前此支持仅在 UFS2 文件系统中存在。策略作者可以选择使用一个或多个扩展属性来实现多标签文件系统对象标签。出于效率考虑，vnode 标签（`v_label`）是任何磁盘标签的缓存；策略可以在 vnode 实例化时将值加载到缓存中，并根据需要更新缓存。因此，每次访问控制检查时不需要直接访问扩展属性。

>**注意**
>
>当前，如果一个标记策略允许动态卸载，则其状态槽不能被回收，这对标记策略的卸载-重新加载操作数量设置了严格（且相对较低）的上限。

### 6.5.10. 系统调用

MAC 框架实现了多个系统调用：这些调用中的大多数支持暴露给用户应用程序的与策略无关的标签检索和操作 API。

标签管理调用接受一个标签描述结构体 `struct mac`，该结构体包含一系列 MAC 标签元素。每个元素包含一个字符字符串名称和字符字符串值。每个策略都将有机会声明一个特定的元素名称，从而允许策略暴露多个独立的元素（如果需要的话）。策略模块通过入口点执行内核标签和用户提供的标签之间的内化和外化，从而支持各种语义。标签管理系统调用通常通过用户库函数进行包装，以执行内存分配和错误处理，从而简化需要管理标签的用户应用程序。

以下是 FreeBSD 内核中存在的 MAC 相关系统调用：

* `mac_get_proc()` 可用于检索当前进程的标签。
* `mac_set_proc()` 可用于请求更改当前进程的标签。
* `mac_get_fd()` 可用于检索由文件描述符引用的对象（文件、套接字、管道等）的标签。
* `mac_get_file()` 可用于检索由文件系统路径引用的对象的标签。
* `mac_set_fd()` 可用于请求更改由文件描述符引用的对象（文件、套接字、管道等）的标签。
* `mac_set_file()` 可用于请求更改由文件系统路径引用的对象的标签。
* `mac_syscall()` 允许策略模块在不修改系统调用表的情况下创建新的系统调用；它接受目标策略名称、操作编号和供策略使用的不透明参数。
* `mac_get_pid()` 可用于请求通过进程 ID 获取另一个进程的标签。
* `mac_get_link()` 与 `mac_get_file()` 相同，只是如果路径的最后一项是符号链接，它不会跟随符号链接，因此可以用于检索符号链接的标签。
* `mac_set_link()` 与 `mac_set_file()` 相同，只是如果路径的最后一项是符号链接，它不会跟随符号链接，因此可以用于操作符号链接的标签。
* `mac_execve()` 与 `execve()` 系统调用相同，只是它还接受一个请求的标签，用于在开始执行新程序时设置进程的标签。这种执行时的标签更改被称为“过渡”。
* `mac_get_peer()` 实际上通过套接字选项实现，用于检索远程对等端在套接字上的标签（如果可用）。

除了这些系统调用外，`SIOCSIGMAC` 和 `SIOCSIFMAC` 网络接口 ioctl 允许检索和设置网络接口上的标签。

## 6.6. MAC 策略架构

安全策略要么直接链接到内核中，要么编译成可加载的内核模块，可以在启动时或通过运行时的模块加载系统调用动态加载。策略模块通过一组声明的入口点与系统交互，提供对系统事件流的访问，并允许策略影响访问控制决策。每个策略包含以下几个元素：

* 策略的可选配置参数。
* 策略逻辑和参数的集中实现。
* 策略生命周期事件的可选实现，例如初始化和销毁。
* 可选的支持，用于在选定的内核对象上初始化、维护和销毁标签。
* 可选的支持，用于用户进程检查和修改选定对象的标签。
* 实现策略感兴趣的选定访问控制入口点。
* 声明策略身份、模块入口点和策略属性。

### 6.6.1. 策略声明

模块可以使用 `MAC_POLICY_SET()` 宏声明，该宏为策略命名，提供指向 MAC 入口点向量的引用，提供决定策略框架如何处理策略的加载时标志，并可选择性地请求框架分配标签状态。

```
static struct mac_policy_ops mac_policy_ops =
{
        .mpo_destroy = mac_policy_destroy,
        .mpo_init = mac_policy_init,
        .mpo_init_bpfdesc_label = mac_policy_init_bpfdesc_label,
        .mpo_init_cred_label = mac_policy_init_label,
/* ... */
        .mpo_check_vnode_setutimes = mac_policy_check_vnode_setutimes,
        .mpo_check_vnode_stat = mac_policy_check_vnode_stat,
        .mpo_check_vnode_write = mac_policy_check_vnode_write,
};
```

MAC 策略入口点向量，在本例中为 `mac_policy_ops`，将模块中定义的函数与特定的入口点关联起来。可用入口点及其原型的完整列表可以在 MAC 入口点参考部分找到。在模块注册期间，.mpo\_init 和 .mpo\_destroy 入口点尤其重要。`mpo_init` 会在策略成功注册到模块框架后，但在其他任何入口点变为活跃之前调用。这样，策略就可以执行任何策略特定的分配和初始化，例如初始化数据或锁。`mpo_destroy` 会在卸载策略模块时调用，用于释放分配的内存并销毁锁。目前，这两个入口点是在持有 MAC 策略列表互斥锁的情况下调用的，以防止调用其他入口点：这将发生变化，但在此之前，策略应小心调用内核原语，以避免锁顺序或睡眠问题。

策略声明中的模块名称字段存在，以便为模块提供唯一标识，用于模块依赖关系。应选择一个适当的字符串。策略的完整字符串名称将在加载和卸载事件期间通过内核日志显示，并在提供用户空间进程的状态信息时导出。

### 6.6.2. 策略标志

策略声明的标志字段允许模块在加载时向框架提供有关其功能的信息。目前，定义了三个标志：

* `MPC_LOADTIME_FLAG_UNLOADOK`：此标志指示策略模块可以卸载。如果未提供此标志，则策略框架将拒绝卸载模块的请求。此标志可能用于分配了标签状态且无法在运行时释放该状态的模块。

* `MPC_LOADTIME_FLAG_NOTLATE`：此标志指示策略模块必须在启动过程中尽早加载和初始化。如果指定了此标志，则尝试在启动后注册模块将被拒绝。此标志可用于需要对所有系统对象进行普遍标记的策略，并且无法处理未由策略正确初始化的对象。

* `MPC_LOADTIME_FLAG_LABELMBUFS`：此标志指示策略模块需要对 Mbufs 进行标记，并且应始终为 Mbuf 标签分配内存。默认情况下，MAC 框架不会为 Mbuf 分配标签存储，除非至少一个已加载的策略设置了此标志。当策略不需要 Mbuf 标记时，这会显著提高网络性能。存在一个内核选项 `MAC_ALWAYS_LABEL_MBUF`，该选项强制 MAC 框架分配 Mbuf 标签存储，无论此标志的设置如何，并且在某些环境中可能有用。

>**注意**
>
> 使用 `MPC_LOADTIME_FLAG_LABELMBUFS` 而未设置 `MPC_LOADTIME_FLAG_NOTLATE` 标志的策略必须能够正确处理传入入口点的 `NULL` Mbuf 标签指针。这是必要的，因为在启用 Mbuf 标记的策略加载后，未存储标签的在飞 Mbuf 可能会持续存在。如果策略在网络子系统启用之前加载（即策略不是在后期加载的），那么所有 Mbuf 都保证有标签存储。

### 6.6.3. 策略入口点

框架提供了四类入口点供已注册策略使用：与策略注册和管理相关的入口点，表示内核对象的初始化、创建、销毁和其他生命周期事件的入口点，策略模块可能影响的访问控制决策相关的事件，以及与对象标签管理相关的调用。此外，提供了 `mac_syscall()` 入口点，以便策略可以扩展内核接口，而无需注册新的系统调用。

策略模块的编写者应注意内核锁策略，以及在不同入口点期间可用的对象锁。编写者应尽量避免通过在入口点内部获取非叶锁而导致死锁场景，并遵循对象访问和修改的锁定协议。特别地，编写者应注意，虽然访问对象及其标签所需的锁通常已经持有，但对于所有入口点，可能并不总是有足够的锁来修改对象或其标签。参数的锁定信息已在 MAC 框架入口点文档中进行记录。

策略入口点会传递对象标签的引用以及对象本身。这使得带标签的策略可以在不知道对象内部细节的情况下，仍然基于标签做出决策。例外情况是进程凭证，策略假定它作为内核中的一类安全对象是被理解的。

## 6.7. MAC 策略入口点参考

### 6.7.1. 通用模块入口点

#### 6.7.1.1. `mpo_init`

```
void mpo_init(struct mac_policy_conf *conf);
```

| 参数     | 描述       | 锁定 |
| ------ | -------- | -- |
| `conf` | MAC 策略定义 |    |

策略加载事件。持有策略列表互斥锁，因此不能执行睡眠操作，并且调用其他内核子系统时必须小心。如果策略初始化期间需要潜在的睡眠内存分配，则应通过单独的模块 `SYSINIT()` 来进行。

#### 6.7.1.2. `mpo_destroy`

```c
void mpo_destroy(struct mac_policy_conf *conf);
```

| 参数     | 描述       | 锁定 |
| ------ | -------- | -- |
| `conf` | MAC 策略定义 |    |

策略卸载事件。持有策略列表互斥锁，因此应谨慎处理。

#### 6.7.1.3. `mpo_syscall`

```c
int mpo_syscall(struct thread *td, int call, void *arg);
```

| 参数     | 描述          | 锁定 |
| ------ | ----------- | -- |
| `td`   | 调用线程        |    |
| `call` | 策略特定的系统调用编号 |    |
| `arg`  | 指向系统调用参数的指针 |    |

该入口点提供了一个策略多路复用的系统调用，允许策略为用户进程提供额外的服务，而无需注册特定的系统调用。在注册时提供的策略名称用于从用户空间解复用调用，并且参数将被转发到此入口点。在实现新服务时，安全模块应确保根据需要调用 MAC 框架中的适当访问控制检查。例如，如果某个策略实现了扩展的信号功能，则应调用必要的信号访问控制检查以调用 MAC 框架和其他已注册策略。

>**注意**
>
>  当前模块必须自行执行 `copyin()` 来处理系统调用数据。

#### 6.7.1.4. `mpo_thread_userret`

```c
void mpo_thread_userret(struct thread *td);
```

| 参数   | 描述    | 锁定 |
| ---- | ----- | -- |
| `td` | 返回的线程 |    |

此入口点允许策略模块在线程返回到用户空间时，通过系统调用返回、陷阱返回或其他方式执行与 MAC 相关的事件。这对于具有浮动进程标签的策略是必需的，因为在系统调用处理过程中，通常无法在堆栈中的任意点获取进程锁；进程标签可能表示传统的认证数据、进程历史信息或其他数据。为了使用此机制，计划更改进程凭证标签的数据可以存储在 `p_label` 中，受每策略自旋锁保护，然后设置每线程的 `TDF_ASTPENDING` 标志和每进程的 `PS_MACPENDM` 标志，以便调度调用 `userret` 入口点。从此入口点，策略可以创建替代凭证，而无需过多担心锁定上下文。策略编写者需要注意，调度 AST 和 AST 执行之间的事件排序可能是复杂的，并且在多线程应用程序中可能是交错的。

### 6.7.2. 标签操作

#### 6.7.2.1. `mpo_init_bpfdesc_label`

```c
void mpo_init_bpfdesc_label(struct label *label);
```

| 参数      | 描述    | 锁定 |
| ------- | ----- | -- |
| `label` | 新标签应用 |    |

初始化新创建的 bpfdesc（BPF 描述符）上的标签。允许睡眠。

#### 6.7.2.2. `mpo_init_cred_label`

```c
void mpo_init_cred_label(struct label *label);
```

| 参数      | 描述      | 锁定 |
| ------- | ------- | -- |
| `label` | 初始化的新标签 |    |

初始化新创建的用户凭证上的标签。允许睡眠。

#### 6.7.2.3. `mpo_init_devfsdirent_label`

```c
void mpo_init_devfsdirent_label(struct label *label);
```

| 参数      | 描述    | 锁定 |
| ------- | ----- | -- |
| `label` | 新标签应用 |    |

初始化新创建的 devfs 条目上的标签。允许睡眠。

#### 6.7.2.4. `mpo_init_ifnet_label`

```c
void mpo_init_ifnet_label(struct label *label);
```

| 参数      | 描述    | 锁定 |
| ------- | ----- | -- |
| `label` | 新标签应用 |    |

初始化新创建的网络接口上的标签。允许睡眠。

#### 6.7.2.5. `mpo_init_ipq_label`

```c
void mpo_init_ipq_label(struct label *label, int flag);
```

| 参数      | 描述                                                                                             | 锁定 |
| ------- | ---------------------------------------------------------------------------------------------- | -- |
| `label` | 新标签应用                                                                                          |    |
| `flag`  | 睡眠/非睡眠 [malloc(9)](https://man.freebsd.org/cgi/man.cgi?query=malloc&sektion=9&format=html)；见下文 |    |

初始化新创建的 IP 分片重组队列上的标签。`flag` 字段可以是 M\_WAITOK 或 M\_NOWAIT，用于避免在此初始化调用过程中执行睡眠的 [malloc(9)](https://man.freebsd.org/cgi/man.cgi?query=malloc&sektion=9&format=html)。IP 分片重组队列的分配通常发生在性能敏感的环境中，因此实现时应小心避免睡眠或长时间操作。此入口点可能会失败，导致无法分配 IP 分片重组队列。

#### 6.7.2.6. `mpo_init_mbuf_label`

```c
void mpo_init_mbuf_label(int flag, struct label *label);
```

| 参数      | 描述                                                                                             | 锁定 |
| ------- | ---------------------------------------------------------------------------------------------- | -- |
| `flag`  | 睡眠/非睡眠 [malloc(9)](https://man.freebsd.org/cgi/man.cgi?query=malloc&sektion=9&format=html)；见下文 |    |
| `label` | 要初始化的策略标签                                                                                      |    |

初始化新创建的 mbuf 包头（`mbuf`）上的标签。`flag` 字段可以是 M\_WAITOK 或 M\_NOWAIT，用于避免在此初始化调用过程中执行睡眠的 [malloc(9)](https://man.freebsd.org/cgi/man.cgi?query=malloc&sektion=9&format=html)。mbuf 的分配通常发生在性能敏感的环境中，因此实现时应小心避免睡眠或长时间操作。此入口点可能会失败，导致无法分配 mbuf 头部。

#### 6.7.2.7. `mpo_init_mount_label`

```c
void mpo_init_mount_label(struct label *mntlabel, struct label *fslabel);
```

| 参数         | 描述           | 锁定 |
| ---------- | ------------ | -- |
| `mntlabel` | 用于挂载点本身的策略标签 |    |
| `fslabel`  | 用于文件系统的策略标签  |    |

初始化新创建的挂载点上的标签。允许睡眠。

#### 6.7.2.8. `mpo_init_mount_fs_label`

```c
void mpo_init_mount_fs_label(struct label *label);
```

| 参数      | 描述       | 锁定 |
| ------- | -------- | -- |
| `label` | 用于初始化的标签 |    |

初始化新挂载的文件系统上的标签。允许睡眠。

#### 6.7.2.9. `mpo_init_pipe_label`

```c
void mpo_init_pipe_label(struct label *label);
```

| 参数      | 描述       | 锁定 |
| ------- | -------- | -- |
| `label` | 用于填充的新标签 |    |

初始化新创建的管道上的标签。允许睡眠。

#### 6.7.2.10. `mpo_init_socket_label`

```c
void mpo_init_socket_label(struct label *label, int flag);
```

| 参数      | 描述                                                                                     | 锁定 |
| ------- | -------------------------------------------------------------------------------------- | -- |
| `label` | 要初始化的新标签                                                                               |    |
| `flag`  | [malloc(9)](https://man.freebsd.org/cgi/man.cgi?query=malloc&sektion=9&format=html) 标志 |    |

初始化新创建的套接字上的标签。`flag` 字段可以是 M\_WAITOK 或 M\_NOWAIT，用于避免在此初始化调用过程中执行睡眠的 [malloc(9)](https://man.freebsd.org/cgi/man.cgi?query=malloc&sektion=9&format=html)。

#### 6.7.2.11. `mpo_init_socket_peer_label`

```
void mpo_init_socket_peer_label(struct label *label, int flag);
```

| 参数      | 描述                                                                                     | 锁定 |
| ------- | -------------------------------------------------------------------------------------- | -- |
| `label` | 要初始化的新标签                                                                               |    |
| `flag`  | [malloc(9)](https://man.freebsd.org/cgi/man.cgi?query=malloc&sektion=9&format=html) 标志 |    |

初始化新创建的套接字对等标签。`flag` 字段可以是 M\_WAITOK 或 M\_NOWAIT，用于避免在此初始化调用过程中执行睡眠的 [malloc(9)](https://man.freebsd.org/cgi/man.cgi?query=malloc&sektion=9&format=html)。

#### 6.7.2.12. `mpo_init_proc_label`

```c
void mpo_init_proc_label(struct label *label);
```

| 参数      | 描述        | 锁定 |
| ------- | --------- | -- |
| `label` | 要初始化的进程标签 |    |

初始化新创建的进程标签。允许睡眠。

#### 6.7.2.13. `mpo_init_vnode_label`

```c
void mpo_init_vnode_label(struct label *label);
```

| 参数      | 描述             | 锁定 |
| ------- | -------------- | -- |
| `label` | 要初始化的 vnode 标签 |    |

初始化新创建的 vnode 上的标签。允许睡眠。

#### 6.7.2.14. `mpo_destroy_bpfdesc_label`

```c
void mpo_destroy_bpfdesc_label(struct label *label);
```

| 参数      | 描述         | 锁定 |
| ------- | ---------- | -- |
| `label` | bpfdesc 标签 |    |

销毁 BPF 描述符上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.15. `mpo_destroy_cred_label`

```c
void mpo_destroy_cred_label(struct label *label);
```

| 参数      | 描述      | 锁定 |
| ------- | ------- | -- |
| `label` | 正在销毁的标签 |    |

销毁凭证上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.16. `mpo_destroy_devfsdirent_label`

```c
void mpo_destroy_devfsdirent_label(struct label *label);
```

| 参数      | 描述      | 锁定 |
| ------- | ------- | -- |
| `label` | 正在销毁的标签 |    |

销毁 devfs 条目上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.17. `mpo_destroy_ifnet_label`

```c
void mpo_destroy_ifnet_label(struct label *label);
```

| 参数      | 描述      | 锁定 |
| ------- | ------- | -- |
| `label` | 正在销毁的标签 |    |

销毁移除的网络接口上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.18. `mpo_destroy_ipq_label`

```c
void mpo_destroy_ipq_label(struct label *label);
```

| 参数      | 描述      | 锁定 |
| ------- | ------- | -- |
| `label` | 正在销毁的标签 |    |

销毁 IP 分片队列上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.19. `mpo_destroy_mbuf_label`

```c
void mpo_destroy_mbuf_label(struct label *label);
```

| 参数      | 描述      | 锁定 |
| ------- | ------- | -- |
| `label` | 正在销毁的标签 |    |

销毁 mbuf 头部上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.20. `mpo_destroy_mount_label`

```c
void mpo_destroy_mount_label(struct label *label);
```

| 参数      | 描述         | 锁定 |
| ------- | ---------- | -- |
| `label` | 正在销毁的挂载点标签 |    |

销毁挂载点上的标签。在此入口点，策略模块应释放与 `mntlabel` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.21. `mpo_destroy_mount_label`

```c
void mpo_destroy_mount_label(struct label *mntlabel, struct label *fslabel);
```

| 参数         | 描述          | 锁定 |
| ---------- | ----------- | -- |
| `mntlabel` | 正在销毁的挂载点标签  |    |
| `fslabel`  | 正在销毁的文件系统标签 |    |

销毁挂载点上的标签。在此入口点，策略模块应释放与 `mntlabel` 和 `fslabel` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.22. `mpo_destroy_socket_label`

```c
void mpo_destroy_socket_label(struct label *label);
```

| 参数      | 描述         | 锁定 |
| ------- | ---------- | -- |
| `label` | 正在销毁的套接字标签 |    |

销毁套接字上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.23. `mpo_destroy_socket_peer_label`

```c
void mpo_destroy_socket_peer_label(struct label *peerlabel);
```

| 参数          | 描述           | 锁定 |
| ----------- | ------------ | -- |
| `peerlabel` | 正在销毁的套接字对等标签 |    |

销毁套接字上的对等标签。在此入口点，策略模块应释放与 `peerlabel` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.24. `mpo_destroy_pipe_label`

```c
void mpo_destroy_pipe_label(struct label *label);
```

| 参数      | 描述   | 锁定 |
| ------- | ---- | -- |
| `label` | 管道标签 |    |

销毁管道上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.25. `mpo_destroy_proc_label`

```c
void mpo_destroy_proc_label(struct label *label);
```

| 参数      | 描述   | 锁定 |
| ------- | ---- | -- |
| `label` | 进程标签 |    |

销毁进程上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.26. `mpo_destroy_vnode_label`

```c
void mpo_destroy_vnode_label(struct label *label);
```

| 参数      | 描述   | 锁定 |
| ------- | ---- | -- |
| `label` | 进程标签 |    |

销毁 vnode 上的标签。在此入口点，策略模块应释放与 `label` 相关的任何内部存储，以便将其销毁。

#### 6.7.2.27. `mpo_copy_mbuf_label`

```c
void mpo_copy_mbuf_label(struct label *src, struct label *dest);
```

| 参数     | 描述   | 锁定 |
| ------ | ---- | -- |
| `src`  | 源标签  |    |
| `dest` | 目标标签 |    |

将 `src` 中的标签信息复制到 `dest`。

#### 6.7.2.28. `mpo_copy_pipe_label`

```c
void mpo_copy_pipe_label(struct label *src, struct label *dest);
```

| 参数     | 描述   | 锁定 |
| ------ | ---- | -- |
| `src`  | 源标签  |    |
| `dest` | 目标标签 |    |

将 `src` 中的标签信息复制到 `dest`。

#### 6.7.2.29. `mpo_copy_vnode_label`

```c
void mpo_copy_vnode_label(struct label *src, struct label *dest);
```

| 参数     | 描述   | 锁定 |
| ------ | ---- | -- |
| `src`  | 源标签  |    |
| `dest` | 目标标签 |    |

将 `src` 中的标签信息复制到 `dest`。

#### 6.7.2.30. `mpo_externalize_cred_label`

```c
int mpo_externalize_cred_label(struct label *label, char *element_name,
    struct sbuf *sb, int *claimed);
```

| 参数             | 描述                            | 锁定 |
| -------------- | ----------------------------- | -- |
| `label`        | 要外部化的标签                       |    |
| `element_name` | 需要外部化标签的策略名称                  |    |
| `sb`           | 用于填充标签文本表示的字符串缓冲区             |    |
| `claimed`      | 当 `element_data` 可以填充时，应该递增该值 |    |

根据传递的标签结构生成外部化标签。外部化标签是标签内容的文本表示，可以供用户态应用程序使用，并由用户读取。目前，所有策略的 `externalize` 入口点都会被调用，因此实现应检查 `element_name` 的内容，在尝试填充 `sb` 之前。如果 `element_name` 与您的策略名称不匹配，直接返回 0。只有在外部化标签数据时发生错误时，才返回非零值。一旦策略填充了 `element_data`，`*claimed` 应递增。

#### 6.7.2.31. `mpo_externalize_ifnet_label`

```c
int mpo_externalize_ifnet_label(struct label *label, char *element_name,
    struct sbuf *sb, int *claimed);
```

| 参数             | 描述                            | 锁定 |
| -------------- | ----------------------------- | -- |
| `label`        | 要外部化的标签                       |    |
| `element_name` | 需要外部化标签的策略名称                  |    |
| `sb`           | 用于填充标签文本表示的字符串缓冲区             |    |
| `claimed`      | 当 `element_data` 可以填充时，应该递增该值 |    |

根据传递的标签结构生成外部化标签。外部化标签是标签内容的文本表示，可以供用户态应用程序使用，并由用户读取。目前，所有策略的 `externalize` 入口点都会被调用，因此实现应检查 `element_name` 的内容，在尝试填充 `sb` 之前。如果 `element_name` 与您的策略名称不匹配，直接返回 0。只有在外部化标签数据时发生错误时，才返回非零值。一旦策略填充了 `element_data`，`*claimed` 应递增。

#### 6.7.2.32. `mpo_externalize_pipe_label`

```c
int mpo_externalize_pipe_label(struct label *label, char *element_name,
    struct sbuf *sb, int *claimed);
```

| 参数             | 描述                            | 锁定 |
| -------------- | ----------------------------- | -- |
| `label`        | 要外部化的标签                       |    |
| `element_name` | 需要外部化标签的策略名称                  |    |
| `sb`           | 用于填充标签文本表示的字符串缓冲区             |    |
| `claimed`      | 当 `element_data` 可以填充时，应该递增该值 |    |

根据传递的标签结构生成外部化标签。外部化标签是标签内容的文本表示，可以供用户态应用程序使用，并由用户读取。目前，所有策略的 `externalize` 入口点都会被调用，因此实现应检查 `element_name` 的内容，在尝试填充 `sb` 之前。如果 `element_name` 与您的策略名称不匹配，直接返回 0。只有在外部化标签数据时发生错误时，才返回非零值。一旦策略填充了 `element_data`，`*claimed` 应递增。

#### 6.7.2.33. `mpo_externalize_socket_label`

```c
int mpo_externalize_socket_label(struct label *label, char *element_name,
    struct sbuf *sb, int *claimed);
```

| 参数             | 描述                            | 锁定 |
| -------------- | ----------------------------- | -- |
| `label`        | 要外部化的标签                       |    |
| `element_name` | 需要外部化标签的策略名称                  |    |
| `sb`           | 用于填充标签文本表示的字符串缓冲区             |    |
| `claimed`      | 当 `element_data` 可以填充时，应该递增该值 |    |

根据传递的标签结构生成外部化标签。外部化标签是标签内容的文本表示，可以供用户态应用程序使用，并由用户读取。目前，所有策略的 `externalize` 入口点都会被调用，因此实现应检查 `element_name` 的内容，在尝试填充 `sb` 之前。如果 `element_name` 与您的策略名称不匹配，直接返回 0。只有在外部化标签数据时发生错误时，才返回非零值。一旦策略填充了 `element_data`，`*claimed` 应递增。

#### 6.7.2.34. `mpo_externalize_socket_peer_label`

```c
int mpo_externalize_socket_peer_label(struct label *label, char *element_name,
    struct sbuf *sb, int *claimed);
```

| 参数             | 描述                            | 锁定 |
| -------------- | ----------------------------- | -- |
| `label`        | 要外部化的标签                       |    |
| `element_name` | 需要外部化标签的策略名称                  |    |
| `sb`           | 用于填充标签文本表示的字符串缓冲区             |    |
| `claimed`      | 当 `element_data` 可以填充时，应该递增该值 |    |

根据传递的标签结构生成外部化标签。外部化标签是标签内容的文本表示，可以供用户态应用程序使用，并由用户读取。目前，所有策略的 `externalize` 入口点都会被调用，因此实现应检查 `element_name` 的内容，在尝试填充 `sb` 之前。如果 `element_name` 与您的策略名称不匹配，直接返回 0。只有在外部化标签数据时发生错误时，才返回非零值。一旦策略填充了 `element_data`，`*claimed` 应递增。

#### 6.7.2.35. `mpo_externalize_vnode_label`

```c
int mpo_externalize_vnode_label(struct label *label, char *element_name,
    struct sbuf *sb, int *claimed);
```

| 参数             | 描述                            | 锁定 |
| -------------- | ----------------------------- | -- |
| `label`        | 要外部化的标签                       |    |
| `element_name` | 需要外部化标签的策略名称                  |    |
| `sb`           | 用于填充标签文本表示的字符串缓冲区             |    |
| `claimed`      | 当 `element_data` 可以填充时，应该递增该值 |    |

根据传递的标签结构生成外部化标签。外部化标签是标签内容的文本表示，可以供用户态应用程序使用，并由用户读取。目前，所有策略的 `externalize` 入口点都会被调用，因此实现应检查 `element_name` 的内容，在尝试填充 `sb` 之前。如果 `element_name` 与您的策略名称不匹配，直接返回 0。只有在外部化标签数据时发生错误时，才返回非零值。一旦策略填充了 `element_data`，`*claimed` 应递增。

#### 6.7.2.36. `mpo_internalize_cred_label`

```c
int mpo_internalize_cred_label(struct label *label, char *element_name,
    char *element_data, int *claimed);
```

| 参数             | 描述                 | 锁定 |
| -------------- | ------------------ | -- |
| `label`        | 要填充的标签             |    |
| `element_name` | 需要内部化标签的策略名称       |    |
| `element_data` | 需要内部化的文本数据         |    |
| `claimed`      | 当数据能够成功内部化时，应该递增该值 |    |

根据外部化标签数据的文本格式生成内部标签结构。目前，所有策略的 `internalize` 入口点都会在请求内部化时被调用，因此实现应将 `element_name` 的内容与其策略名称进行比较，以确保应该内部化 `element_data` 中的数据。与 `externalize` 入口点一样，如果 `element_name` 与策略名称不匹配，则返回 0；如果数据能够成功内部化，则递增 `*claimed`。

#### 6.7.2.37. `mpo_internalize_ifnet_label`

```c
int mpo_internalize_ifnet_label(struct label *label, char *element_name,
    char *element_data, int *claimed);
```

| 参数             | 描述                 | 锁定 |
| -------------- | ------------------ | -- |
| `label`        | 要填充的标签             |    |
| `element_name` | 需要内部化标签的策略名称       |    |
| `element_data` | 需要内部化的文本数据         |    |
| `claimed`      | 当数据能够成功内部化时，应该递增该值 |    |

根据外部化标签数据的文本格式生成内部标签结构。目前，所有策略的 `internalize` 入口点都会在请求内部化时被调用，因此实现应将 `element_name` 的内容与其策略名称进行比较，以确保应该内部化 `element_data` 中的数据。与 `externalize` 入口点一样，如果 `element_name` 与策略名称不匹配，则返回 0；如果数据能够成功内部化，则递增 `*claimed`。

#### 6.7.2.38. `mpo_internalize_pipe_label`

```c
int mpo_internalize_pipe_label(struct label *label, char *element_name,
    char *element_data, int *claimed);
```

| 参数             | 描述                 | 锁定 |
| -------------- | ------------------ | -- |
| `label`        | 要填充的标签             |    |
| `element_name` | 需要内部化标签的策略名称       |    |
| `element_data` | 需要内部化的文本数据         |    |
| `claimed`      | 当数据能够成功内部化时，应该递增该值 |    |

根据外部化标签数据的文本格式生成内部标签结构。目前，所有策略的 `internalize` 入口点都会在请求内部化时被调用，因此实现应将 `element_name` 的内容与其策略名称进行比较，以确保应该内部化 `element_data` 中的数据。与 `externalize` 入口点一样，如果 `element_name` 与策略名称不匹配，则返回 0；如果数据能够成功内部化，则递增 `*claimed`。

#### 6.7.2.39. `mpo_internalize_socket_label`

```c
int mpo_internalize_socket_label(struct label *label, char *element_name,
    char *element_data, int *claimed);
```

| 参数             | 描述                 | 锁定 |
| -------------- | ------------------ | -- |
| `label`        | 要填充的标签             |    |
| `element_name` | 需要内部化标签的策略名称       |    |
| `element_data` | 需要内部化的文本数据         |    |
| `claimed`      | 当数据能够成功内部化时，应该递增该值 |    |

根据外部化标签数据的文本格式生成内部标签结构。目前，所有策略的 `internalize` 入口点都会在请求内部化时被调用，因此实现应将 `element_name` 的内容与其策略名称进行比较，以确保应该内部化 `element_data` 中的数据。与 `externalize` 入口点一样，如果 `element_name` 与策略名称不匹配，则返回 0；如果数据能够成功内部化，则递增 `*claimed`。

#### 6.7.2.40. `mpo_internalize_vnode_label`

```c
int mpo_internalize_vnode_label(struct label *label, char *element_name,
    char *element_data, int *claimed);
```

| 参数             | 描述                 | 锁定 |
| -------------- | ------------------ | -- |
| `label`        | 要填充的标签             |    |
| `element_name` | 需要内部化标签的策略名称       |    |
| `element_data` | 需要内部化的文本数据         |    |
| `claimed`      | 当数据能够成功内部化时，应该递增该值 |    |

根据外部化标签数据的文本格式生成内部标签结构。目前，所有策略的 `internalize` 入口点都会在请求内部化时被调用，因此实现应将 `element_name` 的内容与其策略名称进行比较，以确保应该内部化 `element_data` 中的数据。与 `externalize` 入口点一样，如果 `element_name` 与策略名称不匹配，则返回 0；如果数据能够成功内部化，则递增 `*claimed`。

### 6.7.3. Label Events

这一类入口点由 MAC 框架使用，允许策略维护内核对象的标签信息。对于每个对 MAC 策略感兴趣的被标记内核对象，可以为相关生命周期事件注册入口点。所有对象都实现了初始化、创建和销毁挂钩。一些对象还实现了重新标记，允许用户进程更改对象上的标签。某些对象还实现了特定于对象的事件，例如与 IP 重组相关的标签事件。一个典型的标记对象会有以下生命周期的入口点：

```c
标签初始化                o
(特定对象等待)              \
标签创建                    o
                             \
重新标记事件,                 o--<--.
各种特定对象的,              |     |
访问控制事件                 ~-->--o
                                       \
标签销毁                       o
```

标签初始化允许策略在没有对象使用上下文的情况下分配内存并设置标签的初始值。分配给策略的标签槽将默认被清零，因此某些策略可能不需要执行初始化。

标签创建发生在内核结构与实际内核对象关联时。例如，mbuf 可能会被分配并保持未使用状态，直到需要它们。mbuf 分配会导致 mbuf 上的标签初始化，但 mbuf 创建发生在 mbuf 与数据报文关联时。通常，在创建事件中会提供上下文，包括创建的情况以及创建过程中其他相关对象的标签。例如，当一个 mbuf 从套接字创建时，套接字及其标签将与新创建的 mbuf 及其标签一起呈现给注册的策略。创建事件中不鼓励进行内存分配，因为它可能发生在内核性能敏感部分；此外，创建调用不允许失败，因此内存分配失败无法报告。

特定对象事件通常不属于标签事件的其他广泛类别，但通常会提供在额外上下文的基础上修改或更新对象标签的机会。例如，IP 分片重组队列上的标签可能会在 MAC\_UPDATE\_IPQ 入口点中更新，这是由于接受额外的 mbuf 到该队列中。

访问控制事件将在以下部分详细讨论。

标签销毁允许策略释放与标签相关的存储或状态，以便支持对象的内核数据结构可以被重用或释放。

除了与特定内核对象关联的标签之外，还有一种额外的标签类别：临时标签。这些标签用于存储由用户进程提交的更新信息。这些标签与其他标签类型一样进行初始化和销毁，但创建事件是 MAC\_INTERNALIZE，接受用户标签并将其转换为内核表示。

#### 6.7.3.1. File System Object Labeling Event Operations

##### 6.7.3.1.1. `mpo_associate_vnode_devfs`

```c
void mpo_associate_vnode_devfs(struct mount *mp, struct label *fslabel,
    struct devfs_dirent *de, struct label *delabel, struct vnode *vp,
    struct label *vlabel);
```

| 参数        | 描述                             | 锁定 |
| --------- | ------------------------------ | -- |
| `mp`      | Devfs 挂载点                      |    |
| `fslabel` | Devfs 文件系统标签（`mp→mnt_fslabel`） |    |
| `de`      | Devfs 目录项                      |    |
| `delabel` | 与 `de` 关联的策略标签                 |    |
| `vp`      | 与 `de` 关联的 vnode               |    |
| `vlabel`  | 与 `vp` 关联的策略标签                 |    |

根据传入的 Devfs 目录项 `de` 及其标签，填充新创建的 Devfs vnode 的标签 (`vlabel`)。

##### 6.7.3.1.2. `mpo_associate_vnode_extattr`

```c
int mpo_associate_vnode_extattr(struct mount *mp, struct label *fslabel,
    struct vnode *vp, struct label *vlabel);
```

| 参数        | 描述             | 锁定 |
| --------- | -------------- | -- |
| `mp`      | 文件系统挂载点        |    |
| `fslabel` | 文件系统标签         |    |
| `vp`      | 要标记的 vnode     |    |
| `vlabel`  | 与 `vp` 关联的策略标签 |    |

尝试从文件系统扩展属性中检索 `vp` 的标签。成功时返回值为 `0`。如果不支持扩展属性检索，则可以接受的备用方案是将 `fslabel` 复制到 `vlabel` 中。如果发生错误，则应返回适当的 `errno` 值。

##### 6.7.3.1.3. `mpo_associate_vnode_singlelabel`

```c
void mpo_associate_vnode_singlelabel(struct mount *mp, struct label *fslabel,
    struct vnode *vp, struct label *vlabel);
```

| 参数        | 描述             | 锁定 |
| --------- | -------------- | -- |
| `mp`      | 文件系统挂载点        |    |
| `fslabel` | 文件系统标签         |    |
| `vp`      | 要标记的 vnode     |    |
| `vlabel`  | 与 `vp` 关联的策略标签 |    |

在非多标签文件系统上，调用此入口点根据文件系统标签 `fslabel` 设置 `vp` 的策略标签。

##### 6.7.3.1.4. `mpo_create_devfs_device`

```c
void mpo_create_devfs_device(dev_t dev, struct devfs_dirent *devfs_dirent,
    struct label *label);
```

| 参数             | 描述                     | 锁定 |
| -------------- | ---------------------- | -- |
| `dev`          | 与 `devfs_dirent` 相关的设备 |    |
| `devfs_dirent` | 要标记的 Devfs 目录项         |    |
| `label`        | 要填充的 `devfs_dirent` 标签 |    |

为传入的设备创建的 `devfs_dirent` 填充标签。此调用会在设备文件系统挂载、重新生成或新设备可用时触发。

##### 6.7.3.1.5. `mpo_create_devfs_directory`

```c
void mpo_create_devfs_directory(char *dirname, int dirnamelen,
    struct devfs_dirent *devfs_dirent, struct label *label);
```

| 参数             | 描述                | 锁定 |
| -------------- | ----------------- | -- |
| `dirname`      | 被创建目录的名称          |    |
| `namelen`      | 字符串 `dirname` 的长度 |    |
| `devfs_dirent` | 被创建的目录的 Devfs 目录项 |    |

为传入的目录创建的 `devfs_dirent` 填充标签。此调用会在设备文件系统挂载、重新生成或新设备需要特定目录层次时触发。

##### 6.7.3.1.6. `mpo_create_devfs_symlink`

```c
void mpo_create_devfs_symlink(struct ucred *cred, struct mount *mp,
    struct devfs_dirent *dd, struct label *ddlabel, struct devfs_dirent *de,
    struct label *delabel);
```

| 参数        | 描述                 | 锁定 |
| --------- | ------------------ | -- |
| `cred`    | 主体凭证               |    |
| `mp`      | Devfs 挂载点          |    |
| `dd`      | 符号链接的目标            |    |
| `ddlabel` | 与目标 `dd` 关联的标签     |    |
| `de`      | 符号链接条目             |    |
| `delabel` | 与符号链接条目 `de` 关联的标签 |    |

为新创建的 Devfs 符号链接条目填充标签 (`delabel`)。

##### 6.7.3.1.7. `mpo_create_vnode_extattr`

```c
int mpo_create_vnode_extattr(struct ucred *cred, struct mount *mp,
    struct label *fslabel, struct vnode *dvp, struct label *dlabel,
    struct vnode *vp, struct label *vlabel, struct componentname *cnp);
```

| 参数        | 描述               | 锁定 |
| --------- | ---------------- | -- |
| `cred`    | 主体凭证             |    |
| `mp`      | 文件系统挂载点          |    |
| `fslabel` | 文件系统标签           |    |
| `dvp`     | 父目录 vnode        |    |
| `dlabel`  | 与父目录 `dvp` 关联的标签 |    |
| `vp`      | 新创建的 vnode       |    |
| `vlabel`  | 与 `vp` 关联的策略标签   |    |
| `cnp`     | `vp` 的组件名称       |    |

将 `vp` 的标签写入适当的扩展属性。如果写入成功，填充 `vlabel` 并返回 `0`。如果失败，返回适当的错误值。

##### 6.7.3.1.8. `mpo_create_mount`

```c
void mpo_create_mount(struct ucred *cred, struct mount *mp, struct label *mnt,
    struct label *fslabel);
```

| 参数         | 描述          | 锁定 |
| ---------- | ----------- | -- |
| `cred`     | 主体凭证        |    |
| `mp`       | 被挂载的文件系统对象  |    |
| `mntlabel` | 填充的文件系统挂载标签 |    |
| `fslabel`  | 文件系统的策略标签   |    |

根据传入的主体凭证填充挂载点的标签。此调用会在新文件系统挂载时触发。

##### 6.7.3.1.9. `mpo_create_root_mount`

```c
void mpo_create_root_mount(struct ucred *cred, struct mount *mp,
    struct label *mntlabel, struct label *fslabel);
```

| 参数                                                                                                 | 描述 | 锁定 |
| -------------------------------------------------------------------------------------------------- | -- | -- |
| 参见 [`mpo_create_mount`](https://docs.freebsd.org/en/books/arch-handbook/mac/#mac-mpo-create-mount) |    |    |

根据传入的主体凭证填充根文件系统挂载点的标签。此调用会在根文件系统挂载后、`mpo_create_mount` 调用之后触发。

##### 6.7.3.1.10. `mpo_relabel_vnode`

```c
void mpo_relabel_vnode(struct ucred *cred, struct vnode *vp,
    struct label *vnodelabel, struct label *newlabel);
```

| 参数           | 描述                           | 锁定 |
| ------------ | ---------------------------- | -- |
| `cred`       | 主体凭证                         |    |
| `vp`         | 要重新标记的 vnode                 |    |
| `vnodelabel` | `vp` 当前的策略标签                 |    |
| `newlabel`   | 替换 `vnodelabel` 的新标签，可能是部分标签 |    |

根据传入的更新 vnode 标签和主体凭证更新传入的 vnode 标签。

##### 6.7.3.1.11. `mpo_setlabel_vnode_extattr`

```c
int mpo_setlabel_vnode_extattr(struct ucred *cred, struct vnode *vp,
    struct label *vlabel, struct label *intlabel);
```

| 参数         | 描述             | 锁定 |
| ---------- | -------------- | -- |
| `cred`     | 主体凭证           |    |
| `vp`       | 要写入标签的 vnode   |    |
| `vlabel`   | 与 `vp` 关联的策略标签 |    |
| `intlabel` | 要写入的标签         |    |

将 `intlabel` 中的策略标签写入扩展属性。这是从 `vop_stdcreatevnode_ea` 调用时进行的。

##### 6.7.3.1.12. `mpo_update_devfsdirent`

```c
void mpo_update_devfsdirent(struct devfs_dirent *devfs_dirent,
    struct label *direntlabel, struct vnode *vp, struct label *vnodelabel);
```

| 参数             | 描述                         | 锁定 |
| -------------- | -------------------------- | -- |
| `devfs_dirent` | Devfs 目录项                  |    |
| `direntlabel`  | 需要更新的 `devfs_dirent` 的策略标签 |    |
| `vp`           | 父目录 vnode                  | 锁定 |
| `vnodelabel`   | 与父目录 `vp` 关联的策略标签          |    |

根据传入的 devfs vnode 标签更新 `devfs_dirent` 的标签。当 devfs vnode 成功重新标记后，会调用此函数以确保标签更改持久化，即使 vnode 被回收。同时，在 devfs 创建符号链接时，也会调用此函数，在 `mac_vnode_create_from_vnode` 调用之后初始化 vnode 标签。

#### 6.7.3.2. IPC 对象标签操作

##### 6.7.3.2.1. `mpo_create_mbuf_from_socket`

```c
void mpo_create_mbuf_from_socket(struct socket *so, struct label *socketlabel,
    struct mbuf *m, struct label *mbuflabel);
```

| 参数            | 描述                 | 锁定        |
| ------------- | ------------------ | --------- |
| `socket`      | 套接字                | 套接字锁定正在进行 |
| `socketlabel` | 与 `socket` 关联的策略标签 |           |
| `m`           | mbuf 对象            |           |
| `mbuflabel`   | 为 `m` 填充的策略标签      |           |

根据传入的套接字标签设置新创建的 mbuf 头的标签。此调用在套接字生成新数据报或消息并存储到传入的 mbuf 时触发。

##### 6.7.3.2.2. `mpo_create_pipe`

```c
void mpo_create_pipe(struct ucred *cred, struct pipe *pipe,
    struct label *pipelabel);
```

| 参数          | 描述               | 锁定 |
| ----------- | ---------------- | -- |
| `cred`      | 主体凭证             |    |
| `pipe`      | 管道               |    |
| `pipelabel` | 与 `pipe` 关联的策略标签 |    |

根据传入的主体凭证设置新创建的管道的标签。此调用在新管道创建时触发。

##### 6.7.3.2.3. `mpo_create_socket`

```c
void mpo_create_socket(struct ucred *cred, struct socket *so,
    struct label *socketlabel);
```

| 参数            | 描述           | 锁定 |
| ------------- | ------------ | -- |
| `cred`        | 主体凭证         | 不变 |
| `so`          | 要标记的套接字      |    |
| `socketlabel` | 为 `so` 填充的标签 |    |

根据传入的主体凭证为新创建的套接字设置标签。此调用在套接字创建时触发。

##### 6.7.3.2.4. `mpo_create_socket_from_socket`

```c
void mpo_create_socket_from_socket(struct socket *oldsocket,
    struct label *oldsocketlabel, struct socket *newsocket,
    struct label *newsocketlabel);
```

| 参数               | 描述                    | 锁定 |
| ---------------- | --------------------- | -- |
| `oldsocket`      | 监听套接字                 |    |
| `oldsocketlabel` | 与 `oldsocket` 关联的策略标签 |    |
| `newsocket`      | 新的套接字                 |    |
| `newsocketlabel` | 与 `newsocket` 关联的策略标签 |    |

根据 [accept(2)](https://man.freebsd.org/cgi/man.cgi?query=accept&sektion=2&format=html) 接受的新套接字 `newsocket` 为其设置标签，标签来源于 [listen(2)](https://man.freebsd.org/cgi/man.cgi?query=listen&sektion=2&format=html) 中的监听套接字 `oldsocket`。

##### 6.7.3.2.5. `mpo_relabel_pipe`

```c
void mpo_relabel_pipe(struct ucred *cred, struct pipe *pipe,
    struct label *oldlabel, struct label *newlabel);
```

| 参数         | 描述            | 锁定 |
| ---------- | ------------- | -- |
| `cred`     | 主体凭证          |    |
| `pipe`     | 管道            |    |
| `oldlabel` | 当前与管道关联的策略标签  |    |
| `newlabel` | 要应用于管道的策略标签更新 |    |

将新的标签 `newlabel` 应用到管道 `pipe` 上。

##### 6.7.3.2.6. `mpo_relabel_socket`

```c
void mpo_relabel_socket(struct ucred *cred, struct socket *so,
    struct label *oldlabel, struct label *newlabel);
```

| 参数         | 描述             | 锁定 |
| ---------- | -------------- | -- |
| `cred`     | 主体凭证           | 不变 |
| `so`       | 套接字对象          |    |
| `oldlabel` | 当前套接字 `so` 的标签 |    |
| `newlabel` | 套接字 `so` 的标签更新 |    |

根据传入的标签更新，更新套接字 `so` 的标签。

##### 6.7.3.2.7. `mpo_set_socket_peer_from_mbuf`

```c
void mpo_set_socket_peer_from_mbuf(struct mbuf *mbuf, struct label *mbuflabel,
    struct label *oldlabel, struct label *newlabel);
```

| 参数          | 描述              | 锁定 |
| ----------- | --------------- | -- |
| `mbuf`      | 通过套接字接收到的第一个数据报 |    |
| `mbuflabel` | 与 `mbuf` 关联的标签  |    |
| `oldlabel`  | 套接字当前的标签        |    |
| `newlabel`  | 要为套接字设置的对端标签    |    |

根据传入的 mbuf 标签为流式套接字设置对端标签。此调用在流式套接字接收到第一个数据报时触发，Unix 域套接字除外。

##### 6.7.3.2.8. `mpo_set_socket_peer_from_socket`

```c
void mpo_set_socket_peer_from_socket(struct socket *oldsocket,
    struct label *oldsocketlabel, struct socket *newsocket,
    struct label *newsocketpeerlabel);
```

| 参数                   | 描述                       | 锁定 |
| -------------------- | ------------------------ | -- |
| `oldsocket`          | 本地套接字                    |    |
| `oldsocketlabel`     | 与 `oldsocket` 关联的策略标签    |    |
| `newsocket`          | 对端套接字                    |    |
| `newsocketpeerlabel` | 要为 `newsocket` 填充的对端策略标签 |    |

根据传入的远程套接字端点为流式 Unix 域套接字设置对端标签。此调用将在套接字对连接时触发，并将为两个端点进行调用。

#### 6.7.3.3. 网络对象标记事件操作

##### 6.7.3.3.1. `mpo_create_bpfdesc`

```c
void mpo_create_bpfdesc(struct ucred *cred, struct bpf_d *bpf_d,
    struct label *bpflabel);
```

| 参数      | 描述                 | 锁定 |
| ------- | ------------------ | -- |
| `cred`  | 主体凭证               | 不变 |
| `bpf_d` | 对象；BPF 描述符         |    |
| `bpf`   | 要为 `bpf_d` 填充的策略标签 |    |

根据传入的主体凭证为新创建的 BPF 描述符设置标签。此调用将在进程打开 BPF 设备节点时触发。

##### 6.7.3.3.2. `mpo_create_ifnet`

```c
void mpo_create_ifnet(struct ifnet *ifnet, struct label *ifnetlabel);
```

| 参数           | 描述                 | 锁定 |
| ------------ | ------------------ | -- |
| `ifnet`      | 网络接口               |    |
| `ifnetlabel` | 要为 `ifnet` 填充的策略标签 |    |

为新创建的网络接口设置标签。此调用可能在新物理接口可用时或在启动过程中由用户操作触发，当伪接口被实例化时也会触发。

##### 6.7.3.3.3. `mpo_create_ipq`

```c
void mpo_create_ipq(struct mbuf *fragment, struct label *fragmentlabel,
    struct ipq *ipq, struct label *ipqlabel);
```

| 参数              | 描述                   | 锁定 |
| --------------- | -------------------- | -- |
| `fragment`      | 第一个接收到的 IP 数据片段      |    |
| `fragmentlabel` | 与 `fragment` 关联的策略标签 |    |
| `ipq`           | 要标记的 IP 重组队列         |    |
| `ipqlabel`      | 要为 `ipq` 填充的策略标签     |    |

根据第一个接收到的数据片段的 mbuf 标头设置新创建的 IP 重组队列的标签。

##### 6.7.3.3.4. `mpo_create_datagram_from_ipq`

```c
void mpo_create_create_datagram_from_ipq(struct ipq *ipq,
    struct label *ipqlabel, struct mbuf *datagram, struct label *datagramlabel);
```

| 参数              | 描述                    | 锁定 |
| --------------- | --------------------- | -- |
| `ipq`           | IP 重组队列               |    |
| `ipqlabel`      | 与 `ipq` 关联的策略标签       |    |
| `datagram`      | 要标记的数据报               |    |
| `datagramlabel` | 要为 `datagram` 填充的策略标签 |    |

根据 IP 重组队列中生成的数据报从中提取标签，为其设置标签。

##### 6.7.3.3.5. `mpo_create_fragment`

```c
void mpo_create_fragment(struct mbuf *datagram, struct label *datagramlabel,
    struct mbuf *fragment, struct label *fragmentlabel);
```

| 参数              | 描述                    | 锁定 |
| --------------- | --------------------- | -- |
| `datagram`      | 数据报                   |    |
| `datagramlabel` | 与 `datagram` 关联的策略标签  |    |
| `fragment`      | 要标记的数据片段              |    |
| `fragmentlabel` | 要为 `fragment` 填充的策略标签 |    |

根据数据报的 mbuf 标头的标签为新创建的 IP 数据片段的 mbuf 标头设置标签。

##### 6.7.3.3.6. `mpo_create_mbuf_from_mbuf`

```c
void mpo_create_mbuf_from_mbuf(struct mbuf *oldmbuf, struct label *oldmbuflabel,
    struct mbuf *newmbuf, struct label *newmbuflabel);
```

| 参数             | 描述                   | 锁定 |
| -------------- | -------------------- | -- |
| `oldmbuf`      | 现有的（源）mbuf           |    |
| `oldmbuflabel` | 与 `oldmbuf` 关联的策略标签  |    |
| `newmbuf`      | 新的 mbuf 要被标记         |    |
| `newmbuflabel` | 要为 `newmbuf` 填充的策略标签 |    |

根据现有数据报的 mbuf 标头为新创建的数据报的 mbuf 标头设置标签。此调用可能在多种情况下触发，包括当 mbuf 因对齐目的而重新分配时。

##### 6.7.3.3.7. `mpo_create_mbuf_linklayer`

```c
void mpo_create_mbuf_linklayer(struct ifnet *ifnet, struct label *ifnetlabel,
    struct mbuf *mbuf, struct label *mbuflabel);
```

| 参数           | 描述                | 锁定 |
| ------------ | ----------------- | -- |
| `ifnet`      | 网络接口              |    |
| `ifnetlabel` | 与 `ifnet` 关联的策略标签 |    |
| `mbuf`       | 新数据报的 mbuf 标头     |    |
| `mbuflabel`  | 要为 `mbuf` 填充的策略标签 |    |

根据传入的网络接口为新创建的数据报设置 mbuf 标头标签。此调用可能在多种情况下触发，包括 ARP 或 IPv4 和 IPv6 堆栈中的 ND6 响应。

##### 6.7.3.3.8. `mpo_create_mbuf_from_bpfdesc`

```c
void mpo_create_mbuf_from_bpfdesc(struct bpf_d *bpf_d, struct label *bpflabel,
    struct mbuf *mbuf, struct label *mbuflabel);
```

| 参数          | 描述                | 锁定 |
| ----------- | ----------------- | -- |
| `bpf_d`     | BPF 描述符           |    |
| `bpflabel`  | 与 `bpf_d` 关联的策略标签 |    |
| `mbuf`      | 新的 mbuf 要被标记      |    |
| `mbuflabel` | 要为 `mbuf` 填充的策略标签 |    |

根据传入的 BPF 描述符为新创建的数据报设置 mbuf 标头标签。此调用在对与传入的 BPF 描述符关联的 BPF 设备执行写操作时触发。

##### 6.7.3.3.9. `mpo_create_mbuf_from_ifnet`

```c
void mpo_create_mbuf_from_ifnet(struct ifnet *ifnet, struct label *ifnetlabel,
    struct mbuf *mbuf, struct label *mbuflabel);
```

| 参数           | 描述                | 锁定 |
| ------------ | ----------------- | -- |
| `ifnet`      | 网络接口              |    |
| `ifnetlabel` | 与 `ifnet` 关联的策略标签 |    |
| `mbuf`       | 新数据报的 mbuf 标头     |    |
| `mbuflabel`  | 要为 `mbuf` 填充的策略标签 |    |

根据传入的网络接口为新创建的数据报设置 mbuf 标头标签。

##### 6.7.3.3.10. `mpo_create_mbuf_multicast_encap`

```c
void mpo_create_mbuf_multicast_encap(struct mbuf *oldmbuf,
    struct label *oldmbuflabel, struct ifnet *ifnet, struct label *ifnetlabel,
    struct mbuf *newmbuf, struct label *newmbuflabel);
```

| 参数             | 描述                   | 锁定 |
| -------------- | -------------------- | -- |
| `oldmbuf`      | 现有数据报的 mbuf 标头       |    |
| `oldmbuflabel` | 与 `oldmbuf` 关联的策略标签  |    |
| `ifnet`        | 网络接口                 |    |
| `ifnetlabel`   | 与 `ifnet` 关联的策略标签    |    |
| `newmbuf`      | 要标记的新数据报的 mbuf 标头    |    |
| `newmbuflabel` | 要为 `newmbuf` 填充的策略标签 |    |

根据现有数据报生成的新数据报，在经过传入的多播封装接口处理时设置其 mbuf 标头标签。此调用发生在 mbuf 通过虚拟接口交付时。

##### 6.7.3.3.11. `mpo_create_mbuf_netlayer`

```c
void mpo_create_mbuf_netlayer(struct mbuf *oldmbuf, struct label *oldmbuflabel,
    struct mbuf *newmbuf, struct label *newmbuflabel);
```

| 参数             | 描述                  | 锁定 |
| -------------- | ------------------- | -- |
| `oldmbuf`      | 接收到的数据报             |    |
| `oldmbuflabel` | 与 `oldmbuf` 关联的策略标签 |    |
| `newmbuf`      | 新创建的数据报             |    |
| `newmbuflabel` | 与 `newmbuf` 关联的策略标签 |    |

根据现有接收到的数据报（`oldmbuf`）生成的新数据报，设置其 mbuf 标头标签。此调用可能在多种情况下触发，包括响应 ICMP 请求数据报时。

##### 6.7.3.3.12. `mpo_fragment_match`

```c
int mpo_fragment_match(struct mbuf *fragment, struct label *fragmentlabel,
    struct ipq *ipq, struct label *ipqlabel);
```

| 参数              | 描述                   | 锁定 |
| --------------- | -------------------- | -- |
| `fragment`      | IP 数据报片段             |    |
| `fragmentlabel` | 与 `fragment` 关联的策略标签 |    |
| `ipq`           | IP 片段重组队列            |    |
| `ipqlabel`      | 与 `ipq` 关联的策略标签      |    |

确定包含 IP 数据报片段（`fragment`）的 mbuf 标头是否与传入的 IP 片段重组队列（`ipq`）的标签匹配。如果匹配，则返回（1）；否则返回（0）。此调用发生在 IP 堆栈尝试为新接收的片段找到现有的片段重组队列时；如果失败，可能会为该片段实例化一个新的片段重组队列。策略可以使用此入口点，防止在策略不允许基于标签或其他信息重组匹配的 IP 片段时进行重组。

##### 6.7.3.3.13. `mpo_relabel_ifnet`

```c
void mpo_relabel_ifnet(struct ucred *cred, struct ifnet *ifnet,
    struct label *ifnetlabel, struct label *newlabel);
```

| 参数           | 描述                 | 锁定 |
| ------------ | ------------------ | -- |
| `cred`       | 主体凭证               |    |
| `ifnet`      | 对象；网络接口            |    |
| `ifnetlabel` | 与 `ifnet` 关联的策略标签  |    |
| `newlabel`   | 要应用于 `ifnet` 的标签更新 |    |

根据传入的更新标签（`newlabel`）和传入的主体凭证（`cred`），更新网络接口 `ifnet` 的标签。

##### 6.7.3.3.14. `mpo_update_ipq`

```c
void mpo_update_ipq(struct mbuf *fragment, struct label *fragmentlabel,
    struct ipq *ipq, struct label *ipqlabel);
```

| 参数          | 描述               | 锁定 |
| ----------- | ---------------- | -- |
| `mbuf`      | IP 片段            |    |
| `mbuflabel` | 与 `mbuf` 关联的策略标签 |    |
| `ipq`       | IP 片段重组队列        |    |
| `ipqlabel`  | 要更新的 `ipq` 的策略标签 |    |

根据接收的 IP 片段 mbuf 标头（`mbuf`），更新 IP 片段重组队列（`ipq`）的标签。

#### 6.7.3.4. 进程标签事件操作

##### 6.7.3.4.1. `mpo_create_cred`

```c
void mpo_create_cred(struct ucred *parent_cred, struct ucred *child_cred);
```

| 参数            | 描述    | 锁定 |
| ------------- | ----- | -- |
| `parent_cred` | 父主体凭证 |    |
| `child_cred`  | 子主体凭证 |    |

根据传入的主体凭证为新创建的主体凭证设置标签。当调用 [crcopy(9)](https://man.freebsd.org/cgi/man.cgi?query=crcopy&sektion=9&format=html) 时，此调用将在新创建的 `struct ucred` 上触发。此调用不应与进程的分叉或创建事件混淆。

##### 6.7.3.4.2. `mpo_execve_transition`

```c
void mpo_execve_transition(struct ucred *old, struct ucred *new,
    struct vnode *vp, struct label *vnodelabel);
```

| 参数           | 描述             | 锁定  |
| ------------ | -------------- | --- |
| `old`        | 现有主体凭证         | 不可变 |
| `new`        | 要标记的新主体凭证      |     |
| `vp`         | 要执行的文件         | 已锁定 |
| `vnodelabel` | 与 `vp` 关联的策略标签 |     |

根据执行传入的 vnode（`vp`）引起的标签过渡，从传入的现有主体凭证（`old`）更新新创建的主体凭证（`new`）的标签。此调用在进程执行传入的 vnode 时发生，并且如果某个策略从 `mpo_execve_will_transition` 入口点返回成功，才会触发。策略可以选择通过调用 `mpo_create_cred` 并传递这两个主体凭证来实现此调用，而无需实现过渡事件。如果策略实现了 `mpo_create_cred`，则不应将此入口点留空，即使它们没有实现 `mpo_execve_will_transition`。

##### 6.7.3.4.3. `mpo_execve_will_transition`

```c
int mpo_execve_will_transition(struct ucred *old, struct vnode *vp,
    struct label *vnodelabel);
```

| 参数           | 描述                                                                                           | 锁定  |
| ------------ | -------------------------------------------------------------------------------------------- | --- |
| `old`        | [execve(2)](https://man.freebsd.org/cgi/man.cgi?query=execve&sektion=2&format=html) 调用前的主体凭证 | 不可变 |
| `vp`         | 要执行的文件                                                                                       |     |
| `vnodelabel` | 与 `vp` 关联的策略标签                                                                               |     |

确定策略是否希望因执行传入的 vnode（`vp`）而进行标签过渡。若需要过渡，则返回 1；否则返回 0。即使策略返回 0，它在遇到 `mpo_execve_transition` 被意外调用时也应表现正确，因为该调用可能是由另一个策略请求过渡时触发的。

##### 6.7.3.4.4. `mpo_create_proc0`

```c
void mpo_create_proc0(struct ucred *cred);
```

| 参数     | 描述       | 锁定 |
| ------ | -------- | -- |
| `cred` | 要填充的主体凭证 |    |

创建进程 0 的主体凭证，进程 0 是所有内核进程的父进程。

##### 6.7.3.4.5. `mpo_create_proc1`

```c
void mpo_create_proc1(struct ucred *cred);
```

| 参数     | 描述       | 锁定 |
| ------ | -------- | -- |
| `cred` | 要填充的主体凭证 |    |

创建进程 1 的主体凭证，进程 1 是所有用户进程的父进程。

##### 6.7.3.4.6. `mpo_relabel_cred`

```c
void mpo_relabel_cred(struct ucred *cred, struct label *newlabel);
```

| 参数         | 描述                | 锁定 |
| ---------- | ----------------- | -- |
| `cred`     | 主体凭证              |    |
| `newlabel` | 要应用于 `cred` 的标签更新 |    |

根据传入的更新标签（`newlabel`），更新主体凭证（`cred`）的标签。

### 6.7.4. 访问控制检查

访问控制入口点允许策略模块影响内核做出的访问控制决策。通常，尽管并非总是如此，访问控制入口点的参数将包括一个或多个授权凭证、可能包括标签的其他对象的信息。访问控制入口点可以返回 0 以允许操作，或者返回一个 [errno(2)](https://man.freebsd.org/cgi/man.cgi?query=errno&sektion=2&format=html) 错误值。各个注册策略模块调用该入口点的结果将按照以下方式组合：如果所有模块都允许操作成功，则返回成功。如果一个或多个模块返回失败，则返回失败。如果多个模块返回失败，则返回给用户的 errno 值将根据以下优先级选择，该优先级由 **kern\_mac.c** 中的 `error_select()` 函数实现：

| 优先级最高 | EDEADLK |
| ----- | ------- |
|       | EINVAL  |
|       | ESRCH   |
|       | EACCES  |
| 优先级最低 | EPERM   |

如果所有模块返回的错误值都不在优先级图表中，则将从该集中的任意选定值返回。通常，规则按以下顺序提供错误优先级：内核失败、无效参数、对象不存在、访问不允许、其他。

#### 6.7.4.1. `mpo_check_bpfdesc_receive`

```c
int mpo_check_bpfdesc_receive(struct bpf_d *bpf_d, struct label *bpflabel,
    struct ifnet *ifnet, struct label *ifnetlabel);
```

| 参数           | 描述            | 锁定 |
| ------------ | ------------- | -- |
| `bpf_d`      | 主体；BPF 描述符    |    |
| `bpflabel`   | `bpf_d` 的策略标签 |    |
| `ifnet`      | 对象；网络接口       |    |
| `ifnetlabel` | `ifnet` 的策略标签 |    |

确定 MAC 框架是否应允许来自传入接口的数据报传递到传入的 BPF 描述符的缓冲区。若允许，则返回 0；否则返回 `errno` 值。建议的失败值：标签不匹配时返回 EACCES，缺少权限时返回 EPERM。

#### 6.7.4.2. `mpo_check_kenv_dump`

```c
int mpo_check_kenv_dump(struct ucred *cred);
```

| 参数     | 描述   | 锁定 |
| ------ | ---- | -- |
| `cred` | 主体凭证 |    |

确定主体是否应被允许检索内核环境（参见 [kenv(2)](https://man.freebsd.org/cgi/man.cgi?query=kenv&sektion=2&format=html)）。

#### 6.7.4.3. `mpo_check_kenv_get`

```c
int mpo_check_kenv_get(struct ucred *cred, char *name);
```

| 参数     | 描述       | 锁定 |
| ------ | -------- | -- |
| `cred` | 主体凭证     |    |
| `name` | 内核环境变量名称 |    |

确定主体是否应被允许检索指定内核环境变量的值。

#### 6.7.4.4. `mpo_check_kenv_set`

```c
int mpo_check_kenv_set(struct ucred *cred, char *name);
```

| 参数     | 描述       | 锁定 |
| ------ | -------- | -- |
| `cred` | 主体凭证     |    |
| `name` | 内核环境变量名称 |    |

确定主体是否应被允许设置指定的内核环境变量。

#### 6.7.4.5. `mpo_check_kenv_unset`

```c
int mpo_check_kenv_unset(struct ucred *cred, char *name);
```

| 参数     | 描述       | 锁定 |
| ------ | -------- | -- |
| `cred` | 主体凭证     |    |
| `name` | 内核环境变量名称 |    |

确定主体是否应被允许取消设置指定的内核环境变量。

#### 6.7.4.6. `mpo_check_kld_load`

```c
int mpo_check_kld_load(struct ucred *cred, struct vnode *vp,
    struct label *vlabel);
```

| 参数       | 描述           | 锁定 |
| -------- | ------------ | -- |
| `cred`   | 主体凭证         |    |
| `vp`     | 内核模块 vnode   |    |
| `vlabel` | 与 `vp` 关联的标签 |    |

确定主体是否应被允许加载指定的模块文件。

#### 6.7.4.7. `mpo_check_kld_stat`

```c
int mpo_check_kld_stat(struct ucred *cred);
```

| 参数     | 描述   | 锁定 |
| ------ | ---- | -- |
| `cred` | 主体凭证 |    |

确定主体是否应被允许检索已加载的内核模块文件及其相关统计信息。

#### 6.7.4.8. `mpo_check_kld_unload`

```c
int mpo_check_kld_unload(struct ucred *cred);
```

| 参数     | 描述   | 锁定 |
| ------ | ---- | -- |
| `cred` | 主体凭证 |    |

确定主体是否应被允许卸载内核模块。

#### 6.7.4.9. `mpo_check_pipe_ioctl`

```c
int mpo_check_pipe_ioctl(struct ucred *cred, struct pipe *pipe,
    struct label *pipelabel, unsigned long cmd, void *data);
```

| 参数          | 描述                                                                                   | 锁定 |
| ----------- | ------------------------------------------------------------------------------------ | -- |
| `cred`      | 主体凭证                                                                                 |    |
| `pipe`      | 管道                                                                                   |    |
| `pipelabel` | 与 `pipe` 关联的策略标签                                                                     |    |
| `cmd`       | [ioctl(2)](https://man.freebsd.org/cgi/man.cgi?query=ioctl&sektion=2&format=html) 命令 |    |
| `data`      | [ioctl(2)](https://man.freebsd.org/cgi/man.cgi?query=ioctl&sektion=2&format=html) 数据 |    |

确定主体是否应被允许执行指定的 [ioctl(2)](https://man.freebsd.org/cgi/man.cgi?query=ioctl&sektion=2&format=html) 调用。

#### 6.7.4.10. `mpo_check_pipe_poll`

```c
int mpo_check_pipe_poll(struct ucred *cred, struct pipe *pipe,
    struct label *pipelabel);
```

| 参数          | 描述               | 锁定 |
| ----------- | ---------------- | -- |
| `cred`      | 主体凭证             |    |
| `pipe`      | 管道               |    |
| `pipelabel` | 与 `pipe` 关联的策略标签 |    |

确定主体是否应被允许对 `pipe` 进行轮询操作。

#### 6.7.4.11. `mpo_check_pipe_read`

```c
int mpo_check_pipe_read(struct ucred *cred, struct pipe *pipe,
    struct label *pipelabel);
```

| 参数          | 描述               | 锁定 |
| ----------- | ---------------- | -- |
| `cred`      | 主体凭证             |    |
| `pipe`      | 管道               |    |
| `pipelabel` | 与 `pipe` 关联的策略标签 |    |

确定主体是否应被允许读取 `pipe`。

#### 6.7.4.12. `mpo_check_pipe_relabel`

```c
int mpo_check_pipe_relabel(struct ucred *cred, struct pipe *pipe,
    struct label *pipelabel, struct label *newlabel);
```

| 参数          | 描述                     | 锁定 |
| ----------- | ---------------------- | -- |
| `cred`      | 主体凭证                   |    |
| `pipe`      | 管道                     |    |
| `pipelabel` | 当前与 `pipe` 关联的策略标签     |    |
| `newlabel`  | 更新后的标签，应用于 `pipelabel` |    |

确定主体是否应被允许重新标记 `pipe`。

#### 6.7.4.13. `mpo_check_pipe_stat`

```c
int mpo_check_pipe_stat(struct ucred *cred, struct pipe *pipe,
    struct label *pipelabel);
```

| 参数          | 描述               | 锁定 |
| ----------- | ---------------- | -- |
| `cred`      | 主体凭证             |    |
| `pipe`      | 管道               |    |
| `pipelabel` | 与 `pipe` 关联的策略标签 |    |

确定主体是否应被允许检索与 `pipe` 相关的统计信息。

#### 6.7.4.14. `mpo_check_pipe_write`

```c
int mpo_check_pipe_write(struct ucred *cred, struct pipe *pipe,
    struct label *pipelabel);
```

| 参数          | 描述               | 锁定 |
| ----------- | ---------------- | -- |
| `cred`      | 主体凭证             |    |
| `pipe`      | 管道               |    |
| `pipelabel` | 与 `pipe` 关联的策略标签 |    |

确定主体是否应被允许向 `pipe` 写入数据。

#### 6.7.4.15. `mpo_check_socket_bind`

```c
int mpo_check_socket_bind(struct ucred *cred, struct socket *socket,
    struct label *socketlabel, struct sockaddr *sockaddr);
```

| 参数            | 描述                 | 锁定 |
| ------------- | ------------------ | -- |
| `cred`        | 主体凭证               |    |
| `socket`      | 待绑定的套接字            |    |
| `socketlabel` | 与 `socket` 关联的策略标签 |    |
| `sockaddr`    | `socket` 的地址       |    |

#### 6.7.4.16. `mpo_check_socket_connect`

```c
int mpo_check_socket_connect(struct ucred *cred, struct socket *socket,
    struct label *socketlabel, struct sockaddr *sockaddr);
```

| 参数            | 描述                 | 锁定 |
| ------------- | ------------------ | -- |
| `cred`        | 主体凭证               |    |
| `socket`      | 待连接的套接字            |    |
| `socketlabel` | 与 `socket` 关联的策略标签 |    |
| `sockaddr`    | `socket` 的地址       |    |

确定主体凭证 (`cred`) 是否可以将传入的套接字 (`socket`) 连接到传入的套接字地址 (`sockaddr`)。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），EPERM（缺乏权限）。

#### 6.7.4.17. `mpo_check_socket_receive`

```c
int mpo_check_socket_receive(struct ucred *cred, struct socket *so,
    struct label *socketlabel);
```

| 参数            | 描述             | 锁定 |
| ------------- | -------------- | -- |
| `cred`        | 主体凭证           |    |
| `so`          | 套接字            |    |
| `socketlabel` | 与 `so` 关联的策略标签 |    |

确定主体是否应被允许从套接字 `so` 接收信息。

#### 6.7.4.18. `mpo_check_socket_send`

```c
int mpo_check_socket_send(struct ucred *cred, struct socket *so,
    struct label *socketlabel);
```

| 参数            | 描述             | 锁定 |
| ------------- | -------------- | -- |
| `cred`        | 主体凭证           |    |
| `so`          | 套接字            |    |
| `socketlabel` | 与 `so` 关联的策略标签 |    |

确定主体是否应被允许通过套接字 `so` 发送信息。

#### 6.7.4.19. `mpo_check_cred_visible`

```c
int mpo_check_cred_visible(struct ucred *u1, struct ucred *u2);
```

| 参数   | 描述   | 锁定 |
| ---- | ---- | -- |
| `u1` | 主体凭证 |    |
| `u2` | 对象凭证 |    |

确定主体凭证 `u1` 是否能够“查看”其他具有传入主体凭证 `u2` 的主体。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），EPERM（缺乏权限），或 ESRCH（隐藏可见性）。此调用可在多种情况下使用，包括通过 `ps` 使用的进程间状态 sysctl，以及在 procfs 查找时。

#### 6.7.4.20. `mpo_check_socket_visible`

```c
int mpo_check_socket_visible(struct ucred *cred, struct socket *socket,
    struct label *socketlabel);
```

| 参数            | 描述                 | 锁定 |
| ------------- | ------------------ | -- |
| `cred`        | 主体凭证               |    |
| `socket`      | 对象；套接字             |    |
| `socketlabel` | 与 `socket` 关联的策略标签 |    |

#### 6.7.4.21. `mpo_check_ifnet_relabel`

```c
int mpo_check_ifnet_relabel(struct ucred *cred, struct ifnet *ifnet,
    struct label *ifnetlabel, struct label *newlabel);
```

| 参数           | 描述                  | 锁定 |
| ------------ | ------------------- | -- |
| `cred`       | 主体凭证                |    |
| `ifnet`      | 对象；网络接口             |    |
| `ifnetlabel` | 与 `ifnet` 关联的现有策略标签 |    |
| `newlabel`   | 将应用于 `ifnet` 的标签更新  |    |

确定主体凭证是否可以将传入的网络接口 `ifnet` 重新标记为传入的标签更新 `newlabel`。

#### 6.7.4.22. `mpo_check_socket_relabel`

```c
int mpo_check_socket_relabel(struct ucred *cred, struct socket *socket,
    struct label *socketlabel, struct label *newlabel);
```

| 参数            | 描述                       | 锁定 |
| ------------- | ------------------------ | -- |
| `cred`        | 主体凭证                     |    |
| `socket`      | 对象；套接字                   |    |
| `socketlabel` | 与 `socket` 关联的现有策略标签     |    |
| `newlabel`    | 将应用于 `socketlabel` 的标签更新 |    |

确定主体凭证是否可以将传入的套接字 `socket` 重新标记为传入的标签更新 `newlabel`。

#### 6.7.4.23. `mpo_check_cred_relabel`

```c
int mpo_check_cred_relabel(struct ucred *cred, struct label *newlabel);
```

| 参数         | 描述                | 锁定 |
| ---------- | ----------------- | -- |
| `cred`     | 主体凭证              |    |
| `newlabel` | 将应用于 `cred` 的标签更新 |    |

确定主体凭证是否可以将其自身重新标记为传入的标签更新 `newlabel`。

#### 6.7.4.24. `mpo_check_vnode_relabel`

```c
int mpo_check_vnode_relabel(struct ucred *cred, struct vnode *vp,
    struct label *vnodelabel, struct label *newlabel);
```

| 参数           | 描述               | 锁定  |
| ------------ | ---------------- | --- |
| `cred`       | 主体凭证             | 不可变 |
| `vp`         | 对象；vnode         | 锁定  |
| `vnodelabel` | 与 `vp` 关联的现有策略标签 |     |
| `newlabel`   | 将应用于 `vp` 的标签更新  |     |

确定主体凭证是否可以将传入的 vnode 重新标记为传入的标签更新。

#### 6.7.4.25. `mpo_check_mount_stat`

```c
int mpo_check_mount_stat(struct ucred *cred, struct mount *mp,
    struct label *mountlabel);
```

| 参数           | 描述             | 锁定 |
| ------------ | -------------- | -- |
| `cred`       | 主体凭证           |    |
| `mp`         | 对象；文件系统挂载      |    |
| `mountlabel` | 与 `mp` 关联的策略标签 |    |

确定主体凭证是否可以查看对文件系统执行 statfs 时的结果。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配）或 EPERM（缺乏权限）。此调用可在多种情况下使用，包括调用 [statfs(2)](https://man.freebsd.org/cgi/man.cgi?query=statfs&sektion=2&format=html) 和相关调用时，以及在调用 [getfsstat(2)](https://man.freebsd.org/cgi/man.cgi?query=getfsstat&sektion=2&format=html) 时，确定要排除哪些文件系统。

#### 6.7.4.26. `mpo_check_proc_debug`

```c
int mpo_check_proc_debug(struct ucred *cred, struct proc *proc);
```

| 参数     | 描述    | 锁定  |
| ------ | ----- | --- |
| `cred` | 主体凭证  | 不可变 |
| `proc` | 对象；进程 |     |

确定主体凭证是否可以调试传入的进程。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），EPERM（缺乏权限），或 ESRCH（隐藏目标的可见性）。此调用可在多种情况下使用，包括使用 [ptrace(2)](https://man.freebsd.org/cgi/man.cgi?query=ptrace&sektion=2&format=html) 和 [ktrace(2)](https://man.freebsd.org/cgi/man.cgi?query=ktrace&sektion=2&format=html) API 时，以及某些类型的 procfs 操作时。

#### 6.7.4.27. `mpo_check_vnode_access`

```c
int mpo_check_vnode_access(struct ucred *cred, struct vnode *vp,
    struct label *label, int flags);
```

| 参数      | 描述                                                                                     | 锁定 |
| ------- | -------------------------------------------------------------------------------------- | -- |
| `cred`  | 主体凭证                                                                                   |    |
| `vp`    | 对象；vnode                                                                               |    |
| `label` | 与 `vp` 关联的策略标签                                                                         |    |
| `flags` | [access(2)](https://man.freebsd.org/cgi/man.cgi?query=access&sektion=2&format=html) 标志 |    |

确定主体凭证在对传入的 vnode 执行 `access(2)` 和相关调用时，如何根据传入的访问标志返回结果。此功能应使用与 `mpo_check_vnode_open` 相同的语义实现。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配）或 EPERM（缺乏权限）。

#### 6.7.4.28. `mpo_check_vnode_chdir`

```c
int mpo_check_vnode_chdir(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel);
```

| 参数       | 描述                                                                                                 | 锁定 |
| -------- | -------------------------------------------------------------------------------------------------- | -- |
| `cred`   | 主体凭证                                                                                               |    |
| `dvp`    | 对象；要通过 [chdir(2)](https://man.freebsd.org/cgi/man.cgi?query=chdir&sektion=2&format=html) 进入的 vnode |    |
| `dlabel` | 与 `dvp` 关联的策略标签                                                                                    |    |

确定主体凭证是否可以将进程的工作目录更改为传入的 vnode。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.29. `mpo_check_vnode_chroot`

```c
int mpo_check_vnode_chroot(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel);
```

| 参数       | 描述              | 锁定 |
| -------- | --------------- | -- |
| `cred`   | 主体凭证            |    |
| `dvp`    | 目录 vnode        |    |
| `dlabel` | 与 `dvp` 关联的策略标签 |    |

确定主体凭证是否允许 [chroot(2)](https://man.freebsd.org/cgi/man.cgi?query=chroot&sektion=2&format=html) 进入指定的目录（`dvp`）。

#### 6.7.4.30. `mpo_check_vnode_create`

```c
int mpo_check_vnode_create(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel, struct componentname *cnp, struct vattr *vap);
```

| 参数       | 描述              | 锁定 |
| -------- | --------------- | -- |
| `cred`   | 主体凭证            |    |
| `dvp`    | 对象；父目录 vnode    |    |
| `dlabel` | 与 `dvp` 关联的策略标签 |    |
| `cnp`    | 与 `dvp` 关联的组件名称 |    |
| `vap`    | vnode 的属性       |    |

确定主体凭证是否可以在传入的父目录、传入的名称信息和传入的属性信息的基础上创建一个 vnode。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。此调用可在多种情况下使用，包括作为对 [open(2)](https://man.freebsd.org/cgi/man.cgi?query=open&sektion=2&format=html) 带有 O\_CREAT 标志、[mkfifo(2)](https://man.freebsd.org/cgi/man.cgi?query=mkfifo&sektion=2&format=html) 等的调用的响应。

#### 6.7.4.31. `mpo_check_vnode_delete`

```c
int mpo_check_vnode_delete(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel, struct vnode *vp, void *label,
    struct componentname *cnp);
```

| 参数       | 描述              | 锁定 |
| -------- | --------------- | -- |
| `cred`   | 主体凭证            |    |
| `dvp`    | 父目录 vnode       |    |
| `dlabel` | 与 `dvp` 关联的策略标签 |    |
| `vp`     | 对象；要删除的 vnode   |    |
| `label`  | 与 `vp` 关联的策略标签  |    |
| `cnp`    | 与 `vp` 关联的组件名称  |    |

确定主体凭证是否可以从传入的父目录中删除 vnode，并且可以基于传入的名称信息删除 vnode。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。此调用可在多种情况下使用，包括作为对 [unlink(2)](https://man.freebsd.org/cgi/man.cgi?query=unlink&sektion=2&format=html) 和 [rmdir(2)](https://man.freebsd.org/cgi/man.cgi?query=rmdir&sektion=2&format=html) 的调用的响应。实现此接口的策略应当同时实现 `mpo_check_rename_to` 以授权删除作为重命名目标的对象。

#### 6.7.4.32. `mpo_check_vnode_deleteacl`

```c
int mpo_check_vnode_deleteacl(struct ucred *cred, struct vnode *vp,
    struct label *label, acl_type_t type);
```

| 参数      | 描述             | 锁定  |
| ------- | -------------- | --- |
| `cred`  | 主体凭证           | 不可变 |
| `vp`    | 对象；vnode       | 锁定  |
| `label` | 与 `vp` 关联的策略标签 |     |
| `type`  | ACL 类型         |     |

确定主体凭证是否可以删除传入 vnode 的指定类型的 ACL。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.33. `mpo_check_vnode_exec`

```c
int mpo_check_vnode_exec(struct ucred *cred, struct vnode *vp,
    struct label *label);
```

| 参数      | 描述             | 锁定 |
| ------- | -------------- | -- |
| `cred`  | 主体凭证           |    |
| `vp`    | 对象；要执行的 vnode  |    |
| `label` | 与 `vp` 关联的策略标签 |    |

确定主体凭证是否可以执行传入的 vnode。执行权限的确定与任何过渡事件的决策是分开进行的。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.34. `mpo_check_vnode_getacl`

```c
int mpo_check_vnode_getacl(struct ucred *cred, struct vnode *vp,
    struct label *label, acl_type_t type);
```

| 参数      | 描述             | 锁定 |
| ------- | -------------- | -- |
| `cred`  | 主体凭证           |    |
| `vp`    | 对象；vnode       |    |
| `label` | 与 `vp` 关联的策略标签 |    |
| `type`  | ACL 类型         |    |

确定主体凭证是否可以从传入的 vnode 获取指定类型的 ACL。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.35. `mpo_check_vnode_getextattr`

```c
int mpo_check_vnode_getextattr(struct ucred *cred, struct vnode *vp,
    struct label *label, int attrnamespace, const char *name, struct uio *uio);
```

| 参数              | 描述                                                                                        | 锁定 |
| --------------- | ----------------------------------------------------------------------------------------- | -- |
| `cred`          | 主体凭证                                                                                      |    |
| `vp`            | 对象；vnode                                                                                  |    |
| `label`         | 与 `vp` 关联的策略标签                                                                            |    |
| `attrnamespace` | 扩展属性命名空间                                                                                  |    |
| `name`          | 扩展属性名称                                                                                    |    |
| `uio`           | I/O 结构指针；参见 [uio(9)](https://man.freebsd.org/cgi/man.cgi?query=uio&sektion=9&format=html) |    |

确定主体凭证是否可以从传入的 vnode 获取指定命名空间和名称的扩展属性。实现使用扩展属性的标记策略的政策可能需要特别处理这些扩展属性上的操作。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.36. `mpo_check_vnode_link`

```c
int mpo_check_vnode_link(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel, struct vnode *vp, struct label *label,
    struct componentname *cnp);
```

| 参数       | 描述              | 锁定 |
| -------- | --------------- | -- |
| `cred`   | 主体凭证            |    |
| `dvp`    | 目录 vnode        |    |
| `dlabel` | 与 `dvp` 关联的策略标签 |    |
| `vp`     | 链接目标 vnode      |    |
| `label`  | 与 `vp` 关联的策略标签  |    |
| `cnp`    | 创建链接的组件名称       |    |

确定主体是否应被允许通过 `cnp` 指定的名称创建一个指向 vnode `vp` 的链接。

#### 6.7.4.37. `mpo_check_vnode_mmap`

```c
int mpo_check_vnode_mmap(struct ucred *cred, struct vnode *vp,
    struct label *label, int prot);
```

| 参数      | 描述                                                                                            | 锁定 |
| ------- | --------------------------------------------------------------------------------------------- | -- |
| `cred`  | 主体凭证                                                                                          |    |
| `vp`    | 要映射的 vnode                                                                                    |    |
| `label` | 与 `vp` 关联的策略标签                                                                                |    |
| `prot`  | mmap 保护标志（参见 [mmap(2)](https://man.freebsd.org/cgi/man.cgi?query=mmap&sektion=2&format=html)） |    |

确定主体是否应该被允许以 `prot` 指定的保护方式映射 vnode `vp`。

#### 6.7.4.38. `mpo_check_vnode_mmap_downgrade`

```c
void mpo_check_vnode_mmap_downgrade(struct ucred *cred, struct vnode *vp,
    struct label *label, int *prot);
```

| 参数      | 描述                                                                                                         | 锁定 |
| ------- | ---------------------------------------------------------------------------------------------------------- | -- |
| `cred`  | 参见 [`mpo_check_vnode_mmap`](https://docs.freebsd.org/en/books/arch-handbook/mac/#mac-mpo-check-vnode-mmap) |    |
| `vp`    |                                                                                                            |    |
| `label` |                                                                                                            |    |
| `prot`  | 需要降级的 mmap 保护标志                                                                                            |    |

基于主体和对象标签降级 mmap 保护标志。

#### 6.7.4.39. `mpo_check_vnode_mprotect`

```c
int mpo_check_vnode_mprotect(struct ucred *cred, struct vnode *vp,
    struct label *label, int prot);
```

| 参数     | 描述        | 锁定 |
| ------ | --------- | -- |
| `cred` | 主体凭证      |    |
| `vp`   | 映射的 vnode |    |
| `prot` | 内存保护标志    |    |

确定主体是否应该被允许为从 vnode `vp` 映射的内存设置指定的内存保护。

#### 6.7.4.40. `mpo_check_vnode_poll`

```c
int mpo_check_vnode_poll(struct ucred *active_cred, struct ucred *file_cred,
    struct vnode *vp, struct label *label);
```

| 参数            | 描述             | 锁定 |
| ------------- | -------------- | -- |
| `active_cred` | 主体凭证           |    |
| `file_cred`   | 与文件结构关联的凭证     |    |
| `vp`          | 被轮询的 vnode     |    |
| `label`       | 与 `vp` 关联的策略标签 |    |

确定主体是否应被允许轮询 vnode `vp`。

#### 6.7.4.41. `mpo_check_vnode_rename_from`

```c
int mpo_vnode_rename_from(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel, struct vnode *vp, struct label *label,
    struct componentname *cnp);
```

| 参数       | 描述              | 锁定 |
| -------- | --------------- | -- |
| `cred`   | 主体凭证            |    |
| `dvp`    | 目录 vnode        |    |
| `dlabel` | 与 `dvp` 关联的策略标签 |    |
| `vp`     | 要重命名的 vnode     |    |
| `label`  | 与 `vp` 关联的策略标签  |    |
| `cnp`    | `vp` 的组件名称      |    |

确定主体是否应被允许将 vnode `vp` 重命名为其他名称。

#### 6.7.4.42. `mpo_check_vnode_rename_to`

```c
int mpo_check_vnode_rename_to(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel, struct vnode *vp, struct label *label, int samedir,
    struct componentname *cnp);
```

| 参数        | 描述                      | 锁定 |
| --------- | ----------------------- | -- |
| `cred`    | 主体凭证                    |    |
| `dvp`     | 目录 vnode                |    |
| `dlabel`  | 与 `dvp` 关联的策略标签         |    |
| `vp`      | 被覆盖的 vnode              |    |
| `label`   | 与 `vp` 关联的策略标签          |    |
| `samedir` | 布尔值；如果源目录和目标目录相同，则为 `1` |    |
| `cnp`     | 目标组件名称                  |    |

确定主体是否应被允许将 vnode `vp` 重命名到目录 `dvp` 或 `cnp` 表示的名称。如果没有现有文件要覆盖，则 `vp` 和 `label` 将为 NULL。

#### 6.7.4.43. `mpo_check_socket_listen`

```c
int mpo_check_socket_listen(struct ucred *cred, struct socket *socket,
    struct label *socketlabel);
```

| 参数            | 描述                 | 锁定 |
| ------------- | ------------------ | -- |
| `cred`        | 主体凭证               |    |
| `socket`      | 对象；socket          |    |
| `socketlabel` | 与 `socket` 关联的策略标签 |    |

确定主体凭证是否可以在传入的 socket 上监听。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.44. `mpo_check_vnode_lookup`

```c
int mpo_check_vnode_lookup(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel, struct componentname *cnp);
```

| 参数       | 描述              | 锁定 |
| -------- | --------------- | -- |
| `cred`   | 主体凭证            |    |
| `dvp`    | 对象；vnode        |    |
| `dlabel` | 与 `dvp` 关联的策略标签 |    |
| `cnp`    | 正在查找的组件名称       |    |

确定主体凭证是否可以在传入的目录 vnode 中查找传入的名称。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.45. `mpo_check_vnode_open`

```c
int mpo_check_vnode_open(struct ucred *cred, struct vnode *vp,
    struct label *label, int acc_mode);
```

| 参数         | 描述                                                                                   | 锁定 |
| ---------- | ------------------------------------------------------------------------------------ | -- |
| `cred`     | 主体凭证                                                                                 |    |
| `vp`       | 对象；vnode                                                                             |    |
| `label`    | 与 `vp` 关联的策略标签                                                                       |    |
| `acc_mode` | [open(2)](https://man.freebsd.org/cgi/man.cgi?query=open&sektion=2&format=html) 访问模式 |    |

确定主体凭证是否可以以指定的访问模式对传入的 vnode 执行打开操作。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.46. `mpo_check_vnode_readdir`

```c
int mpo_check_vnode_readdir(struct ucred *cred, struct vnode *dvp,
    struct label *dlabel);
```

| 参数       | 描述              | 锁定 |
| -------- | --------------- | -- |
| `cred`   | 主体凭证            |    |
| `dvp`    | 对象；目录 vnode     |    |
| `dlabel` | 与 `dvp` 关联的策略标签 |    |

确定主体凭证是否可以对传入的目录 vnode 执行 `readdir` 操作。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.47. `mpo_check_vnode_readlink`

```c
int mpo_check_vnode_readlink(struct ucred *cred, struct vnode *vp,
    struct label *label);
```

| 参数      | 描述             | 锁定 |
| ------- | -------------- | -- |
| `cred`  | 主体凭证           |    |
| `vp`    | 对象；vnode       |    |
| `label` | 与 `vp` 关联的策略标签 |    |

确定主体凭证是否可以对传入的符号链接 vnode 执行 `readlink` 操作。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。此调用可能在多个场景中发生，包括用户进程显式调用 `readlink`，或者在进程的名称查找过程中隐式调用 `readlink`。

#### 6.7.4.48. `mpo_check_vnode_revoke`

```c
int mpo_check_vnode_revoke(struct ucred *cred, struct vnode *vp,
    struct label *label);
```

| 参数      | 描述             | 锁定 |
| ------- | -------------- | -- |
| `cred`  | 主体凭证           |    |
| `vp`    | 对象；vnode       |    |
| `label` | 与 `vp` 关联的策略标签 |    |

确定主体凭证是否可以撤销对传入 vnode 的访问权限。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.49. `mpo_check_vnode_setacl`

```c
int mpo_check_vnode_setacl(struct ucred *cred, struct vnode *vp,
    struct label *label, acl_type_t type, struct acl *acl);
```

| 参数      | 描述             | 锁定 |
| ------- | -------------- | -- |
| `cred`  | 主体凭证           |    |
| `vp`    | 对象；vnode       |    |
| `label` | 与 `vp` 关联的策略标签 |    |
| `type`  | ACL 类型         |    |
| `acl`   | ACL            |    |

确定主体凭证是否可以在传入的 vnode 上设置传入类型的 ACL。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.50. `mpo_check_vnode_setextattr`

```c
int mpo_check_vnode_setextattr(struct ucred *cred, struct vnode *vp,
    struct label *label, int attrnamespace, const char *name, struct uio *uio);
```

| 参数              | 描述                                                                                        | 锁定 |
| --------------- | ----------------------------------------------------------------------------------------- | -- |
| `cred`          | 主体凭证                                                                                      |    |
| `vp`            | 对象；vnode                                                                                  |    |
| `label`         | 与 `vp` 关联的策略标签                                                                            |    |
| `attrnamespace` | 扩展属性命名空间                                                                                  |    |
| `name`          | 扩展属性名称                                                                                    |    |
| `uio`           | I/O 结构指针；参见 [uio(9)](https://man.freebsd.org/cgi/man.cgi?query=uio&sektion=9&format=html) |    |

确定主体凭证是否可以在传入的 vnode 上设置传入名称和命名空间的扩展属性。实现安全标签并将其嵌入扩展属性的策略可能希望对这些属性提供额外的保护。此外，策略应避免根据 `uio` 中引用的数据做出决策，因为在此检查与实际操作之间可能存在竞争条件。如果正在执行删除操作，`uio` 可能为 `NULL`。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.51. `mpo_check_vnode_setflags`

```c
int mpo_check_vnode_setflags(struct ucred *cred, struct vnode *vp,
    struct label *label, u_long flags);
```

| 参数      | 描述                                                                                            | 锁定 |
| ------- | --------------------------------------------------------------------------------------------- | -- |
| `cred`  | 主体凭证                                                                                          |    |
| `vp`    | 对象；vnode                                                                                      |    |
| `label` | 与 `vp` 关联的策略标签                                                                                |    |
| `flags` | 文件标志；参见 [chflags(2)](https://man.freebsd.org/cgi/man.cgi?query=chflags&sektion=2&format=html) |    |

确定主体凭证是否可以在传入的 vnode 上设置传入的标志。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.52. `mpo_check_vnode_setmode`

```c
int mpo_check_vnode_setmode(struct ucred *cred, struct vnode *vp,
    struct label *label, mode_t mode);
```

| 参数      | 描述                                                                                        | 锁定 |
| ------- | ----------------------------------------------------------------------------------------- | -- |
| `cred`  | 主体凭证                                                                                      |    |
| `vp`    | 对象；vnode                                                                                  |    |
| `label` | 与 `vp` 关联的策略标签                                                                            |    |
| `mode`  | 文件模式；参见 [chmod(2)](https://man.freebsd.org/cgi/man.cgi?query=chmod&sektion=2&format=html) |    |

确定主体凭证是否可以在传入的 vnode 上设置传入的文件模式。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.53. `mpo_check_vnode_setowner`

```c
int mpo_check_vnode_setowner(struct ucred *cred, struct vnode *vp,
    struct label *label, uid_t uid, gid_t gid);
```

| 参数      | 描述             | 锁定 |
| ------- | -------------- | -- |
| `cred`  | 主体凭证           |    |
| `vp`    | 对象；vnode       |    |
| `label` | 与 `vp` 关联的策略标签 |    |
| `uid`   | 用户 ID          |    |
| `gid`   | 组 ID           |    |

确定主体凭证是否可以将传入的 uid 和 gid 设置为传入 vnode 的文件用户 ID 和文件组 ID。如果 ID 设置为 (`-1`)，则表示请求不更新。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.54. `mpo_check_vnode_setutimes`

```c
int mpo_check_vnode_setutimes(struct ucred *cred, struct vnode *vp,
    struct label *label, struct timespec atime, struct timespec mtime);
```

| 参数      | 描述                                                                                          | 锁定 |
| ------- | ------------------------------------------------------------------------------------------- | -- |
| `cred`  | 主体凭证                                                                                        |    |
| `vp`    | 对象；vnode                                                                                    |    |
| `label` | 与 `vp` 关联的策略标签                                                                              |    |
| `atime` | 访问时间；参见 [utimes(2)](https://man.freebsd.org/cgi/man.cgi?query=utimes&sektion=2&format=html) |    |
| `mtime` | 修改时间；参见 [utimes(2)](https://man.freebsd.org/cgi/man.cgi?query=utimes&sektion=2&format=html) |    |

确定主体凭证是否可以在传入的 vnode 上设置传入的访问时间戳。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.55. `mpo_check_proc_sched`

```c
int mpo_check_proc_sched(struct ucred *ucred, struct proc *proc);
```

| 参数     | 描述    | 锁定 |
| ------ | ----- | -- |
| `cred` | 主体凭证  |    |
| `proc` | 对象；进程 |    |

确定主体凭证是否可以更改传入进程的调度参数。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），EPERM（缺乏权限），或 ESRCH（限制可见性）。

参见 [setpriority(2)](https://man.freebsd.org/cgi/man.cgi?query=setpriority&sektion=2&format=html) 获取更多信息。

#### 6.7.4.56. `mpo_check_proc_signal`

```c
int mpo_check_proc_signal(struct ucred *cred, struct proc *proc, int signal);
```

| 参数       | 描述                                                                                    | 锁定 |
| -------- | ------------------------------------------------------------------------------------- | -- |
| `cred`   | 主体凭证                                                                                  |    |
| `proc`   | 对象；进程                                                                                 |    |
| `signal` | 信号；参见 [kill(2)](https://man.freebsd.org/cgi/man.cgi?query=kill&sektion=2&format=html) |    |

确定主体凭证是否可以向传入的进程发送传入的信号。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），EPERM（缺乏权限），或 ESRCH（限制可见性）。

#### 6.7.4.57. `mpo_check_vnode_stat`

```c
int mpo_check_vnode_stat(struct ucred *cred, struct vnode *vp,
    struct label *label);
```

| 参数      | 描述             | 锁定 |
| ------- | -------------- | -- |
| `cred`  | 主体凭证           |    |
| `vp`    | 对象；vnode       |    |
| `label` | 与 `vp` 关联的策略标签 |    |

确定主体凭证是否可以对传入的 vnode 执行 `stat` 操作。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

参见 [stat(2)](https://man.freebsd.org/cgi/man.cgi?query=stat&sektion=2&format=html) 获取更多信息。

#### 6.7.4.58. `mpo_check_ifnet_transmit`

```c
int mpo_check_ifnet_transmit(struct ucred *cred, struct ifnet *ifnet,
    struct label *ifnetlabel, struct mbuf *mbuf, struct label *mbuflabel);
```

| 参数           | 描述                | 锁定 |
| ------------ | ----------------- | -- |
| `cred`       | 主体凭证              |    |
| `ifnet`      | 网络接口              |    |
| `ifnetlabel` | 与 `ifnet` 关联的策略标签 |    |
| `mbuf`       | 对象；待发送的 mbuf      |    |
| `mbuflabel`  | 与 `mbuf` 关联的策略标签  |    |

确定网络接口是否可以发送传入的 mbuf。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.59. `mpo_check_socket_deliver`

```c
int mpo_check_socket_deliver(struct ucred *cred, struct ifnet *ifnet,
    struct label *ifnetlabel, struct mbuf *mbuf, struct label *mbuflabel);
```

| 参数           | 描述                | 锁定 |
| ------------ | ----------------- | -- |
| `cred`       | 主体凭证              |    |
| `ifnet`      | 网络接口              |    |
| `ifnetlabel` | 与 `ifnet` 关联的策略标签 |    |
| `mbuf`       | 对象；待发送的 mbuf      |    |
| `mbuflabel`  | 与 `mbuf` 关联的策略标签  |    |

确定套接字是否可以接收传入的 mbuf 数据报头中的数据。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），或 EPERM（缺乏权限）。

#### 6.7.4.60. `mpo_check_socket_visible`

```c
int mpo_check_socket_visible(struct ucred *cred, struct socket *so,
    struct label *socketlabel);
```

| 参数            | 描述             | 锁定  |
| ------------- | -------------- | --- |
| `cred`        | 主体凭证           | 不可变 |
| `so`          | 对象；套接字         |     |
| `socketlabel` | 与 `so` 关联的策略标签 |     |

确定主体凭证是否可以通过系统监控功能“查看”传入的套接字（`socket`），例如 [netstat(8)](https://man.freebsd.org/cgi/man.cgi?query=netstat&sektion=8&format=html) 和 [sockstat(1)](https://man.freebsd.org/cgi/man.cgi?query=sockstat&sektion=1&format=html) 使用的功能。返回 0 表示成功，或返回一个 `errno` 值表示失败。建议的失败：EACCES（标签不匹配），EPERM（缺乏权限），或 ESRCH（隐藏可见性）。

#### 6.7.4.61. `mpo_check_system_acct`

```c
int mpo_check_system_acct(struct ucred *ucred, struct vnode *vp,
    struct label *vlabel);
```

| 参数       | 描述                                                                                      | 锁定 |
| -------- | --------------------------------------------------------------------------------------- | -- |
| `ucred`  | 主体凭证                                                                                    |    |
| `vp`     | 会计文件；参见 [acct(5)](https://man.freebsd.org/cgi/man.cgi?query=acct&sektion=5&format=html) |    |
| `vlabel` | 与 `vp` 关联的标签                                                                            |    |

确定主体是否被允许启用会计功能，基于其标签和会计日志文件的标签。

#### 6.7.4.62. `mpo_check_system_nfsd`

```c
int mpo_check_system_nfsd(struct ucred *cred);
```

| 参数     | 描述   | 锁定 |
| ------ | ---- | -- |
| `cred` | 主体凭证 |    |

确定主体是否被允许调用 [nfssvc(2)](https://man.freebsd.org/cgi/man.cgi?query=nfssvc&sektion=2&format=html)。

#### 6.7.4.63. `mpo_check_system_reboot`

```c
int mpo_check_system_reboot(struct ucred *cred, int howto);
```

| 参数      | 描述                                                                                                  | 锁定 |
| ------- | --------------------------------------------------------------------------------------------------- | -- |
| `cred`  | 主体凭证                                                                                                |    |
| `howto` | 来自 [reboot(2)](https://man.freebsd.org/cgi/man.cgi?query=reboot&sektion=2&format=html) 的 `howto` 参数 |    |

确定主体是否被允许以指定的方式重启系统。

#### 6.7.4.64. `mpo_check_system_settime`

```c
int mpo_check_system_settime(struct ucred *cred);
```

| 参数     | 描述   | 锁定 |
| ------ | ---- | -- |
| `cred` | 主体凭证 |    |

确定用户是否被允许设置系统时钟。

#### 6.7.4.65. `mpo_check_system_swapon`

```c
int mpo_check_system_swapon(struct ucred *cred, struct vnode *vp,
    struct label *vlabel);
```

| 参数       | 描述           | 锁定 |
| -------- | ------------ | -- |
| `cred`   | 主体凭证         |    |
| `vp`     | 交换设备         |    |
| `vlabel` | 与 `vp` 关联的标签 |    |

确定主体是否被允许将 `vp` 添加为交换设备。

#### 6.7.4.66. `mpo_check_system_sysctl`

```c
int mpo_check_system_sysctl(struct ucred *cred, int *name, u_int *namelen,
    void *old, size_t *oldlenp, int inkernel, void *new, size_t newlen);
```

| 参数         | 描述                                                                                     | 锁定 |
| ---------- | -------------------------------------------------------------------------------------- | -- |
| `cred`     | 主体凭证                                                                                   |    |
| `name`     | 参见 [sysctl(3)](https://man.freebsd.org/cgi/man.cgi?query=sysctl&sektion=3&format=html) |    |
| `namelen`  |                                                                                        |    |
| `old`      |                                                                                        |    |
| `oldlenp`  |                                                                                        |    |
| `inkernel` | 布尔值；如果来自内核，则为 `1`                                                                      |    |
| `new`      | 参见 [sysctl(3)](https://man.freebsd.org/cgi/man.cgi?query=sysctl&sektion=3&format=html) |    |
| `newlen`   |                                                                                        |    |

确定主体是否应被允许进行指定的 [sysctl(3)](https://man.freebsd.org/cgi/man.cgi?query=sysctl&sektion=3&format=html) 操作。

### 6.7.5. 标签管理调用

当用户进程请求修改对象的标签时，会发生标签重标事件。该过程分为两个阶段：首先执行访问控制检查，以确定更新是否有效且被允许；然后通过单独的入口点执行更新。标签重标入口点通常接受对象、对象标签引用以及进程提交的更新标签。重标期间不建议进行内存分配，因为重标调用不允许失败（失败应在重标检查阶段之前报告）。

## 6.8. 用户空间架构

TrustedBSD MAC 框架包括许多与策略无关的元素，其中包括用于抽象管理标签的 MAC 库接口、对系统凭证管理和登录库的修改，以支持将 MAC 标签分配给用户，以及一组用于监视和修改进程、文件和网络接口标签的工具。有关用户架构的更多细节将在近期添加到本节中。

### 6.8.1. 与策略无关的标签管理 API

TrustedBSD MAC 框架提供了许多库和系统调用，允许应用程序使用与策略无关的接口管理对象的 MAC 标签。这使得应用程序可以操作多种策略的标签，而无需为特定的策略编写支持。这些接口被通用工具如 [ifconfig(8)](https://man.freebsd.org/cgi/man.cgi?query=ifconfig&sektion=8&format=html)、[ls(1)](https://man.freebsd.org/cgi/man.cgi?query=ls&sektion=1&format=html) 和 [ps(1)](https://man.freebsd.org/cgi/man.cgi?query=ps&sektion=1&format=html) 用于查看网络接口、文件和进程上的标签。API 还支持 MAC 管理工具，包括 [getfmac(8)](https://man.freebsd.org/cgi/man.cgi?query=getfmac&sektion=8&format=html)、[getpmac(8)](https://man.freebsd.org/cgi/man.cgi?query=getpmac&sektion=8&format=html)、[setfmac(8)](https://man.freebsd.org/cgi/man.cgi?query=setfmac&sektion=8&format=html)、[setfsmac(8)](https://man.freebsd.org/cgi/man.cgi?query=setfsmac&sektion=8&format=html) 和 [setpmac(8)](https://man.freebsd.org/cgi/man.cgi?query=setpmac&sektion=8&format=html)。MAC API 在 [mac(3)](https://man.freebsd.org/cgi/man.cgi?query=mac&sektion=3&format=html) 中有文档说明。

应用程序处理 MAC 标签的两种形式：一种是内部化的形式，用于返回和设置进程和对象上的标签（`mac_t`），另一种是外部化的形式，基于 C 字符串，适用于存储在配置文件中、显示给用户或从用户输入。每个 MAC 标签包含多个元素，每个元素由名称和值对组成。内核中的策略模块绑定到特定名称，并以特定于策略的方式解释值。在外部化字符串形式中，标签由用 `/` 字符分隔的名称和值对组成。可以使用提供的 API 直接将标签转换为文本并返回；当从内核检索标签时，必须首先为所需的标签元素集准备内部标签存储。通常，这通过使用 [mac\_prepare(3)](https://man.freebsd.org/cgi/man.cgi?query=mac_prepare&sektion=3&format=html) 和一个任意的标签元素列表，或通过调用变体从 [mac.conf(5)](https://man.freebsd.org/cgi/man.cgi?query=mac.conf&sektion=5&format=html) 配置文件加载默认元素集来完成。每个对象的默认值允许应用程序编写者在不了解系统中存在的策略的情况下，有效地显示与对象关联的标签。

>**注意**
>
> 当前，MAC 库不支持直接操作标签元素，除了通过转换为文本字符串、编辑字符串并再转换为内部标签的方式。如果有必要，后续可能会添加这样的接口。

### 6.8.2. 标签绑定到用户

标准用户上下文管理接口 [setusercontext(3)](https://man.freebsd.org/cgi/man.cgi?query=setusercontext&sektion=3&format=html) 已被修改，以从 [login.conf(5)](https://man.freebsd.org/cgi/man.cgi?query=login.conf&sektion=5&format=html) 检索与用户类相关的 MAC 标签。这些标签随后在指定 `LOGIN_SETALL` 或显式指定 `LOGIN_SETMAC` 时，与其他用户上下文一起设置。

>**注意**
>
>  预计在未来版本的 FreeBSD 中，MAC 标签数据库将与 **login.conf** 用户类抽象分离，并维护在一个独立的数据库中。然而，[setusercontext(3)](https://man.freebsd.org/cgi/man.cgi?query=setusercontext&sektion=3&format=html) API 在这种变更之后应保持不变。

## 6.9. 结论

TrustedBSD MAC 框架允许内核模块以高度集成的方式增强系统安全策略。它们可以基于现有的对象属性，或基于与 MAC 框架协作维护的标签数据来实现这一点。该框架足够灵活，能够实现多种策略类型，包括信息流安全策略（如 MLS 和 Biba），以及基于现有 BSD 凭证或文件保护的策略。策略作者在实现新的安全服务时，可能需要参考本文档以及现有的安全模块。
