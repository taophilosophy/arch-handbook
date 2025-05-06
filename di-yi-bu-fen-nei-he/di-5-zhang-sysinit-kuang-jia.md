# 第 5 章 SYSINIT 框架

## 5.1. 术语

**链接器集（Linker Set）**
一种链接器技术，链接器将程序源代码中静态声明的数据收集成一个连续可寻址的数据单元。

## 5.2. SYSINIT 操作

SYSINIT 依赖于链接器将多个位置声明的静态数据组合为一个连续的数据块。这个链接器技术被称为“链接器集（linker set）”。SYSINIT 使用两个链接器集来维护包含每个消费者调用顺序、函数和要传递给该函数的数据的两个数据集。

SYSINIT 在排序执行函数时使用两个优先级。第一个优先级是子系统 ID，提供 SYSINIT 调度函数的整体顺序。当前预声明的 ID 列表位于 **\<sys/kernel.h>** 中的枚举列表 `sysinit_sub_id`。第二个优先级是子系统内的元素顺序。当前预声明的子系统元素顺序位于 **\<sys/kernel.h>** 中的枚举列表 `sysinit_elem_order`。

SYSINIT 目前有两个用途：系统启动时的函数调度和内核模块加载时的调度，以及系统关闭时的函数调度和内核模块卸载时的调度。内核子系统通常使用系统启动时的 SYSINIT 来初始化数据结构，例如进程调度子系统使用 SYSINIT 来初始化运行队列数据结构。设备驱动程序应避免直接使用 `SYSINIT()`。相反，属于总线结构的实际设备驱动程序应使用 `DRIVER_MODULE()` 来提供一个函数，该函数检测设备并在设备存在时初始化设备。它会做一些设备特定的操作，然后调用 `SYSINIT()`。对于非总线结构的伪设备，应使用 `DEV_MODULE()`。

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

`SYSINIT()` 宏在 SYSINIT 的启动数据集中创建必要的 SYSINIT 数据，以便 SYSINIT 能对系统启动和模块加载时的函数进行排序和调度。`SYSINIT()` 需要一个唯一标识符，用于标识特定的函数调度数据，子系统顺序，子系统元素顺序，要调用的函数，以及传递给该函数的数据。所有函数必须接受一个常量指针参数。

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

请注意，`SI_SUB_FOO` 和 `SI_ORDER_FOO` 需要在 `sysinit_sub_id` 和 `sysinit_elem_order` 枚举列表中，如上所述。可以使用现有的枚举值，或者添加自己的枚举值。您也可以使用数学来微调 SYSINIT 的执行顺序。此示例显示了一个需要在处理内核参数调优的 SYSINIT 之前运行的 SYSINIT。

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

`SYSUNINIT()` 宏与 `SYSINIT()` 宏的行为类似，不同之处在于它将 SYSINIT 数据添加到 SYSINIT 的关机数据集中。

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
