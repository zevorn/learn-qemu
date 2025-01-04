---
layout: center
---

# 浅析 QEMU 的 QOM 实现

QEMU/KVM 源码解析与应用-李强

---

## QOM 介绍

QOM 的全称是 QEMU Object Model，是 QEMU 使用面向对象思想实现的抽象层，用来组织 QEMU 中的各种组件（比如设备模拟、后端组件 MemoryRegion、Machine 等）。类似于 C++ 的类，但是 QOM 是用纯 C 语言来实现的。

QOM 支持的面向对象特性有：继承、封装、多态。

一个简单例子，QEMU 命令行创建 edu1 和 edu2 两个对象，它们的种类都是 edu 类型：

```
$ qemu-system-riscv64 \
-device edu,id=edu1, \
-device edu,id=edu2
```

QOM 的运作过程包含三个部分：类型的注册、类型的初始化、对象的初始化：

```
    |--类型注册     ---> type_init()
    |                   register_module_init()
    |                   type_register()
QOM-|--类型的初始化  ---> type_initialize()
    |--对象的初始化  ---> object_new()
    |                   object_initialize()
    |                   object_initialize_with_type()
```
---

## 1 类型的注册

在面向对象的思想中，说到对象时都会提到它所属的类， QEMU 也需要实现一个类型系统。下面举例：

```c {*}{maxHeight:'380px'}
// hw/miscs/edu.c
static void pci_edu_register_types(void)
{
    static InterfaceInfo interfaces[] = {
        { INTERFACE_CONVENTIONAL_PCI_DEVICE },
        { },
    };
    static const TypeInfo edu_info = {
        .name          = TYPE_PCI_EDU_DEVICE,
        .parent        = TYPE_PCI_DEVICE,
        .instance_size = sizeof(EduState),
        .instance_init = edu_instance_init,
        .class_init    = edu_class_init,
        .interfaces = interfaces,
    };

    type_register_static(&edu_info);
}
type_init(pci_edu_register_types)

// include/qemu/module.h
#define type_init(function) module_init(function, MODULE_INIT_QOM)

#ifdef BUILD_DSO
void DSO_STAMP_FUN(void);
/* This is a dummy symbol to identify a loaded DSO as a QEMU module, so we can
 * distinguish "version mismatch" from "not a QEMU module", when the stamp
 * check fails during module loading */
void qemu_module_dummy(void);

#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_dso_module_init(function, type);                               \
}
#else
/* This should not be used directly.  Use block_init etc. instead.  */
#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
#endif

// utils/module.c

void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;

    l = find_type(type);

    QTAILQ_INSERT_TAIL(l, e, node);
}
```

---

register_module_init() 以类型的初始化函数,以及所属类型（对 QOM 类型来说是 MODULE_INIT_QOM）构建出一个 ModuleEntry ，然后插入到对应 module 所属的链表中，所有 module 的链表存放在一个 init_type_list 数组中。

```
               pci_edu_register_types                    
                     ^                                   
                     |      vmxnet3_register_types       
                     |           ^                       
                     +---+       |    intc_register_types
 init_type_list          |       |           ^           
+--------------------+   |       +--------+  +--------------+
| MODULE_INIT_BLOCK  |   |                |                 |
+--------------------+   |     +------+   |     +------+    |  +------+
| MODULE_INIT_OPTS   |   +-----+ init |   +-----+ init |    +--+ init |
+--------------------+         +------+         +------+       +------+
| MODULE_INIT_QOM    +-------->+ node +-------->+ node +------>+ node |
+--------------------+         +------+         +------+       +------+
| MODULE_INIT_TRACE  |         | type |         | type |       | type |
+--------------------+         +------+         +------+       +------+
| ...                |
+--------------------+
```

QEMU 使用的各个类型在 main 函数执行之前就统一注册到了 `init_type_list[MODULE_INIT_QOM]` 这个链表中。

---

进入 main 函数不就以后，就以 MODULE_INIT_QOM 为参数调用了函数 module_call_init, 这个函数执行了 `init_type_list[MODULE_INIT_QOM]` 链表上每一个 ModuleEntry 的 init 函数。

