# 第 9 章 编写 FreeBSD 设备驱动程序


## 9.1. 介绍

本章简要介绍了为 FreeBSD 编写设备驱动程序的内容。在这里，设备是指系统中大多数用于硬件相关的东西，如磁盘、打印机或带有键盘的图形显示器。设备驱动程序是操作系统的软件组件，用于控制特定设备。还有所谓的伪设备，其中设备驱动程序在软件中模拟设备的行为，而没有任何特定的底层硬件。设备驱动程序可以静态地编译到系统中，也可以通过动态内核链接器 `kld' 按需加载。

UNIX®-类操作系统中的大多数设备通过设备节点访问，有时也称为特殊文件。这些文件通常位于文件系统层次结构中的 /dev 目录下。

设备驱动程序大致可以分为两类；字符设备驱动程序和网络设备驱动程序。

## 9.2. 动态内核链接器功能 - KLD

kld 接口允许系统管理员动态地向运行中的系统添加和删除功能。这允许设备驱动程序编写者将其新更改加载到运行中的内核中，而无需不断重新启动以测试更改。

kld 接口用于：

* kldload - 加载一个新的内核模块
* kldunload - 卸载一个内核模块
* kldstat - 列出加载的模块

内核模块的骨架布局

```
/*
 * KLD Skeleton
 * Inspired by Andrew Reiter's Daemonnews article
 */

#include <sys/types.h>
#include <sys/systm.h>  /* uprintf */
#include <sys/errno.h>
#include <sys/param.h>  /* defines used in kernel.h */
#include <sys/module.h>
#include <sys/kernel.h> /* types used in module initialization */

/*
 * Load handler that deals with the loading and unloading of a KLD.
 */

static int
skel_loader(struct module *m, int what, void *arg)
{
	int err = 0;

	switch (what) {
	case MOD_LOAD:                /* kldload */
		uprintf("Skeleton KLD loaded.\n");
		break;
	case MOD_UNLOAD:
		uprintf("Skeleton KLD unloaded.\n");
		break;
	default:
		err = EOPNOTSUPP;
		break;
	}
	return(err);
}

/* Declare this module to the rest of the kernel */

static moduledata_t skel_mod = {
	"skel",
	skel_loader,
	NULL
};

DECLARE_MODULE(skeleton, skel_mod, SI_SUB_KLD, SI_ORDER_ANY);
```

### 9.2.1. Makefile

FreeBSD 提供了一个系统 makefile 来简化编译内核模块的过程。

```
SRCS=skeleton.c
KMOD=skeleton

.include <bsd.kmod.mk>
```

运行 make ，将使用这个 makefile 创建一个名为 skeleton.ko 的文件，可以通过键入以下命令将其加载到内核中：

```
# kldload -v ./skeleton.ko
```

## 9.3. 字符设备

一个字符设备驱动程序是直接向用户进程传输数据的驱动程序。这是最常见的设备驱动程序类型，在源树中有许多简单的示例。

这个简单的伪设备示例会记住写入它的任何值，并在读取时将它们回显出来。

示例 1。适用于 FreeBSD 10.X - 12.X 的样本回显伪设备驱动程序示例。

```
/*
 * Simple Echo pseudo-device KLD
 *
 * Murray Stokely
 * Søren (Xride) Straarup
 * Eitan Adler
 */

#include <sys/types.h>
#include <sys/systm.h>  /* uprintf */
#include <sys/param.h>  /* defines used in kernel.h */
#include <sys/module.h>
#include <sys/kernel.h> /* types used in module initialization */
#include <sys/conf.h>   /* cdevsw struct */
#include <sys/uio.h>    /* uio struct */
#include <sys/malloc.h>

#define BUFFERSIZE 255

/* Function prototypes */
static d_open_t      echo_open;
static d_close_t     echo_close;
static d_read_t      echo_read;
static d_write_t     echo_write;

/* Character device entry points */
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

/* vars */
static struct cdev *echo_dev;
static struct s_echo *echomsg;

MALLOC_DECLARE(M_ECHOBUF);
MALLOC_DEFINE(M_ECHOBUF, "echobuffer", "buffer for echo module");

