# 第 6 章 TrustedBSD MAC 框架

## 6.1. MAC 文档版权

本文档是由 FreeBSD 项目的 Chris Costello 在 Safeport Network Services 和 Network Associates Laboratories 开发的，Network Associates, Inc.的安全研究部门在 DARPA/SPAWAR 合同 N66001-01-C-8035（“CBOSS”）下开发，作为 DARPA CHATS 研究计划的一部分。

在源代码（SGML DocBook）和“编译”形式（SGML，HTML，PDF，PostScript，RTF 等）中重新分发和使用，无论是否经过修改，只要满足以下条件：

1. 源代码( SGML DocBook)的再分发必须保留本文件中的版权声明、这份条件列表以及以下免责声明作为本文件的第一行，不得进行修改。
2. 在编译形式中(转换为其他 DTD、转换为 PDF、PostScript、RTF 和其他格式)，必须在文档和/或其他随附分发的材料中复制上述版权声明、此条件列表以及以下免责声明。

|  | 本文档由 Networks Associates Technology, Inc 提供,并且"按现状"提供,任何明示或暗示的担保,包括但不限于对特定目的的适销性和适用性的担保,均不承认。在任何情况下,Networks Associates Technology, Inc 不对本文档的使用以任何方式引起的任何直接、间接、附带、特殊、示例性或后果性损害承担责任(包括但不限于替代商品或服务的采购,使用、数据或利润的损失或业务中断),无论是在合同、严格责任还是侵权行为(包括疏忽或其他情形)的任何责任理论下,即使已被告知可能发生此类损害也是如此。 |
| -- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

## 6.2. 概要

FreeBSD 包括对几种强制访问控制策略的实验性支持，以及一个用于内核安全可扩展性的框架，即 TrustedBSD MAC 框架。MAC 框架是一个可插拔的访问控制框架，允许轻松地将新的安全策略链接到内核中，在启动时加载，或在运行时动态加载。该框架提供了各种功能，使实现新的安全策略变得更加容易，包括轻松地将安全标签（如机密信息）附加到系统对象上的能力。

本章介绍了 MAC 策略框架，并为一个示例 MAC 策略模块提供了文档。

## 6.3. 介绍

TrustedBSD MAC 框架提供了一种机制，允许在编译时或运行时扩展内核访问控制模型。 新的系统策略可以作为内核模块实现，并链接到内核; 如果存在多个策略模块，则它们的结果将被组合。 MAC 框架为协助策略编写人员提供各种访问控制基础设施服务，包括支持瞬态和持久的与策略无关的对象安全标签。 目前，此支持被视为实验性的。

本章提供了适用于策略模块开发者以及潜在的 MAC 启用环境消费者的信息，了解 MAC 框架如何支持内核访问控制扩展。

## 6.4. 政策背景

强制访问控制（MAC）是指操作系统对用户强制执行的一组访问控制策略。MAC 策略可以与用户自主保护对象的自主访问控制（DAC）保护形成对比。在传统的 UNIX 系统中，DAC 保护包括文件权限和访问控制列表；MAC 保护包括防止用户间调试和防火墙的进程控制。操作系统设计者和安全研究人员制定了各种 MAC 策略，包括多级安全（MLS）保密策略，Biba 完整性策略，基于角色的访问控制（RBAC），域和类型强制（DTE），以及类型强制（TE）。每个模型都会基于各种因素做出决策，包括用户身份，角色和安全许可，以及表示数据敏感性和完整性等概念的对象的安全标签。

TrustedBSD MAC 框架能够支持实现所有这些策略的策略模块，以及一类广泛的系统加固策略，这些策略可能使用现有的安全属性，如用户和组 ID，以及文件的扩展属性和其他系统属性。此外，尽管名字叫做 MAC 框架，但策略模块有很大的灵活性，可以实现纯自主策略的授权保护。

## 6.5. MAC 框架内核架构

TrustedBSD MAC 框架允许内核模块扩展操作系统安全策略，同时提供许多访问控制模块所需的基础功能。如果同时加载多个策略，则 MAC 框架将有用地（某个定义来说）组合这些策略的结果。

### 6.5.1. 内核元素

MAC 框架包含许多内核元素：

* 框架管理接口
* 并发和同步原语。
* 策略注册
* 内核对象的可扩展安全标签
* 策略入口点组合运算符
* 标签管理基元
* 内核服务调用的入口点 API
* 策略模块的入口点 API
* 入口点实现（策略生命周期，对象生命周期/标签管理，访问控制检查）。
* 与策略无关的标签管理系统调用
* mac_syscall() 多路复用系统调用
* 实现为 MAC 策略模块的各种安全策略

### 6.5.2. 框架管理接口

TrustedBSD MAC 框架可以直接使用 sysctl、加载器可调参数和系统调用进行管理。

在大多数情况下，相同名称的 sysctl 和加载器可调整相同的参数，并控制与各种内核子系统相关的保护执行等行为。此外，如果 MAC 调试支持编译到内核中，将维护多个计数器以跟踪标签分配。通常建议不要使用每个子系统执行控件来控制生产环境中的策略行为，因为它们会广泛影响所有活动策略的操作。相反，应优先使用每个策略控件，因为它们提供了更大的粒度和更大的操作一致性，适用于策略模块。

使用系统模块管理系统调用和其他系统接口，包括引导加载程序变量，执行策略模块的加载和卸载；策略模块将有机会影响加载和卸载事件，包括阻止不希望的策略卸载。

### 6.5.3. 策略列表并发性和同步化

由于活动策略集可能在运行时发生变化，并且入口点的调用是非原子的，因此需要同步以防止在入口点调用正在进行时加载或卸载策略，冻结活动策略集以防止加载或卸载策略，直到入口点调用完成。这是通过一个框架繁忙计数来实现的：每当进入一个入口点时，繁忙计数会递增；每当退出时，繁忙计数会递减。在繁忙计数升高时，不允许策略列表更改，并且试图修改策略列表的线程将休眠，直到列表不再繁忙。繁忙计数受互斥锁保护，并且条件变量用于唤醒等待策略列表修改的线程。这种同步模型的一个副作用是允许从策略模块内部递归进入 MAC 框架，尽管通常不会使用。

采用各种优化措施来减少繁忙计数的开销，包括避免在列表为空或仅包含静态条目（在系统启动前加载的策略，无法卸载）时增加和减少的全部成本。还提供了一个编译时选项，可以防止在运行时更改加载的策略集，从而消除了与支持动态加载和卸载策略相关的互斥锁定成本，因为不再需要同步。

由于 MAC 框架在某些入口点不允许阻塞，因此不能使用普通的睡眠锁；因此，尝试加载或卸载可能会阻塞相当长的时间，等待框架变得空闲。

### 6.5.4. 标签同步

由于感兴趣的内核对象通常可以同时从多个线程访问，并且允许多个线程同时进入 MAC 框架，因此 MAC 框架维护的安全属性存储被仔细同步。一般情况下，现有的内核对象数据上的内核同步用于保护对象上的 MAC 框架安全标签：例如，套接字上的 MAC 标签使用现有的套接字互斥锁进行保护。同样，对于并发访问的语义通常与容器对象的语义相同：对于凭据，与凭据结构的其余部分一样，标签内容保持写时复制语义。当使用对象引用调用 MAC 框架时，MAC 框架会对对象施加必要的锁定。策略作者必须了解这些同步语义，因为它们有时会限制对标签允许的访问类型：例如，当只读引用传递给策略的凭据时，只允许对附加到凭据的标签状态进行读取操作。

### 6.5.5. 策略同步和并发

策略模块必须编写，以假定许多内核线程可能同时进入一个或多个策略入口点，这是因为 FreeBSD 内核的并行和抢占性质。如果策略模块使用可变状态，这可能需要在策略内部使用同步原语，以防止对该状态产生不一致的视图，导致策略操作不正确。策略通常可以利用现有的 FreeBSD 同步原语来实现这一目的，包括互斥锁、睡眠锁、条件变量和计数信号量。但是，策略应该小心编写这些原语，尊重现有的内核锁顺序，并认识到一些入口点不允许休眠，限制这些入口点中原语的使用到互斥锁和唤醒操作。

当策略模块调用其他内核子系统时，它们通常需要释放任何策略锁，以避免违反内核锁顺序或冒险锁递归。这将保持策略锁作为全局锁顺序中的叶锁，有助于避免死锁。

### 6.5.6. 策略注册

MAC 框架维护两个活动策略列表：一个静态列表和一个动态列表。这两个列表在锁定语义上略有不同：使用静态列表不需要提升参考计数。当加载包含 MAC 框架策略的内核模块时，策略模块将使用 SYSINIT 调用注册函数；当卸载策略模块时， SYSINIT 同样会调用注销函数。如果策略模块加载超过一次，注册可能会失败，如果注册所需资源不足（例如，策略可能需要标记，但可用的标记状态不足），或者其他策略前提条件可能无法满足（某些策略可能仅能在引导之前加载）。同样，如果一个策略被标记为不可卸载，注销可能会失败。

### 6.5.7. 入口点

内核服务以两种方式与 MAC 框架交互：它们调用一系列 API 来通知框架相关事件，并在安全相关对象中提供一个策略不可知的标签结构指针。标签指针通过标签管理入口点由 MAC 框架维护，并允许框架通过对维护对象的内核子系统进行相对非侵入性的更改为策略模块提供标记服务。例如，标签指针已经添加到进程、进程凭据、套接字、管道、vnodes、Mbufs、网络接口、IP 重组队列以及各种其他安全相关结构中。内核服务还会在执行重要安全决策时调用 MAC 框架，允许策略模块基于自己的标准（可能包括存储在安全标签中的数据）增强这些决策。这些安全关键决策大多是显式访问控制检查；然而，某些影响更广泛决策函数的功能，例如套接字的数据包匹配和程序执行时的标签转换。

### 6.5.8. 策略组合

当有多个策略模块同时加载到内核中时，策略模块的结果将由框架使用组合运算符组合。该运算符目前是硬编码的，要求所有活动策略必须批准请求才能返回成功。由于策略可能返回各种错误条件（成功、访问被拒绝、对象不存在等），一个优先级运算符从策略返回的错误集合中选择结果错误。一般来说，指示对象不存在的错误会优先于指示对象访问被拒绝的错误。虽然不能保证结果的组合会是有用或安全的，但我们发现对于许多有用的策略选择而言，确实如此。例如，传统的受信任系统通常会附带两个或更多使用类似组合的策略。

### 6.5.9. 标签支持

由于许多有趣的访问控制扩展依赖于对象上的安全标签，因此 MAC 框架提供了一套基于策略的标签管理系统调用，涵盖各种用户可见对象。常见的标签类型包括分区标识符、敏感性标签、完整性标签、隔间、域、角色和类型。所谓策略不可知是指策略模块能够完全定义与对象关联的元数据的语义。策略模块参与提供给用户应用程序的基于字符串的标签的内部化和外部化，并且如果需要，可以向应用程序公开多个标签元素。

内存中的标签存储在分配的 slab 中，其中包含一个固定长度的联合数组，每个联合数组包含一个指针和一个。注册标签存储的策略将被分配一个“槽”标识符，该标识符可用于取消引用标签存储。存储的语义完全由策略模块决定：模块提供了与内核对象生命周期相关联的各种入口点，包括初始化、关联/创建和销毁。使用这些接口，可以实现引用计数和其他存储模型。通常策略模块不需要直接访问对象结构以检索标签，因为 MAC 框架通常会将对象指针和对象标签的直接指针传递给入口点。这一规则的主要例外是进程凭据，必须手动取消引用才能访问凭据标签。这在 MAC 框架的将来版本中可能会改变。

初始化条目点经常包括一个睡眠倾向标志，指示初始化是否允许睡眠；如果不允许睡眠，则可能返回失败以取消标签（因此对象）的分配。例如，在中断处理期间的网络堆栈中，不允许睡眠，或者在调用者持有互斥锁时可能发生这种情况。由于在飞行中网络数据包（Mbufs）上维护标签的性能成本，策略必须明确声明需要分配 Mbuf 标签。动态加载的策略使用标签时必须能够处理其初始化函数未在对象上调用的情况，因为在加载策略时对象可能已经存在。MAC 框架保证未初始化的标签槽将保存 0 或 NULL 值，策略可以使用这些值来检测未初始化的值。但是，由于 Mbuf 标签的分配是有条件的，策略还必须能够处理 Mbuf 的 NULL 标签指针，如果它们已经被动态加载。

对于文件系统标签，提供了特殊支持，用于在扩展属性中持久存储安全标签。在可用的情况下，扩展属性事务用于允许对 vnodes 上的安全标签进行一致的复合更新 — 目前此支持仅存在于 UFS2 文件系统中。策略作者可以选择使用一个（或多个）扩展属性来实现多标签文件系统对象标签。出于效率原因，vnode 标签（ v_label ）是任何磁盘上标签的缓存；策略能够在实例化 vnode 时将值加载到缓存中，并根据需要更新缓存。因此，在每次访问控制检查时，不需要直接访问扩展属性。

|  | 目前，如果一个带标签的策略允许动态卸载，那么它的状态槽将无法被回收，这对于带标签的策略的卸载重载操作次数设置了严格（相对较低）的限制。 |
| -- | ---------------------------------------------------------------------------------------------------------------------------------------- |

### 6.5.10. 系统调用

MAC 框架实现了许多系统调用：其中大多数调用支持面向策略的标签检索和操作 API，这些 API 暴露给用户应用程序。

标签管理调用接受一个标签描述结构， struct mac ，其中包含一系列 MAC 标签元素。每个元素包含一个字符串名称和字符串值。每个策略将有机会声明特定元素名称，允许策略在需要时公开多个独立元素。策略模块通过入口点在内核标签和用户提供的标签之间执行内部化和外部化，允许多种语义。标签管理系统调用通常由用户库函数包装，以执行内存分配和错误处理，简化必须管理标签的用户应用程序。

FreeBSD 内核中存在以下与 MAC 相关的系统调用：

* mac_get_proc() 可用于检索当前进程的标签。
* mac_set_proc() 可用于请求更改当前进程的标签。
* mac_get_fd() 可用于检索由文件描述符引用的对象（文件、套接字、管道等）的标签。
* mac_get_file() 可用于检索由文件系统路径引用的对象的标签。
* mac_set_fd() 可用于请求更改由文件描述符引用的对象（文件，套接字，管道等）的标签。
* mac_set_file() 可用于请求更改由文件系统路径引用的对象的标签。
* mac_syscall() 允许策略模块创建新的系统调用，而无需修改系统调用表； 它接受目标策略名称，操作编号和策略使用的不透明参数。
* mac_get_pid() 可用于通过进程 id 请求另一个进程的标签。
* mac_get_link() 与 mac_get_file() 相同，只是如果它是路径中的最后一个条目，则不会跟随符号链接，因此可用于检索符号链接上的标签。
* mac_set_link() 与 mac_set_file() 相同，只是如果它是路径中的最后一个条目，则不会跟随符号链接，因此可用于操作符号链接上的标签。
* mac_execve() 等同于 execve() 系统调用，只是它还接受一个请求的标签，在开始执行新程序时将进程标签设置为该标签。执行时标签的变化称为"过渡"。
* mac_get_peer() 实际上是通过套接字选项实现的，如果可用，它检索套接字上远程对等体的标签。

