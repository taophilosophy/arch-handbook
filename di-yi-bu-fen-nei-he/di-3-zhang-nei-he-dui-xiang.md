# 第 3 章 内核对象


内核对象，或 Kobj 为内核提供了一个面向对象的 C 编程系统。因此，被操作的数据携带了如何操作它的描述。这允许在运行时向接口添加和移除操作，而不会破坏二进制兼容性。

## 3.1. 术语

对象一组数据 - 数据结构 - 数据分配。

方法-函数操作。

类-一个或多个方法。

接口-一个或多个方法的标准集合。

## 3.2. Kobj 操作

Kobj 通过生成方法描述来工作。每个描述都包含一个唯一的 id 和一个默认函数。描述的地址用于在类的方法表中唯一标识该方法。

通过创建一个将一个或多个函数与方法描述关联的方法表来构建一个类。在使用之前，对该类进行编译。编译会分配一个缓存并将其与类关联起来。如果另一个引用类的编译尚未完成，那么会为类的方法表中的每个方法描述分配一个唯一的 id。为了使用每个方法，脚本会生成一个函数来验证参数并自动引用方法描述以进行查找。生成的函数通过使用与方法描述关联的唯一 id 作为哈希值到与对象类关联的缓存中查找该方法。如果方法未被缓存，生成的函数会继续使用类的表来查找方法。如果找到方法，则使用类中关联的函数；否则，使用与方法描述关联的默认函数。

这些间接引用可以可视化为以下内容：

```
object->cache<->class
```

## 3.3. 使用 Kobj

### 3.3.1. 结构

```
struct kobj_method
```

### 3.3.2. 函数

```
void kobj_class_compile(kobj_class_t cls);
void kobj_class_compile_static(kobj_class_t cls, kobj_ops_t ops);
void kobj_class_free(kobj_class_t cls);
kobj_t kobj_create(kobj_class_t cls, struct malloc_type *mtype, int mflags);
void kobj_init(kobj_t obj, kobj_class_t cls);
void kobj_delete(kobj_t obj, struct malloc_type *mtype);
```

### 3.3.3. 宏

```
KOBJ_CLASS_FIELDS
KOBJ_FIELDS
DEFINE_CLASS(name, methods, size)
KOBJMETHOD(NAME, FUNC)
```

### 3.3.4. 头文件

```
<sys/param.h>
<sys/kobj.h>
```

### 3.3.5. 创建一个接口模版

使用 Kobj 的第一步是创建一个接口。创建接口涉及创建一个模版，脚本 src/sys/kern/makeobjops.pl 可以用来生成方法声明和方法查找函数的头文件和代码。

在这个模版中，使用以下关键字： #include ， INTERFACE ， CODE ， EPILOG ， HEADER ， METHOD ， PROLOG ， STATICMETHOD 和 DEFAULT 。

#include 语句及其后的内容将逐字复制到生成的代码文件的开头。

 例如：

```
#include <sys/foo.h>
```

INTERFACE 关键字用于定义接口名称。此名称与每个方法名称连接为[接口名称]_[方法名称]。其语法为 INTERFACE [接口名称];。

 例如：

```
INTERFACE foo;
```

CODE 关键字会将其参数逐字复制到代码文件中。其语法是 CODE { [whatever] };

 例如：

```
CODE {
	struct foo * foo_alloc_null(struct bar *)
	{
		return NULL;
	}
};
```

HEADER 关键字会将其参数逐字复制到头文件中。它的语法是 HEADER { [whatever] };

 例如：

```
HEADER {
        struct mumble;
        struct grumble;
};
```

METHOD 关键字描述一个方法。它的语法是 METHOD [return type] [method name] { [object [, arguments]] };

 例如：

```
METHOD int bar {
	struct object *;
	struct foo *;
	struct bar;
};
```

DEFAULT 关键字可以跟在 METHOD 关键字后面。它扩展了 METHOD 关键字以包括方法的默认功能。扩展的语法是 METHOD [return type] [method name] { [object; [other arguments]] }DEFAULT [default function];

 例如：

```
METHOD int bar {
	struct object *;
	struct foo *;
	int bar;
} DEFAULT foo_hack;
```

STATICMETHOD 关键字与 METHOD 关键字相似，不同之处在于 kobj 数据不在对象结构的头部，因此将其强制转换为 kobj_t 是不正确的。相反， STATICMETHOD 依赖于 Kobj 数据被引用为“ops”。这也非常有用，可以直接从类的方法表中调用方法。

PROLOG 和 EPILOG 关键字在它们附加的地方之前或之后立即插入代码。此功能主要用于分析情况，以便以其他方式难以获取信息的情况。

其他完整示例：

```
src/sys/kern/bus_if.m
src/sys/kern/device_if.m
```

### 3.3.6. 创建一个类

使用 Kobj 的第二步是创建一个类。一个类包括一个名称、一个方法表和对象大小（如果使用 Kobj 的对象处理功能）。要创建类，请使用宏 DEFINE_CLASS() 。要创建方法表，请创建一个以 NULL 条目结尾的 kobj_method_t 数组。每个非 NULL 条目可以使用宏 KOBJMETHOD() 创建。

 例如：

```
DEFINE_CLASS(fooclass, foomethods, sizeof(struct foodata));

kobj_method_t foomethods[] = {
	KOBJMETHOD(bar_doo, foo_doo),
	KOBJMETHOD(bar_foo, foo_foo),
	{ NULL, NULL}
};
```

类必须被“编译”。根据类初始化时系统的状态，必须使用静态分配的缓存“操作表”。这可以通过声明一个 struct kobj_ops 并使用 kobj_class_compile_static(); 来实现，否则，应该使用 kobj_class_compile() 。

### 3.3.7. 创建一个对象

使用 Kobj 的第三步涉及如何定义对象。Kobj 对象创建例程假设 Kobj 数据位于对象的头部。如果这不合适，你将不得不自己分配对象，然后在其 Kobj 部分上使用 kobj_init() ；否则，你可以使用 kobj_create() 来自动分配和初始化对象的 Kobj 部分。 kobj_init() 也可以用来改变对象使用的类。

将 Kobj 集成到对象中，应使用宏 KOBJ_FIELDS。

 例如

```
struct foo_data {
	KOBJ_FIELDS;
	foo_foo;
	foo_bar;
};
```

### 3.3.8. 调用方法

Kobj 的最后一步是简单地使用生成的函数来使用对象的类中的所需方法。这就像使用接口名称和方法名称一样简单，只需进行一些修改。应该使用下划线将接口名称与方法名称连接起来，全部使用大写。

例如，如果接口名称是 foo，方法是 bar，那么调用将是：

```
[return value = ] FOO_BAR(object [, other parameters]);
```

### 3.3.9. 清理工作

当通过 kobj_create() 分配的对象不再需要时，可以调用 kobj_delete() ，当类不再被使用时，可以调用 kobj_class_free() 。
