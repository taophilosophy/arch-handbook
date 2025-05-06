# 第 11 章 PCI 设备

本章将介绍 FreeBSD 在 PCI 总线上编写设备驱动程序的机制。

## 11.1. 探测与附加

这里介绍了 PCI 总线代码如何遍历未附加的设备，并查看新加载的 kld 是否会附加到它们中的任何一个。

### 11.1.1. 示例驱动程序源代码 (**mypci.c**)

```c
/*
 * 简单的 KLD，用于测试 PCI 函数。
 *
 * Murray Stokely
 */

#include <sys/param.h>		/* kernel.h 中使用的定义 */
#include <sys/module.h>
#include <sys/systm.h>
#include <sys/errno.h>
#include <sys/kernel.h>		/* 模块初始化时使用的类型 */
#include <sys/conf.h>		/* cdevsw 结构 */
#include <sys/uio.h>		/* uio 结构 */
#include <sys/malloc.h>
#include <sys/bus.h>		/* pci 总线相关的结构、原型和 DEVMETHOD 宏 */

#include <machine/bus.h>
#include <sys/rman.h>
#include <machine/resource.h>

#include <dev/pci/pcivar.h>	/* 用于 pci_get 宏！ */
#include <dev/pci/pcireg.h>

/* softc 保存每个实例的数据。 */
struct mypci_softc {
	device_t	my_dev;
	struct cdev	*my_cdev;
};

/* 函数原型 */
static d_open_t		mypci_open;
static d_close_t	mypci_close;
static d_read_t		mypci_read;
static d_write_t	mypci_write;

/* 字符设备入口点 */

static struct cdevsw mypci_cdevsw = {
	.d_version =	D_VERSION,
	.d_open =	mypci_open,
	.d_close =	mypci_close,
	.d_read =	mypci_read,
	.d_write =	mypci_write,
	.d_name =	"mypci",
};

/*
 * 在 cdevsw 例程中，我们通过使用 struct cdev 的 si_drv1 成员来查找我们的 softc。
 * 我们在附加例程中创建 /dev 条目时将此变量设置为指向我们的 softc。
 */

int
mypci_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
	struct mypci_softc *sc;

	/* 查找我们的 softc。 */
	sc = dev->si_drv1;
	device_printf(sc->my_dev, "打开成功。\n");
	return (0);
}

int
mypci_close(struct cdev *dev, int fflag, int devtype, struct thread *td)
{
	struct mypci_softc *sc;

	/* 查找我们的 softc。 */
	sc = dev->si_drv1;
	device_printf(sc->my_dev, "已关闭。\n");
	return (0);
}

int
mypci_read(struct cdev *dev, struct uio *uio, int ioflag)
{
	struct mypci_softc *sc;

	/* 查找我们的 softc。 */
	sc = dev->si_drv1;
	device_printf(sc->my_dev, "请求读取 %zd 字节。\n", uio->uio_resid);
	return (0);
}

int
mypci_write(struct cdev *dev, struct uio *uio, int ioflag)
{
	struct mypci_softc *sc;

	/* 查找我们的 softc。 */
	sc = dev->si_drv1;
	device_printf(sc->my_dev, "请求写入 %zd 字节。\n", uio->uio_resid);
	return (0);
}

/* PCI 支持函数 */

/*
 * 将此设备的设备 ID 与此驱动程序支持的 ID 进行比较。
 * 如果匹配，设置描述并返回成功。
 */
static int
mypci_probe(device_t dev)
{

	device_printf(dev, "MyPCI 探测\n供应商 ID : 0x%x\n设备 ID : 0x%x\n",
	    pci_get_vendor(dev), pci_get_device(dev));

	if (pci_get_vendor(dev) == 0x11c1) {
		printf("我们得到了 Winmodem，探测成功！\n");
		device_set_desc(dev, "WinModem");
		return (BUS_PROBE_DEFAULT);
	}
	return (ENXIO);
}

/* 只有在探测成功时才会调用附加函数。 */

static int
mypci_attach(device_t dev)
{
	struct mypci_softc *sc;

	printf("MyPCI 附加：设备 ID : 0x%x\n", pci_get_devid(dev));

	/* 查找我们的 softc 并初始化其字段。 */
	sc = device_get_softc(dev);
	sc->my_dev = dev;

	/*
	 * 为此设备创建 /dev 条目。内核会自动分配给我们一个主设备号。
	 * 我们使用此设备的单元号作为次设备号，并将字符设备命名为 "mypci<unit>"。
	 */
	sc->my_cdev = make_dev(&mypci_cdevsw, device_get_unit(dev),
	    UID_ROOT, GID_WHEEL, 0600, "mypci%u", device_get_unit(dev));
	sc->my_cdev->si_drv1 = sc;
	printf("Mypci 设备加载完毕。\n");
	return (0);
}

/* 卸载设备。 */

static int
mypci_detach(device_t dev)
{
	struct mypci_softc *sc;

	/* 清理我们在附加例程中创建的 softc 状态。 */
	sc = device_get_softc(dev);
	destroy_dev(sc->my_cdev);
	printf("Mypci 卸载！\n");
	return (0);
}

/* 系统关闭时调用，先同步后执行。 */

static int
mypci_shutdown(device_t dev)
{

	printf("Mypci 关闭！\n");
	return (0);
}

/*
 * 设备挂起例程。
 */
static int
mypci_suspend(device_t dev)
{

	printf("Mypci 挂起！\n");
	return (0);
}

/*
 * 设备恢复例程。
 */
static int
mypci_resume(device_t dev)
{

	printf("Mypci 恢复！\n");
	return (0);
}

static device_method_t mypci_methods[] = {
	/* 设备接口 */
	DEVMETHOD(device_probe,		mypci_probe),
	DEVMETHOD(device_attach,	mypci_attach),
	DEVMETHOD(device_detach,	mypci_detach),
	DEVMETHOD(device_shutdown,	mypci_shutdown),
	DEVMETHOD(device_suspend,	mypci_suspend),
	DEVMETHOD(device_resume,	mypci_resume),

	DEVMETHOD_END
};

static devclass_t mypci_devclass;

DEFINE_CLASS_0(mypci, mypci_driver, mypci_methods, sizeof(struct mypci_softc));
DRIVER_MODULE(mypci, pci, mypci_driver, mypci_devclass, 0, 0);
```

