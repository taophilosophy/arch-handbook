# 第 4 章 jail 子系统

在大多数 UNIX® 系统上， root 拥有无所不能的权力。这促进了不安全性。如果黑客在系统上获得了 root ，他将可以使用手边的每个功能。在 FreeBSD 中，有一些 sysctls 降低了 root 的权限，以最小化黑客造成的破坏。具体来说，其中一个功能被称为 secure levels 。同样地，另一个功能从 FreeBSD 4.0 开始存在，是一个称为 jail(8)的实用程序。Jail 为一个环境创建了 chroot 并对在 jail 内分叉的进程设置了某些限制。例如，一个被 jail 的进程无法影响 jail 外部的进程、利用某些系统调用或对主机环境造成任何破坏。

Jail 正在成为新的安全模型。人们在 jail 内运行潜在易受攻击的服务器，比如 Apache、BIND 和 sendmail，因此，如果黑客在 jail 内获得了 root ，那只是个恼人的大问题，而不是灾难。本文主要关注 jail 的内部（源代码）。关于如何设置 jail 的信息，请参阅有关 jails 的手册条目。

## 4.1. 架构

Jail 由两个领域组成：用户空间程序 jail(8)和内核中实现的代码：系统调用 jail(2)及相关限制。我将讨论用户空间程序，然后讨论 jail 如何在内核中实现。

### 4.1.1. 用户空间代码

用户空间 jail 的源代码位于/usr/src/usr.sbin/jail，包括一个文件 jail.c。该程序接受这些参数：jail 的路径，主机名，IP 地址和要执行的命令。

#### 4.1.1.1. 数据结构

在 jail.c 中，我要注意的第一件事是从 /usr/include/sys/jail.h 包含的一个重要结构 struct jail j; 的声明。

jail 结构的定义是：

```
/usr/include/sys/jail.h:

struct jail {
        u_int32_t       version;
        char            *path;
        char            *hostname;
        u_int32_t       ip_number;
};
```

正如您所看到的，对于传递给 jail(8) 程序的每个参数，确实在其执行期间设置。

```
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

传递给 jail(8) 程序的参数之一是一个 IP 地址，通过该 IP 地址可以访问 jail。jail(8) 将给定的 IP 地址转换为主机字节顺序，然后将其存储在 j ( jail 结构体) 中。

```
/usr/src/usr.sbin/jail/jail.c:
struct in_addr in;
...
if (inet_aton(argv[2], &in) == 0)
    errx(1, "Could not make sense of ip-number: %s", argv[2]);
j.ip_number = ntohl(in.s_addr);
```

inet_aton(3) 函数会“将指定的字符串解释为一个互联网地址，并将该地址放入所提供的结构中。” 当 IP 地址由 inet_aton(3) 放入结构中，并且通过 ntohl(3) 被转换为主机字节顺序时， ip_number 结构中的 jail 成员才会被设置。

#### 4.1.1.3. 限制进程

最后，用户空间程序 jails 进程。现在，进程成为一个被 jail 的进程，并使用 execv(3) 执行给定的命令。

```
/usr/src/usr.sbin/jail/jail.c
i = jail(&j);
...
if (execv(argv[3], argv + 3) != 0)
    err(1, "execv: %s", argv[3]);
```

As you can see, the `jail()` function is called, and its argument is the `jail` structure which has been filled with the arguments given to the program. Finally, the program you specify is executed. I will now discuss how jail is implemented within the kernel.

### 4.1.2. Kernel Space

We will now be looking at the file /usr/src/sys/kern/kern_jail.c. This is the file where the [jail(2)](https://man.freebsd.org/cgi/man.cgi?query=jail&sektion=2&format=html) system call, appropriate sysctls, and networking functions are defined.

#### 4.1.2.1。 Sysctls

在 kern_jail.c 中，定义了以下 sysctls：

```
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

用户可以通过 sysctl(8)程序访问这些 sysctl 中的每一个。在整个内核中，这些特定 sysctl 通过其名称来识别。例如，第一个 sysctl 的名称是 security.jail.set_hostname_allowed 。

#### 4.1.2.2. jail(2) 系统调用

像所有系统调用一样，jail(2) 系统调用接受两个参数， struct thread *td 和 struct jail_args *uap 。 td 是描述调用线程的结构体指针。在这种情况下， uap 是指向用户态传递的 jail 结构体的指针所包含的结构体指针。当我之前描述用户态程序时，你看到 jail(2) 系统调用被赋予一个 jail 结构体作为自己的参数。

```
/usr/src/sys/kern/kern_jail.c:
/*
 * struct jail_args {
 *  struct jail *jail;
 * };
 */
int
jail(struct thread *td, struct jail_args *uap)
```

