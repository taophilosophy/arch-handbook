# 第 11 章 PCI设备

本章将讨论有关在 PCI 总线上为设备编写设备驱动程序的 FreeBSD 机制。

## 11.1. 探测和附加

这里的信息是关于 PCI 总线代码如何迭代未连接的设备，并查看新加载的 kld 是否会附加到它们中的任何一个。

### 11.1.1. 样例驱动程序源码（mypci.c）

```
/*
 * Simple KLD to play with the PCI functions.
 *
 * Murray Stokely
 */

#include <sys/param.h>		/* defines used in kernel.h */
#include <sys/module.h>
#include <sys/systm.h>
#include <sys/errno.h>
#include <sys/kernel.h>		/* types used in module initialization */
#include <sys/conf.h>		/* cdevsw struct */
#include <sys/uio.h>		/* uio struct */
#include <sys/malloc.h>
#include <sys/bus.h>		/* structs, prototypes for pci bus stuff and DEVMETHOD macros! */

#include <machine/bus.h>
#include <sys/rman.h>
#include <machine/resource.h>

#include <dev/pci/pcivar.h>	/* For pci_get macros! */
#include <dev/pci/pcireg.h>

/* The softc holds our per-instance data. */
struct mypci_softc {
	device_t	my_dev;
	struct cdev	*my_cdev;
};

/* Function prototypes */
static d_open_t		mypci_open;
static d_close_t	mypci_close;
static d_read_t		mypci_read;
static d_write_t	mypci_write;

/* Character device entry points */

static struct cdevsw mypci_cdevsw = {
	.d_version =	D_VERSION,
	.d_open =	mypci_open,
	.d_close =	mypci_close,
	.d_read =	mypci_read,
	.d_write =	mypci_write,
	.d_name =	"mypci",
};

/*
 * In the cdevsw routines, we find our softc by using the si_drv1 member
 * of struct cdev.  We set this variable to point to our softc in our
 * attach routine when we create the /dev entry.
 */

int
mypci_open(struct cdev *dev, int oflags, int devtype, struct thread *td)
{
	struct mypci_softc *sc;

	/* Look up our softc. */
	sc = dev->si_drv1;
	device_printf(sc->my_dev, "Opened successfully.\n");
	return (0);
}

int
mypci_close(struct cdev *dev, int fflag, int devtype, struct thread *td)
{
	struct mypci_softc *sc;

	/* Look up our softc. */
	sc = dev->si_drv1;
	device_printf(sc->my_dev, "Closed.\n");
	return (0);
}

int
mypci_read(struct cdev *dev, struct uio *uio, int ioflag)
{
	struct mypci_softc *sc;

	/* Look up our softc. */
	sc = dev->si_drv1;
	device_printf(sc->my_dev, "Asked to read %zd bytes.\n", uio->uio_resid);
	return (0);
}

int
mypci_write(struct cdev *dev, struct uio *uio, int ioflag)
{
	struct mypci_softc *sc;

	/* Look up our softc. */
	sc = dev->si_drv1;
	device_printf(sc->my_dev, "Asked to write %zd bytes.\n", uio->uio_resid);
	return (0);
}

/* PCI Support Functions */

/*
 * Compare the device ID of this device against the IDs that this driver
 * supports.  If there is a match, set the description and return success.
 */
static int
mypci_probe(device_t dev)
{

	device_printf(dev, "MyPCI Probe\nVendor ID : 0x%x\nDevice ID : 0x%x\n",
	    pci_get_vendor(dev), pci_get_device(dev));

	if (pci_get_vendor(dev) == 0x11c1) {
		printf("We've got the Winmodem, probe successful!\n");
		device_set_desc(dev, "WinModem");
		return (BUS_PROBE_DEFAULT);
	}
	return (ENXIO);
}

/* Attach function is only called if the probe is successful. */

static int
mypci_attach(device_t dev)
{
	struct mypci_softc *sc;

	printf("MyPCI Attach for : deviceID : 0x%x\n", pci_get_devid(dev));

	/* Look up our softc and initialize its fields. */
	sc = device_get_softc(dev);
	sc->my_dev = dev;

	/*
	 * Create a /dev entry for this device.  The kernel will assign us
	 * a major number automatically.  We use the unit number of this
	 * device as the minor number and name the character device
	 * "mypci<unit>".
	 */
	sc->my_cdev = make_dev(&mypci_cdevsw, device_get_unit(dev),
	    UID_ROOT, GID_WHEEL, 0600, "mypci%u", device_get_unit(dev));
	sc->my_cdev->si_drv1 = sc;
	printf("Mypci device loaded.\n");
	return (0);
}

/* Detach device. */

static int
mypci_detach(device_t dev)
{
	struct mypci_softc *sc;

	/* Teardown the state in our softc created in our attach routine. */
	sc = device_get_softc(dev);
	destroy_dev(sc->my_cdev);
	printf("Mypci detach!\n");
	return (0);
}

/* Called during system shutdown after sync. */

static int
mypci_shutdown(device_t dev)
{

	printf("Mypci shutdown!\n");
	return (0);
}

/*
 * Device suspend routine.
 */
static int
mypci_suspend(device_t dev)
{

	printf("Mypci suspend!\n");
	return (0);
}

/*
 * Device resume routine.
 */
static int
mypci_resume(device_t dev)
{

	printf("Mypci resume!\n");
	return (0);
}

static device_method_t mypci_methods[] = {
	/* Device interface */
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

### 11.1.2. 样例驱动程序的 Makefile

```
# Makefile for mypci driver

KMOD=	mypci
SRCS=	mypci.c
SRCS+=	device_if.h bus_if.h pci_if.h

