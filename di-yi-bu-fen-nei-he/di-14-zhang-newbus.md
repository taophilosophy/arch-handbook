# 第 14 章 Newbus

*特别感谢 Matthew N. Dodd、Warner Losh、Bill Paul、Doug Rabson、Mike Smith、Peter Wemm 和 Scott Long*。

本章详细解释了 Newbus 设备框架。

## 14.1. 设备驱动程序

### 14.1.1. 设备驱动程序的目的

设备驱动程序是一个软件组件，它提供了内核的外围设备通用视图（例如，磁盘、网络适配器）与外围设备实际实现之间的接口。\*设备驱动程序接口（DDI）\*是内核与设备驱动程序组件之间定义的接口。

### 14.1.2. 设备驱动程序的类型

在 UNIX® 和 FreeBSD 的早期，有四种定义的设备类型：

* 块设备驱动程序
* 字符设备驱动程序
* 网络设备驱动程序
* 虚拟设备驱动程序

*块设备*以使用固定大小的数据块的方式进行操作。这种类型的驱动程序依赖于所谓的 *缓冲区缓存*，它将访问过的数据块缓存在内存的专用部分中。通常，这个缓冲区缓存是基于写入延迟的，这意味着当数据在内存中被修改时，它会在系统进行定期磁盘刷新时同步到磁盘，从而优化写操作。

### 14.1.3. 字符设备

然而，从 FreeBSD 4.0 版本开始，块设备和字符设备之间的区别不再存在。

## 14.2. Newbus 概述

*Newbus* 是一种基于抽象层的新总线架构实现，首次出现在 FreeBSD 3.0 中，当时 Alpha 端口被引入到源代码树中。直到 4.0 版本，它才成为默认的设备驱动系统。其目标是为操作系统提供一种更面向对象的方式，用于连接主机系统提供的各种总线和设备。

它的主要特点包括：

* 动态附加
* 驱动程序的模块化简化
* 虚拟总线

其中一个最显著的变化是从扁平化和临时系统迁移到设备树布局。

在顶层是 *“root”* 设备，它是所有其他设备的父设备。对于每种架构，通常有一个“root”的子设备，它连接诸如 *host-to-PCI 桥* 等设备。对于 x86 架构，这个“root”设备是 *“nexus”* 设备。对于 Alpha 架构，不同型号的 Alpha 对应着不同的顶层设备，这些设备对应不同的硬件芯片组，包括 *lca*、*apecs*、*cia* 和 *tsunami* 等。

在 Newbus 的上下文中，设备表示系统中的单个硬件实体。例如，每个 PCI 设备由一个 Newbus 设备表示。系统中的任何设备都可以有子设备；具有子设备的设备通常被称为 *“总线”*。系统中常见的总线有 ISA 和 PCI，它们分别管理附加到 ISA 和 PCI 总线的设备列表。

不同种类的总线之间的连接通常由 *“桥接”* 设备表示，该设备通常具有一个附加总线的子设备。例如，*PCI-to-PCI 桥* 在父 PCI 总线上表示为设备 *pcibN*，并为附加的总线提供一个子设备 *pciN*。这种布局简化了 PCI 总线树的实现，使得可以使用通用代码处理顶层总线和桥接总线。

在 Newbus 架构中，每个设备都会请求其父设备来映射其资源。父设备会继续请求自己的父设备，直到达到 nexus。所以，基本上，nexus 是 Newbus 系统中唯一了解所有资源的部分。

>**技巧**
>
>一个 ISA 设备可能想要映射其 IO 端口 `0x230`，它会请求其父设备，即 ISA 总线。ISA 总线将请求交给 PCI-to-ISA 桥接设备，后者再请求 PCI 总线，最终达到主机到 PCI 的桥接设备，最后到达 nexus。这个向上的过渡的优点是，可以在此过程中翻译请求。例如，在 MIPS 系统上，`0x230` IO 端口请求可能会通过 PCI 桥接转换为内存映射地址 `0xb0000230`。

资源分配可以在设备树的任何位置进行控制。例如，在许多 Alpha 平台上，ISA 中断与 PCI 中断是分别管理的，ISA 中断的资源分配由 Alpha 的 ISA 总线设备管理。而在 IA-32 平台上，ISA 和 PCI 中断都由顶级的 nexus 设备管理。对于端口、内存和端口地址空间，IA-32 的资源由 nexus 管理，而 Alpha 上则由相关的芯片组驱动程序管理（例如，CIA 或 tsunami）。

为了规范化对内存和端口映射资源的访问，Newbus 集成了来自 NetBSD 的 `bus_space` API。这些 API 提供了一个单一接口，取代了 inb/outb 和直接的内存读写操作。其优点是，单个驱动程序可以轻松地使用内存映射寄存器或端口映射寄存器（某些硬件同时支持两者）。

这种支持被集成到资源分配机制中。当分配资源时，驱动程序可以从资源中检索相关的 `bus_space_tag_t` 和 `bus_space_handle_t`。

Newbus 还允许在专门的文件中定义接口方法。这些文件通常是 **.m** 文件，位于 **src/sys** 目录下。

