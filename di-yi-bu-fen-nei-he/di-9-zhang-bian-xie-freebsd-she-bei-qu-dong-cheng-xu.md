# 第 9 章 编写 FreeBSD 设备驱动程序

## 9.1. 介绍

本章简要介绍了如何为 FreeBSD 编写设备驱动程序。在这个上下文中，“设备”通常指的是与硬件相关的系统组件，例如磁盘、打印机或带有键盘的显示器。设备驱动程序是操作系统中控制特定设备的软件组件。还有所谓的伪设备，设备驱动程序通过软件模拟设备的行为，而没有任何特定的底层硬件。设备驱动程序可以静态地编译进系统，或者通过动态内核链接器设施 `kld` 按需加载。

在类 UNIX® 操作系统中，大多数设备通过设备节点进行访问，这些节点有时也称为特殊文件。这些文件通常位于文件系统层次结构中的 **/dev** 目录下。

设备驱动程序大致可以分为两类：字符设备驱动程序和网络设备驱动程序。

## 9.2. 动态内核链接器设施 - KLD

`kld` 接口允许系统管理员动态地为运行中的系统添加和移除功能。这使得设备驱动程序的编写者能够在不需要不断重启来测试更改的情况下，将新的更改加载到运行中的内核中。

通过以下命令使用 `kld` 接口：

* `kldload` - 加载一个新的内核模块
* `kldunload` - 卸载一个内核模块
* `kldstat` - 列出已加载的模块

内核模块的框架布局

```c
/*
 * KLD 框架
 * 灵感来源于 Andrew Reiter 的 Daemonnews 文章
 */

#include <sys/types.h>
#include <sys/systm.h>  /* uprintf */
#include <sys/errno.h>
#include <sys/param.h>  /* kernel.h 中使用的定义 */
#include <sys/module.h>
#include <sys/kernel.h> /* 模块初始化中使用的类型 */

/*
 * 处理 KLD 加载和卸载的加载处理函数。
 */

static int
skel_loader(struct module *m, int what, void *arg)
{
	int err = 0;

	switch (what) {
	case MOD_LOAD:                /* kldload */
		uprintf("Skeleton KLD 加载。\n");
		break;
	case MOD_UNLOAD:
		uprintf("Skeleton KLD 卸载。\n");
		break;
	default:
		err = EOPNOTSUPP;
		break;
	}
	return(err);
}

/* 向内核声明此模块 */

static moduledata_t skel_mod = {
	"skel",
	skel_loader,
	NULL
};

DECLARE_MODULE(skeleton, skel_mod, SI_SUB_KLD, SI_ORDER_ANY);
```

### 9.2.1. Makefile

FreeBSD 提供了一个系统 Makefile 来简化编译内核模块。

```c
SRCS=skeleton.c
KMOD=skeleton

.include <bsd.kmod.mk>
```

使用此 Makefile 运行 `make` 将生成一个文件 **skeleton.ko**，可以通过以下命令将其加载到内核中：

```sh
# kldload -v ./skeleton.ko
```

## 9.3. 字符设备

字符设备驱动程序是直接与用户进程交换数据的驱动程序。这是最常见的设备驱动程序类型，源代码树中有许多简单的示例。

这个简单的伪设备例子记住了写入它的所有值，并且在读取时可以将其回显出来。

