# 第 4 章 jail 子系统

在大多数 UNIX® 系统中，`root` 拥有无上的权限。这容易导致不安全。如果攻击者获得了 `root` 权限，他将能够轻易地访问所有功能。在 FreeBSD 中，有一些 sysctl 可以稀释 `root` 的权限，从而最小化攻击者造成的损害。具体来说，其中一个功能叫做“secure levels”（安全级别）。另一个自 FreeBSD 4.0 起提供的功能是一个名为 [jail(8)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=8&format=html) 的实用程序。Jail 将环境 chroot 并设置对在 jail 中派生的进程的某些限制。例如，jail 中的进程无法影响 jail 外部的进程，无法使用某些系统调用，也无法对主机环境造成任何损害。

Jail 正在成为新的安全模型。人们将一些可能存在漏洞的服务器（如 Apache、BIND 和 sendmail）运行在 jails 中，以便如果攻击者在 jail 中获得了 `root` 权限，这只是一个麻烦，而不是一场灾难。本文主要关注 jail 的内部实现（源代码）。有关如何设置 jail 的信息，请参见 [手册中的 jail 条目](https://docs.freebsd.org/en/books/handbook/#jails)。

## 4.1. 架构

Jail 由两个部分组成：用户空间程序 [jail(8)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=8&format=html) 和内核中实现的代码：系统调用 [jail(2)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2&format=html) 及其相关限制。我将讨论用户空间程序，然后讨论 jail 在内核中的实现。

### 4.1.1. 用户空间代码

用户空间 jail 的源代码位于 **/usr/src/usr.sbin/jail** 目录，包含一个文件 **jail.c**。该程序接受以下参数：jail 的路径、主机名、IP 地址以及要执行的命令。

#### 4.1.1.1. 数据结构

在 **jail.c** 中，首先需要注意的是声明了一个重要的结构 `struct jail j;`，该结构包含自 **/usr/include/sys/jail.h** 引入的内容。

`jail` 结构的定义如下：

```c
/usr/include/sys/jail.h:

struct jail {
        u_int32_t       version;
        char            *path;
        char            *hostname;
        u_int32_t       ip_number;
};
```

如你所见，每个传递给 [jail(8)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=8&format=html) 程序的参数都有一个对应的条目，实际上它们会在程序执行期间被设置。

```c
/usr/src/usr.sbin/jail/jail.c
char path[PATH_MAX];
...
if (realpath(argv[0], path) == NULL)
    err(1, "realpath: %s", argv[0]);
if (chdir(path) != 0)
    err(1, "chdir: %s", path);
memset(&j, 0, sizeof(j));
j.version = 0;
j.path = path;
j.hostname = argv[1];
```

#### 4.1.1.2. 网络

[jail(8)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=8&format=html) 程序的一个参数是用于通过网络访问 jail 的 IP 地址。[jail(8)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=8&format=html) 将给定的 IP 地址转换为主机字节序，然后将其存储在 `j`（`jail` 结构）中。

```c
/usr/src/usr.sbin/jail/jail.c:
struct in_addr in;
...
if (inet_aton(argv[2], &in) == 0)
    errx(1, "Could not make sense of ip-number: %s", argv[2]);
j.ip_number = ntohl(in.s_addr);
```

[inet\_aton(3)](https://man.freebsd.org/cgi/man.cgi?query=inet_aton&sektion=3&format=html) 函数“将指定的字符字符串解释为互联网地址，并将该地址存放到提供的结构中。”`ip_number` 成员仅在通过 [inet\_aton(3)](https://man.freebsd.org/cgi/man.cgi?query=inet_aton&sektion=3&format=html) 将 IP 地址存储到 `in` 结构中，并通过 [ntohl(3)](https://man.freebsd.org/cgi/man.cgi?query=ntohl&sektion=3&format=html) 转换为主机字节序时被设置。

#### 4.1.1.3. 限制进程

最后，用户空间程序将进程限制在 jail 中。此时，jail 也成为一个被监禁的进程，并执行通过 [execv(3)](https://man.freebsd.org/cgi/man.cgi?query=execv&sektion=3&format=html) 提供的命令。

```c
/usr/src/usr.sbin/jail/jail.c
i = jail(&j);
...
if (execv(argv[3], argv + 3) != 0)
    err(1, "execv: %s", argv[3]);
```

如你所见，调用了 `jail()` 函数，并且其参数是已填充的 `jail` 结构，最后执行了指定的程序。我将讨论 jail 在内核中的实现。

### 4.1.2. 内核空间

现在我们来看一下 **/usr/src/sys/kern/kern\_jail.c** 文件。这是定义了 [jail(2)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2&format=html) 系统调用、相关的 sysctl 以及网络功能的文件。

#### 4.1.2.1. Sysctl

在 **kern\_jail.c** 中，定义了以下 sysctl：

```c
/usr/src/sys/kern/kern_jail.c:
int     jail_set_hostname_allowed = 1;
SYSCTL_INT(_security_jail, OID_AUTO, set_hostname_allowed, CTLFLAG_RW,
    &jail_set_hostname_allowed, 0,
    "Processes in jail can set their hostnames");

int     jail_socket_unixiproute_only = 1;
SYSCTL_INT(_security_jail, OID_AUTO, socket_unixiproute_only, CTLFLAG_RW,
    &jail_socket_unixiproute_only, 0,
    "Processes in jail are limited to creating UNIX/IPv4/route sockets only");

int     jail_sysvipc_allowed = 0;
SYSCTL_INT(_security_jail, OID_AUTO, sysvipc_allowed, CTLFLAG_RW,
    &jail_sysvipc_allowed, 0,
    "Processes in jail can use System V IPC primitives");

static int jail_enforce_statfs = 2;
SYSCTL_INT(_security_jail, OID_AUTO, enforce_statfs, CTLFLAG_RW,
    &jail_enforce_statfs, 0,
    "Processes in jail cannot see all mounted file systems");

int    jail_allow_raw_sockets = 0;
SYSCTL_INT(_security_jail, OID_AUTO, allow_raw_sockets, CTLFLAG_RW,
    &jail_allow_raw_sockets, 0,
    "Prison root can create raw sockets");

int    jail_chflags_allowed = 0;
SYSCTL_INT(_security_jail, OID_AUTO, chflags_allowed, CTLFLAG_RW,
    &jail_chflags_allowed, 0,
    "Processes in jail can alter system file flags");

int     jail_mount_allowed = 0;
SYSCTL_INT(_security_jail, OID_AUTO, mount_allowed, CTLFLAG_RW,
    &jail_mount_allowed, 0,
    "Processes in jail can mount/unmount jail-friendly file systems");
```

每个 sysctl 都可以通过 [sysctl(8)](https://man.freebsd.org/cgi/man.cgi?query=sysctl&sektion=8&format=html) 程序由用户访问。整个内核都可以通过这些特定的 sysctl 名称识别它们。例如，第一个 sysctl 的名称是 `security.jail.set_hostname_allowed`。

#### 4.1.2.2. [jail(2)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2&format=html) 系统调用

像所有系统调用一样，[jail(2)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2&format=html) 系统调用接收两个参数，`struct thread *td` 和 `struct jail_args *uap`。`td` 是指向描述调用线程的 `thread` 结构的指针。在此上下文中，`uap` 是指向一个结构的指针，该结构包含指向用户空间 **jail.c** 传递的 `jail` 结构的指针。当我之前描述用户空间程序时，你看到 [jail(2)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2&format=html) 系统调用接收一个 `jail` 结构作为参数。

```c
/usr/src/sys/kern/kern_jail.c:
/*
 * struct jail_args {
 *  struct jail *jail;
 * };
 */
int
jail(struct thread *td, struct jail_args *uap)
```

因此，可以使用 `uap→jail` 来访问传递给系统调用的 `jail` 结构。接下来，系统调用使用 [copyin(9)](https://man.freebsd.org/cgi/man.cgi?query=copyin&sektion=9&format=html) 函数将 `jail` 结构复制到内核空间。[copyin(9)](https://man.freebsd.org/cgi/man.cgi?query=copyin&sektion=9&format=html) 接受三个参数：要复制到内核空间的数据的地址（即 `uap→jail`），存储目标地址（即 `j`）和存储的大小。传递给 `uap→jail` 的 `jail` 结构被复制到内核空间并存储在另一个 `jail` 结构 `j` 中。

```
/usr/src/sys/kern/kern_jail.c:
error = copyin(uap->jail, &j, sizeof(j));
```

在 **jail.h** 中，还定义了另一个重要的结构 `prison`。`prison` 结构只在内核空间中使用。以下是 `prison` 结构的定义。

```c
/usr/include/sys/jail.h:
struct prison {
        LIST_ENTRY(prison) pr_list;                     /* (a) 所有 prison */
        int              pr_id;                         /* (c) prison ID */
        int              pr_ref;                        /* (p) 引用计数 */
        char             pr_path[MAXPATHLEN];           /* (c) chroot 路径 */
        struct vnode    *pr_root;                       /* (c) 指向 root 的 vnode */
        char             pr_host[MAXHOSTNAMELEN];       /* (p) jail 主机名 */
        u_int32_t        pr_ip;                         /* (c) IP 地址 */
        void            *pr_linux;                      /* (p) linux ABI */
        int              pr_securelevel;                /* (p) 安全级别 */
        struct task      pr_task;                       /* (d) 销毁任务 */
        struct mtx       pr_mtx;
        void            **pr_slots;                     /* (p) 额外数据 */
};
```

[jail(2)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2&format=html) 系统调用接着为 `prison` 结构分配内存，并在 `jail` 和 `prison` 结构之间复制数据。

```c
/usr/src/sys/kern/kern_jail.c:
MALLOC(pr, struct prison *, sizeof(*pr), M_PRISON, M_WAITOK | M_ZERO);
...
error = copyinstr(j.path, &pr->pr_path, sizeof(pr->pr_path), 0);
if (error)
    goto e_killmtx;
...
error = copyinstr(j.hostname, &pr->pr_host, sizeof(pr->pr_host), 0);
if (error)
     goto e_dropvnref;
pr->pr_ip = j.ip_number;
```

接下来，我们将讨论另一个重要的系统调用 [jail\_attach(2)](https://man.freebsd.org/cgi/man.cgi?query=jail_attach&sektion=2&format=html)，它实现了将进程放入 jail 的功能。

```c
/usr/src/sys/kern/kern_jail.c:
/*
 * struct jail_attach_args {
 *      int jid;
 * };
 */
int
jail_attach(struct thread *td, struct jail_attach_args *uap)
```

该系统调用对进程进行一系列更改，使其与未被 jail 限制的进程区分开来。为了理解 [jail\_attach(2)](https://man.freebsd.org/cgi/man.cgi?query=jail_attach&sektion=2&format=html) 的作用，我们需要一些背景信息。

在 FreeBSD 中，每个内核可见的线程都由其 `thread` 结构标识，而进程则由其 `proc` 结构描述。你可以在 **/usr/include/sys/proc.h** 中找到 `thread` 和 `proc` 结构的定义。例如，任何系统调用中的 `td` 参数实际上是指向调用线程的 `thread` 结构的指针。如前所述，`td_proc` 是指向表示包含该线程的进程的 `proc` 结构的指针。`proc` 结构包含可以描述进程所有者身份（`p_ucred`）、进程资源限制（`p_limit`）等信息。在 `proc` 结构中指向的 `ucred` 结构（通过 `p_ucred` 成员访问）中，存在指向 `prison` 结构的指针（`cr_prison`）。

```c
/usr/include/sys/proc.h:
struct thread {
    ...
    struct proc *td_proc;
    ...
};
struct proc {
    ...
    struct ucred *p_ucred;
    ...
};
/usr/include/sys/ucred.h
struct ucred {
    ...
    struct prison *cr_prison;
    ...
};
```

在 **kern\_jail.c** 中，`jail()` 函数调用 `jail_attach()` 并传入给定的 `jid`。`jail_attach()` 然后调用 `change_root()` 函数来更改调用进程的根目录。随后，`jail_attach()` 创建一个新的 `ucred` 结构，并在成功将 `prison` 结构附加到 `ucred` 结构后，将新创建的 `ucred` 结构附加到调用进程。从那时起，调用进程将被视为一个已被 jail 限制的进程。当内核例程 `jailed()` 被调用并传入新创建的 `ucred` 结构时，它将返回 1，表示该凭据与某个 jail 相关联。所有在 jail 内创建的进程的公共祖先进程是运行 [jail(8)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=8&format=html) 的进程，因为它调用了 [jail(2)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2&format=html) 系统调用。当一个程序通过 [execve(2)](https://man.freebsd.org/cgi/man.cgi?query=execve&sektion=2&format=html) 被执行时，它会继承父进程的 `ucred` 结构中的 jail 属性，因此它也会有一个 jail 的 `ucred` 结构。

```c
/usr/src/sys/kern/kern_jail.c
int
jail(struct thread *td, struct jail_args *uap)
{
...
    struct jail_attach_args jaa;
...
    error = jail_attach(td, &jaa);
    if (error)
        goto e_dropprref;
...
}

int
jail_attach(struct thread *td, struct jail_attach_args *uap)
{
    struct proc *p;
    struct ucred *newcred, *oldcred;
    struct prison *pr;
...
    p = td->td_proc;
...
    pr = prison_find(uap->jid);
...
    change_root(pr->pr_root, td);
...
    newcred->cr_prison = pr;
    p->p_ucred = newcred;
...
}
```

当进程从其父进程派生时，[fork(2)](https://man.freebsd.org/cgi/man.cgi?query=fork&sektion=2&format=html) 系统调用使用 `crhold()` 来维持新派生进程的凭据。这本质上保持了新派生子进程的凭据与其父进程的一致性，因此子进程也会被 jail 限制。

```c
/usr/src/sys/kern/kern_fork.c:
p2->p_ucred = crhold(td->td_ucred);
...
td2->td_ucred = crhold(p2->p_ucred);
```

## 4.2. 限制

在内核中，针对被 jail 限制的进程有访问限制。通常，这些限制只会检查进程是否被 jail 限制，如果是，它将返回一个错误。例如：

```c
if (jailed(td->td_ucred))
    return (EPERM);
```

### 4.2.1. SysV IPC

System V IPC 基于消息。进程可以彼此发送这些消息，告诉它们如何操作。与消息相关的函数包括： [msgctl(3)](https://man.freebsd.org/cgi/man.cgi?query=msgctl&sektion=3&format=html)、[msgget(3)](https://man.freebsd.org/cgi/man.cgi?query=msgget&sektion=3&format=html)、[msgsnd(3)](https://man.freebsd.org/cgi/man.cgi?query=msgsnd&sektion=3&format=html) 和 [msgrcv(3)](https://man.freebsd.org/cgi/man.cgi?query=msgrcv&sektion=3&format=html)。之前我提到过，你可以开启或关闭某些 sysctl 来影响 jail 的行为。其中一个 sysctl 是 `security.jail.sysvipc_allowed`。默认情况下，这个 sysctl 被设置为 0。如果它被设置为 1，将破坏 jail 的基本目的；jail 内的特权用户将能够影响 jail 外部的进程。消息和信号之间的区别在于，消息仅包含信号编号。

**/usr/src/sys/kern/sysv\_msg.c**：

* `msgget(key, msgflg)`：`msgget` 返回（并可能创建）一个消息描述符，该描述符表示要在其他函数中使用的消息队列。
* `msgctl(msgid, cmd, buf)`：通过此函数，进程可以查询消息描述符的状态。
* `msgsnd(msgid, msgp, msgsz, msgflg)`：`msgsnd` 将一条消息发送到进程。
* `msgrcv(msgid, msgp, msgsz, msgtyp, msgflg)`：进程使用此函数接收消息。

在与这些函数对应的每个系统调用中，都有以下条件判断：

```c
/usr/src/sys/kern/sysv_msg.c:
if (!jail_sysvipc_allowed && jailed(td->td_ucred))
    return (ENOSYS);
```

信号量系统调用允许进程通过对一组信号量执行原子操作来同步执行。基本上，信号量为进程提供了另一种锁定资源的方法。然而，等待信号量的进程将会进入睡眠状态，直到资源被释放。以下信号量系统调用在 jail 内部被阻止：[semget(2)](https://man.freebsd.org/cgi/man.cgi?query=semget&sektion=2&format=html)、[semctl(2)](https://man.freebsd.org/cgi/man.cgi?query=semctl&sektion=2&format=html) 和 [semop(2)](https://man.freebsd.org/cgi/man.cgi?query=semop&sektion=2&format=html)。

**/usr/src/sys/kern/sysv\_sem.c**：

* `semctl(semid, semnum, cmd, …)`：`semctl` 对由 `semid` 标识的信号量队列执行指定的 `cmd`。
* `semget(key, nsems, flag)`：`semget` 创建一个信号量数组，该数组与 `key` 对应。
  `key 和 flag 的意义与 msgget 中相同。`
* `semop(semid, array, nops)`：`semop` 对由 `semid` 标识的信号量集合执行由 `array` 指定的一组操作。

System V IPC 允许进程共享内存。进程可以通过共享它们的虚拟地址空间的部分内容，并读取和写入存储在共享内存中的数据，从而直接相互通信。这些系统调用在 jail 环境内被阻止：[shmdt(2)](https://man.freebsd.org/cgi/man.cgi?query=shmdt&sektion=2&format=html)、[shmat(2)](https://man.freebsd.org/cgi/man.cgi?query=shmat&sektion=2&format=html)、[shmctl(2)](https://man.freebsd.org/cgi/man.cgi?query=shmctl&sektion=2&format=html) 和 [shmget(2)](https://man.freebsd.org/cgi/man.cgi?query=shmget&sektion=2&format=html)。

**/usr/src/sys/kern/sysv\_shm.c**：

* `shmctl(shmid, cmd, buf)`：`shmctl` 对由 `shmid` 标识的共享内存区域执行各种控制操作。
* `shmget(key, size, flag)`：`shmget` 访问或创建一个大小为 `size` 字节的共享内存区域。
* `shmat(shmid, addr, flag)`：`shmat` 将由 `shmid` 标识的共享内存区域附加到进程的地址空间。
* `shmdt(addr)`：`shmdt` 从地址 `addr` 中分离先前附加的共享内存区域。

### 4.2.2. 套接字

Jail 以特殊的方式处理 [socket(2)](https://man.freebsd.org/cgi/man.cgi?query=socket&sektion=2&format=html) 系统调用和相关的低级套接字函数。为了确定是否允许创建某个套接字，首先检查 sysctl `security.jail.socket_unixiproute_only` 是否被设置。如果设置了，只有当套接字的类型为 `PF_LOCAL`、`PF_INET` 或 `PF_ROUTE` 时，才允许创建该套接字，否则返回错误。

```c
/usr/src/sys/kern/uipc_socket.c:
int
socreate(int dom, struct socket **aso, int type, int proto,
    struct ucred *cred, struct thread *td)
{
    struct protosw *prp;
...
    if (jailed(cred) && jail_socket_unixiproute_only &&
        prp->pr_domain->dom_family != PF_LOCAL &&
        prp->pr_domain->dom_family != PF_INET &&
        prp->pr_domain->dom_family != PF_ROUTE) {
        return (EPROTONOSUPPORT);
    }
...
}
```

### 4.2.3. 伯克利数据包过滤器（BPF）

伯克利数据包过滤器（BPF）提供了一种原始接口，用于在协议无关的方式下访问数据链路层。现在，BPF 是否能在 jail 环境中使用由 [devfs(8)](https://man.freebsd.org/cgi/man.cgi?query=devfs&sektion=8&format=html) 控制。

### 4.2.4. 协议

有一些非常常见的协议，比如 TCP、UDP、IP 和 ICMP。IP 和 ICMP 位于同一层级：网络层 2。在某些情况下，会采取一些预防措施，防止 jailed 进程绑定到某个特定地址，前提是 `nam` 参数被设置。`nam` 是一个指向 `sockaddr` 结构的指针，该结构描述了绑定服务的地址。更准确的定义是，`sockaddr` "可以用作引用每个地址的标识标签和长度的模板"。在函数 `in_pcbbind_setup()` 中，`sin` 是指向 `sockaddr_in` 结构的指针，该结构包含端口、地址、长度和要绑定的套接字的域族。基本上，这不允许 jail 中的任何进程指定不属于该进程所在 jail 的地址。

```c
/usr/src/sys/netinet/in_pcb.c:
int
in_pcbbind_setup(struct inpcb *inp, struct sockaddr *nam, in_addr_t *laddrp,
    u_short *lportp, struct ucred *cred)
{
    ...
    struct sockaddr_in *sin;
    ...
    if (nam) {
        sin = (struct sockaddr_in *)nam;
        ...
        if (sin->sin_addr.s_addr != INADDR_ANY)
            if (prison_ip(cred, 0, &sin->sin_addr.s_addr))
                return(EINVAL);
        ...
        if (lport) {
            ...
            if (prison && prison_ip(cred, 0, &sin->sin_addr.s_addr))
                return (EADDRNOTAVAIL);
            ...
        }
    }
    if (lport == 0) {
        ...
        if (laddr.s_addr != INADDR_ANY)
            if (prison_ip(cred, 0, &laddr.s_addr))
                return (EINVAL);
        ...
    }
...
    if (prison_ip(cred, 0, &laddr.s_addr))
        return (EINVAL);
...
}
```

你可能会想知道 `prison_ip()` 函数是做什么的。`prison_ip()` 函数接受三个参数，一个指向凭证（由 `cred` 表示）的指针、任何标志和一个 IP 地址。如果该 IP 地址不属于 jail，它将返回 1，否则返回 0。从代码中可以看出，如果该 IP 地址确实不属于 jail，则协议不允许绑定到该地址。

```c
/usr/src/sys/kern/kern_jail.c:
int
prison_ip(struct ucred *cred, int flag, u_int32_t *ip)
{
    u_int32_t tmp;

    if (!jailed(cred))
        return (0);
    if (flag)
        tmp = *ip;
    else
        tmp = ntohl(*ip);
    if (tmp == INADDR_ANY) {
        if (flag)
            *ip = cred->cr_prison->pr_ip;
        else
            *ip = htonl(cred->cr_prison->pr_ip);
        return (0);
    }
    if (tmp == INADDR_LOOPBACK) {
        if (flag)
            *ip = cred->cr_prison->pr_ip;
        else
            *ip = htonl(cred->cr_prison->pr_ip);
        return (0);
    }
    if (cred->cr_prison->pr_ip != tmp)
        return (1);
    return (0);
}
```

### 4.2.5. 文件系统

即使是 jail 中的 `root` 用户，在安全级别（securelevel）大于 0 的情况下，也不允许取消或修改任何文件标志，例如不可变（immutable）、仅追加（append-only）和不可删除（undeleteable）标志。

```c
/usr/src/sys/ufs/ufs/ufs_vnops.c:
static int
ufs_setattr(ap)
    ...
{
    ...
        if (!priv_check_cred(cred, PRIV_VFS_SYSFLAGS, 0)) {
            if (ip->i_flags
                & (SF_NOUNLINK | SF_IMMUTABLE | SF_APPEND)) {
                    error = securelevel_gt(cred, 0);
                    if (error)
                        return (error);
            }
            ...
        }
}

/usr/src/sys/kern/kern_priv.c
int
priv_check_cred(struct ucred *cred, int priv, int flags)
{
    ...
    error = prison_priv_check(cred, priv);
    if (error)
        return (error);
    ...
}

/usr/src/sys/kern/kern_jail.c
int
prison_priv_check(struct ucred *cred, int priv)
{
    ...
    switch (priv) {
    ...
    case PRIV_VFS_SYSFLAGS:
        if (jail_chflags_allowed)
            return (0);
        else
            return (EPERM);
    ...
    }
    ...
}
```