除了这些系统调用外， SIOCSIGMAC 和 SIOCSIFMAC 网络接口 ioctls 允许检索和设置网络接口上的标签。

## 6.6. MAC 策略架构

安全策略要么直接链接到内核，要么编译为可在启动时加载的可加载内核模块，要么在运行时使用模块加载系统调用动态加载。策略模块通过一组声明的入口点与系统交互，提供对系统事件流的访问，并允许策略影响访问控制决策。每个策略包含若干元素：

* 策略的可选配置参数。
* 策略逻辑和参数的集中实现。
* 可选实现策略生命周期事件，如初始化和销毁。
* 可选支持在选定的内核对象上初始化、维护和销毁标签。
* 为用户进程检查和修改选定对象的标签提供可选支持。
* 实现与策略相关的选定访问控制入口点。
* 声明策略身份、模块入口点和策略属性。

### 6.6.1. 策略声明

模块可以使用 MAC_POLICY_SET() 宏进行声明，该宏命名策略，提供对 MAC 入口点向量的引用，提供确定策略框架应如何处理策略的加载时标志，并可选择请求框架分配标签状态。

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

MAC 策略入口点向量，在本示例中为 macpolicyops ，将模块中定义的函数与特定入口点关联起来。可在 MAC 入口点参考部分找到所有可用入口点及其原型的完整列表。在模块注册期间特别感兴趣的是.mpo_destroy 和.mpo_init 入口点。.mpo_init 将在成功向模块框架注册策略但在任何其他入口点变为活动之前调用。这允许策略执行任何特定于策略的分配和初始化，例如初始化任何数据或锁。.mpo_destroy 将在卸载策略模块时调用，以允许释放任何已分配的内存和销毁锁。目前，这两个入口点在保持 MAC 策略列表互斥锁的情况下被调用，以防止调用任何其他入口点：这将被更改，但同时，策略应注意调用哪些内核原语，以避免锁定顺序或睡眠问题。

政策声明的模块名称字段存在是为了可以唯一标识模块的依赖关系。应选择一个适当的字符串。政策的完整字符串名称在加载和卸载事件期间通过内核日志向用户显示，并在向用户态进程提供状态信息时导出。

### 6.6.2. 政策标志

政策声明的标志字段允许模块在模块加载时向框架提供关于其能力的信息。目前定义了三个标志：

此标志表示策略模块可以被卸载。如果没有提供此标志，则策略框架将拒绝卸载模块的请求。这个标志可能被那些在运行时无法释放分配的标签状态的模块使用。

此标志表示策略模块必须在引导过程的早期被加载和初始化。如果指定了该标志，将拒绝在引导后尝试注册模块。这个标志可能被需要对所有系统对象进行普遍标记的策略使用，且不能处理未经正确初始化的对象。

此标志表示策略模块需要对 Mbuf 进行标记，并且内存应始终为 Mbuf 标签的存储分配。默认情况下，MAC 框架不会为 Mbuf 分配标签存储，除非至少加载了一个带有此标志设置的策略。当策略不需要对 Mbuf 进行标记时，这显着提高了网络性能。存在一个内核选项， MAC_ALWAYS_LABEL_MBUF ，用于强制 MAC 框架分配 Mbuf 标签存储，而不考虑此标志的设置，并且在某些环境中可能很有用。

|  | 使用未设置 MPC_LOADTIME_FLAG_NOTLATE 标志的策略必须能够正确处理传递到入口点的 NULL Mbuf 标签指针。 这是必要的，因为在加载启用 Mbuf 标签的策略后，没有标签存储的在飞行中的 Mbuf 可能会持续存在。 如果在网络子系统激活之前加载策略（即，在加载策略时不是晚加载），则可以保证所有 Mbuf 都具有标签存储。 |
| -- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

### 6.6.3. 策略入口点

提供了四类与框架注册的策略相关的入口点：与策略的注册和管理相关的入口点，表示内核对象的初始化、创建、销毁和其他生命周期事件的入口点，与策略模块可能影响的访问控制决策相关的事件，以及涉及对象标签管理的调用。此外，还提供了一个 mac_syscall() 入口点，以便策略可以在不注册新系统调用的情况下扩展内核接口。

策略模块编写者应了解内核锁定策略，以及在哪些入口点可以使用哪些对象锁。编写者应尽量避免在入口点内获取非叶锁，同时遵循对象访问和修改的锁定协议以避免死锁情况。特别是，编写者应注意，虽然通常会持有访问对象及其标签所需的锁，但并非所有入口点都具有修改对象或其标签所需的足够锁。参数的锁定信息已记录在 MAC 框架入口点文档中。

策略入口点将传递对象标签的引用以及对象本身。这使得带标签的策略可以不了解对象的内部情况，但仍可根据标签做出决策。唯一的例外是进程凭据，策略假定内核中的策略将其视为第一类安全对象。

## 6.7. MAC 策略入口点参考

### 6.7.1. 通用模块入口点

#### 6.7.1.1. `mpo_init`

```
void mpo_init(	conf);
struct mac_policy_conf *conf;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `conf`     | MAC 策略定义 |      |

策略加载事件。策略列表互斥体被锁定，因此不能执行休眠操作，对其他内核子系统的调用必须谨慎进行。如果在策略初始化期间需要可能会睡眠的内存分配，应使用单独的模块 SYSINIT()进行。

#### 6.7.1.2. `mpo_destroy`

```
void mpo_destroy(	conf);
struct mac_policy_conf *conf;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `conf`     | MAC 政策定义 |      |

策略加载事件。策略列表互斥锁被持有，因此应该谨慎。

#### 6.7.1.3. `mpo_syscall`

```
int mpo_syscall(	td,
 	call,
 	arg);
struct thread *td;
int call;
void *arg;
```

| 参数 | 描述               | 锁定 |
| ------ | -------------------- | ------ |
| `td`     | 调用线程           |      |
| `call`     | 特定策略系统调用号 |      |
| `arg`     | 系统调用参数指针   |      |

此入口点提供策略多路复用系统调用，以便策略可以为用户进程提供附加服务，而无需注册特定系统调用。在注册期间提供的策略名称用于从用户态解多路复用调用，并将参数转发到此入口点。在实现新服务时，安全模块应确保根据需要从 MAC 框架调用适当的访问控制检查。例如，如果策略实现了增强信号功能，应调用必要的信号访问控制检查以调用 MAC 框架和其他已注册策略。

|  | 模块目前必须自行执行系统调用数据的 copyin() 。 |
| -- | ------------------------------------------------ |

#### 6.7.1.4. `mpo_thread_userret`

```
void mpo_thread_userret(	td);
struct thread *td;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `td`     | 返回线程 |      |

此入口点允许策略模块在线程通过系统调用返回、陷阱返回或其他方式返回用户空间时执行与 MAC 相关的事件。对于具有浮动进程标签的策略而言，这是必需的，因为在系统调用处理过程中的任意点获取进程锁并不总是可能的；进程标签可能代表传统认证数据、进程历史信息或其他数据。要使用此机制，对进程凭证标签的预期更改可能存储在由每个策略自旋锁保护的 p_label 中，然后设置每个线程的 TDF_ASTPENDING 标志和每个进程的 PS_MACPENDM 标志以安排调用 userret 入口点。从此入口点，策略可以更轻松地创建替换凭证，而不必过多考虑锁定上下文。策略编写者应当注意，与调度 AST 和执行 AST 相关的事件顺序可能在多线程应用程序中变得复杂且交织在一起。

### 6.7.2. 标签操作

#### 6.7.2.1. `mpo_init_bpfdesc_label`

```
void mpo_init_bpfdesc_label(	label);
struct label *label;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `label`     | 应用新标签 |      |

在新实例化的 bpfdesc（BPF 描述符）上初始化标签。允许睡眠。

#### 6.7.2.2. `mpo_init_cred_label`

```
void mpo_init_cred_label(	label);
struct label *label;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `label`     | 初始化新标签 |      |

为新实例化的用户凭据初始化标签。允许睡眠。

#### 6.7.2.3. `mpo_init_devfsdirent_label`

```
void mpo_init_devfsdirent_label(	label);
struct label *label;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `label`     | 要应用的新标签 |      |

初始化新实例化的 devfs 条目上的标签。允许休眠。

#### 6.7.2.4. `mpo_init_ifnet_label`

```
void mpo_init_ifnet_label(	label);
struct label *label;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `label`     | 新标签应用 |      |

在新实例化的网络接口上初始化标签。允许睡眠。

#### 6.7.2.5. `mpo_init_ipq_label`

```
void mpo_init_ipq_label(	label,
 	flag);
struct label *label;
int flag;
```

| 参数 | 描述                         | 锁定 |
| ------ | ------------------------------ | ------ |
| `label`     | 申请新标签                   |      |
| `flag`     | 睡眠/非睡眠 malloc(9);见下文 |      |

在新实例化的 IP 片段重组队列上初始化标签。 flag 字段可以是 M_WAITOK 和 M_NOWAIT 之一，并应在此初始化调用期间避免执行睡眠 malloc(9)。IP 片段重组队列分配经常发生在性能敏感的环境中，实现应谨慎避免睡眠或长时间运行的操作。此入口点允许失败，导致无法分配 IP 片段重组队列。

#### 6.7.2.6. `mpo_init_mbuf_label`

```
void mpo_init_mbuf_label(	flag,
 	label);
int flag;
struct label *label;
```

| 参数 | 描述                              | 锁定 |
| ------ | ----------------------------------- | ------ |
| `flag`     | 睡眠/非睡眠 malloc(9); 请参见下文 |      |
| `label`     | 初始化策略标签                    |      |

在新实例化的 mbuf 数据包头（ mbuf ）上初始化标签。 flag 字段可以是 M_WAITOK 和 M_NOWAIT 之一，并且应该在此初始化调用期间避免执行睡眠 malloc(9)。mbuf 分配经常发生在性能敏感的环境中，实现应小心避免睡眠或长时间运行的操作。允许此入口点失败，导致无法分配 mbuf 头。

#### 6.7.2.7. `mpo_init_mount_label`

```
void mpo_init_mount_label(	mntlabel,
 	fslabel);
struct label *mntlabel;
struct label *fslabel;
```

| 参数 | 描述                               | 锁定 |
| ------ | ------------------------------------ | ------ |
| `mntlabel`     | 创建要为挂载点本身初始化的策略标签 |      |
| `fslabel`     | 创建要为文件系统初始化的策略标签   |      |

在新实例化的挂载点上初始化标签。允许睡眠。

#### 6.7.2.8. `mpo_init_mount_fs_label`

```
void mpo_init_mount_fs_label(	label);
struct label *label;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `label`     | 初始化标签 |      |

在新挂载的文件系统上初始化标签。允许睡眠

#### 6.7.2.9. `mpo_init_pipe_label`

```
void mpo_init_pipe_label(	label);
struct label*label;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `label`     | 填写标签 |      |

为新实例化的管道初始化标签。允许睡眠。

#### 6.7.2.10. `mpo_init_socket_label`

```
void mpo_init_socket_label(	label,
 	flag);
struct label *label;
int flag;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `label`     | 初始化新标签   |      |
| `flag`     | malloc(9) 标志 |      |

初始化新实例套接字的标签。 flag 字段可以是 M_WAITOK 和 M_NOWAIT 中的一个，并且应该用于避免在此初始化调用期间执行睡眠 malloc(9)。

#### 6.7.2.11. `mpo_init_socket_peer_label`

```
void mpo_init_socket_peer_label(	label,
 	flag);
struct label *label;
int flag;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `label`     | 初始化新标签   |      |
| `flag`     | malloc(9) 标志 |      |

初始化新实例化套接字的对等标签。 flag 字段可以是 M_WAITOK 和 M_NOWAIT 之一，在此初始化调用期间应使用它们，以避免执行睡眠 malloc(9)。

#### 6.7.2.12. `mpo_init_proc_label`

```
void mpo_init_proc_label(	label);
struct label *label;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `label`     | 初始化新标签 |      |

为新实例化的进程初始化标签。允许睡眠。

#### 6.7.2.13. `mpo_init_vnode_label`

```
void mpo_init_vnode_label(	label);
struct label *label;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `label`     | 初始化新标签 |      |

在新实例化的虚拟节点上初始化标签。允许睡眠。

#### 6.7.2.14. `mpo_destroy_bpfdesc_label`

```
void mpo_destroy_bpfdesc_label(	label);
struct label *label;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `label`     | bpfdesc 标签 |      |

销毁 BPF 描述符上的标签。在此入口点，策略应释放与 label 关联的任何内部存储，以便销毁它。

#### 6.7.2.15. `mpo_destroy_cred_label`

```
void mpo_destroy_cred_label(	label);
struct label *label;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `label`     | 标签被销毁 |      |

销毁凭据上的标签。在此入口点，策略模块应释放与 label 关联的任何内部存储，以便销毁。

#### 6.7.2.16. `mpo_destroy_devfsdirent_label`

```
void mpo_destroy_devfsdirent_label(	label);
struct label *label;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `label`     | 被销毁的标签 |      |

在 devfs 条目上销毁标签。在此入口点，策略模块应释放与 label 关联的任何内部存储，以便销毁它。

#### 6.7.2.17. `mpo_destroy_ifnet_label`

```
void mpo_destroy_ifnet_label(	label);
struct label *label;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `label`     | 标签被销毁 |      |

删除被移除接口上的标签。在这个入口点，策略模块应该释放与 label 相关的任何内部存储，以便它可以被销毁。

#### 6.7.2.18. `mpo_destroy_ipq_label`

```
void mpo_destroy_ipq_label(	label);
struct label *label;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `label`     | 标签被销毁 |      |

销毁 IP 片段队列上的标签。在此入口点，策略模块应释放与 label 关联的任何内部存储，以便销毁它。

#### 6.7.2.19. `mpo_destroy_mbuf_label`

```
void mpo_destroy_mbuf_label(	label);
struct label *label;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `label`     | 被销毁的标签 |      |

在 mbuf 标头上销毁标签。在此入口点，策略模块应释放与 label 相关联的任何内部存储，以便销毁它。

#### 6.7.2.20. `mpo_destroy_mount_label`

```
void mpo_destroy_mount_label(	label);
struct label *label;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `label`     | 挂载点标签被销毁 |      |

摧毁挂载点上的标签。在此入口点，策略模块应释放与 mntlabel 关联的内部存储，以便它们可以被销毁。

#### 6.7.2.21. `mpo_destroy_mount_label`

```
void mpo_destroy_mount_label(	mntlabel,
 	fslabel);
struct label *mntlabel;
struct label *fslabel;
```

| 参数 | 描述               | 锁定 |
| ------ | -------------------- | ------ |
| `mntlabel`     | 挂载点标签被破坏   |      |
| `fslabel`     | 文件系统标签被破坏 |      |

摧毁挂载点上的标签。在此入口点，策略模块应释放与 mntlabel 和 fslabel 关联的内部存储，以便可以销毁它们。

#### 6.7.2.22. `mpo_destroy_socket_label`

