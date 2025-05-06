# 第 14 章 Newbus


特别感谢 Matthew N. Dodd, Warner Losh, Bill Paul, Doug Rabson, Mike Smith, Peter Wemm 和 Scott Long。

本章详细解释了 Newbus 设备框架。

## 14.1. 设备驱动程序

### 14.1.1. 设备驱动程序的目的

设备驱动程序是一种软件组件，提供内核对外围设备（例如磁盘、网络适配器）的通用视图与外设的实际实现之间的接口。设备驱动程序接口（DDI）是内核与设备驱动程序组件之间定义的接口。

### 14.1.2. 设备驱动程序的类型

在 UNIX® 以及 FreeBSD 中，曾经定义过四种类型的设备：

* 块设备驱动程序
* 字符设备驱动程序
* 网络设备驱动程序
* 伪设备驱动程序

块设备以固定大小的数据块的方式执行。这种类型的驱动程序依赖于所谓的缓冲区缓存，它在内存的一个专用部分中缓存访问的数据块。通常，这种缓冲区缓存基于后台写操作，这意味着当数据在内存中被修改时，它会在系统进行定期磁盘刷新时将数据同步到磁盘，从而优化写入操作。

### 14.1.3. 字符设备

然而，在 FreeBSD 4.0 及以后的版本中，块设备和字符设备之间的区别变得不存在。

## 14.2. Newbus 概述

Newbus 是基于抽象层的新总线架构的实现，在 FreeBSD 3.0 中引入了 Alpha port后才开始使用。直到 4.0 版本之前，它才成为设备驱动程序的默认系统。其目标是提供一种更面向对象的方式，用于连接主机系统提供给操作系统的各种总线和设备。

其主要特点包括但不限于：

* 动态附加
* 驱动程序的简易模块化
* 伪总线

最显著的变化之一是从平面和特定系统迁移到设备树布局。

在顶层是“根”设备，它是所有其他设备的父设备。对于每种体系结构，通常有一个名为“root”的子设备，附有诸如主机到 PCI 桥等内容。对于 x86，这个“root”设备是“nexus”设备。对于 Alpha，Alpha 的各种不同型号都有不同的顶层设备，对应不同的硬件芯片组，包括 lca、apecs、cia 和 tsunami。

在 Newbus 上下文中，设备表示系统中的单个硬件实体。例如，每个 PCI 设备都由一个 Newbus 设备表示。系统中的任何设备都可以有子设备；具有子设备的设备通常称为“总线” 。系统中常见总线的示例包括 ISA 和 PCI，它们分别管理连接到 ISA 和 PCI 总线的设备列表。

通常，不同类型总线之间的连接由“桥接”设备表示，通常具有一个连接到附加总线的子设备。一个例子是 PCI-to-PCI 桥接器，它在父 PCI 总线上由一个名称为 pcibN 的设备表示，并在附加总线上有一个名为 pciN 的子设备。这种布局简化了 PCI 总线树的实现，允许对顶层总线和桥接总线使用共同的代码。

新总线架构中的每个设备都要求其父级映射其资源。然后父级继续向其父级请求，直到达到中枢。所以，基本上，中枢是新总线系统中唯一知道所有资源的部分。

|  | ISA 设备可能希望在其 IO port处于 0x230 ，因此它请求其父级，即 ISA 总线。ISA 总线将其交给 PCI 到 ISA 桥接器，后者再请求 PCI 总线，达到主机到 PCI 桥接器，最终到达中枢。向上过渡的美妙之处在于有机会对请求进行翻译。例如， 0x230 IO port请求在 MIPS 机顶盒上可能变为内存映射到 PCI 桥接器的 0xb0000230 。 |
| -- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

资源分配可以在设备树中的任何地方控制。例如，在许多 Alpha 平台上，ISA 中断与 PCI 中断分别管理，并且 ISA 中断的资源分配由 Alpha 的 ISA 总线设备管理。在 IA-32 上，ISA 和 PCI 中断都由顶层中枢设备管理。对于 ports，内存和 port 地址空间由单个实体管理 - IA-32 上的中枢和 Alpha 上的相关芯片组驱动程序（例如，CIA 或海啸）。

为了规范对内存和port映射资源的访问，Newbus 集成了来自 NetBSD 的 bus_space API。这些 API 提供了一个单一的 API，用于替换 inb/outb 和直接内存读取/写入。这样做的好处是，一个驱动程序可以轻松地使用内存映射寄存器或port映射寄存器（一些硬件同时支持两者）。

此支持已集成到资源分配机制中。当分配资源时，驱动程序可以从资源中检索相关的 bus_space_tag_t 和 bus_space_handle_t 。

Newbus 还允许在专门用于此目的的文件中定义接口方法。这些文件是在 src/sys 层级结构下找到的.m 文件。

