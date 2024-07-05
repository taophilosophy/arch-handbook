# 第 5 章 SYSINIT 框架



SYSINIT 是用于通用调用排序和分派机制的框架。FreeBSD 目前在内核的动态初始化中使用它。SYSINIT 允许重新排序、添加、移除和替换 FreeBSD 的内核子系统，在内核或其模块加载时，无需编辑静态排序的初始化路由并重新编译内核。该系统还允许内核模块，目前称为 KLD，可以在引导时单独编译、链接和初始化，甚至在系统已经运行时加载。这是通过使用“内核链接器”和“链接器集”来实现的。

## 5.1. 术语

链接器集是一种链接技术，其中链接器将程序源文件中静态声明的数据收集到一个连续可寻址的数据单元中。

## 5.2. SYSINIT 操作

SYSINIT 依赖于链接器能够将程序源代码中在多个位置声明的静态数据作为单个连续数据块进行分组。这种链接器技术称为 "链接器集"。SYSINIT 使用两个链接器集来维护包含每个消费者调用顺序、函数和传递给该函数的数据指针的两个数据集。

SYSINIT 在对函数执行顺序进行排序时使用两个优先级。第一个优先级是子系统 ID，为 SYSINIT 的函数分派提供整体顺序。当前预声明的 ID 在 中的枚举列表中。第二个使用的优先级是子系统内的元素顺序。当前预声明的子系统元素顺序在 中的枚举列表中。

目前 SYSINIT 有两个用途。在系统启动和内核模块加载时进行功能分发，以及在系统关闭和内核模块卸载时进行功能分发。内核子系统经常使用系统启动的 SYSINIT 来初始化数据结构，例如进程调度子系统使用 SYSINIT 来初始化运行队列数据结构。设备驱动程序应避免直接使用 SYSINIT() 。相反，属于总线结构一部分的实际设备的驱动程序应使用 DRIVER_MODULE() 来提供一个检测设备并（如果存在）初始化设备的函数。它将执行一些特定于设备的操作，然后调用 SYSINIT() 本身。对于非属于总线结构的伪设备，请使用 DEV_MODULE() 。

## 使用 SYSINIT

### 5.3.1. 接口

#### 5.3.1.1. 头部

```
<sys/kernel.h>
```

#### 5.3.1.2. 宏

```
SYSINIT(uniquifier, subsystem, order, func, ident)
SYSUNINIT(uniquifier, subsystem, order, func, ident)
```

### 5.3.2. 启动

SYSINIT() 宏在 SYSINIT 的启动数据集中创建必要的 SYSINIT 数据，以便 SYSINIT 对系统启动和模块加载时需要排序和分发的函数进行调用。 SYSINIT() 接受一个唯一标识符，该标识符由 SYSINIT 用于识别特定函数分发数据、子系统顺序、子系统元素顺序、要调用的函数以及要传递给函数的数据。所有函数必须接受一个常量指针参数。

示例 1。 SYSINIT() 的示例

```
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

请注意， SI_SUB_FOO 和 SI_ORDER_FOO 需要出现在上面提到的枚举类型 sysinit_sub_id 和 sysinit_elem_order 中。可以使用现有枚举类型或将自己的枚举类型添加到中。还可以使用数学方法对 SYSINIT 将运行的顺序进行微调。此示例显示了需要在处理调整内核参数的 SYSINIT 之前运行的 SYSINIT。

示例 2. 调整 SYSINIT() 订单的示例

```
static void
mptable_register(void *dummy __unused)
{

	apic_register_enumerator(&mptable_enumerator);
}

SYSINIT(mptable_register, SI_SUB_TUNABLES - 1, SI_ORDER_FIRST,
    mptable_register, NULL);
```

### 5.3.3. 关机

SYSUNINIT() 宏的行为与 SYSINIT() 宏类似，只是它将 SYSINIT 数据添加到 SYSINIT 的关机数据集中。

示例 3. SYSUNINIT() 的示例

```
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
