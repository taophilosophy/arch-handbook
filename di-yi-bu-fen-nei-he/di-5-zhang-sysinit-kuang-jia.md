# 第 5 章 SYSINIT 框架

SYSINIT 是一种通用调用排序和分发机制的框架。FreeBSD 目前将其用于内核的动态初始化。SYSINIT 允许 FreeBSD 的内核子系统在内核或其某个模块加载时的内核链接阶段被重新排序、添加、删除和替换，而无需编辑静态排序的初始化例程并重新编译内核。该系统还允许内核模块（目前称为 *KLD*）在引导时单独编译、链接和初始化，甚至可以在系统已经运行之后再加载。这是通过"内核链接器"和"链接器集"实现的。

## 5.1. 术语

**链接器集（Linker Set）**
一种链接器技术，链接器将程序源代码中静态声明的数据收集成一个连续可寻址的数据单元。

## 5.2. SYSINIT 操作

SYSINIT 依赖于链接器将程序源代码中多个位置声明的静态数据组合为一个连续数据块的能力。这种链接器技术被称为"链接器集"。SYSINIT 使用两个链接器集来维护两个数据集，其中包含每个消费者的调用顺序、函数以及要传递给该函数的数据指针。

SYSINIT 在为函数执行排序时使用两个优先级。第一个优先级是子系统 ID，为 SYSINIT 的函数调度提供整体顺序。当前预声明的 ID 位于 **<sys/kernel.h>** 的 `sysinit_sub_id` 枚举列表中。第二个使用的优先级是子系统内的元素顺序。当前预声明的子系统元素顺序位于 **<sys/kernel.h>** 的 `sysinit_elem_order` 枚举列表中。

SYSINIT 目前有两种用途：系统启动和内核模块加载时的函数调度，以及系统关闭和内核模块卸载时的函数调度。内核子系统通常使用系统启动 SYSINIT 来初始化数据结构，例如进程调度子系统使用 SYSINIT 来初始化运行队列数据结构。设备驱动程序应避免直接使用 `SYSINIT()`。相反，作为总线结构一部分的实际设备驱动程序应使用 `DRIVER_MODULE()` 提供一个函数来检测设备，如果设备存在则初始化设备。它会执行一些特定于设备的操作，然后自行调用 `SYSINIT()`。对于不属于总线结构的伪设备，使用 `DEV_MODULE()`。

## 5.3. 使用 SYSINIT

### 5.3.1. 接口

#### 5.3.1.1. 头文件

```c
<sys/kernel.h>
```

#### 5.3.1.2. 宏

```c
SYSINIT(uniquifier, subsystem, order, func, ident)
SYSUNINIT(uniquifier, subsystem, order, func, ident)
```

### 5.3.2. 启动

`SYSINIT()` 宏在 SYSINIT 的启动数据集中创建必要的 SYSINIT 数据，以便 SYSINIT 在系统启动和模块加载时对函数进行排序和调度。`SYSINIT()` 接受一个唯一标识符（SYSINIT 用其标识特定的函数调度数据）、子系统顺序、子系统元素顺序、要调用的函数以及要传递给该函数的数据。所有函数都必须接受一个常量指针参数。

**示例 1：`SYSINIT()` 示例**

```c
#include <sys/kernel.h>

void foo_null(void *unused)
{
        foo_doo();
}
SYSINIT(foo, SI_SUB_FOO, SI_ORDER_FOO, foo_null, NULL);

struct foo foo_voodoo = {
        FOO_VOODOO;
}

void foo_arg(void *vdata)
{
        struct foo *foo = (struct foo *)vdata;
        foo_data(foo);
}
SYSINIT(bar, SI_SUB_FOO, SI_ORDER_FOO, foo_arg, &foo_voodoo);
```

请注意，`SI_SUB_FOO` 和 `SI_ORDER_FOO` 需要如上所述位于 `sysinit_sub_id` 和 `sysinit_elem_order` 枚举中。可以使用现有的枚举值，或者向枚举中添加你自己的值。你也可以使用数学运算来微调 SYSINIT 的运行顺序。此示例展示了一个需要刚好在处理内核参数调优的 SYSINIT 之前运行的 SYSINIT。

**示例 2：调整 `SYSINIT()` 顺序**

```c
static void
mptable_register(void *dummy __unused)
{

	apic_register_enumerator(&mptable_enumerator);
}

SYSINIT(mptable_register, SI_SUB_TUNABLES - 1, SI_ORDER_FIRST,
    mptable_register, NULL);
```

### 5.3.3. 关闭

`SYSUNINIT()` 宏的行为与 `SYSINIT()` 宏类似，不同之处在于它将 SYSINIT 数据添加到 SYSINIT 的关闭数据集中。

**示例 3：`SYSUNINIT()` 示例**

```c
#include <sys/kernel.h>

void foo_cleanup(void *unused)
{
        foo_kill();
}
SYSUNINIT(foobar, SI_SUB_FOO, SI_ORDER_FOO, foo_cleanup, NULL);

struct foo_stack foo_stack = {
        FOO_STACK_VOODOO;
}

void foo_flush(void *vdata)
{
}
SYSUNINIT(barfoo, SI_SUB_FOO, SI_ORDER_FOO, foo_flush, &foo_stack);
```