因此， uap→jail 可以用来访问传递给系统调用的 jail 结构体。接下来，系统调用使用 copyin(9) 函数将 jail 结构体复制到内核空间。copyin(9) 接受三个参数：要复制到内核空间的数据的地址， uap→jail ，存储位置 j 和存储空间的大小。由 uap→jail 指向的 jail 结构体被复制到内核空间，并存储在另一个 jail 结构体 j 中。

```
/usr/src/sys/kern/kern_jail.c:
error = copyin(uap->jail, &j, sizeof(j));
```

在 jail.h 中定义了另一个重要的结构。这是 prison 结构。 prison 结构专门在内核空间中使用。这是 prison 结构的定义。

```
/usr/include/sys/jail.h:
struct prison {
        LIST_ENTRY(prison) pr_list;                     /* (a) all prisons */
        int              pr_id;                         /* (c) prison id */
        int              pr_ref;                        /* (p) refcount */
        char             pr_path[MAXPATHLEN];           /* (c) chroot path */
        struct vnode    *pr_root;                       /* (c) vnode to rdir */
        char             pr_host[MAXHOSTNAMELEN];       /* (p) jail hostname */
        u_int32_t        pr_ip;                         /* (c) ip addr host */
        void            *pr_linux;                      /* (p) linux abi */
        int              pr_securelevel;                /* (p) securelevel */
        struct task      pr_task;                       /* (d) destroy task */
        struct mtx       pr_mtx;
      void            **pr_slots;                     /* (p) additional data */
};
```

然后，jail(2)系统调用为 prison 结构分配内存，并在 jail 和 prison 结构之间复制数据。

```
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

接下来，我们将讨论另一个重要的系统调用 jail_attach(2)，它实现了将进程放入 jail 的功能。

```
/usr/src/sys/kern/kern_jail.c:
/*
 * struct jail_attach_args {
 *      int jid;
 * };
 */
int
jail_attach(struct thread *td, struct jail_attach_args *uap)
```

这个系统调用会进行改变，以区分被 jail 的进程和未被 jail 的进程。 要理解 jail_attach(2) 对我们的作用，需要一些背景信息。

在 FreeBSD 中，每个内核可见线程由其 thread 结构标识，而进程由其 proc 结构描述。 您可以在 /usr/include/sys/proc.h 中找到 thread 和 proc 结构的定义。 例如，任何系统调用中的 td 参数实际上是指向调用线程的 thread 结构的指针，正如前面所述。 td_proc 结构中的 thread 成员，由 td 指向的结构，是指向包含由 td 表示的线程的进程的 proc 结构的指针。 proc 结构包含可以描述所有者身份（ p_ucred ）、进程资源限制（ p_limit ）等的成员。 在由 proc 结构中的 p_ucred 成员指向的 ucred 结构中，有一个指向 prison 结构（ cr_prison ）的指针。

```
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

在 kern_jail.c 中，函数 jail() 然后使用给定的 jid 调用函数 jail_attach() 。 jail_attach() 调用函数 change_root() 来更改调用进程的根目录。 然后 jail_attach() 创建一个新的 ucred 结构，并在成功将 prison 结构附加到 ucred 结构之后，将新创建的 ucred 结构附加到调用进程。 从那时起，调用进程被认为是被 jail 的。 当在内核中调用带有新创建的 ucred 结构作为其参数的内核例程 jailed() 时，返回 1 以告知凭证与 jail 连接。 在 jail 中生成的所有进程的公共祖先进程，是运行 jail(8) 的进程，因为它调用 jail(2) 系统调用。 通过 execve(2) 执行程序时，它会继承其父进程的 ucred 结构的被 jail 属性，因此它具有一个被 jail 的 ucred 结构。

```
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

当从其父进程派生进程时，fork(2)系统调用使用 crhold() 来维护新派生进程的凭据。它本质上保持新派生子进程的凭据与其父进程一致，因此子进程也被禁闭。

```
/usr/src/sys/kern/kern_fork.c:
p2->p_ucred = crhold(td->td_ucred);
...
td2->td_ucred = crhold(p2->p_ucred);
```

## 4.2. 限制

在整个内核中都存在与被禁闭进程相关的访问限制。通常，这些限制只检查进程是否被禁闭，如果是，则返回一个错误。例如：

```
if (jailed(td->td_ucred))
    return (EPERM);