```
void mpo_destroy_socket_label(	label);
struct label *label;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `label`     | 销毁套接字标签 |      |

销毁套接字上的标签。在此入口点，策略模块应释放与 label 关联的任何内部存储，以便销毁。

#### 6.7.2.23. `mpo_destroy_socket_peer_label`

```
void mpo_destroy_socket_peer_label(	peerlabel);
struct label *peerlabel;
```

| 参数 | 描述               | 锁定 |
| ------ | -------------------- | ------ |
| `peerlabel`     | 销毁套接字对等标签 |      |

销毁套接字上的对等标签。在此入口点，策略模块应释放与 label 关联的任何内部存储，以便销毁它。

#### 6.7.2.24. `mpo_destroy_pipe_label`

```
void mpo_destroy_pipe_label(	label);
struct label *label;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `label`     | 管道标签 |      |

销毁管道上的标签。在此入口点，策略模块应释放与 label 关联的任何内部存储，以便销毁它。

#### 6.7.2.25. `mpo_destroy_proc_label`

```
void mpo_destroy_proc_label(	label);
struct label *label;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `label`     | 过程标签 |      |

销毁进程上的标签。在此入口点，策略模块应释放与 label 关联的任何内部存储，以便可以销毁它。

#### 6.7.2.26. `mpo_destroy_vnode_label`

```
void mpo_destroy_vnode_label(	label);
struct label *label;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `label`     | 进程标签 |      |

销毁 vnode 上的标签。在此入口点，策略模块应释放与 label 关联的任何内部存储，以便销毁它。

#### 6.7.2.27. `mpo_copy_mbuf_label`

```
void mpo_copy_mbuf_label(	src,
 	dest);
struct label *src;
struct label *dest;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `src`     | 源标签     |      |
| `dest`     | 目的地标签 |      |

将 src 中的标签信息复制到 dest 。

#### 6.7.2.28. `mpo_copy_pipe_label`

```
void mpo_copy_pipe_label(	src,
 	dest);
struct label *src;
struct label *dest;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `src`     | 源标签     |      |
| `dest`     | 目的地标签 |      |

将 src 中的标签信息复制到 dest 中。

#### 6.7.2.29. `mpo_copy_vnode_label`

```
void mpo_copy_vnode_label(	src,
 	dest);
struct label *src;
struct label *dest;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `src`     | 源标签     |      |
| `dest`     | 目的地标签 |      |

将 src 中的标签信息复制到 dest 中。

#### 6.7.2.30. `mpo_externalize_cred_label`

```
int mpo_externalize_cred_label(	label,
 	element_name,
 	sb,
 	*claimed);
struct label *label;
char *element_name;
struct sbuf *sb;
int *claimed;
```

| 参数 | 描述                             | 锁定 |
| ------ | ---------------------------------- | ------ |
| `label`     | 标签要外部化                     |      |
| `element_name`     | 应将标签外部化的策略名称         |      |
| `sb`     | 应填入文本表示形式的字符串缓冲区 |      |
| `claimed`     | 当 element_data 可填入时应递增。 |      |

根据传递的标签结构生成外部化标签。外部化标签包括标签内容的文本表示，可供用户空间应用程序使用并供用户阅读。当前将调用所有策略的 externalize 入口点，因此在尝试填充 sb 之前，实现应检查 element_name 的内容。如果 element_name 与你的策略名称不匹配，请简单地返回 0。仅在在外部化标签数据时发生错误时返回非零值。一旦策略填充 element_data ，应递增 *claimed 。

#### 6.7.2.31. `mpo_externalize_ifnet_label`

```
int mpo_externalize_ifnet_label(	label,
 	element_name,
 	sb,
 	*claimed);
struct label *label;
char *element_name;
struct sbuf *sb;
int *claimed;
```

| 参数 | 描述                               | 锁定 |
| ------ | ------------------------------------ | ------ |
| `label`     | 标签要外部化                       |      |
| `element_name`     | 应该外部化标签的策略的名称         |      |
| `sb`     | 要用文本表示填充的字符串缓冲区     |      |
| `claimed`     | 当 element_data 可以填入时应递增。 |      |

根据传递的标签结构生成外部化标签。外部化标签包括标签内容的文本表示，可用于用户空间应用程序，并可被用户阅读。当前，将调用所有策略的 externalize 入口点，因此实现应在尝试填充 sb 之前检查 element_name 的内容。如果 element_name 与你的策略名称不匹配，请简单地返回 0。仅在外部化标签数据时发生错误时返回非零值。一旦策略填充 element_data ，应递增 *claimed 。

#### 6.7.2.32. `mpo_externalize_pipe_label`

```
int mpo_externalize_pipe_label(	label,
 	element_name,
 	sb,
 	*claimed);
struct label *label;
char *element_name;
struct sbuf *sb;
int *claimed;
```

| 参数 | 描述                               | 锁定 |
| ------ | ------------------------------------ | ------ |
| `label`     | 要外部化的标签                     |      |
| `element_name`     | 应该外部化标签的策略名称           |      |
| `sb`     | 要填充文本表示的字符串缓冲器       |      |
| `claimed`     | 当 element_data 可以填入时应递增。 |      |

根据传递的标签结构生成外部化标签。外部化标签包含一个标签内容的文本表示，可以与用户应用程序一起使用并被用户阅读。当前，所有策略的 externalize 入口点将被调用，因此在尝试填充 sb 之前，实现应检查 element_name 的内容。如果 element_name 与你的策略名称不匹配，只需返回 0。只有在外部化标签数据时发生错误时才返回非零值。一旦策略填写 element_data ， *claimed 就应该递增。

#### 6.7.2.33. `mpo_externalize_socket_label`

```
int mpo_externalize_socket_label(	label,
 	element_name,
 	sb,
 	*claimed);
struct label *label;
char *element_name;
struct sbuf *sb;
int *claimed;
```

| 参数 | 描述                               | 锁定 |
| ------ | ------------------------------------ | ------ |
| `label`     | 标签外部化                         |      |
| `element_name`     | 应该外部化标签的策略名称           |      |
| `sb`     | 用于填充文本表示的字符串缓冲区     |      |
| `claimed`     | 当 element_data 可以填充时应递增。 |      |

基于传递的标签结构生成外部化标签。外部化标签包括标签内容的文本表示，可与用户空间应用程序一起使用并由用户阅读。当前，将调用所有策略的 externalize 入口点，因此实现应在尝试填充 sb 之前检查 element_name 的内容。如果 element_name 与你的策略名称不匹配，请简单地返回 0。只有在外部化标签数据时发生错误时才返回非零值。一旦策略填写 element_data ，应递增 *claimed 。

#### 6.7.2.34. `mpo_externalize_socket_peer_label`

```
int mpo_externalize_socket_peer_label(	label,
 	element_name,
 	sb,
 	*claimed);
struct label *label;
char *element_name;
struct sbuf *sb;
int *claimed;
```

| 参数 | 描述                                 | 锁定 |
| ------ | -------------------------------------- | ------ |
| `label`     | 应外部化的标签                       |      |
| `element_name`     | 应外部化标签的策略名称               |      |
| `sb`     | 要填充为标签的文本表示的字符串缓冲区 |      |
| `claimed`     | 当可以填入 element_data 时应递增     |      |

基于传递的标签结构生成外部化标签。外部化标签包括可以与用户端应用一起使用并由用户阅读的标签内容的文本表示。目前，将调用所有策略的 externalize 入口点，因此实现在尝试填充 sb 之前应检查 element_name 的内容。如果 element_name 与你的策略名称不匹配，只需返回 0。仅在在外部化标签数据时发生错误时返回非零值。一旦策略填写 element_data ，应递增 *claimed 。

#### 6.7.2.35. `mpo_externalize_vnode_label`

```
int mpo_externalize_vnode_label(	label,
 	element_name,
 	sb,
 	*claimed);
struct label *label;
char *element_name;
struct sbuf *sb;
int *claimed;
```

| 参数 | 描述                               | 锁定 |
| ------ | ------------------------------------ | ------ |
| `label`     | 要外部化的标签                     |      |
| `element_name`     | 应该外部化标签的策略名称           |      |
| `sb`     | 要填充文本表示的标签的字符串缓冲区 |      |
| `claimed`     | 当 element_data 可以填入时应递增。 |      |

根据传递的标签结构生成外部化标签。外部化标签包括标签内容的文本表示，可用于用户应用程序并被用户阅读。当前，将调用所有策略的 externalize 入口点，因此在尝试填入 sb 之前，实现应检查 element_name 的内容。如果 element_name 与你的策略名称不匹配，只需返回 0。仅在在外部化标签数据时发生错误时返回非零值。一旦策略填入 element_data ，应递增 *claimed 。

#### 6.7.2.36. `mpo_internalize_cred_label`

```
int mpo_internalize_cred_label(	label,
 	element_name,
 	element_data,
 	claimed);
struct label *label;
char *element_name;
char *element_data;
int *claimed;
```

| 参数 | 描述                           | 锁定 |
| ------ | -------------------------------- | ------ |
| `label`     | 需要填写的标签                 |      |
| `element_name`     | 要内部化标签的政策名称         |      |
| `element_data`     | 要内部化的文本数据             |      |
| `claimed`     | 当数据可以成功内部化时应递增。 |      |

基于文本格式的外部标签数据生成内部标签结构。目前，当请求内部化时，会调用所有策略的 internalize 入口点，因此实现应该将 element_name 的内容与其自身名称进行比较，以确保它应该内部化 element_data 中的数据。就像在 externalize 入口点中一样，如果 element_name 与其自身名称不匹配，或当数据成功内部化时，入口点应该返回 0，在这种情况下 *claimed 应该递增。

#### 6.7.2.37. `mpo_internalize_ifnet_label`

```
int mpo_internalize_ifnet_label(	label,
 	element_name,
 	element_data,
 	claimed);
struct label *label;
char *element_name;
char *element_data;
int *claimed;
```

| 参数 | 描述                           | 锁定 |
| ------ | -------------------------------- | ------ |
| `label`     | 要填写的标签                   |      |
| `element_name`     | 应将其标签内部化的策略名称     |      |
| `element_data`     | 内部化的文本数据               |      |
| `claimed`     | 当数据可以成功内部化时应递增。 |      |

根据文本格式中外部化的标签数据生成内部标签结构。当前，当请求内部化时，会调用所有策略的 internalize 入口点，因此实现应该将 element_name 的内容与自身名称进行比较，以确保应该内部化 element_data 中的数据。就像 externalize 入口点一样，如果 element_name 与自身名称不匹配，入口点应返回 0，或者当数据可以成功内部化时， *claimed 应该递增。

#### 6.7.2.38. `mpo_internalize_pipe_label`

```
int mpo_internalize_pipe_label(	label,
 	element_name,
 	element_data,
 	claimed);
struct label *label;
char *element_name;
char *element_data;
int *claimed;
```

| 参数 | 描述                             | 锁定 |
| ------ | ---------------------------------- | ------ |
| `label`     | 要填入的标签                     |      |
| `element_name`     | 应该内部化标签的策略名称         |      |
| `element_data`     | 要内部化的文本数据               |      |
| `claimed`     | 当数据可以成功内部化时，应递增。 |      |

基于文本格式中的外部化标签数据生成内部标签结构。当前，当请求内部化时，所有策略的 internalize 入口点都会被调用，因此实现应该将 element_name 的内容与其自身名称进行比较，以确保应该将 element_data 中的数据内部化。就像 externalize 入口点一样，如果 element_name 与自身名称不匹配，入口点应返回 0，或者当数据可以成功内部化时，应递增 *claimed 。

#### 6.7.2.39. `mpo_internalize_socket_label`

```
int mpo_internalize_socket_label(	label,
 	element_name,
 	element_data,
 	claimed);
struct label *label;
char *element_name;
char *element_data;
int *claimed;
```

| 参数 | 描述                       | 锁定 |
| ------ | ---------------------------- | ------ |
| `label`     | 填写标签                   |      |
| `element_name`     | 应内部化标签的策略名称     |      |
| `element_data`     | 要内部化的文本数据         |      |
| `claimed`     | 当数据成功内部化时应递增。 |      |

根据文本格式中的外部化标签数据生成内部标签结构。当前，当请求内部化时，所有策略的 internalize 入口点都会被调用，因此实现应该将 element_name 的内容与自身名称进行比较，以确保应该将 element_data 中的数据内部化。就像 externalize 入口点一样，如果 element_name 与自身名称不匹配，入口点应该返回 0，或者当数据可以成功内部化时， *claimed 应该递增。

#### 6.7.2.40. `mpo_internalize_vnode_label`

```
int mpo_internalize_vnode_label(	label,
 	element_name,
 	element_data,
 	claimed);
struct label *label;
char *element_name;
char *element_data;
int *claimed;
```

| 参数 | 描述                           | 锁定 |
| ------ | -------------------------------- | ------ |
| `label`     | 要填写的标签                   |      |
| `element_name`     | 应内部化其标签的策略名称       |      |
| `element_data`     | 将文字数据内部化               |      |
| `claimed`     | 当数据可以成功内部化时应递增。 |      |

根据文本格式中的外部化标签数据生成内部标签结构。当前，当请求内部化时，所有策略的 internalize 入口点都被调用，因此实现应将 element_name 的内容与其自身的名称进行比较，以确保应该在 element_data 中将数据内部化。就像在 externalize 入口点中一样，如果 element_name 不匹配其自身的名称，则入口点应返回 0，或者当数据可以成功内部化时，应递增 *claimed 。

### 6.7.3. 标签事件

这个入口点类别是 MAC 框架用来允许策略在内核对象上维护标签信息的。对于每个 MAC 策略感兴趣的带标签内核对象，可以注册相关生命周期事件的入口点。所有对象都实现了初始化、创建和销毁挂钩。一些对象还会实现重新标记，允许用户进程更改对象上的标签。一些对象还会实现特定于对象的事件，比如与 IP 重组相关的标签事件。一个典型的带标签对象将具有以下入口点的生命周期：

```
Label initialization          o
(object-specific wait)         \
Label creation                  o
                                 \
Relabel events,                   o--<--.
Various object-specific,          |     |
Access control events             ~-->--o
                                         \
Label destruction                         o
```

标签初始化允许策略为对象分配内存并设置标签的初始值，而不需要对象的使用上下文。分配给策略的标签槽默认情况下会被清零，所以一些策略可能不需要执行初始化。

标签创建发生在内核结构与实际内核对象关联时。例如，Mbufs 可能会在池中分配并保持未使用，直到它们被需要。mbuf 分配导致在 mbuf 上进行标签初始化，但 mbuf 创建发生在 mbuf 与数据报相关联时。通常，将为创建事件提供上下文，包括创建的情况，以及创建过程中其他相关对象的标签。例如，当从套接字创建 mbuf 时，套接字及其标签将被呈现给已注册策略，除了新的 mbuf 及其标签。在创建事件中进行内存分配是不鼓励的，因为它可能发生在内核的性能敏感 ports;此外，不允许创建调用失败，因此无法报告内存分配失败。

特定对象事件通常不属于其他广泛类别的标签事件，但通常会提供机会根据额外上下文修改或更新对象上的标签。例如，在 MAC_UPDATE_IPQ 入口点期间，IP 片段重组队列上的标签可能会在接受附加 mbuf 到该队列时进行更新。

访问控制事件将在以下部分详细讨论。

标签销毁许可策略允许在标签与对象关联期间释放与标签关联的存储或状态，以便支持对象的内核数据结构可以被重用或释放。

除了与特定内核对象关联的标签之外，还存在另一类标签：临时标签。这些标签用于存储用户进程提交的更新信息。这些标签与其他类型的标签一样被初始化和销毁，但创建事件是 MAC_INTERNALIZE，它接受一个用户标签，将其转换为内核表示形式。

#### 6.7.3.1. 文件系统对象标记事件操作

##### 6.7.3.1.1. `mpo_associate_vnode_devfs`

```
void mpo_associate_vnode_devfs(	mp,
 	fslabel,
 	de,
 	delabel,
 	vp,
 	vlabel);