```c
/*
 * 简单的 Echo 伪设备 KLD
 *
 * Murray Stokely
 * Søren (Xride) Straarup
 * Eitan Adler
 */

#include <sys/types.h>
#include <sys/systm.h>  /* uprintf */
#include <sys/param.h>  /* kernel.h 中使用的定义 */
#include <sys/module.h>
#include <sys/kernel.h> /* 模块初始化中使用的类型 */
#include <sys/conf.h>   /* cdevsw 结构体 */
#include <sys/uio.h>    /* uio 结构体 */
#include <sys/malloc.h>

#define BUFFERSIZE 255

/* 函数原型 */
static d_open_t      echo_open;
static d_close_t     echo_close;
static d_read_t      echo_read;
static d_write_t     echo_write;

/* 字符设备入口点 */
static struct cdevsw echo_cdevsw = {
	.d_version = D_VERSION,
	.d_open = echo_open,
	.d_close = echo_close,
	.d_read = echo_read,
	.d_write = echo_write,
	.d_name = "echo",
};

struct s_echo {
	char msg[BUFFERSIZE + 1];
	int len;
};

/* 变量 */
static struct cdev *echo_dev;
static struct s_echo *echomsg;

MALLOC_DECLARE(M_ECHOBUF);
MALLOC_DEFINE(M_ECHOBUF, "echobuffer", "echo 模块的缓冲区");

/*
 * 该函数由 kld[un]load(2) 系统调用调用，用于
 * 确定加载或卸载模块时需要执行的操作。
 */
static int
echo_loader(struct module *m __unused, int what, void *arg __unused)
{
	int error = 0;

	switch (what) {
	case MOD_LOAD:                /* kldload */
		error = make_dev_p(MAKEDEV_CHECKNAME | MAKEDEV_WAITOK,
		    &echo_dev,
		    &echo_cdevsw,
		    0,
		    UID_ROOT,
		    GID_WHEEL,
		    0600,
		    "echo");
		if (error != 0)
			break;

		echomsg = malloc(sizeof(*echomsg), M_ECHOBUF, M_WAITOK |
		    M_ZERO);
		printf("Echo 设备已加载。\n");
		break;
	case MOD_UNLOAD:
		destroy_dev(echo_dev);
		free(echomsg, M_ECHOBUF);
		printf("Echo 设备已卸载。\n");
		break;
	default:
		error = EOPNOTSUPP;
		break;
	}
	return (error);
}

static int
echo_open(struct cdev *dev __unused, int oflags __unused, int devtype __unused,
    struct thread *td __unused)
{
	int error = 0;

	uprintf("成功打开设备 \"echo\"。\n");
	return (error);
}

static int
echo_close(struct cdev *dev __unused, int fflag __unused, int devtype __unused,
    struct thread *td __unused)
{

	uprintf("关闭设备 \"echo\"。\n");
	return (0);
}

/*
 * 读取函数将通过 echo_write() 保存的 buf 传递回
 * 用户空间供访问。
 * uio(9)
 */
static int
echo_read(struct cdev *dev __unused, struct uio *uio, int ioflag __unused)
{
	size_t amt;
	int error;

	/*
	 * 读取操作的大小是多少？ 可能是用户请求的大小，
	 * 或者是剩余数据的大小。 注意 'len' 不包括尾随的空字符。
	 */
	amt = MIN(uio->uio_resid, uio->uio_offset >= echomsg->len + 1 ? 0 :
	    echomsg->len + 1 - uio->uio_offset);

	if ((error = uiomove(echomsg->msg, amt, uio)) != 0)
		uprintf("uiomove 失败！\n");

	return (error);
}

/*
 * echo_write 接受一个字符字符串并将其保存到
 * buf 中以供以后访问。
 */
static int
echo_write(struct cdev *dev __unused, struct uio *uio, int ioflag __unused)
{
	size_t amt;
	int error;

	/*
	 * 我们要么从头开始写入，要么是追加 -- 不允许随机访问。
	 */
	if (uio->uio_offset != 0 && (uio->uio_offset != echomsg->len))
		return (EINVAL);

	/* 这是一个新消息，重置长度 */
	if (uio->uio_offset == 0)
		echomsg->len = 0;

	/* 将字符串从用户内存复制到内核内存 */
	amt = MIN(uio->uio_resid, (BUFFERSIZE - echomsg->len));

	error = uiomove(echomsg->msg + uio->uio_offset, amt, uio);

	/* 现在我们需要添加空字符并记录长度 */
	echomsg->len = uio->uio_offset;
	echomsg->msg[echomsg->len] = 0;

	if (error != 0)
		uprintf("写入失败：地址错误！\n");
	return (error);
}

DEV_MODULE(echo, echo_loader, NULL);
```

加载此驱动程序后，尝试执行：

```sh
# echo -n "Test Data" > /dev/echo
# cat /dev/echo
成功打开设备 "echo"。
Test Data
关闭设备 "echo"。
```

真实硬件设备将在下一章中描述。

## 9.4. 块设备（已废除）

其他 UNIX® 系统可能支持另一种磁盘设备，称为块设备。块设备是内核提供缓存的磁盘设备。这种缓存使得块设备几乎无法使用，或者至少是不可靠的。缓存会重新排序写操作的顺序，剥夺了应用程序在任何时刻知道磁盘内容的能力。

这使得磁盘上数据结构（文件系统、数据库等）的可预测和可靠的崩溃恢复变得不可能。由于写操作可能被延迟，内核无法报告哪个特定的写操作遇到了写入错误，这进一步加剧了一致性问题。

因此，没有严肃的应用程序依赖于块设备，事实上，几乎所有直接访问磁盘的应用程序都非常注意指定应始终使用字符（或“原始”）设备。由于每个磁盘（分区）别名为两个具有不同语义的设备的实现显著复杂了相关内核代码，FreeBSD 在磁盘 I/O 基础设施现代化过程中放弃了对缓存磁盘设备的支持。

## 9.5. 网络驱动程序

网络设备的驱动程序不使用设备节点来访问。它们的选择基于内核中的其他决策，且通常通过使用系统调用 socket(2) 来引入对网络设备的使用，而不是调用 open()。

有关更多信息，请参见 ifnet(9)，这是环回设备的源代码。
