# 第 2 章 内核中的锁

*本章由 FreeBSD SMP Next Generation 项目维护。*

本文概述了 FreeBSD 内核中使用的锁，以允许内核内的有效多处理。可以通过几种方式实现锁定。数据结构可以通过 mutexes 或 lockmgr(9)锁来保护。一些变量只需始终使用原子操作来访问即可。

## 2.1. 互斥体

互斥锁只是用来保证互斥的锁。具体来说，互斥锁一次只能被一个实体拥有。如果另一个实体希望获取已经拥有的互斥锁，它必须等待直到互斥锁被释放。在 FreeBSD 内核中，互斥锁由进程拥有。

互斥锁可以递归获取，但它们旨在短时间内持有。具体来说，在持有互斥锁时不能休眠。如果需要在休眠期间持有锁，请使用 lockmgr(9)锁。

每个互斥锁都有几个值得关注的属性：

内核源码中 struct mtx 变量的名称。

mtx_init 分配给它的互斥体的逻辑名称。此名称显示在 KTR 跟踪消息和见证错误和警告中，并用于在见证代码中区分互斥体。

互斥体的类型，根据 MTX_* 标志。每个标志的含义与其在 mutex(9)中的文档中记录的含义相关。

  MTX_DEF 一个睡眠互斥体

  MTX_SPIN 一个自旋互斥体

MTX_RECURSE 这个互斥体允许递归。

保护对象一个数据结构或数据结构成员列表，该条目保护这些数据结构或数据结构成员。对于数据结构成员，名称将采用 structure name . member name 的形式。

依赖函数只能在持有此互斥锁时调用的函数。

表 1。互斥锁列表

| 变量名称       | 逻辑名称   | 类型 | 保护对象 | 依赖函数                                                                                                                      |
| ---------------- | ------------ | ------ | ---------- | ------------------------------------------------------------------------------------------------------------------------------- |
| sched\_lock | 调度锁     | `MTX_SPIN`     | `MTX_RECURSE`         | `_gmonparam`, `cnt.v_swtch`, `cp_time`, `curpriority`, `mtx`.`mtx_blocked`, `mtx`.`mtx_contested`, `proc`.`p_procq`, `proc`.`p_slpq`, `proc`.`p_sflag`, `proc`.`p_stat`, `proc`.`p_estcpu`, `proc`.`p_cpticks` `proc`.`p_pctcpu`, `proc`.`p_wchan`, `proc`.`p_wmesg`, `proc`.`p_swtime`, `proc`.`p_slptime`, `proc`.`p_runtime`, `proc`.`p_uu`, `proc`.`p_su`, `proc`.`p_iu`, `proc`.`p_uticks`, `proc`.`p_sticks`, `proc`.`p_iticks`, `proc`.`p_oncpu`, `proc`.`p_lastcpu`, `proc`.`p_rqindex`, `proc`.`p_heldmtx`, `proc`.`p_blocked`, `proc`.`p_mtxname`, `proc`.`p_contested`, `proc`.`p_priority`, `proc`.`p_usrpri`, `proc`.`p_nativepri`, `proc`.`p_nice`, `proc`.`p_rtprio`, `pscnt`, `slpque`, `itqueuebits`, `itqueues`, `rtqueuebits`, `rtqueues`, `queuebits`, `queues`, `idqueuebits`, `idqueues`, `switchtime`, `switchticks` |
| vm86pcb 锁     | vm86pcb 锁 | `MTX_DEF`     | `vm86pcb`         | `vm86_bioscall`                                                                                                                              |
| 巨人           | 巨人       | `MTX_DEF`     | `MTX_RECURSE`         | 几乎所有东西                                                                                                                  |
| 锁定调用       | 锁定调用   | `MTX_SPIN`     | `MTX_RECURSE`         | `callfree`, `callwheel`, `nextsoftcheck`, `proc`.`p_itcallout`, `proc`.`p_slpcallout`, `softticks`, `ticks`                                                                                                                |

## 2.2. 共享独占锁

这些锁提供基本的读者-写者类型功能，并且可能被正在睡眠的进程持有。目前它们由 lockmgr(9)支持。

共享独占锁列表表 2。

| 变量名称 | 保护对象 |
| ---------- | ---------- |
| `allproc_lock`         | `allproc` `zombproc` `pidhashtbl` `proc`.`p_list` `proc`.`p_hash` `nextpid`  |
| `proctree_lock`         | `proc`.`p_children` `proc`.`p_sibling`      |

## 2.3. 原子保护变量

原子保护变量是一种特殊变量，不受显式锁保护。 相反，对变量的所有数据访问都使用特殊的原子操作，如 atomic(9)中所述。 很少有变量是这样处理的，尽管其他同步原语，如互斥锁，是用原子保护变量实现的。

* `mtx`.`mtx_lock`