struct mount *mp;
struct label *fslabel;
struct devfs_dirent *de;
struct label *delabel;
struct vnode *vp;
struct label *vlabel;
```

| 参数 | 描述                                   | 锁定 |
| ------ | ---------------------------------------- | ------ |
| `mp`     | Devfs 挂载点                           |      |
| `fslabel`     | Devfs 文件系统标签 ( mp→mnt_fslabel ) |      |
| `de`     | Devfs 目录条目                         |      |
| `delabel`     | 与 de 关联的策略标签                   |      |
| `vp`     | 与 de 关联的 vnode                     |      |
| `vlabel`     | 与 vp 关联的策略标签                   |      |

根据传递给定的 devfs 目录条目及其标签，为新创建的 devfs vnode 填写标签（ vlabel ）。

##### 6.7.3.1.2. `mpo_associate_vnode_extattr`

```
int mpo_associate_vnode_extattr(	mp,
 	fslabel,
 	vp,
 	vlabel);
struct mount *mp;
struct label *fslabel;
struct vnode *vp;
struct label *vlabel;
```

| 参数 | 描述                 | 锁定 |
| ------ | ---------------------- | ------ |
| `mp`     | 文件系统挂载点       |      |
| `fslabel`     | 文件系统标签         |      |
| `vp`     | Vnode 到标签         |      |
| `vlabel`     | 与 vp 关联的策略标签 |      |

尝试从文件系统扩展属性中检索 vp 的标签。成功时，返回值为 0 。如果不支持扩展属性检索，可以接受的替代方法是将 fslabel 复制到 vlabel 。发生错误时，应返回 errno 的适当值。

##### 6.7.3.1.3. `mpo_associate_vnode_singlelabel`

```
void mpo_associate_vnode_singlelabel(	mp,
 	fslabel,
 	vp,
 	vlabel);
struct mount *mp;
struct label *fslabel;
struct vnode *vp;
struct label *vlabel;
```

| 参数 | 描述                 | 锁定 |
| ------ | ---------------------- | ------ |
| `mp`     | 文件系统挂载点       |      |
| `fslabel`     | 文件系统标签         |      |
| `vp`     | Vnode 到标签         |      |
| `vlabel`     | 与 vp 关联的策略标签 |      |

在非多标签文件系统上，根据文件系统标签 fslabel ，调用此入口点设置 vp 的策略标签。

##### 6.7.3.1.4. `mpo_create_devfs_device`

```
void mpo_create_devfs_device(	dev,
 	devfs_dirent,
 	label);
dev_t dev;
struct devfs_dirent *devfs_dirent;
struct label *label;
```

| 参数 | 描述                           | 锁定 |
| ------ | -------------------------------- | ------ |
| `dev`     | 与 devfs_dirent 对应的设备     |      |
| `devfs_dirent`     | 要标记的 Devfs 目录条目。      |      |
| `label`     | 用于填写 devfs_dirent 的标签。 |      |

填写要为传递的设备创建的 devfs_dirent 上的标签。当设备文件系统被挂载、重新生成或新设备可用时，将调用此函数。

##### 6.7.3.1.5. `mpo_create_devfs_directory`

```
void mpo_create_devfs_directory(	dirname,
 	dirnamelen,
 	devfs_dirent,
 	label);
char *dirname;
int dirnamelen;
struct devfs_dirent *devfs_dirent;
struct label *label;
```

| 参数 | 描述                            | 锁定 |
| ------ | --------------------------------- | ------ |
| `dirname`     | 创建目录的名称                  |      |
| `namelen`     | 字符串的长度 dirname            |      |
| `devfs_dirent`     | 正在创建的目录的 Devfs 目录条目 |      |

填写为传递目录创建的 devfs_dirent 上的标签。当设备文件系统被挂载、重新生成或提供需要特定目录层次结构的新设备时，将调用此函数。

##### 6.7.3.1.6. `mpo_create_devfs_symlink`

```
void mpo_create_devfs_symlink(	cred,
 	mp,
 	dd,
 	ddlabel,
 	de,
 	delabel);
struct ucred *cred;
struct mount *mp;
struct devfs_dirent *dd;
struct label *ddlabel;
struct devfs_dirent *de;
struct label *delabel;
```

| 参数 | 描述               | 锁定 |
| ------ | -------------------- | ------ |
| `cred`     | 主题凭证           |      |
| `mp`     | Devfs 挂载点       |      |
| `dd`     | 链接目标           |      |
| `ddlabel`     | 与 dd 相关联的标签 |      |
| `de`     | 符号链接条目       |      |
| `delabel`     | 与 de 关联的标签   |      |

为新创建的 devfs(5) 符号链接条目填写标签 ( delabel )。

##### 6.7.3.1.7. `mpo_create_vnode_extattr`

```
int mpo_create_vnode_extattr(	cred,
 	mp,
 	fslabel,
 	dvp,
 	dlabel,
 	vp,
 	vlabel,
 	cnp);
struct ucred *cred;
struct mount *mp;
struct label *fslabel;
struct vnode *dvp;
struct label *dlabel;
struct vnode *vp;
struct label *vlabel;
struct componentname *cnp;
```

| 参数 | 描述                 | 锁定 |
| ------ | ---------------------- | ------ |
| `cred`     | 主体凭证             |      |
| `mount`     | 文件系统挂载点       |      |
| `label`     | 文件系统标签         |      |
| `dvp`     | 父目录 vnode         |      |
| `dlabel`     | 与 dvp 关联的标签    |      |
| `vp`     | 新创建的 vnode       |      |
| `vlabel`     | 与 vp 关联的策略标签 |      |
| `cnp`     | vp 的组件名称        |      |

将 vp 的标签写入适当的扩展属性中。如果写入成功，请使用标签填充 vlabel ，然后返回 0。否则，返回适当的错误。

##### 6.7.3.1.8. `mpo_create_mount`

```
void mpo_create_mount(	cred,
 	mp,
 	mnt,
 	fslabel);
struct ucred *cred;
struct mount *mp;
struct label *mnt;
struct label *fslabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `cred`     | 主体凭证               |      |
| `mp`     | 被挂载的对象;文件系统  |      |
| `mntlabel`     | 用于填写 mp 的策略标签 |      |
| `fslabel`     | mp 挂载的策略标签      |      |

通过传递的主题凭据填写正在创建的挂载点上的标签。当挂载新文件系统时将调用此函数。

##### 6.7.3.1.9. `mpo_create_root_mount`

```
void mpo_create_root_mount(	cred,
 	mp,
 	mntlabel,
 	fslabel);
struct ucred *cred;
struct mount *mp;
struct label *mntlabel;
struct label *fslabel;
```

| 参数                           | 描述 | 锁定 |
| -------------------------------- | ------ | ------ |
| 看 mpo\_create\_mount 。 |      |      |

填写由传递的主体凭证创建的挂载点上的标签。此调用将在根文件系统挂载后进行。

##### 6.7.3.1.10. `mpo_relabel_vnode`

```
void mpo_relabel_vnode(	cred,
 	vp,
 	vnodelabel,
 	newlabel);
struct ucred *cred;
struct vnode *vp;
struct label *vnodelabel;
struct label *newlabel;
```

| 参数 | 描述                                   | 锁定 |
| ------ | ---------------------------------------- | ------ |
| `cred`     | 主体凭证                               |      |
| `vp`     | vnode 重新标记                         |      |
| `vnodelabel`     | vp 的现有策略标签                      |      |
| `newlabel`     | 替换 vnodelabel 的新标签，可能是部分的 |      |

根据传递的更新 vnode 标签和传递的主体凭证，在传递的 vnode 上更新标签。

##### 6.7.3.1.11. `mpo_setlabel_vnode_extattr`

```
int mpo_setlabel_vnode_extattr(	cred,
 	vp,
 	vlabel,
 	intlabel);
struct ucred *cred;
struct vnode *vp;
struct label *vlabel;
struct label *intlabel;
```

| 参数 | 描述                 | 锁定 |
| ------ | ---------------------- | ------ |
| `cred`     | 主题凭证             |      |
| `vp`     | 写入标签的 Vnode     |      |
| `vlabel`     | 与 vp 相关的策略标签 |      |
| `intlabel`     | 要写出的标签         |      |

写出从 intlabel 到扩展属性的策略。这称为从 vop_stdcreatevnode_ea 调用。

##### 6.7.3.1.12. `mpo_update_devfsdirent`

```
void mpo_update_devfsdirent(	devfs_dirent,
 	direntlabel,
 	vp,
 	vnodelabel);
struct devfs_dirent *devfs_dirent;
struct label *direntlabel;
struct vnode *vp;
struct label *vnodelabel;
```

| 参数 | 描述                             | 锁定          |
| ------ | ---------------------------------- | --------------- |
| `devfs_dirent`     | 对象；devfs 目录条目             |               |
| `direntlabel`     | 要更新为 devfs_dirent 的策略标签 |               |
| `vp`     | 父 vnode                         | 锁定          |
|      | `vnodelabel`                                 | vp 的策略标签 |

从传递的 devfs vnode 标签更新 devfs_dirent 标签。当 devfs vnode 成功重新标记以提交标签更改时，将调用此调用，以使标签更改持久，即使 vnode 被回收也是如此。当在 devfs 中创建符号链接时，将调用 mac_vnode_create_from_vnode 来初始化 vnode 标签后，也将进行此调用。

#### 6.7.3.2. IPC 对象标记事件操作

##### 6.7.3.2.1. `mpo_create_mbuf_from_socket`

```
void mpo_create_mbuf_from_socket(	so,
 	socketlabel,
 	m,
 	mbuflabel);
struct socket *so;
struct label *socketlabel;
struct mbuf *m;
struct label *mbuflabel;
```

| 参数 | 描述                  | 锁定                     |
| ------ | ----------------------- | -------------------------- |
| `socket`     | 插座                  | 套接字锁定工作正在进行中 |
| `socketlabel`     | socket 的策略标签     |                          |
| `m`     | 对象; mbuf            |                          |
| `mbuflabel`     | 用于填写 m 的策略标签 |                          |

从传递的套接字标签设置新创建的 mbuf 标头上的标签。当套接字生成新的数据报文或消息并存储在传递的 mbuf 中时，会调用此函数。

##### 6.7.3.2.2. `mpo_create_pipe`

```
void mpo_create_pipe(	cred,
 	pipe,
 	pipelabel);
struct ucred *cred;
struct pipe *pipe;
struct label *pipelabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `cred`     | 主体凭证               |      |
| `pipe`     | 管道                   |      |
| `pipelabel`     | 与 pipe 关联的策略标签 |      |

从传递的主体凭据上设置新创建的管道上的标签。在创建新管道时会调用此方法。

##### 6.7.3.2.3. `mpo_create_socket`

```
void mpo_create_socket(	cred,
 	so,
 	socketlabel);
struct ucred *cred;
struct socket *so;
struct label *socketlabel;
```

| 参数 | 描述                | 锁定   |
| ------ | --------------------- | -------- |
| `cred`     | 学科证书            | 不变的 |
| `so`     | 对象;套接字进行标记 |        |
| `socketlabel`     | 用于填充 so 的标签  |        |

从传递的主体凭据设置新创建套接字上的标签。当套接字被创建时进行此调用。

##### 6.7.3.2.4. `mpo_create_socket_from_socket`

```
void mpo_create_socket_from_socket(	oldsocket,
 	oldsocketlabel,
 	newsocket,
 	newsocketlabel);
struct socket *oldsocket;
struct label *oldsocketlabel;
struct socket *newsocket;
struct label *newsocketlabel;
```

| 参数 | 描述                               | 锁定 |
| ------ | ------------------------------------ | ------ |
| `oldsocket`     | 监听套接字                         |      |
| `oldsocketlabel`     | 与 oldsocket 相关联的策略标签      |      |
| `newsocket`     | 新套接字                           |      |
| `newsocketlabel`     | 与 newsocketlabel 相关联的策略标签 |      |

通过基于 listen(2) 套接字 oldsocket 标记套接字 newsocket , 新接受(2) 的

##### 6.7.3.2.5. `mpo_relabel_pipe`

```
void mpo_relabel_pipe(	cred,
 	pipe,
 	oldlabel,
 	newlabel);
struct ucred *cred;
struct pipe *pipe;
struct label *oldlabel;
struct label *newlabel;
```

| 参数 | 描述                         | 锁定 |
| ------ | ------------------------------ | ------ |
| `cred`     | 主题凭证                     |      |
| `pipe`     | 管道                         |      |
| `oldlabel`     | 与 pipe 关联的当前策略标签   |      |
| `newlabel`     | 要应用于 pipe 的策略标签更新 |      |

将一个新标签 newlabel 应用于 pipe 。

##### 6.7.3.2.6. `mpo_relabel_socket`

```
void mpo_relabel_socket(	cred,
 	so,
 	oldlabel,
 	newlabel);
struct ucred *cred;
struct socket *so;
struct label *oldlabel;
struct label *newlabel;
```

| 参数 | 描述          | 锁定     |
| ------ | --------------- | ---------- |
| `cred`     | 主题凭证      | 不可变的 |
| `so`     | 对象; 套接字  |          |
| `oldlabel`     | so 的当前标签 |          |
| `newlabel`     | so 的标签更新 |          |

从传递的套接字标签更新更新套接字上的标签

##### 6.7.3.2.7. `mpo_set_socket_peer_from_mbuf`

```
void mpo_set_socket_peer_from_mbuf(	mbuf,
 	mbuflabel,
 	oldlabel,
 	newlabel);
struct mbuf *mbuf;
struct label *mbuflabel;
struct label *oldlabel;
struct label *newlabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `mbuf`     | 收到的第一份数据报文   |      |
| `mbuflabel`     | mbuf 的标签            |      |
| `oldlabel`     | 套接字的当前标签       |      |
| `newlabel`     | 要填写套接字的策略标签 |      |

从传递的 mbuf 标签在流套接字上设置对等标签。只有在通过流套接字接收到第一个数据报时，才会调用此调用，Unix 域套接字除外。

##### 6.7.3.2.8. `mpo_set_socket_peer_from_socket`

```
void mpo_set_socket_peer_from_socket(	oldsocket,
 	oldsocketlabel,
 	newsocket,
 	newsocketpeerlabel);
struct socket *oldsocket;
struct label *oldsocketlabel;
struct socket *newsocket;
struct label *newsocketpeerlabel;
```

| 参数 | 描述                       | 锁定 |
| ------ | ---------------------------- | ------ |
| `oldsocket`     | 本地套接字                 |      |
| `oldsocketlabel`     | 策略标签用于 oldsocket     |      |
| `newsocket`     | 对等套接字                 |      |
| `newsocketpeerlabel`     | 填写策略标签用于 newsocket |      |

