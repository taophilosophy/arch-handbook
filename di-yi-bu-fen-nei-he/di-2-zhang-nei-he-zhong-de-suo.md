# 第 2 章 内核中的锁

*本章由 FreeBSD 下一代 SMP 项目维护。*

本文档概述了 FreeBSD 内核中用于实现有效多处理的锁机制。锁定可以通过多种方式实现。数据结构可以通过互斥锁（mutex）或 [lockmgr(9)](https://man.freebsd.org/cgi/man.cgi?query=lockmgr&sektion=9&format=html) 锁来保护。有些变量则通过始终使用原子操作来访问，从而保护它们。

## 2.1. 互斥锁（Mutexes）

互斥锁（mutex）仅仅是用来保证互斥性的锁。具体来说，互斥锁一次只能由一个实体拥有。如果其他实体希望获取已经被拥有的互斥锁，它必须等待直到该互斥锁被释放。在 FreeBSD 内核中，互斥锁由进程拥有。

互斥锁可以递归获取，但它们的目的是短时间持有。具体来说，持有互斥锁时不能休眠。如果需要在休眠期间持有锁，请使用 [lockmgr(9)](https://man.freebsd.org/cgi/man.cgi?query=lockmgr&sektion=9&format=html) 锁。

每个互斥锁有几个重要属性：

* **变量名**：内核源代码中 `struct mtx` 变量的名称。
* **逻辑名称**：通过 `mtx_init` 分配给它的互斥锁名称。此名称会显示在 KTR 跟踪消息和 witness 错误及警告中，用于区分 witness 代码中的互斥锁。
* **类型**：互斥锁的类型，使用 `MTX_*` 标志表示。每个标志的含义与 [mutex(9)](https://man.freebsd.org/cgi/man.cgi?query=mutex&sektion=9&format=html) 中的文档相关。

  * `MTX_DEF`：一个睡眠互斥锁
  * `MTX_SPIN`：一个自旋互斥锁
  * `MTX_RECURSE`：此互斥锁允许递归
* **保护对象**：此条目所保护的数据结构或数据结构成员的列表。对于数据结构成员，名称格式为 `结构体名.成员名`。
* **依赖函数**：只有在持有此互斥锁时，才能调用的函数。

### 表 1. 互斥锁列表

| 变量名           | 逻辑名称           | 类型         | 保护对象          | 依赖函数                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |                                                                                                                                                                                                                                                               |
| ------------- | -------------- | ---------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| sched\_lock   | "sched lock"   | `MTX_SPIN` | `MTX_RECURSE` | `_gmonparam`, `cnt.v_swtch`, `cp_time`, `curpriority`, `mtx`.`mtx_blocked`, `mtx`.`mtx_contested`, `proc`.`p_procq`, `proc`.`p_slpq`, `proc`.`p_sflag`, `proc`.`p_stat`, `proc`.`p_estcpu`, `proc`.`p_cpticks`, `proc`.`p_pctcpu`, `proc`.`p_wchan`, `proc`.`p_wmesg`, `proc`.`p_swtime`, `proc`.`p_slptime`, `proc`.`p_runtime`, `proc`.`p_uu`, `proc`.`p_su`, `proc`.`p_iu`, `proc`.`p_uticks`, `proc`.`p_sticks`, `proc`.`p_iticks`, `proc`.`p_oncpu`, `proc`.`p_lastcpu`, `proc`.`p_rqindex`, `proc`.`p_heldmtx`, `proc`.`p_blocked`, `proc`.`p_mtxname`, `proc`.`p_contested`, `proc`.`p_priority`, `proc`.`p_usrpri`, `proc`.`p_nativepri`, `proc`.`p_nice`, `proc`.`p_rtprio`, `pscnt`, `slpque`, `itqueuebits`, `itqueues`, `rtqueuebits`, `rtqueues`, `queuebits`, `queues`, `idqueuebits`, `idqueues`, `switchtime`, `switchticks` | `setrunqueue`, `remrunqueue`, `mi_switch`, `chooseproc`, `schedclock`, `resetpriority`, `updatepri`, `maybe_resched`, `cpu_switch`, `cpu_throw`, `need_resched`, `resched_wanted`, `clear_resched`, `aston`, `astoff`, `astpending`, `calcru`, `proc_compare` |
| vm86pcb\_lock | "vm86pcb lock" | `MTX_DEF`  | `vm86pcb`     | `vm86_bioscall`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                                                                                                                                               |
| Giant         | "Giant"        | `MTX_DEF`  | `MTX_RECURSE` | 几乎所有内容                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | 很多                                                                                                                                                                                                                                                            |
| callout\_lock | "callout lock" | `MTX_SPIN` | `MTX_RECURSE` | `callfree`, `callwheel`, `nextsoftcheck`, `proc`.`p_itcallout`, `proc`.`p_slpcallout`, `softticks`, `ticks`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |                                                                                                                                                                                                                                                               |

## 2.2. 共享独占锁（Shared Exclusive Locks）

这些锁提供基本的读写锁类型功能，可以由睡眠中的进程持有。目前，它们是由 [lockmgr(9)](https://man.freebsd.org/cgi/man.cgi?query=lockmgr&sektion=9&format=html) 支持的。

### 表 2. 共享独占锁列表

| 变量名             | 保护对象                                                                             |
| --------------- | -------------------------------------------------------------------------------- |
| `allproc_lock`  | `allproc`, `zombproc`, `pidhashtbl`, `proc`.`p_list`, `proc`.`p_hash`, `nextpid` |
| `proctree_lock` | `proc`.`p_children`, `proc`.`p_sibling`                                          |

## 2.3. 原子保护变量（Atomically Protected Variables）

原子保护变量是一种特殊的变量，未通过显式锁保护。相反，所有对这些变量的访问都使用特殊的原子操作，如 [atomic(9)](https://man.freebsd.org/cgi/man.cgi?query=atomic&sektion=9&format=html) 中所描述的那样。很少有变量是以这种方式处理的，尽管其他同步原语（如互斥锁）是通过原子保护变量实现的。

* `mtx`.`mtx_lock`