```c
void module_call_init(module_init_type type)
{
    ModuleTypeList *l;
    ModuleEntry *e;

    if (modules_init_done[type]) {
        return;
    }

    l = find_type(type);

    QTAILQ_FOREACH(e, l, node) {
        e->init();
    }

    modules_init_done[type] = true;
}
```

下面以 edu 设备为例，我们通过源码来分析一下该类型的 init 函数是如何初始化的。

主要分析一下 edu_info 是如何被初始化的，初始化阶段最终调用核心函数 `type_register_internal()`。

---

TypeImpl 的数据基本能上都是从 TypeInfo 复制过来的，表示的是一个类型的基本信息。在 C++ 中，可以使用 class 关键字定义一个类型。

QEMU 使用 C 语言实现面向对象时也必须保存对象的类型信息，所以在 TypeInfo 里面指定了类型的基本信息，然后在初始化的时候复制到 TypeImpl 的哈希表中。

TypeImpl 中存放了类型的所有信息，定义如下：

```c {*}{maxHeight:'300px'}
struct TypeImpl
{
    const char *name;

    size_t class_size;

    size_t instance_size;
    size_t instance_align;

    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);

    void *class_data;

    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;

    const char *parent;
    TypeImpl *parent_type;

    ObjectClass *class;

    int num_interfaces;
    InterfaceImpl interfaces[MAX_INTERFACES];
};
```

---

## 2 类型的初始化

在 C++ 等面向对象的编程语言中，当程序声明一个类型的时候，就已经知道了其类型的信息，比如它的对象大小。
但如果使用 C 语言来实现面向对象的这些特性，就需要做特殊的处理，对类进行单独的初始化。

类的初始化使用过 type_initialize() 完成的，这个函数并不长，函数的输入时表示类型信息的
 TypeImpl 类型 ti 。具体函数如下：

```c {*}{maxHeight:'280px'}
static void type_initialize(TypeImpl *ti)
{
    TypeImpl *parent;

    if (ti->class) {
        return;
    }

    // 1. 设置相关 filed
    ti->class_size = type_class_get_size(ti);
    ti->instance_size = type_object_get_size(ti);
    ti->instance_align = type_object_get_align(ti);
    /* Any type with zero instance_size is implicitly abstract.
     * This means interface types are all abstract.
     */
    if (ti->instance_size == 0) {
        ti->abstract = true;
    }
    if (type_is_ancestor(ti, type_interface)) {
        assert(ti->instance_size == 0);
        assert(ti->abstract);
        assert(!ti->instance_init);
        assert(!ti->instance_post_init);
        assert(!ti->instance_finalize);
        assert(!ti->num_interfaces);
    }
    ti->class = g_malloc0(ti->class_size);

    // 2. 初始化所有父类类型
    parent = type_get_parent(ti);
    if (parent) {
        type_initialize(parent);
        GSList *e;
        int i;

        g_assert(parent->class_size <= ti->class_size);
        g_assert(parent->instance_size <= ti->instance_size);
        memcpy(ti->class, parent->class, parent->class_size);
        ti->class->interfaces = NULL;

        for (e = parent->class->interfaces; e; e = e->next) {
            InterfaceClass *iface = e->data;
            ObjectClass *klass = OBJECT_CLASS(iface);

            type_initialize_interface(ti, iface->interface_type, klass->type);
        }

        for (i = 0; i < ti->num_interfaces; i++) {
            TypeImpl *t = type_get_by_name_noload(ti->interfaces[i].typename);
            if (!t) {
                error_report("missing interface '%s' for object '%s'",
                             ti->interfaces[i].typename, parent->name);
                abort();
            }
            for (e = ti->class->interfaces; e; e = e->next) {
                TypeImpl *target_type = OBJECT_CLASS(e->data)->type;

                if (type_is_ancestor(target_type, t)) {
                    break;
                }
            }

            if (e) {
                continue;
            }

            type_initialize_interface(ti, t, t);
        }
    }

    ti->class->properties = g_hash_table_new_full(g_str_hash, g_str_equal, NULL,
                                                  object_property_free);

    ti->class->type = ti;

    // 3. 依次调用所有父类的初始化函数 （与 C++ 类似）
    while (parent) {
        if (parent->class_base_init) {
            parent->class_base_init(ti->class, ti->class_data);
        }
        parent = type_get_parent(parent);
    }

    if (ti->class_init) {
        ti->class_init(ti->class, ti->class_data);
    }
}
```