从传递的远程套接字端点为流 UNIX 域套接字设置对等标签。当套接字对连接时，将调用此调用，并将为两个端点都进行调用。

#### 6.7.3.3. 网络对象标记事件操作

##### 6.7.3.3.1. `mpo_create_bpfdesc`

```
void mpo_create_bpfdesc(	cred,
 	bpf_d,
 	bpflabel);
struct ucred *cred;
struct bpf_d *bpf_d;
struct label *bpflabel;
```

| 参数 | 描述                   | 锁定   |
| ------ | ------------------------ | -------- |
| `cred`     | 主体凭证               | 不变的 |
| `bpf_d`     | 对象; bpf 描述符       |        |
| `bpf`     | 要填写的策略标签 bpf_d |        |

从传递的主体凭证为新创建的 BPF 描述符设置标签。当由具有传递的主体凭证的进程打开 BPF 设备节点时，将调用此函数。

##### 6.7.3.3.2. `mpo_create_ifnet`

```
void mpo_create_ifnet(	ifnet,
 	ifnetlabel);
struct ifnet *ifnet;
struct label *ifnetlabel;
```

| 参数 | 描述                      | 锁定 |
| ------ | --------------------------- | ------ |
| `ifnet`     | 网络接口                  |      |
| `ifnetlabel`     | 用于填写 ifnet 的策略标签 |      |

创建接口的标签。当新的物理接口对系统可用时，或者当通过引导或用户操作实例化伪接口时，可能会发起此调用。

##### 6.7.3.3.3. `mpo_create_ipq`

```
void mpo_create_ipq(	fragment,
 	fragmentlabel,
 	ipq,
 	ipqlabel);
struct mbuf *fragment;
struct label *fragmentlabel;
struct ipq *ipq;
struct label *ipqlabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `fragment`     | 首个接收的 IP 片段     |      |
| `fragmentlabel`     | fragment 的策略标签    |      |
| `ipq`     | 要为 IP 重组队列标记   |      |
| `ipqlabel`     | 要填写的策略标签为 ipq |      |

从第一个接收到的片段的 mbuf 标头上设置新创建的 IP 片段重组队列的标签。

##### 6.7.3.3.4. `mpo_create_datagram_from_ipq`

```
void mpo_create_create_datagram_from_ipq(	ipq,
 	ipqlabel,
 	datagram,
 	datagramlabel);
struct ipq *ipq;
struct label *ipqlabel;
struct mbuf *datagram;
struct label *datagramlabel;
```

| 参数 | 描述                             | 锁定 |
| ------ | ---------------------------------- | ------ |
| `ipq`     | IP 重组队列                      |      |
| `ipqlabel`     | ipq 的策略标签                   |      |
| `datagram`     | 待标记的数据报                   |      |
| `datagramlabel`     | 要填写的策略标签为 datagramlabel |      |

设置从生成它的 IP 片段重组队列中重新组装的 IP 数据报上的标签。

##### 6.7.3.3.5. `mpo_create_fragment`

```
void mpo_create_fragment(	datagram,
 	datagramlabel,
 	fragment,
 	fragmentlabel);
struct mbuf *datagram;
struct label *datagramlabel;
struct mbuf *fragment;
struct label *fragmentlabel;
```

| 参数 | 描述                      | 锁定 |
| ------ | --------------------------- | ------ |
| `datagram`     | 数据包                    |      |
| `datagramlabel`     | datagram 的政策标签       |      |
| `fragment`     | 要标记的片段              |      |
| `fragmentlabel`     | 要填写的政策标签 datagram |      |

将新创建的 IP 片段的 mbuf 标签设置为生成该片段的数据报的 mbuf 标头上的标签。

##### 6.7.3.3.6. `mpo_create_mbuf_from_mbuf`

```
void mpo_create_mbuf_from_mbuf(	oldmbuf,
 	oldmbuflabel,
 	newmbuf,
 	newmbuflabel);
struct mbuf *oldmbuf;
struct label *oldmbuflabel;
struct mbuf *newmbuf;
struct label *newmbuflabel;
```

| 参数 | 描述                      | 锁定 |
| ------ | --------------------------- | ------ |
| `oldmbuf`     | 现有（源）mbuf            |      |
| `oldmbuflabel`     | oldmbuf 的策略标签        |      |
| `newmbuf`     | 新的 mbuf 将被标记        |      |
| `newmbuflabel`     | 要为 newmbuf 填写策略标签 |      |

在从现有数据报的 mbuf 标头到新创建的数据报的 mbuf 标头上设置标签。可以在许多情况下调用此函数，包括为了对齐目的重新分配 mbuf 时。

##### 6.7.3.3.7. `mpo_create_mbuf_linklayer`

```
void mpo_create_mbuf_linklayer(	ifnet,
 	ifnetlabel,
 	mbuf,
 	mbuflabel);
struct ifnet *ifnet;
struct label *ifnetlabel;
struct mbuf *mbuf;
struct label *mbuflabel;
```

| 参数 | 描述                     | 锁定 |
| ------ | -------------------------- | ------ |
| `ifnet`     | 网络接口                 |      |
| `ifnetlabel`     | ifnet 的策略标签         |      |
| `mbuf`     | 新数据报的 mbuf 标头     |      |
| `mbuflabel`     | 用于填写 mbuf 的策略标签 |      |

为通过接口传递链路层响应生成的新创建数据报的 mbuf 标头设置标签。 可在多种情况下调用此函数，包括 IPv4 和 IPv6 堆栈中的 ARP 或 ND6 响应。

##### 6.7.3.3.8. `mpo_create_mbuf_from_bpfdesc`

```
void mpo_create_mbuf_from_bpfdesc(	bpf_d,
 	bpflabel,
 	mbuf,
 	mbuflabel);
struct bpf_d *bpf_d;
struct label *bpflabel;
struct mbuf *mbuf;
struct label *mbuflabel;
```

| 参数 | 描述                     | 锁定 |
| ------ | -------------------------- | ------ |
| `bpf_d`     | BPF 描述符               |      |
| `bpflabel`     | 用于 bpflabel 的策略标签 |      |
| `mbuf`     | 新的 mbuf 将被标记       |      |
| `mbuflabel`     | 用于填写 mbuf 的策略标签 |      |

使用传递的 BPF 描述符生成新创建的数据报时，在 MBUF 标头上设置标签。当向与传递的 BPF 描述符关联的 BPF 设备执行写操作时，将调用此函数。

##### 6.7.3.3.9. `mpo_create_mbuf_from_ifnet`

```
void mpo_create_mbuf_from_ifnet(	ifnet,
 	ifnetlabel,
 	mbuf,
 	mbuflabel);
struct ifnet *ifnet;
struct label *ifnetlabel;
struct mbuf *mbuf;
struct label *mbuflabel;
```

| 参数 | 描述                      | 锁定 |
| ------ | --------------------------- | ------ |
| `ifnet`     | 网络接口                  |      |
| `ifnetlabel`     | ifnetlabel 的策略标签     |      |
| `mbuf`     | 新数据报的 mbuf 头        |      |
| `mbuflabel`     | 待填入的 Policy 标签 mbuf |      |

在从传递的网络接口生成的新创建的数据报的 mbuf 头上设置标签

##### 6.7.3.3.10. `mpo_create_mbuf_multicast_encap`

```
void mpo_create_mbuf_multicast_encap(	oldmbuf,
 	oldmbuflabel,
 	ifnet,
 	ifnetlabel,
 	newmbuf,
 	newmbuflabel);
struct mbuf *oldmbuf;
struct label *oldmbuflabel;
struct ifnet *ifnet;
struct label *ifnetlabel;
struct mbuf *newmbuf;
struct label *newmbuflabel;
```

| 参数 | 描述                       | 锁定 |
| ------ | ---------------------------- | ------ |
| `oldmbuf`     | 现有数据报的 mbuf 标头     |      |
| `oldmbuflabel`     | oldmbuf 的策略标签         |      |
| `ifnet`     | 网络接口                   |      |
| `ifnetlabel`     | 用于 ifnet 的策略标签      |      |
| `newmbuf`     | 用于新数据报的 mbuf 标头   |      |
| `newmbuflabel`     | 用于填写的策略标签 newmbuf |      |

在通过传递的组播封装接口处理时，为新创建的数据报文设置从现有传递的数据报文生成的 mbuf 标签。当要使用虚拟接口传递 mbuf 时，进行此调用。

##### 6.7.3.3.11. `mpo_create_mbuf_netlayer`

```
void mpo_create_mbuf_netlayer(	oldmbuf,
 	oldmbuflabel,
 	newmbuf,
 	newmbuflabel);
struct mbuf *oldmbuf;
struct label *oldmbuflabel;
struct mbuf *newmbuf;
struct label *newmbuflabel;
```

| 参数 | 描述               | 锁定 |
| ------ | -------------------- | ------ |
| `oldmbuf`     | 收到数据包         |      |
| `oldmbuflabel`     | oldmbuf 的策略标签 |      |
| `newmbuf`     | 新创建的数据报     |      |
| `newmbuflabel`     | newmbuf 的策略标签 |      |

在响应现有接收到的数据报时，设置由 IP 栈生成的新创建数据报的 mbuf 标签头（ oldmbuf ）。可以在许多情况下调用此函数，包括响应 ICMP 请求数据报时。

##### 6.7.3.3.12. `mpo_fragment_match`

```
int mpo_fragment_match(	fragment,
 	fragmentlabel,
 	ipq,
 	ipqlabel);
struct mbuf *fragment;
struct label *fragmentlabel;
struct ipq *ipq;
struct label *ipqlabel;
```

| 参数 | 描述                | 锁定 |
| ------ | --------------------- | ------ |
| `fragment`     | IP 数据报片段       |      |
| `fragmentlabel`     | fragment 的策略标签 |      |
| `ipq`     | IP 片段重组队列     |      |
| `ipqlabel`     | ipq 的政策标签      |      |

确定包含 IP 数据报 ( fragment ) 片段的 mbuf 头部是否与传递的 IP 片段重组队列 ( ipq ) 的标签相匹配。对于成功匹配返回 (1)，对于无匹配返回 (0)。在 IP 栈尝试为新收到的片段找到现有的片段重组队列时进行此调用；如果失败，可以为该片段实例化新的片段重组队列。策略可以利用此入口点防止基于标签或其他信息不允许其重新组装的情况下重新组装其他匹配的 IP 片段。

##### 6.7.3.3.13. `mpo_relabel_ifnet`

```
void mpo_relabel_ifnet(	cred,
 	ifnet,
 	ifnetlabel,
 	newlabel);
struct ucred *cred;
struct ifnet *ifnet;
struct label *ifnetlabel;
struct label *newlabel;
```

| 参数 | 描述                    | 锁定 |
| ------ | ------------------------- | ------ |
| `cred`     | 主体凭据                |      |
| `ifnet`     | 对象; 网络接口          |      |
| `ifnetlabel`     | ifnet 的策略标签        |      |
| `newlabel`     | 应用于 ifnet 的标签更新 |      |

根据传递的更新标签 newlabel 和传递的主题凭据 cred 更新网络接口 ifnet 的标签。

##### 6.7.3.3.14. `mpo_update_ipq`

```
void mpo_update_ipq(	fragment,
 	fragmentlabel,
 	ipq,
 	ipqlabel);
struct mbuf *fragment;
struct label *fragmentlabel;
struct ipq *ipq;
struct label *ipqlabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `mbuf`     | IP 片段                |      |
| `mbuflabel`     | mbuf 的策略标签        |      |
| `ipq`     | IP 片段重组队列        |      |
| `ipqlabel`     | 要更新的策略标签为 ipq |      |

根据接受传递的 IP 片段 mbuf 标头（ mbuf ）来更新 IP 片段重组队列（ ipq ）上的标签。

#### 6.7.3.4. 处理标签事件操作

##### 6.7.3.4.1. `mpo_create_cred`

```
void mpo_create_cred(	parent_cred,
 	child_cred);
struct ucred *parent_cred;
struct ucred *child_cred;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `parent_cred`     | 父主体凭证 |      |
| `child_cred`     | 子主体凭证 |      |

从传递的主体凭证设置新创建的主体凭证的标签。当在新创建的 struct ucred 上调用 crcopy(9) 时将进行此调用。此调用不应与进程复制或创建事件混淆。

##### 6.7.3.4.2. `mpo_execve_transition`

```
void mpo_execve_transition(	old,
 	new,
 	vp,
 	vnodelabel);
struct ucred *old;
struct ucred *new;
struct vnode *vp;
struct label *vnodelabel;
```

| 参数 | 描述             | 锁定     |
| ------ | ------------------ | ---------- |
| `old`     | 现有主题凭据     | 不可改变 |
| `new`     | 新主体凭证需标记 |          |
| `vp`     | 要执行的文件     | 已锁定   |
| `vnodelabel`     | vp 的政策标签    |          |

根据执行传递的 vnode（ vp ）引起的标签转换，从已传递的现有主体凭证（ old ）更新新创建的主体凭证（ new ）的标签。当一个过程执行传递的 vnode 并且其中一个策略从 mpo_execve_will_transition 入口点返回成功时，将发生此调用。策略可以选择通过调用 mpo_create_cred 并传递两个主体凭证来实现此调用，以便不实现过渡事件。即使策略不实现 mpo_execve_will_transition ，在实现 mpo_create_cred 时也不应使该入口点未实现。

##### 6.7.3.4.3. `mpo_execve_will_transition`

```
int mpo_execve_will_transition(	old,
 	vp,
 	vnodelabel);
struct ucred *old;
struct vnode *vp;
struct label *vnodelabel;
```

| 参数 | 描述                           | 锁定   |
| ------ | -------------------------------- | -------- |
| `old`     | 在执行 execve(2)之前的主体凭证 | 不可变 |
| `vp`     | 执行的文件                     |        |
| `vnodelabel`     | vp 的策略标签                  |        |

确定策略是否希望根据传递的 vnode 的执行结果以及传递的主体凭证执行过渡事件。如果需要过渡，则返回 1，否则返回 0。即使策略返回 0，在出现对 mpo_execve_transition 的意外调用时也应正确执行，因为该调用可能是另一个策略请求过渡的结果。

##### 6.7.3.4.4. `mpo_create_proc0`

```
void mpo_create_proc0(	cred);
struct ucred *cred;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `cred`     | 需要填写主体凭据 |      |

创建进程 0 的主体凭据，即所有内核进程的父进程。

##### 6.7.3.4.5. `mpo_create_proc1`

```
void mpo_create_proc1(	cred);
struct ucred *cred;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `cred`     | 需填写主体凭证 |      |

创建进程 1 的主体凭证，所有用户进程的父进程。

##### 6.7.3.4.6. `mpo_relabel_cred`

```
void mpo_relabel_cred(	cred,
 	newlabel);