```

### 4.2.1. SysV IPC

System V IPC 基于消息。进程可以相互发送这些告诉它们如何行动的消息。处理消息的函数包括：msgctl(3)、msgget(3)、msgsnd(3)和 msgrcv(3)。前面我提到过，您可以打开或关闭某些 sysctl 以影响 jail 的行为。其中一个 sysctl 是 security.jail.sysvipc_allowed 。默认情况下，此 sysctl 设置为 0。如果设置为 1，则会破坏拥有 jail 的整个目的；特权用户可以影响被 jail 化环境之外的进程。消息和信号之间的区别在于消息只包含信号编号。

/usr/src/sys/kern/sysv_msg.c:

- msgget(key, msgflg) ： msgget 返回（并可能创建）一个消息描述符，指定一个消息队列供其他函数使用。
- msgctl(msgid, cmd, buf) ：使用此函数，进程可以查询消息描述符的状态。
- msgsnd(msgid, msgp, msgsz, msgflg) ： msgsnd 向进程发送消息。
- msgrcv(msgid, msgp, msgsz, msgtyp, msgflg) ：一个进程使用此函数接收消息

在与这些函数对应的每个系统调用中，都有这个条件：

```
/usr/src/sys/kern/sysv_msg.c:
if (!jail_sysvipc_allowed && jailed(td->td_ucred))
    return (ENOSYS);
```

信号量系统调用允许进程通过对一组信号量进行一组原子操作来同步执行。基本上，信号量为进程提供了另一种锁定资源的方式。然而，等待正在使用的信号量的进程将会休眠，直到资源被释放。以下信号量系统调用在一个 jail 内被阻塞：semget(2)，semctl(2)和 semop(2)。

/usr/src/sys/kern/sysv_sem.c:

- semctl(semid, semnum, cmd, …) ： semctl 在由 semid 指示的信号量队列上执行指定的 cmd 。
- semget(key, nsems, flag) ： semget 创建一个与 key 对应的信号量数组。 key and flag take on the same meaning as they do in msgget.
- semop(semid, array, nops) ： semop 执行由 array 指示的一组操作，对 semid 标识的信号量集进行操作。

System V IPC 允许进程共享内存。进程可以通过共享它们的虚拟地址空间的部分，直接与彼此通信，然后读取和写入存储在共享内存中的数据。这些系统调用在受限环境中被阻止：shmdt(2)、shmat(2)、shmctl(2)和 shmget(2)。

/usr/src/sys/kern/sysv_shm.c:

- shmctl(shmid, cmd, buf) ： shmctl 对由 shmid 标识的共享内存区执行各种控制操作。
- shmget(key, size, flag) ： shmget 访问或创建一个 size 字节的共享内存区。
- shmat(shmid, addr, flag) ： shmat 将由 shmid 标识的共享内存区附加到进程的地址空间。
- shmdt(addr) ： shmdt 分离先前附加在 addr 的共享内存区域。

### 4.2.2. 套接字

Jail 特殊处理 socket(2) 系统调用和相关的底层套接字函数。为了确定是否允许创建特定套接字，它首先检查 sysctl security.jail.socket_unixiproute_only 是否设置。如果设置，只允许创建指定族为 PF_LOCAL 、 PF_INET 或 PF_ROUTE 的套接字。否则，将返回错误。

```
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

### 4.2.3. 伯克利数据包过滤器

伯克利数据包过滤器以协议无关的方式为数据链路层提供原始接口。BPF 现在由 devfs(8)控制，是否可以在受限环境中使用。

### 4.2.4. 协议

有一些非常常见的协议，比如 TCP、UDP、IP 和 ICMP。IP 和 ICMP 在同一层级：网络层 2。为了防止被 jail 的进程绑定协议到特定地址，会采取一些预防措施，只有当设置了 nam 参数时才会生效。 nam 是指向 sockaddr 结构的指针，描述要绑定服务的地址。更精确的定义是 sockaddr “可用作引用每个地址的标识标签和长度的模板”。在函数 in_pcbbind_setup() 中， sin 是指向 sockaddr_in 结构的指针，其中包含 port、地址、长度和套接字的域家族，该套接字将被绑定。基本上，这禁止任何进程从 jail 到能够指定不属于调用进程所在 jail 的地址。

```
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

也许你想知道函数 prison_ip() 的作用是什么。 prison_ip() 给出三个参数，一个指向凭证的指针（由 cred 表示），任何标志和一个 IP 地址。如果 IP 地址不属于 jail，则返回 1，否则返回 0。从代码中可以看出，如果确实是一个不属于 jail 的 IP 地址，那么协议就不允许绑定到该地址。

```
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

即使在安全等级高于 0 时，内部用户 root 也不被允许取消或修改任何文件标志，例如不可变、追加和不可删除标志。

```
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