Newbus 系统的核心是一个可扩展的“面向对象编程”模型。系统中的每个设备都有一个它支持的方法表。系统和其他设备使用这些方法来控制设备并请求服务。设备支持的不同方法由多个“接口”定义。“接口”是一组相关方法，可以由设备实现。

在 Newbus 系统中，设备的方法由系统中的各种设备驱动程序提供。当设备在 *自动配置* 过程中附加到驱动程序时，它使用驱动程序声明的方法表。设备可以稍后 *脱离* 驱动程序并 *重新附加* 到新的驱动程序，从而使用新的方法表。这允许动态替换驱动程序，这对于驱动程序开发非常有用。

接口由类似于文件系统中定义 vnode 操作的语言来描述。接口将存储在方法文件中（通常命名为 **foo\_if.m**）。

**示例 1. Newbus 方法**

```c
# Foo 子系统/驱动程序（注释...）

    INTERFACE foo

METHOD int doit {
    device_t dev;
};

# DEFAULT 是当方法没有通过：DEVMETHOD() 提供时将使用的方法

METHOD void doit_to_child {
    device_t dev;
    driver_t child;
} DEFAULT doit_generic_to_child;
```

当这个接口被编译时，它会生成一个头文件 "**foo\_if.h**"，其中包含以下函数声明：

```c
int FOO_DOIT(device_t dev);
int FOO_DOIT_TO_CHILD(device_t dev, device_t child);
```

同时，还会创建一个源文件 "**foo\_if.c**"，该文件与自动生成的头文件一起使用；它包含这些函数的实现，这些实现会查找对象方法表中的相关函数并调用该函数。

系统定义了两个主要接口。第一个基础接口称为 *"device"*，包括与所有设备相关的方法。*"device"* 接口中的方法包括 *"probe"*、*"attach"* 和 *"detach"* 用于控制硬件检测，以及 *"shutdown"*、*"suspend"* 和 *"resume"* 用于关键事件通知。

第二个更复杂的接口是 *"bus"*。这个接口包含适用于具有子设备的设备的方法，包括访问总线特定的每个设备信息的方法^\[[1](https://docs.freebsd.org/en/books/arch-handbook/newbus/#_footnotedef_1 "查看脚注")]^、事件通知（`child_detached`、`driver_added`）和资源管理（`alloc_resource`、`activate_resource`、`deactivate_resource`、`release_resource`）。

"bus" 接口中的许多方法为总线设备的某个子设备提供服务。这些方法通常使用前两个参数来指定提供服务的总线以及请求服务的子设备。为了简化驱动程序代码，许多这些方法具有访问函数，查找父设备并调用父设备上的方法。例如，方法 `BUS_TEARDOWN_INTR(device_t dev, device_t child, …)` 可以通过函数 `bus_teardown_intr(device_t child, …)` 调用。

系统中的某些总线类型定义了额外的接口，以提供对总线特定功能的访问。例如，PCI 总线驱动程序定义了 "pci" 接口，它具有两个方法 `<em>read_config</em>` 和 `<em>write_config</em>`，用于访问 PCI 设备的配置寄存器。

## 14.3. Newbus API

由于 Newbus API 非常庞大，本节努力对其进行文档化。更多信息将在此文档的下一个版本中提供。

### 14.3.1. 重要的源代码层次结构位置

**src/sys/\[arch]/\[arch]** - 特定机器架构的内核代码位于此目录。例如，`i386` 架构或 `SPARC64` 架构。

**src/sys/dev/\[bus]** - 特定 `[bus]` 的设备支持位于此目录。

**src/sys/dev/pci** - PCI 总线支持代码位于此目录。

**src/sys/\[isa|pci]** - PCI/ISA 设备驱动程序位于此目录。PCI/ISA 总线支持代码曾经存在于此目录中，直到 FreeBSD 版本 `4.0`。

### 14.3.2. 重要结构和类型定义

`devclass_t` - 这是指向 `struct devclass` 的指针类型定义。

`device_method_t` - 这是与 `kobj_method_t` 相同的类型（请参见 **src/sys/kobj.h**）。

`device_t` - 这是指向 `struct device` 的指针类型定义。`device_t` 代表系统中的一个设备。它是一个内核对象。有关实现细节，请参见 **src/sys/sys/bus\_private.h**。

`driver_t` - 这是引用 `struct driver` 的类型定义。`driver` 结构是 `device` 内核对象的一种类；它还保存驱动程序的私有数据。

***driver\_t* 实现**

```c
struct driver {
		KOBJ_CLASS_FIELDS;
		void	*priv;			/* 驱动程序的私有数据 */
	  };
```

`device_state_t` 类型，它是一个枚举，表示 Newbus 设备在自动配置过程前后的可能状态。

**设备状态 \_device\_state\_t**

```c
/*
	   * src/sys/sys/bus.h
	   */
	  typedef enum device_state {
		DS_NOTPRESENT,	/* 未探测或探测失败 */
		DS_ALIVE,		/* 探测成功 */
		DS_ATTACHED,	/* 调用了 attach 方法 */
		DS_BUSY			/* 设备正在使用 */
	  } device_state_t;
```