struct ucred *cred;
struct label *newlabel;
```

| 参数 | 描述                  | 锁定 |
| ------ | ----------------------- | ------ |
| `cred`     | 学科凭证              |      |
| `newlabel`     | 更新标签以应用于 cred |      |

更新主题凭据上的标签从传递的更新标签。

### 6.7.4. 访问控制检查

访问控制条目点允许策略模块影响内核所做的访问控制决策。一般来说，虽然不是总是这样，访问控制条目点的参数将包括一个或多个授权凭证，操作涉及的任何其他对象的信息（可能包括标签）。访问控制条目点可能返回 0 以允许操作，或者一个 errno(2)错误值。调用各个已注册的策略模块中的条目点的结果将组合如下：如果所有模块都允许操作成功，则将返回成功。如果一个或多个模块返回失败，则将返回一个失败。如果多个模块返回失败，则将使用以下优先顺序选择要返回给用户的 errno 值，该优先顺序由 kern_mac.c 中的 error_select() 函数实现：

| 最高优先级 | EDEADLK |
| ------------ | --------- |
|            | EINVAL  |
|            | ESRCH   |
|            | EACCES  |
| 最低优先级 | EPERM   |

如果所有模块返回的错误值都不在优先级图表中列出，则将从集合中任意选择一个值返回。一般情况下，规则按以下顺序对错误赋予优先级：内核故障，无效参数，对象不存在，访问不允许，其他。

#### 6.7.4.1. `mpo_check_bpfdesc_receive`

```
int mpo_check_bpfdesc_receive(	bpf_d,
 	bpflabel,
 	ifnet,
 	ifnetlabel);
struct bpf_d *bpf_d;
struct label *bpflabel;
struct ifnet *ifnet;
struct label *ifnetlabel;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `bpf_d`     | 主题; BPF 描述符 |      |
| `bpflabel`     | bpf_d 的策略标签 |      |
| `ifnet`     | 对象; 网络接口   |      |
| `ifnetlabel`     | ifnet 的策略标签 |      |

确定 MAC 框架是否允许从传递的接口传递数据报文到传递的 BPF 描述符的缓冲区。成功返回(0)，失败返回 errno 值。建议的失败原因：标签不匹配为 EACCES，缺乏权限为 EPERM。

#### 6.7.4.2. `mpo_check_kenv_dump`

```
int mpo_check_kenv_dump(	cred);
struct ucred *cred;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `cred`     | 主题凭证 |      |

确定是否应允许主体检索内核环境（请参阅 kenv(2)）。

#### 6.7.4.3. `mpo_check_kenv_get`

```
int mpo_check_kenv_get(	cred,
 	name);
struct ucred *cred;
char *name;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `cred`     | 主题凭证         |      |
| `name`     | 内核环境变量名称 |      |

确定是否应允许主体检索指定内核环境变量的值。

#### 6.7.4.4. `mpo_check_kenv_set`

```
int mpo_check_kenv_set(	cred,
 	name);
struct ucred *cred;
char *name;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `cred`     | 主题凭证       |      |
| `name`     | 内核环境变量名 |      |

确定是否应允许主题设置指定的内核环境变量。

#### 6.7.4.5. `mpo_check_kenv_unset`

```
int mpo_check_kenv_unset(	cred,
 	name);
struct ucred *cred;
char *name;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `cred`     | 主体凭证         |      |
| `name`     | 内核环境变量名称 |      |

确定是否应允许主体取消设置指定的内核环境变量。

#### 6.7.4.6. `mpo_check_kld_load`

```
int mpo_check_kld_load(	cred,
 	vp,
 	vlabel);
struct ucred *cred;
struct vnode *vp;
struct label *vlabel;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `cred`     | 主题凭证         |      |
| `vp`     | 内核模块 vnode   |      |
| `vlabel`     | 与 vp 关联的标签 |      |

确定主题是否应被允许加载指定的模块文件。

#### 6.7.4.7. `mpo_check_kld_stat`

```
int mpo_check_kld_stat(	cred);
struct ucred *cred;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `cred`     | 主题凭证 |      |

确定是否应允许主体检索已加载的内核模块文件列表和相关统计信息。

#### 6.7.4.8. `mpo_check_kld_unload`

```
int mpo_check_kld_unload(	cred);
struct ucred *cred;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `cred`     | 主体凭证 |      |

确定是否应允许主题卸载内核模块。

#### 6.7.4.9. `mpo_check_pipe_ioctl`

```
int mpo_check_pipe_ioctl(	cred,
 	pipe,
 	pipelabel,
 	cmd,
 	data);
struct ucred *cred;
struct pipe *pipe;
struct label *pipelabel;
unsigned long cmd;
void *data;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `cred`     | 主体凭证               |      |
| `pipe`     | 管道                   |      |
| `pipelabel`     | 与 pipe 关联的策略标签 |      |
| `cmd`     | ioctl(2) 命令          |      |
| `data`     | ioctl(2) 数据          |      |

确定是否应允许主体进行指定的 ioctl(2)调用。

#### 6.7.4.10. `mpo_check_pipe_poll`

```
int mpo_check_pipe_poll(	cred,
 	pipe,
 	pipelabel);
struct ucred *cred;
struct pipe *pipe;
struct label *pipelabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `cred`     | 主题凭证               |      |
| `pipe`     | 管道                   |      |
| `pipelabel`     | 与 pipe 关联的策略标签 |      |

确定是否应允许主体轮询 pipe 。

#### 6.7.4.11. `mpo_check_pipe_read`

```
int mpo_check_pipe_read(	cred,
 	pipe,
 	pipelabel);
struct ucred *cred;
struct pipe *pipe;
struct label *pipelabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `cred`     | 主体凭证               |      |
| `pipe`     | 管道                   |      |
| `pipelabel`     | 与 pipe 相关的策略标签 |      |

确定是否应允许主题读取 pipe 的访问。

#### 6.7.4.12. `mpo_check_pipe_relabel`

```
int mpo_check_pipe_relabel(	cred,
 	pipe,
 	pipelabel,
 	newlabel);
struct ucred *cred;
struct pipe *pipe;
struct label *pipelabel;
struct label *newlabel;
```

| 参数 | 描述                       | 锁定 |
| ------ | ---------------------------- | ------ |
| `cred`     | 凭证                       |      |
| `pipe`     | 管道                       |      |
| `pipelabel`     | 与 pipe 相关的当前策略标签 |      |
| `newlabel`     | 标签更新为 pipelabel       |      |

确定是否应允许主体重新标记 pipe 。

#### 6.7.4.13. `mpo_check_pipe_stat`

```
int mpo_check_pipe_stat(	cred,
 	pipe,
 	pipelabel);
struct ucred *cred;
struct pipe *pipe;
struct label *pipelabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `cred`     | 主体凭据               |      |
| `pipe`     | 管道                   |      |
| `pipelabel`     | 与 pipe 相关的策略标签 |      |

确定是否应允许主体检索与 pipe 相关的统计信息

#### 6.7.4.14. `mpo_check_pipe_write`

```
int mpo_check_pipe_write(	cred,
 	pipe,
 	pipelabel);
struct ucred *cred;
struct pipe *pipe;
struct label *pipelabel;
```

| 参数 | 描述                     | 锁定 |
| ------ | -------------------------- | ------ |
| `cred`     | 主体凭证                 |      |
| `pipe`     | 管道                     |      |
| `pipelabel`     | 与 pipe 相关联的策略标签 |      |

确定主题是否应该被允许写入 pipe 。

#### 6.7.4.15. `mpo_check_socket_bind`

```
int mpo_check_socket_bind(	cred,
 	socket,
 	socketlabel,
 	sockaddr);
struct ucred *cred;
struct socket *socket;
struct label *socketlabel;
struct sockaddr *sockaddr;
```

| 参数 | 描述              | 锁定 |
| ------ | ------------------- | ------ |
| `cred`     | 主体凭证          |      |
| `socket`     | 要绑定的套接字    |      |
| `socketlabel`     | socket 的政策标签 |      |
| `sockaddr`     | socket 的地址     |      |

#### 6.7.4.16. `mpo_check_socket_connect`

```
int mpo_check_socket_connect(	cred,
 	socket,
 	socketlabel,
 	sockaddr);
struct ucred *cred;
struct socket *socket;
struct label *socketlabel;
struct sockaddr *sockaddr;
```

| 参数 | 描述              | 锁定 |
| ------ | ------------------- | ------ |
| `cred`     | 主体凭证          |      |
| `socket`     | 需要连接的插座    |      |
| `socketlabel`     | socket 的策略标签 |      |
| `sockaddr`     | socket 的地址     |      |

确定主体凭证（ cred ）是否可以将传递的套接字（ socket ）连接到传递的套接字地址（ sockaddr ）。成功返回 0，或者失败返回 errno 值。建议的失败: 标签不匹配的话返回 EACCES，缺乏特权返回 EPERM。

#### 6.7.4.17. `mpo_check_socket_receive`

```
int mpo_check_socket_receive(	cred,
 	so,
 	socketlabel);
struct ucred *cred;
struct socket *so;
struct label *socketlabel;
```

| 参数 | 描述                 | 锁定 |
| ------ | ---------------------- | ------ |
| `cred`     | 主体凭证             |      |
| `so`     | 套接字               |      |
| `socketlabel`     | 与 so 关联的策略标签 |      |

确定是否应允许主体从套接字 so 接收信息。

#### 6.7.4.18. `mpo_check_socket_send`

```
int mpo_check_socket_send(	cred,
 	so,
 	socketlabel);
struct ucred *cred;
struct socket *so;
struct label *socketlabel;
```

| 参数 | 描述                   | 锁定 |
| ------ | ------------------------ | ------ |
| `cred`     | 主题凭证               |      |
| `so`     | 套接字                 |      |
| `socketlabel`     | 与 so 相关联的策略标签 |      |

确定主体是否应该被允许通过套接字 so 发送信息。

#### 6.7.4.19. `mpo_check_cred_visible`

```
int mpo_check_cred_visible(	u1,
 	u2);
struct ucred *u1;
struct ucred *u2;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `u1`     | 主体凭证 |      |
| `u2`     | 客体凭证 |      |

确定是否通过传递的主体凭证 u1 可以“看见”具有传递的主体凭证 u2 的其他主体。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，缺乏特权的 EPERM，或者隐藏可见性的 ESRCH。此调用可能在许多情况下进行，包括由 ps 使用的进程间状态 sysctl 和 procfs 查找。

#### 6.7.4.20. `mpo_check_socket_visible`

```
int mpo_check_socket_visible(	cred,
 	socket,
 	socketlabel);
struct ucred *cred;
struct socket *socket;
struct label *socketlabel;
```

| 参数 | 描述              | 锁定 |
| ------ | ------------------- | ------ |
| `cred`     | 凭证主体          |      |
| `socket`     | 对象; 套接字      |      |
| `socketlabel`     | socket 的策略标签 |      |

#### 6.7.4.21. `mpo_check_ifnet_relabel`

```
int mpo_check_ifnet_relabel(	cred,
 	ifnet,
 	ifnetlabel,
 	newlabel);
struct ucred *cred;
struct ifnet *ifnet;
struct label *ifnetlabel;
struct label *newlabel;
```

| 参数 | 描述                           | 锁定 |
| ------ | -------------------------------- | ------ |
| `cred`     | 主体凭证                       |      |
| `ifnet`     | 对象; 网络接口                 |      |
| `ifnetlabel`     | ifnet 的现有策略标签           |      |
| `newlabel`     | 更新策略标签，稍后应用于 ifnet |      |

确定主体凭据是否可以将传递的网络接口重新标记为传递的标签更新。

#### 6.7.4.22. `mpo_check_socket_relabel`

```
int mpo_check_socket_relabel(	cred,
 	socket,
 	socketlabel,
 	newlabel);
struct ucred *cred;
struct socket *socket;
struct label *socketlabel;
struct label *newlabel;
```

| 参数 | 叙述                               | 锁定 |
| ------ | ------------------------------------ | ------ |
| `cred`     | 主体凭证                           |      |
| `socket`     | 对象; 插座                         |      |
| `socketlabel`     | socket 的现有策略标签              |      |
| `newlabel`     | 标签更新，稍后将应用于 socketlabel |      |

确定主体凭据是否可以重新标记传递的套接字以更新传递的标签。

#### 6.7.4.23. `mpo_check_cred_relabel`

```
int mpo_check_cred_relabel(	cred,
 	newlabel);
struct ucred *cred;
struct label *newlabel;
```

| 参数 | 描述                        | 锁定 |
| ------ | ----------------------------- | ------ |
| `cred`     | 主题凭证                    |      |
| `newlabel`     | 标签更新，稍后将应用于 cred |      |

确定主体凭据是否可以重新标记自身以传递的标签更新。

#### 6.7.4.24. `mpo_check_vnode_relabel`

```
int mpo_check_vnode_relabel(	cred,
 	vp,
 	vnodelabel,
 	newlabel);
struct ucred *cred;
struct vnode *vp;
struct label *vnodelabel;
struct label *newlabel;
```

| 参数 | 描述                      | 锁定   |
| ------ | --------------------------- | -------- |
| `cred`     | 主题凭证                  | 不可变 |
| `vp`     | 对象; vnode               | 已锁定 |
| `vnodelabel`     | vp 的现有策略标签         |        |
| `newlabel`     | vp 后续应用的策略标签更新 |        |

确定主体凭证是否可以将传递的 vnode 重新标记为传递的标签更新

#### 6.7.4.25. `mpo_check_mount_stat`

```
int mpo_check_mount_stat(	cred,
 	mp,
 	mountlabel);
struct ucred *cred;
struct mount *mp;
struct label *mountlabel;
```

| 参数 | 描述               | 锁定 |
| ------ | -------------------- | ------ |
| `cred`     | 主题凭证           |      |
| `mp`     | 对象；文件系统挂载 |      |
| `mountlabel`     | mp 的策略标签      |      |

确定主体凭据是否可以看到对文件系统执行的 statfs 的结果。成功返回 0，失败返回 errno 值。建议的失败情况包括标签不匹配的 EACCES 或缺乏特权的 EPERM。此调用可以在许多情况下进行，包括在调用 statfs(2)和相关调用时，以及确定要从文件系统列表中排除哪些文件系统时，例如在调用 getfsstat(2)时。

#### 6.7.4.26. `mpo_check_proc_debug`

```
int mpo_check_proc_debug(	cred,
 	proc);
struct ucred *cred;
struct proc *proc;
```

| 参数 | 描述       | 锁定     |
| ------ | ------------ | ---------- |
| `cred`     | 主体凭证   | 不可变的 |
| `proc`     | 对象; 过程 |          |

确定主体凭证是否可以调试传递的进程。成功返回 0，失败返回一个 errno 值。建议的失败情况包括：标签不匹配返回 EACCES，缺乏权限返回 EPERM，或者为了隐藏目标的可见性返回 ESRCH。可以在许多情况下进行此调用，包括使用 ptrace(2)和 ktrace(2) API，以及某些类型的 procfs 操作。

#### 6.7.4.27. `mpo_check_vnode_access`

```
int mpo_check_vnode_access(	cred,
 	vp,
 	label,
 	flags);
struct ucred *cred;
struct vnode *vp;
struct label *label;
int flags;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `cred`     | 主题凭证       |      |
| `vp`     | 对象; 虚结点   |      |
| `label`     | vp 的策略标签  |      |
| `flags`     | access(2) 标志 |      |

确定当主体凭证对传递的 vnode 使用传递的访问标志执行 access(2)和相关调用时应如何返回。一般应该使用与 mpo_check_vnode_open 相同的语义实现。成功返回 0，失败返回 errno 值。建议的失败：EACCES 用于标签不匹配或 EPERM 用于缺乏权限。

#### 6.7.4.28. `mpo_check_vnode_chdir`

```
int mpo_check_vnode_chdir(	cred,
 	dvp,
 	dlabel);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