---

## 3. 类型的层次结构

从 type_initialize 可以看到，类型初始化的时候也会初始化父类型。我们从这里展开继续讲讲类型的层次结构。
QOM 通过这种层次结构实现类似 C++ 中的继承概念。

下面基于以 edu 设备为例进行分析：

```c {*}{maxHeight:'250px'}
// hw/misc/edu.c
    static const TypeInfo edu_info = {
        .name          = TYPE_PCI_EDU_DEVICE,
        .parent        = TYPE_PCI_DEVICE,
        ...
    };
// hw/pci/pci.c
static const TypeInfo pci_device_type_info = {
    .name = TYPE_PCI_DEVICE,
    .parent = TYPE_DEVICE,
    ...
};
// hw/core/qdev.c
static const TypeInfo device_type_info = {
    .name = TYPE_DEVICE,
    .parent = TYPE_OBJECT,
    .class_init = device_class_init,
    .abstract = true,
    ...
};
// qom/object.c
static const TypeInfo object_info = {
    .name = TYPE_OBJECT,
    .instance_size = sizeof(Object),
    .class_init = object_class_init,
    .abstract = true,
};
```

这个 edu 类型的层次关系：

```
TYPE_PCI_DEVICE -> TYPE_DEVICE -> TYPE_OBJECT
```

---

下面再从数据结构方面谈一谈类型的层次结构:

在类型初始化函数 type_initialize 中会调用 ti->class=g_malloc0(ti->class_size) 语句分配类型的 class 结构，这个结构实际上代表了类型的信息。类似于 C++ 定义的一个类。

class_size 是 TypeImpl 的一个字段，如果这个类型没有指明它，则会使用父类的 class_size 进行初始化。

edu 设备类型本身没有定义，所以它的 class_size 为 TYPE_DEVICE 中定义的值，即 `sizeof(PCIDevieClass)`。

```c {*}{maxHeight:'260px'}
// include/hw/pci/pci_device.h (qemu v9.2.0)
struct PCIDeviceClass {
    // 第一个域： 属于“设备类型”的类型所具备的一些属性。
    DeviceClass parent_class; // 它的父类是 ObjectClass（所有类型的基础）

    void (*realize)(PCIDevice *dev, Error **errp);
    PCIUnregisterFunc *exit;
    PCIConfigReadFunc *config_read;
    PCIConfigWriteFunc *config_write;

    uint16_t vendor_id;
    uint16_t device_id;
    uint8_t revision;
    uint16_t class_id;
    uint16_t subsystem_vendor_id;       /* only for header type = 0 */
    uint16_t subsystem_id;              /* only for header type = 0 */

    const char *romfile;                /* rom bar */
};
```
---

下面给出 ObjectClass、DeviceClass、PCIDeviceClass 三者之间的关系图：

```
                     +----------------+                   
                 +-- |                | --+               
                 |   |  ObjectClass   |   |               
                 |   |                |   |               
                 |   +----------------+   +--- DeviceClass
                 |   |                |   |               
PCIDeviceClass --+   |  DeviceClass   |   |               
                 |   |  other fileds  |   |               
                 |   |                | --+               
                 |   +----------------+                   
                 |   |                |                   
                 |   | PCIDeviceClass |                   
                 |   |  other fileds  |                   
                 +-- |                |                   
                     +----------------+                   
```

可以看出来它们之间的包含与被包含的关系，事实上，编译器为 C++ 继承结构编译出来的内存分布与这里是类似的。
问题来了，父类的成员域，是什么时候被初始化的呢？

```c
// qoom/object.c, type_initialize()
memcpy(ti->class, parent->class, parent->class_size);
```
我们继续回到源代码分析。

---

## 4 对象的构造与初始化

先做一下简单总结：

1. 首先每个类型指定一个 TypeInfo 注册到系统中；
2. 接着系统运行初始化的时候会把 TypeInfo 转变成 TypeImpl 放到一个哈希表中；
3. 系统会对这个哈希表中的每个类型进行初始化；
4. 接下来根据 QEMU 命令行参数，创建对应的实例对象。