新总线系统的核心是可扩展的“基于对象的编程”模型。系统中的每个设备都有一个支持的方法表。系统和其他设备使用这些方法来控制设备并请求服务。设备支持的不同方法由多个“接口”定义。一个“接口”只是由设备实现的一组相关方法。

在新总线系统中，设备的方法由系统中的各种设备驱动程序提供。当设备在自动配置期间连接到驱动程序时，它使用驱动程序声明的方法表。设备可以稍后与其驱动程序分离，并重新连接到具有新方法表的新驱动程序。这允许动态替换驱动程序，这对驱动程序开发很有用。

接口由类似于用于定义文件系统的 vnode 操作的语言描述的接口定义语言描述。接口将存储在一个方法文件中（通常命名为 foo_if.m）。

示例 1. Newbus 方法

```
      # Foo subsystem/driver (a comment...)

	  INTERFACE foo

	METHOD int doit {
		device_t dev;
	};

	# DEFAULT is the method that will be used, if a method was not
	# provided via: DEVMETHOD()

	METHOD void doit_to_child {
		device_t dev;
		driver_t child;
	} DEFAULT doit_generic_to_child;
```

当编译此接口时，它会生成一个名为 "foo_if.h" 的头文件，其中包含函数声明：

```
      int FOO_DOIT(device_t dev);
      int FOO_DOIT_TO_CHILD(device_t dev, device_t child);
```

还会创建一个源文件 "foo_if.c"，与自动生成的头文件配套；它包含这些函数的实现，这些函数查找对象方法表中相关函数的位置并调用该函数。

系统定义了两个主要接口。第一个基本接口称为 "设备" ，包含所有设备相关的方法。"设备" 接口中的方法包括 "探测"，"连接" 和 "断开" 用于控制硬件检测，以及 "关机"，"暂停" 和 "恢复" 用于关键事件通知。

第二个更复杂的接口是 "总线" 。该接口包含适用于具有子设备的设备的方法，包括用于访问特定于总线的每个设备信息^[1]^，事件通知（ child_detached ， driver_added ）和资源管理（ alloc_resource ， activate_resource ， deactivate_resource ， release_resource ）。

"总线" 接口中的许多方法为总线设备的某个子设备提供服务。这些方法通常使用前两个参数来指定提供服务的总线和请求服务的子设备。为了简化驱动程序代码，这些方法中的许多都有访问器函数，这些函数查找父级并在父级上调用方法。例如，方法 BUS_TEARDOWN_INTR(device_t dev, device_t child, …) 可以使用函数 bus_teardown_intr(device_t child, …) 调用。

系统中的一些总线类型定义了附加接口，以提供对总线特定功能的访问。例如，PCI 总线驱动程序定义了具有两种方法 read_config 和 write_config 用于访问 PCI 设备配置寄存器的 "pci" 接口。

## 14.3. Newbus API

由于 Newbus API 非常庞大，本节尝试对其进行一些文档化。更多信息将在本文档的下一个修订版中提供。

### 源层次结构中的重要位置

src/sys/[arch]/[arch] - 特定机器架构的内核代码存放在这个目录中。例如， i386 架构，或者 SPARC64 架构。

src/sys/dev/[bus] - 特定 [bus] 的设备支持存放在这个目录中。

该目录包含 PCI 总线支持代码。

PCI/ISA 设备驱动程序位于此目录中。在 FreeBSD 版本 4.0 中，PCI/ISA 总线支持代码曾经存在于此目录中。

### 14.3.2. 重要结构和类型定义

devclass_t - 这是指向 struct devclass 的指针类型定义。

device_method_t - 这与 kobj_method_t 相同（请参阅 src/sys/kobj.h）。

device_t - 这是指向 struct device 的指针类型定义。 device_t 代表系统中的设备。这是一个内核对象。有关实现细节，请参阅 src/sys/sys/bus_private.h。

driver_t - 这是引用 struct driver 的类型定义。 driver 结构体是 device 内核对象的一类；它还保存了驱动程序私有的数据。

***driver_t*** **Implementation**

```
	  struct driver {
		KOBJ_CLASS_FIELDS;
		void	*priv;			/* driver private data */
	  };
```

一个 device_state_t 类型，它是一个枚举， device_state 。它包含 Newbus 设备在自动配置过程之前和之后的可能状态。

**设备状态 _device_state_t**

```
	  /*
	   * src/sys/sys/bus.h
	   */
	  typedef enum device_state {
		DS_NOTPRESENT,	/* not probed or probe failed */
		DS_ALIVE,		/* probe succeeded */
		DS_ATTACHED,	/* attach method called */
		DS_BUSY			/* device is open */
	  } device_state_t;
```

---

1. bus_generic_read_ivar(9) 和 bus_generic_write_ivar(9)