```

| 参数 | 描述                         | 锁定 |
| ------ | ------------------------------ | ------ |
| `cred`     | 主体凭证                     |      |
| `dvp`     | 对象；vnode 到 chdir(2) 进入 |      |
| `dlabel`     | dvp 的策略标签               |      |

确定主题凭证是否可以将进程的工作目录更改为传递的 vnode。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，或者缺乏特权的 EPERM。

#### 6.7.4.29. `mpo_check_vnode_chroot`

```
int mpo_check_vnode_chroot(	cred,
 	dvp,
 	dlabel);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
```

| 参数 | 描述                  | 锁定 |
| ------ | ----------------------- | ------ |
| `cred`     | 主体凭证              |      |
| `dvp`     | 目录虚节点            |      |
| `dlabel`     | 与 dvp 关联的策略标签 |      |

确定主体是否应被允许 chroot(2) 进入指定目录 ( dvp )。

#### 6.7.4.30. `mpo_check_vnode_create`

```
int mpo_check_vnode_create(	cred,
 	dvp,
 	dlabel,
 	cnp,
 	vap);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
struct componentname *cnp;
struct vattr *vap;
```

| 参数 | 描述              | 锁定 |
| ------ | ------------------- | ------ |
| `cred`     | 主题凭证          |      |
| `dvp`     | 对象; vnode       |      |
| `dlabel`     | dvp 的策略标签    |      |
| `cnp`     | dvp 的组件名称    |      |
| `vap`     | vap 的 vnode 属性 |      |

确定主题凭证是否可以使用传递的父目录、传递的名称信息和传递的属性信息创建 vnode。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，或者权限不足的 EPERM。在许多情况下可能会调用此函数，包括作为对使用 O_CREAT 打开(2)、mkfifo(2)和其他调用的结果.

#### 6.7.4.31. `mpo_check_vnode_delete`

```
int mpo_check_vnode_delete(	cred,
 	dvp,
 	dlabel,
 	vp,
 	label,
 	cnp);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
struct vnode *vp;
void *label;
struct componentname *cnp;
```

| 参数 | 描述                | 锁定 |
| ------ | --------------------- | ------ |
| `cred`     | 主体凭证            |      |
| `dvp`     | 父目录 vnode        |      |
| `dlabel`     | dvp 的策略标签      |      |
| `vp`     | 要删除的对象; vnode |      |
| `label`     | vp 的策略标签       |      |
| `cnp`     | vp 的组件名称       |      |

确定主体凭证是否可以从传递的父目录和传递的名称信息中删除一个 vnode。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的情况下为 EACCES，或者权限不足的情况下为 EPERM。可以在许多情况下调用此函数，包括由 unlink(2)和 rmdir(2)调用的结果。实现此入口点的策略还应实现 mpo_check_rename_to 以授权由于成为重命名目标而删除对象。

#### 6.7.4.32. `mpo_check_vnode_deleteacl`

```
int mpo_check_vnode_deleteacl(	cred,
 	vp,
 	label,
 	type);
struct ucred *cred;
struct vnode *vp;
struct label *label;
acl_type_t type;
```

| 参数 | 描述           | 锁定     |
| ------ | ---------------- | ---------- |
| `cred`     | 凭证           | 不可变的 |
| `vp`     | 对象；虚拟节点 | 锁定     |
| `label`     | vp 的策略标签  |          |
| `type`     | ACL 类型       |          |

确定主体凭证是否可以从传递的 vnode 中删除传递类型的 ACL。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，或者权限不足的 EPERM。

#### 6.7.4.33. `mpo_check_vnode_exec`

```
int mpo_check_vnode_exec(	cred,
 	vp,
 	label);
struct ucred *cred;
struct vnode *vp;
struct label *label;
```

| 参数 | 描述                 | 锁定 |
| ------ | ---------------------- | ------ |
| `cred`     | 主题凭证             |      |
| `vp`     | 对象；要执行的 vnode |      |
| `label`     | vp 的策略标签        |      |

确定主体凭证是否可以执行传递的 vnode。执行特权的确定与关于任何过渡事件的决定分开进行。成功返回 0，失败返回 errno 值。建议的失败：标签不匹配为 EACCES，或者权限不足为 EPERM。

#### 6.7.4.34. `mpo_check_vnode_getacl`

```
int mpo_check_vnode_getacl(	cred,
 	vp,
 	label,
 	type);
struct ucred *cred;
struct vnode *vp;
struct label *label;
acl_type_t type;
```

| 参数 | 描述          | 锁定 |
| ------ | --------------- | ------ |
| `cred`     | 主题凭证      |      |
| `vp`     | 对象; vnode   |      |
| `label`     | vp 的策略标签 |      |
| `type`     | ACL 类型      |      |

确定主体凭据是否可以从传递的 vnode 中检索传递类型的 ACL。成功返回 0，或失败返回 errno 值。建议的故障：标签不匹配的 EACCES，或缺乏特权的 EPERM。

#### 6.7.4.35. `mpo_check_vnode_getextattr`

```
int mpo_check_vnode_getextattr(	cred,
 	vp,
 	label,
 	attrnamespace,
 	name,
 	uio);
struct ucred *cred;
struct vnode *vp;
struct label *label;
int attrnamespace;
const char *name;
struct uio *uio;
```

| 参数 | 描述                      | 锁定 |
| ------ | --------------------------- | ------ |
| `cred`     | 主体凭证                  |      |
| `vp`     | 对象; vnode               |      |
| `label`     | vp 的策略标签             |      |
| `attrnamespace`     | 扩展属性命名空间          |      |
| `name`     | 扩展属性名称              |      |
| `uio`     | I/O 结构指针；参见 uio(9) |      |

确定主题凭证是否可以从传递的 vnode 中检索具有传递命名空间和名称的扩展属性。使用扩展属性实现标记的策略可能对这些扩展属性的操作有特殊处理。返回 0 表示成功，或返回 errno 值表示失败。建议的失败：标签不匹配的 EACCES，或缺乏权限的 EPERM。

#### 6.7.4.36. `mpo_check_vnode_link`

```
int mpo_check_vnode_link(	cred,
 	dvp,
 	dlabel,
 	vp,
 	label,
 	cnp);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
struct vnode *vp;
struct label *label;
struct componentname *cnp;
```

| 参数 | 描述                     | 锁定 |
| ------ | -------------------------- | ------ |
| `cred`     | 主体凭证                 |      |
| `dvp`     | 目录 vnode               |      |
| `dlabel`     | 与 dvp 关联的策略标签    |      |
| `vp`     | 链接目标 vnode           |      |
| `label`     | 与 vp 相关的策略标签     |      |
| `cnp`     | 正在创建的链接的组件名称 |      |

确定是否允许主体使用由 cnp 指定名称创建到 v 节点 vp 的链接

#### 6.7.4.37. `mpo_check_vnode_mmap`

```
int mpo_check_vnode_mmap(	cred,
 	vp,
 	label,
 	prot);
struct ucred *cred;
struct vnode *vp;
struct label *label;
int prot;
```

| 参数 | 描述                           | 锁定 |
| ------ | -------------------------------- | ------ |
| `cred`     | 主体凭证                       |      |
| `vp`     | Vnode 映射                     |      |
| `label`     | 与 vp 相关联的策略标签         |      |
| `prot`     | 内存映射保护（请参阅 mmap(2)） |      |

确定主体是否应该被允许使用指定的保护来映射 vnode vp 。

#### 6.7.4.38. `mpo_check_vnode_mmap_downgrade`

```
void mpo_check_vnode_mmap_downgrade(	cred,
 	vp,
 	label,
 	prot);
struct ucred *cred;
struct vnode *vp;
struct label *label;
int *prot;
```

| 参数 | 描述                                     | 锁定 |
| ------ | ------------------------------------------ | ------ |
| `cred`     | 查看 mpo\_check\_vnode\_mmap 。 |      |
| `vp`     |                                          |      |
| `label`     |                                          |      |
| `prot`     | 降低内存映射保护                         |      |

根据主体和客体标签降低内存映射保护

#### 6.7.4.39. `mpo_check_vnode_mprotect`

```
int mpo_check_vnode_mprotect(	cred,
 	vp,
 	label,
 	prot);
struct ucred *cred;
struct vnode *vp;
struct label *label;
int prot;
```

| 参数 | 描述         | 锁定 |
| ------ | -------------- | ------ |
| `cred`     | 主题凭证     |      |
| `vp`     | 映射的 vnode |      |
| `prot`     | 内存保护     |      |

确定主体是否应被允许在从 vnode 映射的内存上设置指定的内存保护 vp 。

#### 6.7.4.40. `mpo_check_vnode_poll`

```
int mpo_check_vnode_poll(	active_cred,
 	file_cred,
 	vp,
 	label);
struct ucred *active_cred;
struct ucred *file_cred;
struct vnode *vp;
struct label *label;
```

| 参数 | 描述                 | 锁定 |
| ------ | ---------------------- | ------ |
| `active_cred`     | 主题凭证             |      |
| `file_cred`     | 与结构文件相关的凭证 |      |
| `vp`     | 轮询的 vnode         |      |
| `label`     | 与 vp 关联的策略标签 |      |

确定是否应允许主体轮询 vnode vp 。

#### 6.7.4.41. `mpo_check_vnode_rename_from`

```
int mpo_vnode_rename_from(	cred,
 	dvp,
 	dlabel,
 	vp,
 	label,
 	cnp);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
struct vnode *vp;
struct label *label;
struct componentname *cnp;
```

| 参数 | 描述                  | 锁定 |
| ------ | ----------------------- | ------ |
| `cred`     | 主题凭证              |      |
| `dvp`     | 目录 vnode            |      |
| `dlabel`     | 与 dvp 关联的策略标签 |      |
| `vp`     | 将要重命名的 vnode    |      |
| `label`     | 与 vp 相关的策略标签  |      |
| `cnp`     | vp 的组件名称         |      |

确定主体是否应被允许将 vnode vp 重命名为其他内容。

#### 6.7.4.42. `mpo_check_vnode_rename_to`

```
int mpo_check_vnode_rename_to(	cred,
 	dvp,
 	dlabel,
 	vp,
 	label,
 	samedir,
 	cnp);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
struct vnode *vp;
struct label *label;
int samedir;
struct componentname *cnp;
```

| 参数 | 描述                                     | 锁定 |
| ------ | ------------------------------------------ | ------ |
| `cred`     | 主体凭证                                 |      |
| `dvp`     | 目录 vnode                               |      |
| `dlabel`     | 与 dvp 关联的策略标签                    |      |
| `vp`     | 覆盖的 vnode                             |      |
| `label`     | 与 vp 相关的策略标签                     |      |
| `samedir`     | 布尔值；如果源目录和目标目录相同，则为 1 |      |
| `cnp`     | 目的地组件名称                           |      |

确定是否应允许主题重命名到 vnode vp ，到目录 dvp ，或到 cnp 表示的名称。如果没有现有文件可覆盖， vp 和 label 将为 NULL。

#### 6.7.4.43. `mpo_check_socket_listen`

```
int mpo_check_socket_listen(	cred,
 	socket,
 	socketlabel);
struct ucred *cred;
struct socket *socket;
struct label *socketlabel;
```

| 参数 | 描述              | 锁定 |
| ------ | ------------------- | ------ |
| `cred`     | 主体凭证          |      |
| `socket`     | 对象; 套接字      |      |
| `socketlabel`     | socket 的策略标签 |      |

确定主体凭证是否可以监听传递的套接字。成功返回 0，或失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，或者缺乏特权的 EPERM。

#### 6.7.4.44. `mpo_check_vnode_lookup`

```
int mpo_check_vnode_lookup(	,
 	,
 	,
 	cnp);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
struct componentname *cnp;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `cred`     | 主题凭证       |      |
| `dvp`     | 对象; vnode    |      |
| `dlabel`     | dvp 的策略标签 |      |
| `cnp`     | 查找的组件名称 |      |

确定主体凭据是否可以在传递的目录 vnode 中查找传递的名称。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的情况下返回 EACCES，或者缺乏权限的情况下返回 EPERM。

#### 6.7.4.45. `mpo_check_vnode_open`

```
int mpo_check_vnode_open(	cred,
 	vp,
 	label,
 	acc_mode);
struct ucred *cred;
struct vnode *vp;
struct label *label;
int acc_mode;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `cred`     | 主体凭证         |      |
| `vp`     | 对象; vnode      |      |
| `label`     | vp 的策略标签    |      |
| `acc_mode`     | open(2) 访问模式 |      |

确定主体凭证是否可以对传递的 vnode 执行开放操作，并使用传递的访问模式。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的情况下返回 EACCES，或者缺乏特权的情况下返回 EPERM。

#### 6.7.4.46. `mpo_check_vnode_readdir`

```
int mpo_check_vnode_readdir(	,
 	,
 	);
struct ucred *cred;
struct vnode *dvp;
struct label *dlabel;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `cred`     | 主题凭证         |      |
| `dvp`     | 对象; 目录 vnode |      |
| `dlabel`     | dvp 的策略标签   |      |

确定主体凭证是否可以在传递的目录 vnode 上执行 readdir 操作。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配为 EACCES，权限不足为 EPERM。

#### 6.7.4.47. `mpo_check_vnode_readlink`

```
int mpo_check_vnode_readlink(	cred,
 	vp,
 	label);
struct ucred *cred;
struct vnode *vp;
struct label *label;
```

| 参数 | 描述          | 锁定 |
| ------ | --------------- | ------ |
| `cred`     | 主题凭证      |      |
| `vp`     | 对象; vnode   |      |
| `label`     | vp 的策略标签 |      |

确定主体凭证是否可以在传递的符号链接 vnode 上执行 readlink 操作。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，或者权限不足的 EPERM。此调用可以在许多情况下进行，包括用户进程的显式 readlink 调用，或者作为进程进行名称查找期间的隐式 readlink 的结果。

#### 6.7.4.48. `mpo_check_vnode_revoke`

```
int mpo_check_vnode_revoke(	cred,
 	vp,
 	label);
struct ucred *cred;
struct vnode *vp;
struct label *label;
```

| 参数 | 描述           | 锁定 |
| ------ | ---------------- | ------ |
| `cred`     | 证书主题       |      |
| `vp`     | 对象; 虚拟节点 |      |
| `label`     | vp 的策略标签  |      |

确定主体凭证是否可以撤销对传递的 vnode 的访问权限。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，或者缺乏特权的 EPERM。

#### 6.7.4.49. `mpo_check_vnode_setacl`

```
int mpo_check_vnode_setacl(	cred,
 	vp,
 	label,
 	type,
 	acl);
struct ucred *cred;
struct vnode *vp;
struct label *label;
acl_type_t type;
struct acl *acl;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `cred`     | 主题凭证         |      |
| `vp`     | 对象；vnode      |      |
| `label`     | " vp "的策略标签 |      |
| `type`     | ACL 类型         |      |
| `acl`     | ACL              |      |

确定主体凭证是否可以在传递的 vnode 上设置传递的 ACL 的类型。成功返回 0，失败返回 errno 值。建议失败情况: 标签不匹配的 EACCES，或缺乏权限的 EPERM。

#### 6.7.4.50. `mpo_check_vnode_setextattr`

```
int mpo_check_vnode_setextattr(	cred,
 	vp,
 	label,
 	attrnamespace,
 	name,
 	uio);