.include <bsd.kmod.mk>
```

如果你将上述源文件和 Makefile 放入一个目录，你可以运行 make 来编译样例驱动程序。此外，你可以运行 make load 将驱动程序加载到当前运行的内核中，运行 make unload 来在加载后卸载驱动程序。

### 11.1.3. 其他资源

* [PCI 特别兴趣小组](http://www.pcisig.org/)
* PCI 系统架构，第四版，Tom Shanley 等著

## 11.2. 公共汽车资源

FreeBSD 提供了一种从父总线请求资源的面向对象机制。 几乎所有设备都将是某种总线（PCI、ISA、USB、SCSI 等）的子成员，这些设备需要从其父总线获取资源（如内存段、中断线或 DMA 通道）。

### 11.2.1. 基址寄存器

要对 PCI 设备执行任何特别有用的操作，你需要从 PCI 配置空间获取基址寄存器（BARs）。获取 BAR 的 PCI 特定细节在 bus_alloc_resource() 函数中进行了抽象处理。

例如，典型的驱动程序可能在 attach() 函数中有类似以下内容：

```
    sc->bar0id = PCIR_BAR(0);
    sc->bar0res = bus_alloc_resource(dev, SYS_RES_MEMORY, &sc->bar0id,
				  0, ~0, 1, RF_ACTIVE);
    if (sc->bar0res == NULL) {
        printf("Memory allocation of PCI base register 0 failed!\n");
        error = ENXIO;
        goto fail1;
    }

    sc->bar1id = PCIR_BAR(1);
    sc->bar1res = bus_alloc_resource(dev, SYS_RES_MEMORY, &sc->bar1id,
				  0, ~0, 1, RF_ACTIVE);
    if (sc->bar1res == NULL) {
        printf("Memory allocation of PCI base register 1 failed!\n");
        error =  ENXIO;
        goto fail2;
    }
    sc->bar0_bt = rman_get_bustag(sc->bar0res);
    sc->bar0_bh = rman_get_bushandle(sc->bar0res);
    sc->bar1_bt = rman_get_bustag(sc->bar1res);
    sc->bar1_bh = rman_get_bushandle(sc->bar1res);
```

每个基址寄存器的句柄都保存在 softc 结构中，以便稍后用于向设备写入。

这些句柄可以用于使用 bus_space_* 函数读取或写入设备寄存器。例如，驱动程序可能包含一个快捷函数来从特定于板的寄存器读取，像这样：

```
uint16_t
board_read(struct ni_softc *sc, uint16_t address)
{
    return bus_space_read_2(sc->bar1_bt, sc->bar1_bh, address);
}
```

类似地，可以用以下方式写入寄存器：

```
void
board_write(struct ni_softc *sc, uint16_t address, uint16_t value)
{
    bus_space_write_2(sc->bar1_bt, sc->bar1_bh, address, value);
}
```

这些功能存在 8 位、16 位和 32 位版本，你应相应地使用 bus_space_{read|write}_{1|2|4} 。

```
uint16_t
board_read(struct ni_softc *sc, uint16_t address)
{
	return (bus_read(sc->bar1res, address));
}
```

### 11.2.2. 中断

中断是从面向对象的总线代码中分配的，类似于内存资源。首先，必须从父总线中分配一个 IRQ 资源，然后必须设置中断处理程序来处理此 IRQ。

同样，来自设备 attach() 功能的示例胜过言辞。

```
/* Get the IRQ resource */

    sc->irqid = 0x0;
    sc->irqres = bus_alloc_resource(dev, SYS_RES_IRQ, &(sc->irqid),
				  0, ~0, 1, RF_SHAREABLE | RF_ACTIVE);
    if (sc->irqres == NULL) {
	printf("IRQ allocation failed!\n");
	error = ENXIO;
	goto fail3;
    }

    /* Now we should set up the interrupt handler */

    error = bus_setup_intr(dev, sc->irqres, INTR_TYPE_MISC,
			   my_handler, sc, &(sc->handler));
    if (error) {
	printf("Couldn't set up irq\n");
	goto fail4;
    }
```

驱动程序的分离例程中必须小心处理。你必须使设备的中断流静止，并移除中断处理程序。一旦 bus_teardown_intr() 返回，你就知道你的中断处理程序将不再被调用，并且可能正在执行此中断处理程序的所有线程都已返回。由于此函数可能会休眠，因此在调用此函数时不得持有任何互斥锁。

### 11.2.3. DMA

此部分已过时，仅出于历史原因而存在。处理这些问题的正确方法是改用 bus_space_dma*() 函数。当此部分更新以反映该用法时，可以删除本段。但是，目前 API 处于一种不稳定状态，因此一旦稳定下来，最好更新此部分以反映该状态。

在 PC 上，希望进行总线主控 DMA 的外围设备必须处理物理地址。这是一个问题，因为 FreeBSD 使用虚拟内存并几乎完全处理虚拟地址。幸运的是，有一个名为 vtophys() 的函数来帮助。

```
#include <vm/vm.h>
#include <vm/pmap.h>

#define vtophys(virtual_address) (...)
```

然而，在 alpha 上，解决方案有点不同，我们真正想要的是一个名为 vtobus() 的函数。

```
#if defined(__alpha__)
#define vtobus(va)      alpha_XXX_dmamap((vm_offset_t)va)
#else
#define vtobus(va)      vtophys(va)
#endif
```

### 11.2.4. 释放资源

在 attach() 期间分配的所有资源进行释放非常重要。必须小心释放正确的资源，即使在失败的情况下，以便系统在驱动程序停止运行时仍然可用。