这里我们分析对象的构造流程，主要是通过 object_new 函数来实现，调用链如下：

```
object_new() -> object_new_with_type() -> object_initialize_with_type() -> object_init_with_type()
```

分析一下 object_init_with_type() ：

```c {*}{maxHeight:'130px'}
static void object_init_with_type(Object *obj, TypeImpl *ti)
{
    if (type_has_parent(ti)) {
        object_init_with_type(obj, type_get_parent(ti));
    }

    if (ti->instance_init) {
        ti->instance_init(obj);
    }
}
```

---

下面以 edu 的 TypeInfo 为例介绍对象的初始化：

```c
// hw/misc/edu.c
static const TypeInfo edu_info = {
    .name          = TYPE_PCI_EDU_DEVICE,
    .parent        = TYPE_PCI_DEVICE,
    .instance_size = sizeof(EduState),
    .instance_init = edu_instance_init,
    .class_init    = edu_class_init,
    .interfaces = interfaces,
};
```

对象类型的层次关系：

```c {*}{maxHeight:'180px'}
// hw/misc/edu.c
struct EduState {
    PCIDevice pdev;
    MemoryRegion mmio;
...
} EduState;

// include/hw/pci/pci_device.h （qemu v9.2.0）
struct PCIDevice {
    DeviceState qdev;
    bool partially_hotplugged;
...
};

// include/hw/qdev-core.h
struct DeviceState {
    /* private: */
    Object parent_obj;
    /* public: */
};
```

类型和对象之间是通过 Object 的 class 域联系在一起：`obj->class=type->class`。

---

可以把 QOM 的对象构造分成 3 部分：

1. 类型的构造，通过 TypeInfo 构造一个 TypeImpl 的哈希表，在 main 之前完成；
2. 类型的初始化，在 main 中进行，前两个都是全局性的，编译进去的 QOM 对象都会调用；
3. 类对象的构造，构造具体的实例对象，只会对指定的设备，创建对象。

现在只是构造出了对象，并完成了对象初始化，但是还没有对 EduState 的数据内容进行填充。

这个时候 edu 设备还是不可用的，对设备而言，还需要设置它的 realized 属性为 true 才行。

在 qdev_device_add 函数的后面，还有这样一句：

```c {*}{maxHeight:'200px'}
// system/qdev-monitor.c
/* Takes ownership of @opts on success */
DeviceState *qdev_device_add(QemuOpts *opts, Error **errp)
{
    QDict *qdict = qemu_opts_to_qdict(opts, NULL);
    DeviceState *ret;

    ret = qdev_device_add_from_qdict(qdict, false, errp); // call qdev_realize()
    if (ret) {
        qemu_opts_del(opts);
    }
    qobject_unref(qdict);
    return ret;
}
// hw/core/qdev.c
bool qdev_realize(DeviceState *dev, BusState *bus, Error **errp)
{
    assert(!dev->realized && !dev->parent_bus);

    if (bus) {
        if (!qdev_set_parent_bus(dev, bus, errp)) {
            return false;
        }
    } else {
        assert(!DEVICE_GET_CLASS(dev)->bus_type);
    }

    return object_property_set_bool(OBJECT(dev), "realized", true, errp);
}

```

---

## 5 属性

QOM 实现了类似 C++的基于类的多态，一个对象按照继承体系，可以是 Object、DeviceState、PCIDevice 等。
在 QOM 中为了便于管理对象，还给每种类型已经对象增加了属性。其中：

1. 类属性存在于 ObjectClass 的 properties 域中，在 type_initialize 中构造；
2. 对象属性存在于 Object 的 properties 域中，这个域在 object_initialize_with_type 中构造；

两者皆为一个哈希表，存在属性名字到 ObjectProperty 的映射。

属性由 ObjectProperty 表示：

```c {*}{maxHeight:'210px'}
// include/qom/object.h
struct ObjectProperty
{
    char *name;
    char *type;
    char *description;
    ObjectPropertyAccessor *get;
    ObjectPropertyAccessor *set;
    ObjectPropertyResolve *resolve;
    ObjectPropertyRelease *release;
    ObjectPropertyInit *init;
    void *opaque;
    QObject *defval;
};
```