struct ucred *cred;
struct vnode *vp;
struct label *label;
int attrnamespace;
const char *name;
struct uio *uio;
```

| 参数 | 描述                      | 锁定 |
| ------ | --------------------------- | ------ |
| `cred`     | 主体凭证                  |      |
| `vp`     | 对象; 虚节点              |      |
| `label`     | vp 的策略标签             |      |
| `attrnamespace`     | 扩展属性命名空间          |      |
| `name`     | 扩展属性名称              |      |
| `uio`     | I/O 结构指针；参见 uio(9) |      |

确定主题凭证是否可以在传递的 vnode 上设置传递名称和传递命名空间的扩展属性。 实现安全标签的策略将希望为这些属性提供额外的保护。 此外，策略不应该基于从 uio 引用的数据做出决定，因为在此检查和实际操作之间存在潜在的竞争条件。 如果执行删除操作，则 uio 也可能是 NULL 。 为成功返回 0，失败返回 errno 值。 建议失败：标签不匹配的 EACCES，或者缺乏权限的 EPERM。

#### 6.7.4.51. `mpo_check_vnode_setflags`

```
int mpo_check_vnode_setflags(	cred,
 	vp,
 	label,
 	flags);
struct ucred *cred;
struct vnode *vp;
struct label *label;
u_long flags;
```

| 参数 | 描述                      | 锁定 |
| ------ | --------------------------- | ------ |
| `cred`     | 主题凭据                  |      |
| `vp`     | 对象；vnode               |      |
| `label`     | vp 的策略标签             |      |
| `flags`     | 文件标志; 参见 chflags(2) |      |

确定主体凭证是否可以在传递的 vnode 上设置传递的标志。成功返回 0，失败返回 errno 值。建议的失败原因: 标签不匹配使用 EACCES，或者权限不足使用 EPERM。

#### 6.7.4.52. `mpo_check_vnode_setmode`

```
int mpo_check_vnode_setmode(	cred,
 	vp,
 	label,
 	mode);
struct ucred *cred;
struct vnode *vp;
struct label *label;
mode_t mode;
```

| 参数 | 描述                      | 锁定 |
| ------ | --------------------------- | ------ |
| `cred`     | 主题凭证                  |      |
| `vp`     | 对象; vnode               |      |
| `label`     | vp 的策略标签             |      |
| `mode`     | 文件模式；请参阅 chmod(2) |      |

确定主体凭据是否可以在传递的 vnode 上设置传递的模式。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配为 EACCES，或者权限不足为 EPERM。

#### 6.7.4.53. `mpo_check_vnode_setowner`

```
int mpo_check_vnode_setowner(	cred,
 	vp,
 	label,
 	uid,
 	gid);
struct ucred *cred;
struct vnode *vp;
struct label *label;
uid_t uid;
gid_t gid;
```

| 参数 | 描述          | 锁定 |
| ------ | --------------- | ------ |
| `cred`     | 主题凭证      |      |
| `vp`     | 对象; vnode   |      |
| `label`     | vp 的策略标签 |      |
| `uid`     | 用户 ID       |      |
| `gid`     | 组 ID         |      |

确定主体凭据是否可以将传递的 UID 和 GID 设置为文件 UID 和文件 GID 在传递的 vnode 上。这些 ID 可设置为（ -1 ），以请求不进行更新。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配为 EACCES，权限不足为 EPERM。

#### 6.7.4.54. `mpo_check_vnode_setutimes`

```
int mpo_check_vnode_setutimes(	,
 	,
 	,
 	,
 	);
struct ucred *cred;
struct vnode *vp;
struct label *label;
struct timespec atime;
struct timespec mtime;
```

| 参数 | 描述                       | 锁定 |
| ------ | ---------------------------- | ------ |
| `cred`     | 主体凭证                   |      |
| `vp`     | 对象; vp                   |      |
| `label`     | vp 的策略标签              |      |
| `atime`     | 访问时间; 请参阅 utimes(2) |      |
| `mtime`     | 修改时间；请参阅 utimes(2) |      |

确定主体凭证是否可以在传递的 vnode 上设置传递的访问时间戳。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，或者权限不足的 EPERM。

#### 6.7.4.55. `mpo_check_proc_sched`

```
int mpo_check_proc_sched(	ucred,
 	proc);
struct ucred *ucred;
struct proc *proc;
```

| 参数 | 描述       | 锁定 |
| ------ | ------------ | ------ |
| `cred`     | 主体凭证   |      |
| `proc`     | 对象; 进程 |      |

确定主体凭据是否可以更改传递进程的调度参数。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，缺乏特权的 EPERM，或限制可见性的 ESRCH。

有关更多信息，请参阅 setpriority(2)。

#### 6.7.4.56. `mpo_check_proc_signal`

```
int mpo_check_proc_signal(	cred,
 	proc,
 	signal);
struct ucred *cred;
struct proc *proc;
int signal;
```

| 参数 | 描述               | 锁定 |
| ------ | -------------------- | ------ |
| `cred`     | 主体凭证           |      |
| `proc`     | 对象; 进程         |      |
| `signal`     | 信号; 参见 kill(2) |      |

确定主体凭据是否可以将传递的信号传递给传递的进程。成功返回 0，或者失败返回 errno 值。建议失败：标签不匹配使用 EACCES，缺乏特权使用 EPERM，或限制可见性使用 ESRCH。

#### 6.7.4.57. `mpo_check_vnode_stat`

```
int mpo_check_vnode_stat(	cred,
 	vp,
 	label);
struct ucred *cred;
struct vnode *vp;
struct label *label;
```

| 参数 | 描述          | 锁定 |
| ------ | --------------- | ------ |
| `cred`     | 主体凭证      |      |
| `vp`     | 对象; vnode   |      |
| `label`     | vp 的政策标签 |      |

确定主体凭证是否可以 stat 传递的 vnode。成功返回 0，或失败返回 errno 值。建议的失败：标签不匹配的 EACCES，或缺乏特权的 EPERM。

查看 stat(2) 以获取更多信息。

#### 6.7.4.58. `mpo_check_ifnet_transmit`

```
int mpo_check_ifnet_transmit(	cred,
 	ifnet,
 	ifnetlabel,
 	mbuf,
 	mbuflabel);
struct ucred *cred;
struct ifnet *ifnet;
struct label *ifnetlabel;
struct mbuf *mbuf;
struct label *mbuflabel;
```

| 参数 | 描述                | 锁定 |
| ------ | --------------------- | ------ |
| `cred`     | 主题凭证            |      |
| `ifnet`     | 网络接口            |      |
| `ifnetlabel`     | 对 ifnet 的策略标签 |      |
| `mbuf`     | 要发送的对象;mbuf   |      |
| `mbuflabel`     | mbuf 的策略标签     |      |

确定网络接口是否可以传输传递的 mbuf。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的话返回 EACCES，缺乏权限的话返回 EPERM。

#### 6.7.4.59. `mpo_check_socket_deliver`

```
int mpo_check_socket_deliver(	cred,
 	ifnet,
 	ifnetlabel,
 	mbuf,
 	mbuflabel);
struct ucred *cred;
struct ifnet *ifnet;
struct label *ifnetlabel;
struct mbuf *mbuf;
struct label *mbuflabel;
```

| 参数 | 描述                | 锁定 |
| ------ | --------------------- | ------ |
| `cred`     | 主体凭证            |      |
| `ifnet`     | 网络接口            |      |
| `ifnetlabel`     | ifnet 的策略标签    |      |
| `mbuf`     | 对象; 待传递的 mbuf |      |
| `mbuflabel`     | mbuf 的策略标签     |      |

确定套接字是否可以接收传递的 mbuf 头中存储的数据报。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的 EACCES，或者缺乏特权的 EPERM。

#### 6.7.4.60. `mpo_check_socket_visible`

```
int mpo_check_socket_visible(	cred,
 	so,
 	socketlabel);
struct ucred *cred;
struct socket *so;
struct label *socketlabel;
```

| 参数 | 描述          | 锁定       |
| ------ | --------------- | ------------ |
| `cred`     | 主体凭证      | 不可改变的 |
| `so`     | 对象; 套接字  |            |
| `socketlabel`     | so 的策略标签 |            |

使用系统监控功能（例如 netstat(8)和 sockstat(1)使用的功能）确定主体凭证 cred 是否可以“看到”传递的套接字（ socket ）。成功返回 0，失败返回 errno 值。建议的失败情况：标签不匹配的情况下为 EACCES，缺乏权限的情况下为 EPERM，隐藏可见性的情况下为 ESRCH。

#### 6.7.4.61. `mpo_check_system_acct`

```
int mpo_check_system_acct(	ucred,
 	vp,
 	vlabel);
struct ucred *ucred;
struct vnode *vp;
struct label *vlabel;
```

| 参数 | 描述               | 锁定 |
| ------ | -------------------- | ------ |
| `ucred`     | 主体凭证           |      |
| `vp`     | 会计文件; acct(5)  |      |
| `vlabel`     | 与 vp 相关联的标签 |      |

根据主体的标签和会计日志文件的标签，确定是否应允许主体启用会计。

#### 6.7.4.62. `mpo_check_system_nfsd`

```
int mpo_check_system_nfsd(	cred);
struct ucred *cred;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `cred`     | 主题凭证 |      |

确定是否允许主体调用 nfssvc(2)。

#### 6.7.4.63. `mpo_check_system_reboot`

```
int mpo_check_system_reboot(	cred,
 	howto);
struct ucred *cred;
int howto;
```

| 参数 | 描述                        | 锁定 |
| ------ | ----------------------------- | ------ |
| `cred`     | 主体凭证                    |      |
| `howto`     | 从 reboot(2)中的 howto 参数 |      |

确定是否应允许主体以指定的方式重新启动系统。

#### 6.7.4.64. `mpo_check_system_settime`

```
int mpo_check_system_settime(	cred);
struct ucred *cred;
```

| 参数 | 描述     | 锁定 |
| ------ | ---------- | ------ |
| `cred`     | 主题凭证 |      |

确定用户是否应被允许设置系统时钟。

#### 6.7.4.65. `mpo_check_system_swapon`

```
int mpo_check_system_swapon(	cred,
 	vp,
 	vlabel);
struct ucred *cred;
struct vnode *vp;
struct label *vlabel;
```

| 参数 | 描述             | 锁定 |
| ------ | ------------------ | ------ |
| `cred`     | 主题凭证         |      |
| `vp`     | 交换设备         |      |
| `vlabel`     | 与 vp 关联的标签 |      |

确定是否应允许主体将 vp 添加为交换设备。

#### 6.7.4.66. `mpo_check_system_sysctl`

```
int mpo_check_system_sysctl(	cred,
 	name,
 	namelen,
 	old,
 	oldlenp,
 	inkernel,
 	new,
 	newlen);
struct ucred *cred;
int *name;
u_int *namelen;
void *old;
size_t *oldlenp;
int inkernel;
void *new;
size_t newlen;
```

| 参数 | 描述                         | 锁定 |
| ------ | ------------------------------ | ------ |
| `cred`     | 主体凭证                     |      |
| `name`     | 看 sysctl(3)                 |      |
| `namelen`     |                              |      |
| `old`     |                              |      |
| `oldlenp`     |                              |      |
| `inkernel`     | 布尔；如果从内核调用，则为 1 |      |
| `new`     | 看 sysctl(3)                 |      |
| `newlen`     |                              |      |

确定是否应允许主体进行指定的 sysctl(3)事务。

### 6.7.5. 标签管理调用

当用户进程请求修改对象的标签时，会发生重新标记事件。更新分两个阶段进行：首先，将执行访问控制检查以确定更新是否合法和允许，然后通过一个单独的入口点执行更新。重新标记入口点通常接受对象、对象标签引用和由进程提交的更新标签。在重新标记期间不鼓励内存分配，因为重新标记调用不允许失败（失败应在重新标记检查中提前报告）。

## 6.8. 用户空间架构

TrustedBSD MAC 框架包括许多与策略无关的元素，包括用于抽象管理标签的 MAC 库接口，对系统凭据管理和登录库的修改以支持给用户分配 MAC 标签，以及一组工具来监视和修改进程、文件和网络接口上的标签。有关用户架构的更多详细信息将在不久的将来添加到本节。

### 6.8.1. 用于策略无关标签管理的 API

TrustedBSD MAC Framework 提供了许多库和系统调用，允许应用程序使用与策略无关的接口管理对象上的 MAC 标签。这使得应用程序可以在不支持特定策略的情况下操作各种策略的标签。这些接口被通用工具（如 ifconfig(8)、ls(1)和 ps(1)）使用，用于查看网络接口、文件和进程上的标签。这些 API 还支持包括 getfmac(8)、getpmac(8)、setfmac(8)、setfsmac(8)和 setpmac(8)在内的 MAC 管理工具。MAC API 的文档在 mac(3)中。

应用程序以两种形式处理 MAC 标签：一种是内部形式，用于返回和设置进程和对象上的标签（ mac_t ），另一种是基于 C 字符串的外部形式，适合存储在配置文件中、显示给用户或用户输入。每个 MAC 标签包含多个元素，每个元素由名称和值对组成。内核中的策略模块绑定到特定名称，并以特定于策略的方式解释值。在外部化字符串形式中，标签由逗号分隔的名称和值对列表表示，用 / 字符分隔。标签可以直接使用提供的 API 转换为文本，从内核检索标签时，必须首先准备所需的标签元素集的内部化标签存储。通常有两种方法来完成这个过程：使用 mac_prepare(3)和任意的所需标签元素列表，或者使用从 mac.conf(5)配置文件加载默认元素集的调用变体之一。对象默认值允许应用程序编写者有用地显示与对象关联的标签，而不必了解系统中存在的策略。

|  | 目前，MAC 库不支持除将标签元素直接操作转换为文本字符串、字符串编辑和转换回内部化标签之外的其他操作。如果这些接口对应用程序编写者证明必要，将来可能会添加这样的接口。 |
| -- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

### 6.8.2. 将标签绑定到用户

标准用户上下文管理接口 setusercontext(3)已经修改，以从 login.conf(5)中检索与用户类别关联的 MAC 标签。当指定 LOGIN_SETALL 或明确指定 LOGIN_SETMAC 时，这些标签将与其他用户上下文一起设置。

|  | 预计在将来的 FreeBSD 版本中，MAC 标签数据库将与 login.conf 用户类抽象分离，并将在一个单独的数据库中进行维护。不过，在这样的更改之后，setusercontext(3) API 应保持不变。 |
| -- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

## 结论

TrustedBSD MAC 框架允许内核模块以高度集成的方式增强系统安全策略。它们可以基于现有对象属性或基于在 MAC 框架的帮助下维护的标签数据来执行此操作。该框架具有足够的灵活性，可以实现各种策略类型，包括诸如 MLS 和 Biba 之类的信息流安全策略，以及基于现有 BSD 凭据或文件保护的策略。策略作者在实现新的安全服务时可能希望参考本文档以及现有安全模块。