### 11.1.2. 示例驱动程序的 **Makefile**

```c
# mypci 驱动程序的 Makefile

KMOD=	mypci
SRCS=	mypci.c
SRCS+=	device_if.h bus_if.h pci_if.h

.include <bsd.kmod.mk>
```

如果将上述源文件和 **Makefile** 放入一个目录中，你可以运行 `make` 来编译示例驱动程序。还可以运行 `make load` 将驱动程序加载到当前正在运行的内核中，运行 `make unload` 在加载后卸载该驱动程序。

### 11.1.3. 其他资源

* [PCI 特别兴趣小组](http://www.pcisig.org/)
* 《PCI 系统架构》第四版，Tom Shanley 等著

## 11.2. 总线资源

FreeBSD 提供了一种面向对象的机制，用于从父总线请求资源。几乎所有设备都是某种总线（PCI、ISA、USB、SCSI 等）的子设备，这些设备需要从其父总线获取资源（如内存段、中断线或 DMA 通道）。

### 11.2.1. 基地址寄存器

要对 PCI 设备执行任何特别有用的操作，你需要从 PCI 配置空间获取 *基地址寄存器*（BAR）。获取 BAR 的 PCI 特定细节在 `bus_alloc_resource()` 函数中被抽象化。

例如，一个典型的驱动程序可能在 `attach()` 函数中包含如下代码：

```c
sc->bar0id = PCIR_BAR(0);
    sc->bar0res = bus_alloc_resource(dev, SYS_RES_MEMORY, &sc->bar0id,
				  0, ~0, 1, RF_ACTIVE);
    if (sc->bar0res == NULL) {
        printf("PCI 基地址寄存器 0 的内存分配失败！\n");
        error = ENXIO;
        goto fail1;
    }

    sc->bar1id = PCIR_BAR(1);
    sc->bar1res = bus_alloc_resource(dev, SYS_RES_MEMORY, &sc->bar1id,
				  0, ~0, 1, RF_ACTIVE);
    if (sc->bar1res == NULL) {
        printf("PCI 基地址寄存器 1 的内存分配失败！\n");
        error =  ENXIO;
        goto fail2;
    }
    sc->bar0_bt = rman_get_bustag(sc->bar0res);
    sc->bar0_bh = rman_get_bushandle(sc->bar0res);
    sc->bar1_bt = rman_get_bustag(sc->bar1res);
    sc->bar1_bh = rman_get_bushandle(sc->bar1res);
```

每个基地址寄存器的句柄保存在 `softc` 结构中，以便稍后用于写入设备。

这些句柄可以用来通过 `bus_space_*` 函数从设备寄存器读取或写入。例如，驱动程序可能包含一个简化函数，用于从特定寄存器读取数据，如下所示：

```c
uint16_t
board_read(struct ni_softc *sc, uint16_t address)
{
    return bus_space_read_2(sc->bar1_bt, sc->bar1_bh, address);
}
```

类似地，可以使用以下代码向寄存器写入数据：

```c
void
board_write(struct ni_softc *sc, uint16_t address, uint16_t value)
{
    bus_space_write_2(sc->bar1_bt, sc->bar1_bh, address, value);
}
```

这些函数有 8 位、16 位和 32 位版本，你应根据需要使用 `bus_space_{read|write}_{1|2|4}`。

>**注意**
>
>在 FreeBSD 7.0 及更高版本中，你可以使用 `bus_*` 函数来代替 `bus_space_*` 函数。`bus_*` 函数使用 `struct resource *` 指针，而不是总线标签和总线句柄。因此，你可以删除 `softc` 结构中的总线标签和总线句柄成员，并将 `board_read()` 函数重写为：
>
>```sh
>uint16_t
>board_read(struct ni_softc *sc, uint16_t address)
>{
>	return (bus_read(sc->bar1res, address));
>}
>```

### 11.2.2. 中断

中断从面向对象的总线代码中以类似于内存资源的方式分配。首先，必须从父总线分配一个 IRQ 资源，然后必须设置中断处理程序来处理这个 IRQ。

以下是来自设备 `attach()` 函数的一个示例，它比文字更能说明问题。

```c
/* 获取 IRQ 资源 */

    sc->irqid = 0x0;
    sc->irqres = bus_alloc_resource(dev, SYS_RES_IRQ, &(sc->irqid),
				  0, ~0, 1, RF_SHAREABLE | RF_ACTIVE);
    if (sc->irqres == NULL) {
	printf("IRQ 分配失败！\n");
	error = ENXIO;
	goto fail3;
    }

    /* 现在我们应该设置中断处理程序 */

    error = bus_setup_intr(dev, sc->irqres, INTR_TYPE_MISC,
			   my_handler, sc, &(sc->handler));
    if (error) {
	printf("无法设置 irq\n");
	goto fail4;
    }
```

在驱动程序的 `detach` 例程中必须特别小心。你必须使设备的中断流保持安静，并移除中断处理程序。一旦 `bus_teardown_intr()` 返回，你就可以确定中断处理程序将不再被调用，且所有可能正在执行此中断处理程序的线程都已返回。由于此函数可以休眠，因此在调用此函数时，你不能持有任何互斥锁。

### 11.2.3. DMA

此部分已过时，仅供历史参考。正确的方法是使用 `bus_space_dma*()` 函数来处理这些问题。该段内容可以在更新为反映这些用法时删除。然而，目前，API 仍在变化中，因此一旦稳定下来，最好更新此部分以反映相关用法。

在 PC 上，想要进行总线主控 DMA 的外设必须处理物理地址。这个问题在于 FreeBSD 使用虚拟内存并几乎完全处理虚拟地址。幸运的是，存在一个名为 `vtophys()` 的函数来帮助解决这个问题。

```c
#include <vm/vm.h>
#include <vm/pmap.h>

#define vtophys(virtual_address) (...)
```

然而，在 Alpha 架构上，解决方案有所不同，我们真正需要的是一个名为 `vtobus()` 的函数。

```c
#if defined(__alpha__)
#define vtobus(va)      alpha_XXX_dmamap((vm_offset_t)va)
#else
#define vtobus(va)      vtophys(va)
#endif
```

### 11.2.4. 资源的释放

在 `attach()` 过程中分配的所有资源必须在驱动程序卸载时释放。即使在发生失败条件时，也必须小心地释放正确的资源，以确保系统在驱动程序退出时仍然可用。