---

每一种具体的属性，都会有一个结构体来描述它。下面举例：

```c
// qom/object.c
typedef struct {
    union {
        Object **targetp;
        Object *target; /* if OBJ_PROP_LINK_DIRECT, when holding the pointer  */
        ptrdiff_t offset; /* if OBJ_PROP_LINK_CLASS */
    };
    void (*check)(const Object *, const char *, Object *, Error **);
    ObjectPropertyLinkFlags flags;
} LinkProperty;

typedef struct StringProperty
{
    char *(*get)(Object *, Error **);
    void (*set)(Object *, const char *, Error **);
} StringProperty;

typedef struct BoolProperty
{
    bool (*get)(Object *, Error **);
    void (*set)(Object *, bool, Error **);
} BoolProperty;
```

---

属性的添加，分为类属性的添加和对象属性的添加，以对象属性添加为例，它的属性添加是通过 object_property_add 接口完成的。

```
+----------------+                                                                 
| ...            |                                                                 
+----------------+                                                                 
|   properties   +-------------+-------------------------------------------------  
+----------------+             |                                                   
| ...            |         +---+----+                                              
|                |         | name   |                                              
|                |         +--------+                                              
|                |         | type   |                                              
+----------------+         +--------+                                              
Object                     | set    +---> property_set_bool                        
                           +--------+                                              
                           | get    +---> property_get_bool                        
                           +--------+                                              
                           | opaque +---------> +-------+                          
                           +--------+           |  get  +--> memfd_backend_get_seal
                          ObjectProperty        +-------+                          
                                                |  set  +--> memfd_backend_set_seal
                                                +-------+                          
                                               BoolProperty                    
```
接下来通过源码分析 objetct_property_add()，以及 edu 设备对象的 realized 属性怎么被添加的。

---

介绍两个比较特殊的属性：

1. child 属性，表述对象之间的从属关系，父对象的 child 属性指向子对象，添加 child 属性的函数为 object_property_add_child：

```c {*}{maxHeight:'150px'}
// qom/object.c
ObjectProperty *
object_property_add_child(Object *obj, const char *name,
                          Object *child)
{
    return object_property_try_add_child(obj, name, child, &error_abort);
}
ObjectProperty *
object_property_try_add_child(Object *obj, const char *name,
                              Object *child, Error **errp)
{
    g_autofree char *type = NULL;
    ObjectProperty *op;

    assert(!child->parent);

    type = g_strdup_printf("child<%s>", object_get_typename(child));

    op = object_property_try_add(obj, name, type, object_get_child_property,
                                 NULL, object_finalize_child_property,
                                 child, errp);
    if (!op) {
        return NULL;
    }
    op->resolve = object_resolve_child_property;
    object_ref(child);
    child->parent = obj;
    return op;
}
```

2. link 属性，表示一种连接关系，代表一个设备引用了另一个设备，添加 link 属性的函数为
object_property_add_link ：

```c {*}{maxHeight:'150px'}
// qom/object.c
ObjectProperty *
object_property_add_link(Object *obj, const char *name,
                         const char *type, Object **targetp,
                         void (*check)(const Object *, const char *,
                                       Object *, Error **),
                         ObjectPropertyLinkFlags flags)
{
    return object_add_link_prop(obj, name, type, targetp, check, flags);
}
static ObjectProperty *
object_add_link_prop(Object *obj, const char *name,
                     const char *type, void *ptr,
                     void (*check)(const Object *, const char *,
                                   Object *, Error **),
                     ObjectPropertyLinkFlags flags)
{
    LinkProperty *prop = g_malloc(sizeof(*prop));
    g_autofree char *full_type = NULL;
    ObjectProperty *op;

    if (flags & OBJ_PROP_LINK_DIRECT) {
        prop->target = ptr;
    } else {
        prop->targetp = ptr;
    }
    prop->check = check;
    prop->flags = flags;

    full_type = g_strdup_printf("link<%s>", type);

    op = object_property_add(obj, name, full_type,
                             object_get_link_property,
                             check ? object_set_link_property : NULL,
                             object_release_link_property,
                             prop);
    op->resolve = object_resolve_link_property;
    return op;
}
```