/*
 * This function is called by the kld[un]load(2) system calls to
 * determine what actions to take when a module is loaded or unloaded.
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
		printf("Echo device loaded.\n");
		break;
	case MOD_UNLOAD:
		destroy_dev(echo_dev);
		free(echomsg, M_ECHOBUF);
		printf("Echo device unloaded.\n");
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

	uprintf("Opened device \"echo\" successfully.\n");
	return (error);
}

static int
echo_close(struct cdev *dev __unused, int fflag __unused, int devtype __unused,
    struct thread *td __unused)
{

	uprintf("Closing device \"echo\".\n");
	return (0);
}

/*
 * The read function just takes the buf that was saved via
 * echo_write() and returns it to userland for accessing.
 * uio(9)
 */
static int
echo_read(struct cdev *dev __unused, struct uio *uio, int ioflag __unused)
{
	size_t amt;
	int error;

	/*
	 * How big is this read operation?  Either as big as the user wants,
	 * or as big as the remaining data.  Note that the 'len' does not
	 * include the trailing null character.
	 */
	amt = MIN(uio->uio_resid, uio->uio_offset >= echomsg->len + 1 ? 0 :
	    echomsg->len + 1 - uio->uio_offset);

	if ((error = uiomove(echomsg->msg, amt, uio)) != 0)
		uprintf("uiomove failed!\n");

	return (error);
}

/*
 * echo_write takes in a character string and saves it
 * to buf for later accessing.
 */
static int
echo_write(struct cdev *dev __unused, struct uio *uio, int ioflag __unused)
{
	size_t amt;
	int error;

	/*
	 * We either write from the beginning or are appending -- do
	 * not allow random access.
	 */
	if (uio->uio_offset != 0 && (uio->uio_offset != echomsg->len))
		return (EINVAL);

	/* This is a new message, reset length */
	if (uio->uio_offset == 0)
		echomsg->len = 0;

	/* Copy the string in from user memory to kernel memory */
	amt = MIN(uio->uio_resid, (BUFFERSIZE - echomsg->len));

	error = uiomove(echomsg->msg + uio->uio_offset, amt, uio);

	/* Now we need to null terminate and record the length */
	echomsg->len = uio->uio_offset;
	echomsg->msg[echomsg->len] = 0;

	if (error != 0)
		uprintf("Write failed: bad address!\n");
	return (error);
}

DEV_MODULE(echo, echo_loader, NULL);
```

有了这个驱动程序加载，请尝试：

```
# echo -n "Test Data" > /dev/echo
# cat /dev/echo
Opened device "echo" successfully.
Test Data
Closing device "echo".
```

真实的硬件设备在下一章中描述。

## 9.4. 块设备（已消失）

其他 UNIX®系统可能支持第二种称为块设备的磁盘设备。块设备是指内核提供缓存的磁盘设备。这种缓存使块设备几乎无法使用，或者至少是非常不可靠的。缓存会重新排序写操作的顺序，使应用程序无法在任一特定时刻知道确切的磁盘内容。

这导致无法预测和可靠地恢复磁盘数据结构（文件系统、数据库等）。由于写入可能会延迟，内核无法向应用程序报告哪个特定的写操作遇到写错误，这进一步加剧了一致性问题。

出于这个原因，没有严肃的应用程序依赖块设备，事实上，几乎所有直接访问磁盘的应用程序都非常努力地指定应始终使用字符（或“原始”）设备。由于将每个磁盘（分区）的别名实现为具有不同语义的两个设备会显著复杂化相关的内核代码，FreeBSD 在现代化磁盘 I/O 基础架构的一部分中放弃了对缓存磁盘设备的支持。

## 9.5. 网络驱动程序

与设备节点不同，网络设备的驱动程序不使用设备节点来进行访问。它们的选择是基于内核内部做出的其他决策，而不是调用 open()，使用网络设备通常是通过使用系统调用 socket(2)来引入的。

更多信息请参见 ifnet(9)，回环设备的源代码以及 Bill Paul 的网络驱动程序。
