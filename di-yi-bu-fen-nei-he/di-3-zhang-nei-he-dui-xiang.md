# 第 3 章 内核对象

Kernel Objects，或称 *Kobj*，为内核提供了一个面向对象的 C 编程系统。因此，操作的数据包含了如何操作它的描述。这使得操作可以在运行时动态地添加或移除，并且不会破坏二进制兼容性。

## 3.1. 术语

**Object**

一组数据——数据结构——数据分配。

**Method**

一种操作——函数。

**Class**

一个或多个方法。

**Interface**

一组标准的一个或多个方法。

## 3.2. Kobj 操作

Kobj 通过生成方法描述来工作。每个描述包含一个唯一的 id 和一个默认函数。描述的地址用于在类的函数表中唯一地标识该方法。

通过创建一个方法表，将一个或多个函数与方法描述关联来构建一个类。在使用之前，类需要被编译。编译过程会分配一个缓存，并将其与类关联。如果方法描述尚未由其他引用类的编译分配唯一 id，则为类的方法表中的每个方法描述分配一个唯一的 id。为了使用每个方法，脚本会生成一个函数，用于验证参数，并自动引用方法描述进行查找。生成的函数通过使用与方法描述关联的唯一 id，作为哈希值查找与对象类关联的缓存。如果方法未被缓存，生成的函数将继续使用类的表格查找方法。如果找到该方法，则使用类中关联的函数；否则，使用方法描述中关联的默认函数。

这些间接引用可以用以下方式可视化：

```c
object->cache<->class
```

## 3.3. 使用 Kobj

### 3.3.1. 结构

```c
struct kobj_method
```

### 3.3.2. 函数

```c
void kobj_class_compile(kobj_class_t cls);
void kobj_class_compile_static(kobj_class_t cls, kobj_ops_t ops);
void kobj_class_free(kobj_class_t cls);
kobj_t kobj_create(kobj_class_t cls, struct malloc_type *mtype, int mflags);
void kobj_init(kobj_t obj, kobj_class_t cls);
void kobj_delete(kobj_t obj, struct malloc_type *mtype);
```

### 3.3.3. 宏

```c
KOBJ_CLASS_FIELDS
KOBJ_FIELDS
DEFINE_CLASS(name, methods, size)
KOBJMETHOD(NAME, FUNC)
```

### 3.3.4. 头文件

```c
<sys/param.h>
<sys/kobj.h>
```

### 3.3.5. 创建接口模板

使用 Kobj 的第一步是创建接口。创建接口包括创建一个模板，脚本 **src/sys/kern/makeobjops.pl** 可以用来生成方法声明和方法查找函数的头文件和代码。

在这个模板中使用以下关键字：`#include`、`INTERFACE`、`CODE`、`EPILOG`、`HEADER`、`METHOD`、`PROLOG`、`STATICMETHOD` 和 `DEFAULT`。

`#include` 语句及其后面的内容会被原样复制到生成的代码文件的开头。

例如：

```c
#include <sys/foo.h>
```

`INTERFACE` 关键字用于定义接口名称。该名称会与每个方法名称连接成 `[interface name]_[method name]`。其语法为 `INTERFACE [interface name];`。

例如：

```c
INTERFACE foo;
```

`CODE` 关键字会将其参数原样复制到代码文件中。其语法为 `CODE { [whatever] };`

例如：

```c
CODE {
	struct foo * foo_alloc_null(struct bar *)
	{
		return NULL;
	}
};
```

`HEADER` 关键字会将其参数原样复制到头文件中。其语法为 `HEADER { [whatever] };`

例如：

```c
HEADER {
        struct mumble;
        struct grumble;
};
```

`METHOD` 关键字描述一个方法。其语法为 `METHOD [return type] [method name] { [object [, arguments]] };`

例如：

```c
METHOD int bar {
	struct object *;
	struct foo *;
	struct bar;
};
```

`DEFAULT` 关键字可以跟在 `METHOD` 关键字后面。它扩展了 `METHOD` 关键字，包含方法的默认函数。扩展后的语法为 `METHOD [return type] [method name] { [object; [other arguments]] }DEFAULT [default function];`

例如：

```c
METHOD int bar {
	struct object *;
	struct foo *;
	int bar;
} DEFAULT foo_hack;
```

`STATICMETHOD` 关键字的使用方式与 `METHOD` 类似，但 Kobj 数据不在对象结构的开头，因此将其强制转换为 kobj\_t 会不正确。`STATICMETHOD` 依赖于 Kobj 数据被引用为 'ops'。这对于直接从类的函数表调用方法也很有用。

`PROLOG` 和 `EPILOG` 关键字分别在 `METHOD` 前后插入代码。这一功能主要用于分析情境，其中很难通过其他方式获取信息。

其他完整示例：

```c
src/sys/kern/bus_if.m
src/sys/kern/device_if.m
```

### 3.3.6. 创建类

使用 Kobj 的第二步是创建类。类由名称、方法表和如果使用 Kobj 的对象处理功能时的对象大小组成。要创建类，请使用宏 `DEFINE_CLASS()`。要创建方法表，创建一个由 kobj\_method\_t 组成的数组，以 NULL 条目结束。每个非 NULL 条目可以使用宏 `KOBJMETHOD()` 创建。

例如：

```c
DEFINE_CLASS(fooclass, foomethods, sizeof(struct foodata));

kobj_method_t foomethods[] = {
	KOBJMETHOD(bar_doo, foo_doo),
	KOBJMETHOD(bar_foo, foo_foo),
	{ NULL, NULL}
};
```

类必须被“编译”。根据系统在初始化类时的状态，可能需要使用静态分配的缓存、“ops 表”。这可以通过声明 `struct kobj_ops` 并使用 `kobj_class_compile_static();` 来实现。否则，应该使用 `kobj_class_compile()`。

### 3.3.7. 创建对象

使用 Kobj 的第三步是定义对象。Kobj 对象创建例程假定 Kobj 数据位于对象的开头。如果这种假设不适用，你将需要自行分配对象，并在对象的 Kobj 部分使用 `kobj_init()`；否则，你可以使用 `kobj_create()` 来自动分配和初始化对象的 Kobj 部分。`kobj_init()` 也可以用于更改对象使用的类。

为了将 Kobj 集成到对象中，你应该使用宏 `KOBJ_FIELDS`。

例如：

```c
struct foo_data {
	KOBJ_FIELDS;
	foo_foo;
	foo_bar;
};
```

### 3.3.8. 调用方法

使用 Kobj 的最后一步是使用生成的函数来调用对象类中的所需方法。这与使用接口名称和方法名称，并进行一些修改非常相似。接口名称应与方法名称连接，中间使用下划线，并且全部大写。

例如，如果接口名称是 foo，方法是 bar，那么调用方式将是：

```c
[return value = ] FOO_BAR(object [, other parameters]);
```

### 3.3.9. 清理

当通过 `kobj_create()` 分配的对象不再需要时，可以调用 `kobj_delete()` 对其进行清理；当类不再使用时，可以调用 `kobj_class_free()` 对其进行清理。
