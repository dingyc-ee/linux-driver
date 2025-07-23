# 设备模型

## 第1章 Linux设备模型

Linux设备模型，是Linux内核中用于**统一管理硬件设备、驱动程序及总线关系**的核心框架。它通过抽象化设备、请不动、总线和类等概念，实现设备的动态管理、热插拔支持、电源管理及用户空间交互。

### 1.1 Linux设备模型的核心组成

1. 设备(`Device`)
    + 由`struct device`表示，描述硬件设备的属性(如名称、父设备、所属总线)
    + 作用：在`/sys/devices/`下生成设备树状结构，反应物理连接层级(如SoC内部设备 通过父设备关联)
2. 驱动(`Driver`)
    + 由`struct device_driver`定义，包含`probe()`(设备初始化)、`remove()`(资源清理)等函数
    + 匹配机制：总线通过`match()`函数，将设备与驱动绑定(如设备树`cimpatible`属性匹配驱动的`of_match_table`)
3. 总线(`Bus`)
    + 由`struct bus_type`表示，分为物理总线(如USB、PCI)和虚拟总线(如platform总线)
    + 作用：管理设备与驱动的注册，提供匹配规则(如platform总线匹配SoC外设)
4. 类(`Class`)
    + 由`struct class`定义，按功能聚合设备(如`/sys/calss/net/包含所有网卡`)
    + 优势：抽象共性操作(如电源管理)，减少重复代码

### 1.2 为什么需要设备模型

1. 解决字符设备驱动的局限性
    + 硬编码问题：字符设备驱动需手动分配主、次设备号(`register_chrdev()`)，无法动态适应硬件变化
    + 无统一管理：设备间依赖关系(如父子设备)需手动维护，易引发资源泄露或冲突
    + 功能缺失：不支持热插拔、电源管理等高级特性，需驱动自行实现

2. 设备模型的核心优势

    | 特性 | 字符设备驱动 | 设备模型 | 优势说明 |
    | - | - | - | - |
    | 动态管理 | 静态注册设备号 | 自动匹配设备与驱动 | 支持热插拔(如USB设备插入时加载驱动) |
    | 资源管理 | 手动释放资源 | 自动释放依赖资源(`devm_* API`) | 防止内存泄露 |
    | 用户空间接口 | 依赖/dev节点 | 通过sysfs暴露设备属性 | 支持`echo 1 > /sys/class/leds/led0/brightness控制硬件` |
    | 电源管理 | 需自行实现 | 统一休眠/唤醒接口(如`suspend()`) | 简化功耗优化 |

### 1.3 典型场景：设备模型 vs 字符设备驱动

1. 场景1：多设备支持
    + 字符设备：为每个设备手动分配次设备号，代码冗余(如10个同类型传感器需10此注册)
    + 设备模型：通过总线自动枚举设备(如I2C总线扫描从机地址)，驱动只需实现一次`probe()`
2. 场景2：热插拔处理
    + 字符设备：无法动态响应设备插拔，需重启或手动加载驱动
    + 设备模型：通过`uevent`通知用户空间(如udev自动创建/dev节点)

### 1.4 设备模型的底层实现：`kobject`和`sysfs`

+ `kobject`：设备模型的原子单位，实现引用计数和sysfs目录映射
+ `sysfs`映射规则：
    | 内核对象 | sysfs表现 | 示例路径 |
    | - | - | - |
    | 设备(Deivce) | 目录 | `/sys/devices/platform/soc/leds` |
    | 属性(attribute) | 文件 | `/sys/class/net/eth0/mtu` |
    | 关系(RelationShip) | 符号链接 | `/sys/bus/usb/devices/1-1 -> ../../devices/pci0000:00` |

### 1.5 总结：设备模型的价值与适用性

1. 必要性：设备模型通过统一抽象(设备、驱动、总线、类)，解决了硬件拓扑管理、动态资源分配和标准化接口三大问题，是支持现代硬件(如热插拔SSID、多核SoC)的计时
2. 字符设备驱动的定位：
    + 仍适用于：简单设备(如按键、LED)或无需动态管理的场景
    + 局限性：在复杂系统中，字符设备需额外开发资源管理和热插拔支持，反而增加维护成本

## 第2章 `kobject`

### 2.1 `kobject`原理

Linux设备模型的核心是kobject机制，它为内核对线(如设备、驱动、总线)提供统一的层次结构管理、引用计数和sysfs接口支持。

```c
struct kobject {
	const char		*name;          // sysfs目录名
	struct list_head	entry;      // 用于挂载到kset的链表
	struct kobject		*parent;    // 父对象指针(构建层次结构)
	struct kset		*kset;          // 所属的kset集合
	struct kobj_type	*ktype;     // 对象类型(定义行为)
	struct kernfs_node	*sd;        // sysfs目录项
	struct kref		kref;           // 引用计数器

	unsigned int state_initialized:1;   // 状态标志
	unsigned int state_in_sysfs:1;
    // ... (其他状态标志)
};
```

这似乎有点抽象，我们根据实例来理解。在Linux中，每个kobject都会对应系统/sys下的一个目录，如下图所示。

![](./src/0001.jpg)

这些kobject是怎么来的？以`/sys/dev`为例，我们来看下代码：

1. 创建了`dev`这个kobject，父对象为NULL，所以直接出现在`/sys/`目录下
2. 分别创建了`block`和`char`这两个kobject，父对象为`dev`，所以他们出现在`/sys/dev/`目录下

```c
int __init devices_init(void)
{
	dev_kobj = kobject_create_and_add("dev", NULL);

	sysfs_dev_block_kobj = kobject_create_and_add("block", dev_kobj);

	sysfs_dev_char_kobj = kobject_create_and_add("char", dev_kobj);

	return 0;
}
```

现在我们知道了。kobject表示系统/sys下的一个目录，而目录又是由多个层次，所以对应kobject的树状关系如下图所示。*与设备节点`device_node`的数据结构有一点点类似，有parent指针，但没有child和sibling指针，我们不需要平行关系。*

![](./src/0002.jpg)

### 2.2 `kobject`函数

#### 2.2.1 `kobject_create_and_add()`函数

##### 2.2.1.1 函数功能与原型

动态分配内存、初始化kobject，并在sysfs中创建对应目录。

```c
#include <linux/kobject.h>
struct kobject *kobject_create_and_add(const char *name, struct kobject *parent);
```

+ `name`: sysfs目录名
+ `parent`: 父object指针(若为NULL, 则挂载到/sys根目录)

##### 2.2.1.2 内部实现分步解析

1. 动态内存分配: `kobject_create()`

    ```c
    struct kobject *kobj = kzalloc(sizeof(*kobj), GFP_KERNEL);
    ```

    + 通过kzalloc分配归零内存，避免未初始化字段
    + 若分配失败，返回NULL

2. 初始化object: `kobject_init()`

    ```c
    kobject_init(kobj, &dynamic_kobj_ktype);
    ```

3. 注册到sysfs: `kobject_add()`

    ```c
    retval = kobject_add(kobj, parent, "%s", name);
    ```

    1. 设置名称
    2. 设置父对象: `kobj->parent = parent`，决定sysfs路径(如父路径为`kernel_kobj`，则路径为`/sys/kernel_kobj/name`)
    3. 处理kset关系: 若`kobj->kset`非空，加入其链表(*但本函数默认不绑定`kset`*)
    4. 创建sysfs目录: `create_dir(kobj)` -> 调用`sysfs_create_dir_ns()`生成目录
        ```c
        int sysfs_create_dir_ns(struct kobject *kobj, const void *ns)
        {
            struct kernfs_node *parent, *kn;

            if (kobj->parent)
                parent = kobj->parent->sd;
            else
                parent = sysfs_root_kn;

            kn = kernfs_create_dir_ns(parent, kobject_name(kobj));
 
            return 0;
        }
        ```

##### 2.2.1.3 关键特性与行为规则

1. 默认ktype的局限性
    + 使用`dynamic_kobj_ktype`，仅支持基础内存释放，无默认属性文件(如uevent)
    + 需手动添加属性: 通过`sysfs_create_file()`或`sysfs_create_group()`扩展
2. 声明周期管理
    + 初识引用计数: kref = 1
    + 释放方式: 必须调用`kobject_put(kobj)`，触发引用计数减1，归零时执行`release()`释放内存
3. sysfs目录位置规则
    | parent参数 | sysfs路径 | 示例 |
    | - | - | - |
    | NULL | /sys根目录 | /sys/my_kobj/ |
    | 父kobject指针 | 父对象目录下 | /sys/devices/platform/my/ |
    | 绑定kset(需手动) | kset目录下 | /sys/bus/usb/devices/my/ |

##### 2.2.1.4 `kobject`初始状态图示

```
+---------------------------------------------+
|           struct kobject (kobj)             |
+---------------------------------------------+
| name:      "dynamic_kobj" (动态分配字符串)   |  🔸 由参数`name`生成
| entry:     {prev: →, next: →} (空链表节点)   |  🔸 通过`INIT_LIST_HEAD`初始化
| parent:    NULL 或指向父对象                 |  🔸 由调用参数决定
| kset:     NULL                              |  🔸 **未绑定任何kset**
| ktype:    &dynamic_kobj_ktype               |  🔸 固定类型，不可自定义
| sd:       NULL                              |  🔸 sysfs目录创建后由内核赋值
| kref:     { refcount: 1 } (原子计数)         |  🔸 初始引用计数为1
|---------------------------------------------|
| state_initialized:      1                   |  🔸 已初始化
| state_in_sysfs:         1                   |  🔸 已注册到sysfs
| state_add_uevent_sent:  0                   |  🔸 未发送ADD事件
| state_remove_uevent_sent:0                 |  🔸 未发送REMOVE事件
| uevent_suppress:        0                   |  🔸 允许发送uevent
+---------------------------------------------+
```

##### 2.2.1.5 设计意义与局限

+ 优势：简化动态kobject创建流程，无需预分配内存或自定义ktype
+ 局限：
    + 功能单一：不自动绑定kset，缺乏分组管理能力
    + 无默认属性：需手动添加sysfs文件，不适用于复杂设备模型
+ 使用场景：临时调试接口、简单内核模块的sysfs暴露

#### 2.2.2 `kobject_init_and_add()`函数

##### 2.2.2.1 函数功能与原型

对*预分配内存的kobject*进行初始化，关联其类型(kobj_type)，并注册到`sysfs`文件系统中。

```c
int kobject_init_and_add(struct kobject *kobj, 
                         struct kobj_type *ktype,
                         struct kobject *parent, 
                         const char *fmt, ...);
```

+ `kobj`: *预分配*的kobject指针(通常嵌入在自定义结构体中)
+ `ktype`: 定义对象的声明周期回调(`release`)和属性操作(`sysfs_ops`)
+ `parent`: 父对象指针，决定sysfs目录层级(若为NULL，则使用`kobj->kset`或根目录)
+ `fmt`: 格式化字符串指定sysfs目录名(如`"device%d"`)

##### 2.2.2.2 内部实现流程

1. 初始化kobject: 调用`kobject_init(kobj, ktype)`
    + 校验`kobj`和`ktype`非空
    + 初始化引用计数`kref_init(&kobj->kref)`，设置初始值1
    + 初始化链表头`INIT_LIST_HEAD(&kobj->entry)`
2. 设置名称与父对象
    + 通过`kobject_set_name_vargs()`解析`fmt`参数，生成`kobj->name`(即`sysfs`目录名)
    + 若`parent`非空，设置`kobj->parent = parent`；否则检查`kobj->kset`，将其`kobject`作为默认父节点
3. 注册到`sysfs`: 调用`kobject_add_internal()`
    + 若`kobj->kset`存在，将对象加入其链表
    + 在 sysfs 中创建目录：`sysfs_create_dir_ns()`
    + 生成默认属性文件：遍历`ktype->default_attrs`，调用`sysfs_create_file()`创建文件节点

##### 2.2.2.3 关键特性与设计意义

1. 自定义`ktype`的灵活性
    + 生命周期管理：通过`ktype->release`释放嵌入kobject的父结构(如`container_of`反向解析)
        ```c
        void my_release(struct kobject *kobj) {
            struct my_device *dev = container_of(kobj, struct my_device, kobj);
            kfree(dev);
        }
        ```
    + 属性文件支持：通过`ktype->sysfs_ops`和`default_attrs`定义读写接口
        ```c
        static struct attribute *dev_attrs[] = {
            &dev_attr_name.attr,  // 文本属性文件
            &dev_attr_value.attr,
            NULL
        };
        static struct kobj_type dev_ktype = {
            .release = my_release,
            .sysfs_ops = &dev_sysfs_ops,
            .default_attrs = dev_attrs,
        };
        ```

##### 2.2.2.4 典型应用场景

1. 自定义设备对象管理

    ```c
    struct my_device {
        int id;
        struct kobject kobj;  // 内嵌 kobject
    };

    static struct kobj_type dev_ktype = {
        .release = dev_release,
        .sysfs_ops = &dev_sysfs_ops,
        .default_attrs = dev_attrs,
    };

    int init_device(struct my_device *dev) {
        // 初始化并注册，目录名为 "deviceX"
        return kobject_init_and_add(&dev->kobj, &dev_ktype, NULL, "device%d", dev->id);
    }
    ```

2. 构建父子对象层级

    ```c
    // 父对象
    kobject_init_and_add(&parent_kobj, &ktype, NULL, "parent");
    // 子对象：指定父对象指针
    kobject_init_and_add(&child_kobj, &ktype, &parent_kobj, "child");
    ```

    sysfs结构：

    ```
    /sys/parent/
    ├── child
    │   ├── name
    │   └── value
    └── ...
    ```

##### 2.2.2.5 `kset`行为

通过`kobject_init_and_add()`创建的kobject，是否绑定`kset`取决于开发者是否手动设置其`kset`字段。

1. 默认行为：不自动绑定`kset`

    + 无默认绑定：`kobject_init_and_add`不会自动为kobject关联kset。该函数仅完成以下操作：
        + 初始化kobject：设置kerf引用计数、链表头
        + 根据`parent`参数设置父子关系(在sysfs中创建目录层次)
        + 若`kobj->kset`字段已被*手动赋值*，则将其加入对应kset的链表
    + 关键代码逻辑：在`kobject_add_internal()`函数中判断，若`kobj->kset`为空则跳过此逻辑
        ```c
        if (kobj->kset) { // 检查是否已设置kset
            if (!parent) 
                parent = kobject_get(&kobj->kset->kobj); // 若无父对象，使用kset的kobj作为父节点
            kobj_kset_join(kobj); // 将kobject加入kset的链表
        }
        ```

2. 手动绑定`kset`的方法

    开发者需要在调用`kobject_init_and_add​​()`之前，显式设置`kobject`的`kset`字段：

    ```c
    struct kobject *kobj = &my_device->kobj; // 假设kobject嵌入在自定义结构体中
    kobj->kset = my_kset; // 手动绑定到目标kset
    int ret = kobject_init_and_add(kobj, &my_ktype, NULL, "device%d", id);
    ```

3. 绑定`kset`的实际应用场景

    设备驱动注册：

    ```c
    priv->kobj.kset = bus->p->drivers_kset; // 绑定到总线的drivers_kset
    kobject_init_and_add(&priv->kobj, &driver_ktype, NULL, "%s", drv->name);
    ```

##### 2.2.1.6 `kobject`初始状态图示

```
+-------------------------------------------+
|          struct kobject (kobj)           |
+-------------------------------------------+
| name:     [由fmt生成的动态字符串]          | 🔸 如 `"device0"`，通过 `kobject_set_name_vargs()` 生成 [1,6](@ref)
| entry:    {prev: →, next: →} (空链表节点) | 🔸 `INIT_LIST_HEAD` 初始化，形成自环 [6](@ref)
| parent:   [指向传入的parent指针]           | 🔸 若参数非空则关联父对象；否则为 `NULL` [1](@ref)
| kset:     [调用前手动赋值的kset]           | 🔸 **不自动设置**，需显式赋值（未赋值则为 `NULL`）[1](@ref)
| ktype:    [传入的ktype指针]               | 🔸 **必须非空**，定义生命周期与属性操作 [2](@ref)
| sd:       NULL                            | 🔸 sysfs 目录创建后由内核赋值（`sysfs_create_dir_ns()`）[1](@ref)
| kref:     { refcount: 1 } (原子计数)      | 🔸 初始引用计数为 1（`kref_init()`）[6](@ref)
|-------------------------------------------|
| state_initialized:      1                | 🔸 标记已初始化完成 [6](@ref)
| state_in_sysfs:         1                | 🔸 标记已注册到 sysfs（目录已创建）[1](@ref)
| state_add_uevent_sent:  0                | 🔸 未发送 `KOBJ_ADD` 事件 [2](@ref)
| state_remove_uevent_sent:0               | 🔸 未发送移除事件 [2](@ref)
| uevent_suppress:        0                | 🔸 允许发送 uevent 事件 [2](@ref)
+-------------------------------------------+
```

#### 2.2.3 `kobject_put()`函数

##### 2.2.3.1 函数原型

```c
void kobject_put(struct kobject *kobj);
```

##### 2.2.3.2 核心功能与定位

1. 引用计数管理：`kobject_put()`通过原子操作减少`kobj->kref`的引用计数。计数归零时，自动调用`kobj_type->release`回调函数释放资源
2. 内存释放机制
    + 若`kobject`是动态创建的(如通过`kobject_create`函数)，默认release函数为`dynamic_kobj_release`，直接`kfree(obj)`
    + 若`kobject`嵌入在父结构中(`struct device`)，release需通过`container_of`释放父结构
    ```c
    void device_release(struct kobject *kobj) {
        struct device *dev = container_of(kobj, struct device, kobj);
        kfree(dev); // 释放包含kobject的父结构
    }
    ```
    + 最终释放(`ktype->release`)：由开发者实现的回调负责释放内存

### 2.3 `kobject创建与销毁`：代码实测

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/kobject.h>

static struct kobject *my_kobject_01;
static struct kobject *my_kobject_02;
static struct kobject *my_kobject_03;

static struct kobj_type my_kobj_type;

static int __init my_init(void)
{
    my_kobject_01 = kobject_create_and_add("my_kobject_01", NULL);
    my_kobject_02 = kobject_create_and_add("my_kobject_02", my_kobject_01);

    my_kobject_03 = kzalloc(sizeof(*my_kobject_03), GFP_KERNEL);
    kobject_init_and_add(my_kobject_03, &my_kobj_type, NULL, "%s", "my_kobject_03");

    printk(KERN_INFO "make_kobj init\n");

    return 0;
}

static void __exit my_exit(void)
{
    kobject_put(my_kobject_01);
    kobject_put(my_kobject_02);
    kobject_put(my_kobject_03);

    printk(KERN_INFO "make_kobj exit\n");
}

module_init(my_init);
module_exit(my_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ding");
```

测试结果：

![](./src/0003.jpg)

## 第3章 `kset`

### 3.1 `kset`原理

Linux设备模型中的kset(内核对象集合)，是组织和管理kobject的核心容器，用于构建设备层级结构、管理热插拔事件，并在sysfs中形成逻辑目录结构。

#### 3.1.1 `kset`结构体成员详解

```c
struct kset {
	struct list_head list;      // 管理所有关联kobject的双向链表头
	spinlock_t list_lock;       // 保护链表的自旋锁（防止并发修改）
	struct kobject kobj;        // 内嵌的kobject，代表kset本身在sysfs中的目录
	const struct kset_uevent_ops *uevent_ops;   // 热插拔事件回调函数集
};
```

`kset`结构体的核心成员：

+ `list`与`list_lock`: 所有属于该kset的kobject通过`kobject->entry`嵌入此链表，形成逻辑分组
+ `kobj`: `kset`本身也是`kobject`，因此具备名称、父对象等属性。在sysfs中表现为目录
+ `uevent_ops`: 控制热插拔时间的通知行为，包含三个回调函数：
    + `filter()`: 决定是否发送事件(如过滤特定设备)
    + `name()`: 自定义事件中对象的名称
    + `uevent()`: 添加事件的环境变量

#### 3.1.2 核心应用场景

1. 设备分类管理：将同类设备组织在同一个kset下，形成逻辑分组。
    + 所有PCI设备：`bus_set`的子集`pci_set` (路径: `/sys/bus/pci/devices/`)
    + 所有输入设备：`input_kset` (路径：`/sys/class/input/`)
2. 构建设备树层次结构
    通过`kobject->parent`和`kset->kobj`构建父子关系：
    ```
    /sys/bus/pci/                 ← kset (bus_kset)
        ├── devices/              ← kset (pci_devices_kset)
        │   ├── 0000:00:1f.2/     ← kobject (PCI设备)
        └── drivers/              ← kset (pci_drivers_kset)
            ├── ahci/            ← kobject (驱动)
    ```
3. 热插拔事件管理

#### 3.1.3 `kset`与`kobject`的交互机制

1. 添加kobject到kset: 通过`kobject_add()`实现关联。实现效果: `kobject被加入kset->list链表，并在/sys/bus/pci/devices/下创建目录`

    ```c
    struct kobject *dev_kobj;
    dev_kobj->kset = pci_devices_kset; // 指定所属kset
    kobject_add(dev_kobj, NULL, "0000:00:1f.2"); // 添加到sysfs
    ```

2. 生命周期管理：`kobject`释放时 自动从`kset`链表移除

    ```c
    void kobject_put(struct kobject *kobj) {
        if (kref_put(&kobj->kref, kobject_release)) {
            list_del(&kobj->entry); // 从kset链表中移除
            sysfs_remove_dir(kobj);
        }
    }
    ```

#### 3.1.4 内核源码实例分析

    ```c
    struct kset *pci_bus_kset;
    pci_bus_kset = kset_create_and_add("pci", NULL, &bus_kset.kobj); // 创建/sys/bus/pci/

    struct kset *pci_devs_kset;
    pci_devs_kset = kset_create_and_add("devices", NULL, &pci_bus_kset->kobj); // 创建/sys/bus/pci/devices/

    pci_devs_kset->uevent_ops = &pci_uevent_ops; // 设置热插拔回调
    ```

### 3.2 `kset`与`kobject`

在2.2.2节介绍`kobject_init_and_add()`函数时，我们说这个函数可以手动绑定`kset`。回忆下当时是怎么做的：

```c
struct kobject *kobj = &my_device->kobj; // 假设kobject嵌入在自定义结构体中
kobj->kset = my_kset; // 手动绑定到目标kset
int ret = kobject_init_and_add(kobj, &my_ktype, NULL, "device%d", id);
```

OK。我们假设`kobject`已经手动绑定了`kset`，然后调用`kobject_init_and_add()`，并且`parent = NULL`。看下接下来发生什么：

```c
static int kobject_add_internal(struct kobject *kobj)
{
	int error = 0;
	struct kobject *parent;

	parent = kobject_get(kobj->parent);

	/* join kset if set, use it as parent if we do not already have one */
	if (kobj->kset) {
		if (!parent)
			parent = kobject_get(&kobj->kset->kobj);
		kobj_kset_join(kobj);
		kobj->parent = parent;
	}
}

static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
	__list_add(new, head->prev, head);
}

static void kobj_kset_join(struct kobject *kobj)
{
	if (!kobj->kset)
		return;

	kset_get(kobj->kset);
	spin_lock(&kobj->kset->list_lock);
	list_add_tail(&kobj->entry/* node */, &kobj->kset->list/* list */);
	spin_unlock(&kobj->kset->list_lock);
}
```

从上面的代码中可以看到2个重点：

1. 当手动指定了`kset`时，会把当前`kobject`的`entry`作为节点，插入到`kset->list`的链表尾部
2. 如果`parent = NULL`，会把`kobject`的`parent`父指针，指向`kset->kobj`内嵌的结构

#### 3.2.1 空`kset`初始状态

```
+---------------------------+
|        struct kset        |
|---------------------------|
| list:      ┌─────────────┐ |  
|           →│ 空链表头     │←─────── list.prev/list.next均指向自身
|            └─────────────┘ |  
|---------------------------|
| list_lock: 已初始化的自旋锁  |  ← 未上锁状态
|---------------------------|
| kobj:      ┌─────────────┐ |
|            │ name: "xxx" │←── 创建时指定的名称（如"devices"）
|            │ parent: ptr │←── 指向父kobject（参数指定）或NULL
|            │ kset: NULL  │    ← 内嵌kobject不属于其他集合
|            │ ktype:      │←── 固定为&kset_ktype（含默认release回调）
|            └─────────────┘ |
|---------------------------|
| uevent_ops: 回调函数集      |  ← 由创建参数传入
+---------------------------+
```

关键成员值：

+ `list`: 双向循环链表，`prev`和`next`都指向自身
+ `kobj.kset`: 固定为NULL，表示内嵌kobject独立存在
+ `kobj.parent`: 由`kset_create_and_add()`的父对象参数决定

#### 3.2.2 添加一个`kobject`后的状态

```
+---------------------------+        +-----------------------+
|        struct kset        |        |   struct kobject      |
|---------------------------|        |-----------------------|
| list:      ┌─────────────┐ | ┌───→│ entry.next → list     │
|           →│ 链表头       │─┘ │    │ entry.prev → list     │
|            └─────────────┘   │    | parent: &kset->kobj    |
|---------------------------|  └───┐| kset: 指向当前kset     |
| kobj:      ┌─────────────┐ |      +-----------------------+
|            │ name: "xxx" │←┘
|            │ parent: ...│          
|            │ kset: NULL │          
|            └─────────────┘          
+---------------------------+
```

关键变化：

+ `链表链接`: `kobject`通过`entry`嵌入`kset->list`，形成`链表头 ⇄ kobject`的双向链表
+ `父子关系`:
    + `kobject->parent`: 指向kset的内嵌`kobject`
    + `kobject->set`: 指向当前`kset`，标识归属关系

#### 3.2.3 添加二个`kobject`后的状态

```
+---------------------------+        +-----------------------+      +-----------------------+
|        struct kset        |        |   kobject A           |      |   kobject B           |
|---------------------------|        |-----------------------|      |-----------------------|
| list:      ┌─────────────┐ | ┌───→│ entry.next → B.entry  │┌───→│ entry.next → list     │
|           →│ 链表头       │─┘ │    │ entry.prev → list     ││    │ entry.prev → A.entry  │
|            └─────────────┘   │    | parent: &kset->kobj    ││    | parent: &kset->kobj    |
|---------------------------|   └───┐| kset: 当前kset        │└───┐| kset: 当前kset        |
| kobj: (同上)              |        +-----------------------+      +-----------------------+
+---------------------------+
```

关键变化：

+ `链表扩展`: 新增`kobject`插入链表尾部，形成循环链表(`链表头 ⇄ kobject A ⇄ kobject B ⇄ 链表头`)
+ `统一父对象`: 所有`kobject`的`parent`均指向同一个`kset->kobj`，在sysfs中表现为同级目录(如`/sys/bus/pci/devices/`下的设备)

#### 3.2.4 添加三个`kobject`后的状态

```
+---------------------------+        +-----------------------+      +-----------------------+      +-----------------------+
|        struct kset        |        |   kobject A           |      |   kobject B           |      |   kobject C           |
|---------------------------|        |-----------------------|      |-----------------------|      |-----------------------|
| list:      ┌─────────────┐ | ┌───→│ entry.next → B.entry  │┌───→│ entry.next → C.entry  │┌───→│ entry.next → list     │
|           →│ 链表头       │─┘ │    │ entry.prev → list     ││    │ entry.prev → A.entry  ││    │ entry.prev → B.entry  │
|            └─────────────┘   │    | parent: &kset->kobj    ││    | parent: &kset->kobj    ││    | parent: &kset->kobj    |
|---------------------------|   └───┐| kset: 当前kset        │└───┐| kset: 当前kset        │└───┐| kset: 当前kset        |
| kobj: (同上)              |        +-----------------------+      +-----------------------+      +-----------------------+
+---------------------------+
```

关键变化：

+ `完整循环链表`: 链表结构变为`链表头 ⇄ A ⇄ B ⇄ C ⇄ 链表头`
+ `并发保护`: `list_lock`自旋锁确保多核环境下链表操作的原子性(如并发添加/删除)

### 3.3 `kset_create_and_add()`函数：创建`kset`

```c
struct kset *kset_create_and_add(
    const char *name,                           // [IN] kset在sysfs中的目录名
    const struct kset_uevent_ops *uevent_ops,   // [IN] uevent事件回调函数集
    struct kobject *parent_kobj                 // [IN] 父kobject指针（决定sysfs层级）
);
```

1. `name`: 指定`kset`在sysfs中对应的目录名称，如`"devices"、"block"`
    + 赋值逻辑：通过`kobject_set_name(&kset->jobj, name)`设置内嵌`kobject`的`name`字段
2. `uevent_ops`: 暂时忽略
3. `parent_kobj`: 指定`kset`在`sysfs`中的父目录。若为NULL，则创建在`/sys`根目录下
    + `parent_kobj = kernel_kobj`: 目录位于`/sys/kernel/`
    + `parent_kobj = NULL`： 目录位于`/sys/`

### 3.4 代码实测

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/kobject.h>

static struct kset *my_kset;

static struct kobject *my_kobj_01;
static struct kobject *my_kobj_02;

static struct kobj_type my_ktype;

static int __init my_init(void)
{
    my_kset = kset_create_and_add("my_kset", NULL, NULL);

    my_kobj_01 = kzalloc(sizeof(struct kobject), GFP_KERNEL);
    my_kobj_02 = kzalloc(sizeof(struct kobject), GFP_KERNEL);

    my_kobj_01->kset = my_kset;
    my_kobj_02->kset = my_kset;

    kobject_init_and_add(my_kobj_01, &my_ktype, NULL, "%s", "my_kobj_01");
    kobject_init_and_add(my_kobj_02, &my_ktype, NULL, "%s", "my_kobj_02");

    printk(KERN_INFO "make_kset init\n");

    return 0;
}

static void __exit my_exit(void)
{
    kobject_put(my_kobj_01);
    kobject_put(my_kobj_02);

    printk(KERN_INFO "make_kset exit\n");
}

module_init(my_init);
module_exit(my_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ding");
```

测试结果：

![](./src/0004.jpg)

## 第4章 为什么要引入设备模型

Linux设备模型，是内核用于统一管理硬件设备的框架，其核心包含：总线(Bus)、设备(Device)、驱动(Driver)和类(Class)四个部分。他们共同构建了设备之间的层次关系，支持热插拔、电源管理、sysfs用户交互等功能。

### 4.1 四个核心组件

| 组件 | 定义与作用 | 内核数据结构 | 在sysfs中的位置 |
| - | - | - | - |
| 总线(Bus) | 设备与CPU之间的通信通道(物理或虚拟)，负责管理挂载其上的设备和驱动，并实现二者的匹配。例如PC、USB、I2C总线，以及虚拟的platform总线 | `struct bus_type` | `/sys/bus/` 如(`/sys/bus/pci`) |
| 设备(Device) | 硬件设备的抽象描述(物理或逻辑设备)，包含设备属性、资源(内存地址、IRQ)，所属总线及驱动信息 | `struct device` | `/sys/devices/(按总线层次组织)` |
| 驱动(Driver) | 控制设备的软件模块，实现设备的初始化(probe)、销毁(remove)、电源管理等操作。与设备通过总线匹配后绑定 | `struct device_driver` | `/sys/bus/(总线名)/drivers` 如`/sys/bus/i2c/drivers/tmp102/` |
| 类(Class) | 按功能分配设备的抽象层(如输入设备、显示设备)，提供跨总线的统一接口，简化用户空间操作 | `struct class` | `/sys/class` 如`/sys/class/input/` |

层次结构：总线(`/sys/bus/`)是根节点，设备和驱动挂载在总线下。

![](./src/0005.jpg)

1. 注册与匹配
    + 总线注册(`bus_register`)后，设备和驱动分别通过`device_register`、`driver_register`挂载到总线
    + 总线的`match()`函数比较设备ID(如设备的`compatible`值)与驱动支持的ID表，匹配成功则调用驱动的`probe()`
2. 用户空间交互
    + 通过sysfs暴露设备属性(如`/sys/class/net/eth0/speed`)，用户可直接读写配置
3. 热插拔与电源管理
    + 设备插入时，总线触发`uevent`生成`ACTION=add`事件，通知用户空间(如udev加载驱动)
    + 电源管理时，总线按依赖顺序开关设备(如先启总线再启设备)

### 4.2 总线结构体`struct bus_type`

```c
struct bus_type {
	const char		*name;
	const char		*dev_name;
	struct device		*dev_root;
	struct device_attribute	*dev_attrs;	/* use dev_groups instead */
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	int (*remove)(struct device *dev);
	void (*shutdown)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);
};
```

结构体成员详解：

+ `name`: 总线名称(如`i2c`、`platform`)，对应`/sys/bus/`下的目录名。命名需唯一且不含特殊字符
+ `dev_name`: 设备命名模板(如`i2c-%d`)，用于自动生成未命名设备的名称(如`I2C适配器编号`)
+ `dev_root`: 默认父设备指针，作为新注册设备的父对象(如`platform_bus`作为所有`platform_device`的父设备)
+ `dev_attrs/bus_groups/dev_groups/drv_groups`: 定义总线和设备的默认`sysfs`属性
+ `match`: 设备与驱动匹配的核心逻辑。当设备或驱动注册时调用，返回非0标识匹配成功
    典型实现：
    ```c
    // Platform 总线匹配（drivers/base/platform.c）
    static int platform_match(struct device *dev, struct device_driver *drv) {
        // 检查设备树 compatible 或 ACPI ID
        return of_driver_match_device(dev, drv);
    }
    ```
+ `uevent`: 生成热插拔事件的环境变量(如设备插入时添加`ACTION=add`)，如usb设备插入时触发udev加载驱动
+ `probe`: 设备匹配后初始化，调用驱动的probe函数
+ `remove/shutdown`: 设备移除或关机时的清理操作，如释放资源、断电
+ `suspend/resume`" 设备休眠唤醒时协调电源状态

### 4.3 设备结构体`struct device`

```c
struct device {
	struct device		*parent;
	struct kobject kobj;
	const char		*init_name; /* initial name of the device */
	const struct device_type *type;
	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */
	struct device_node	*of_node; /* associated device tree node */
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	struct class		*class;
};
```

+ `struct device *parent`: 指向父设备(如总线控制器)。设备树中层级关系通过parent构建，例如：
    ```dts
    &i2c1 { // 父设备（I²C 控制器）
        sensor@48 { // 子设备
            compatible = "ti,tmp102";
        };
    };
    ```
    内核中，sensor@48的parent指向i2c1的设备实例
+ `struct kobject kobj`: 在`/sys`生成设备文件。如`/sys/devices/platform/my-device`，用于暴露设备属性
+ `const char *init_name`: 设备初始名称(如`tmp102`)。若未指定，内核自动生成`"bus_id"`
+ `struct bus_type *bus`: 设备所属总线(如`&i2c_bus_type`)。总线定义设备和驱动的匹配规则
+ `struct device_driver *driver`: 绑定后的驱动指针。当总线`match()`成功(如设备树`compatible`匹配驱动的`of_match_table`)，内核调用驱动的`probe()`
+ `void *platform_data`: 板级私有数据(如GPIO配置)。嵌入式开发中常用，避免硬编码
    ```c
    // 板级文件
    static struct tmp102_platform_data my_board_data = {
        .gpio_int = 47,
    };
    platform_device_register_data(dev, &my_board_data);
    ```
+ `struct device_node *of_node`: 指向设备树节点，驱动通过`of_`函数解析资源
    ```c
    // 驱动中解析设备树
    ret = of_property_read_u32(dev->of_node, "reg", &reg_addr); // 获取寄存器地址
    irq = irq_of_parse_and_map(dev->of_node, 0); // 解析中断号
    ```
+ `struct list_head	devres_head`: 设备资源链表(内存、IO)，通过`devm_API`自动管理
    ```c
    res = devm_ioremap_resource(dev, regs); // 自动释放的寄存器映射
    ```
+ `struct class *class`: 设备分类(如`&input_class`)。所有键盘归入`/sys/class/input/`，统一生成uevent事件
+ `dev_t devt`: 设备号(主/次设备号)，用于创建设备文件`/dev/tmp102`
    ```c
    devt = MKDEV(MAJOR_NUM, MINOR_NUM);
    device_create(class, dev, devt, NULL, "tmp102");
    ```

### 4.4 驱动结构体`struct device_driver`

```c
struct device_driver {
	const char		*name;
	struct bus_type		*bus;
	struct module		*owner;
	const struct of_device_id	*of_match_table;

	int (*probe) (struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;
};
```

+ `const char *name`: 驱动名称(如`i2c-tmp102`)，在sysfs中生成`/sys/bus/<所属总线>/drivers/<name>`目录。命名需唯一，用于匹配或调试
+ `struct bus_type *bus`: 指向驱动所属的总线(如`&i2c_bus_type`)。总线负责管理设备与驱动的匹配规则
+ `struct module *owner`: 指向驱动所属的内核模块(如`THIS_MODELE`)，用于模块引用计数管理
+ `const struct of_device_id *of_match_table`: 设备树匹配表
+ `probe/remove/shutdown/suspend/resume`: 驱动的回调函数指针
+ `const struct attribute_group **groups`: 驱动默认属性组，在sysfs中自动生成文件。例如驱动暴露版本

看到这里我们应该思考一个问题。`platform_driver`与`device_driver`有什么关联？

看下`platform_driver`的结构体，和注册函数。做了以下内容：

1. 把驱动所属的总线，设为`platform_bus`
2. 把`platform_driver`的几个函数指针，设置给`device_driver`

```c
struct platform_driver {
	int (*probe)(struct platform_device *);
	int (*remove)(struct platform_device *);
	void (*shutdown)(struct platform_device *);
	int (*suspend)(struct platform_device *, pm_message_t state);
	int (*resume)(struct platform_device *);
	struct device_driver driver;
	const struct platform_device_id *id_table;
	bool prevent_deferred_probe;
};

int __platform_driver_register(struct platform_driver *drv,
				struct module *owner)
{
	drv->driver.owner = owner;
	drv->driver.bus = &platform_bus_type;
	if (drv->probe)
		drv->driver.probe = platform_drv_probe;
	if (drv->remove)
		drv->driver.remove = platform_drv_remove;
	if (drv->shutdown)
		drv->driver.shutdown = platform_drv_shutdown;

	return driver_register(&drv->driver);
}
```

### 4.5 类结构体`struct class`

```c
struct class {
	const char		*name;
	struct module		*owner;

	struct class_attribute		*class_attrs;
	const struct attribute_group	**dev_groups;
	struct kobject			*dev_kobj;

	int (*dev_uevent)(struct device *dev, struct kobj_uevent_env *env);
	char *(*devnode)(struct device *dev, umode_t *mode);

	void (*class_release)(struct class *class);
	void (*dev_release)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);
};
```

+ `const char *name`: 类名称(如`leds`、`input`)，对应`/sys/class/`下的目录名。命名需唯一，用于用户空间分类访问设备
+ `struct module *owner`: 指向拥有该类的内核模块(如`THIS_MODULE`)，用于模块引用计数管理
+ `class_attrs/dev_groups`: 属性文件
+ `char *(*devnode)(struct device *dev, umode_t *mode)`: 自定义设备节点路径(如`/dev/input/event0`)。默认在`/dev/`创建设备，可通过此回调修改路径或权限

### 4.6 `/sys`目录实例分析

我们进入到Linux系统的`/sys`目录下，可以看到如下文件夹。其中和设备模型有关的文件夹为：

![](./src/0006.jpg)

+ `/sys/devices`: 该目录包含了系统中所有设备的子目录。每个设备子目录代表一个具体的设备，通过其路径层次结构和符号链接，反应设备的关系和拓扑结构

    ![](./src/0007.jpg)

+ `/sys/bus`: 该目录包含了总线类型的子目录。每个子目录代表一个特定类型的总线，例如`i2c`、`spi`、`platform`。每个总线子目录中，包含与该总线相关的设备和驱动程序的信息

    ![](./src/0008.jpg)

+ `/sys/class`: 该目录包含了设备类别的子目录。每个子目录代表一个设备类别，例如磁盘、网络接口等。每个设备类别子目录中，包含了属于该类别的设备的信息

    ![](./src/0009.jpg)

### 4.7 设备模型4部分的连接方式

在Linux的sysfs文件系统中，`/sys/class`、`/sys/devices`和`/sys/bus`之间的关联，主要通过符号链接建立。这种设计实现了设备模型的多视角组织：物理拓扑(`/sys/devices`)、功能分类(`/sys/class`)、总线类型(`/sys/bus`)

#### 4.7.1 核心目录的作用与关联方式

以`ethernet`有线网为例说明。我们先看下设备树：有线网设备位于`/soc/aips2(aips-bus@02100000)/fec1(ethernet@02188000)/`这条总线下。

![](./src/0010.jpg)

1. `/sys/devices`: 物理设备的真实存储位置。这是Linux设备模型的核心目录，按硬件连接的物理层级(如总线、父设备)组织起所有设备。例如：
    + 网卡设备路径：`/sys/devices/platform/soc/2100000.aips-bus/2188000.ethernet`
    ![](./src/0011.jpg)
2. `/sys/class`: 按功能分类的符号链接视图
    + 作用：将设备按功能(如输入设备、网络设备)分类，不关心物理连接方式
    + 符号链接：每个设备子目录(如`/sys/class/leds/led1`)中的`device`文件，是指向`/sys/devices`中真实设备的符号链接
    ![](./src/0012.jpg)
3. `/sys/bus`: 按总线类型组织的符号链接试图
    + 作用：按总线类型(如PCI、USB、I2C)组织设备和驱动
    + 符号链接：`/sys/bus/<总线名>/devices`子目录中的条目，是指向`/sys/devices`的符号链接
    ![](./src/0013.jpg)

#### 4.7.2 关联关系图示

![](./src/0014.jpg)

#### 4.7.3 关键点总结

1. 符号链接是核心纽带：`/sys/class`和`/sys/bus/devices`中的条目，均通过符号链接指向`/sys/devices`中的真实设备目录
2. 设计目的：
    + `/sys/devices`: 描述硬件物理层级(如设备依赖关系)
    + `/sys/class`: 提供功能抽象(如统一控制所有LED)
    + `/sys/bus`: 管理总线相关的匹配与驱动绑定
3. 用户空间操作示例：
    + 通过`/sys/class/leds/led1/brightness`控制LED(功能视图)
    + 通过`/sys/bus/pci/drivers/nvidia/bind`手动绑定设备(总线视图)

## 第5章 `kref`引用计数器

### 5.1 什么是引用计数

引用计数是一种内存管理技术，用于跟踪对象或资源的应用数量。它通过在对线被引用时增加计数值，并在引用被减少时释放计数值，以确定何时可以安全的释放对象或资源。

### 5.2 `kref引用计数器`

`kref`是Linux内核中提供的一种引用计数器实现，它是一种轻量级的引用计数技术，用于管理内核中的对象的引用计数。

```c
struct kref {
	atomic_t refcount;
};
```

在使用引用计数器时，通常会将结构体`kref`嵌入到其他结构体中，例如`struct kobject`，以实现引用计数的管理。

```c
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject		*parent;
	struct kset		*kset;
	struct kobj_type	*ktype;
	struct kernfs_node	*sd;
	struct kref		kref;   // 内嵌kref引用计数器
};
```

### 5.3 `kref`常用API函数

#### 5.3.1 `kref_init()`函数：初始化引用计数为1

```c
static inline void kref_init(struct kref *kref)
{
	atomic_set(&kref->refcount, 1);
}
```

#### 5.3.2 `kref_get()`函数：引用计数+1

```c
static inline void kref_get(struct kref *kref)
{
	WARN_ON_ONCE(atomic_inc_return(&kref->refcount) < 2);
}
```

#### 5.3.3 `kref_put()`函数：引用计数-1

```c
static inline int kref_put(struct kref *kref,
	     void (*release)(struct kref *kref))
{
	if (atomic_sub_and_test((int) 1, &kref->refcount)) {
		release(kref);
		return 1;
	}
	return 0;
}
```

### 5.4 引用计数的典型使用场景

#### 5.4.1 多线程数据数据传递

当对象需跨线程传递时，需在传递前增加引用计数，接收方使用后减少引用计数

```c
void worker_thread(void *data) {
    struct my_data *d = data;
    // 操作数据
    kref_put(&d->refcount, data_release);  // 使用完毕释放引用
}

void create_thread() {
    struct my_data *data = kmalloc(..., GFP_KERNEL);
    kref_init(&data->refcount);
    kref_get(&data->refcount);  // 传递前增加引用
    kthread_run(worker_thread, data, "worker");  // 传递指针
    // ... 主线程操作
    kref_put(&data->refcount, data_release);  // 主线程释放引用
}
```

#### 5.4.2 设备驱动资源管理

设备驱动中，`kref`用于跟踪设备的打开次数。*每次open()时初始化计数，每次close()时减少计数，确保无人使用时释放硬件资源。*

```c
struct device_ctx {
    struct kref ref;
    void *hw_reg;
};

static void release_dev(struct kref *kref) {
    struct device_ctx *dev = container_of(kref, struct device_ctx, ref);
    iounmap(dev->hw_reg);  // 解除内存映射
    kfree(dev);
}

int device_open() {
    struct device_ctx *dev = kmalloc(..., GFP_KERNEL);
    kref_init(&dev->ref);
    dev->hw_reg = ioremap(...);
    return 0;
}

void device_close(struct device_ctx *dev) {
    kref_put(&dev->ref, release_dev);  // 关闭时减少引用
}
```

### 5.5 使用规则与陷阱

#### 5.5.1 三条核心规则

1. 传递持久指针前必增计数：若对象指针需长期保存或跨线程传递，必须先调用`kref_get()`
2. 使用完毕后必减计数：通过`kref_put()`释放引用，归零时自动清理
3. 未持有是获取引用需加锁：若代码未持有对象指针却需获取引用(如从链表获取)，必须用锁同步`kref_get`和`kref_put`

#### 5.5.2 常见陷阱

1. 计数不匹配：`init/get`和`put`要成对出现 -> 内存泄露或提前释放
2. 错误释放函数：`release`中未正确使用`container_of` -> 内存损坏
3. 并发漏洞：未加锁保护共享对象的引用操作 -> 竞态条件

### 5.6 运行测试

测试代码：

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/kobject.h>

static struct kobject *my_kobject_01;
static struct kobject *my_kobject_02;
static struct kobject *my_kobject_03;

static struct kobj_type my_kobj_type;

static int __init my_init(void)
{
    my_kobject_01 = kobject_create_and_add("my_kobject_01", NULL);
    printk(KERN_INFO "my_kobject_01 kref:%d\n", my_kobject_01->kref.refcount.counter);

    my_kobject_02 = kobject_create_and_add("my_kobject_02", my_kobject_01);
    printk(KERN_INFO "my_kobject_01 kref:%d\n", my_kobject_01->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_02 kref:%d\n", my_kobject_02->kref.refcount.counter);

    my_kobject_03 = kzalloc(sizeof(*my_kobject_03), GFP_KERNEL);
    kobject_init_and_add(my_kobject_03, &my_kobj_type, NULL, "%s", "my_kobject_03");
    printk(KERN_INFO "my_kobject_03 kref:%d\n", my_kobject_03->kref.refcount.counter);

    return 0;
}

static void __exit my_exit(void)
{
    printk(KERN_INFO "my_kobject_01 kref:%d\n", my_kobject_01->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_02 kref:%d\n", my_kobject_02->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_03 kref:%d\n", my_kobject_03->kref.refcount.counter);

    kobject_put(my_kobject_01);
    printk(KERN_INFO "my_kobject_01 kref:%d\n", my_kobject_01->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_02 kref:%d\n", my_kobject_02->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_03 kref:%d\n", my_kobject_03->kref.refcount.counter);

    kobject_put(my_kobject_02);
    printk(KERN_INFO "my_kobject_01 kref:%d\n", my_kobject_01->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_02 kref:%d\n", my_kobject_02->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_03 kref:%d\n", my_kobject_03->kref.refcount.counter);

    kobject_put(my_kobject_03);
    printk(KERN_INFO "my_kobject_01 kref:%d\n", my_kobject_01->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_02 kref:%d\n", my_kobject_02->kref.refcount.counter);
    printk(KERN_INFO "my_kobject_03 kref:%d\n", my_kobject_03->kref.refcount.counter);
}

module_init(my_init);
module_exit(my_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ding");
```

测试结果：

![](./src/0015.jpg)

## 第6章 `kobject`释放实例分析

通过上个章节的实验，我们已经知道引用计数器是如何工作的。当引用计数器的值为0时，会自动调用自定义的释放函数去执行释放的操作。

### 6.1 创建`kobject`：增加引用计数

我们分析`kobject_create_and_add()`函数，看下引用计数是怎么设置的：

1. `kobject_create_and_add -> kobject_create -> kobject_init -> kobject_init_internal -> kref_init(&kobj->kref)`: 把`kobject`对象的`kref`初始化为1
2. `kobject_create_and_add -> kobject_add -> kobject_add_varg -> kobject_add_internal -> kobject_get(kobj->parent)`: 把`kobject`的`parent`对象的`kref`加1

```c
struct kobject *kobject_create_and_add(const char *name, struct kobject *parent)
{
	struct kobject *kobj;
	int retval;

	kobj = kobject_create();
	retval = kobject_add(kobj, parent, "%s", name);

	return kobj;
}

struct kobject *kobject_create(void)
{
	struct kobject *kobj;

	kobj = kzalloc(sizeof(*kobj), GFP_KERNEL);
	kobject_init(kobj, &dynamic_kobj_ktype);

	return kobj;
}

void kobject_init(struct kobject *kobj, struct kobj_type *ktype)
{
	kobject_init_internal(kobj);
	kobj->ktype = ktype;
	return;
}

static void kobject_init_internal(struct kobject *kobj)
{
	kref_init(&kobj->kref);
}

int kobject_add(struct kobject *kobj, struct kobject *parent,
		const char *fmt, ...)
{
	retval = kobject_add_varg(kobj, parent, fmt, args);
	return retval;
}

static int kobject_add_varg(struct kobject *kobj, struct kobject *parent,
			    const char *fmt, va_list vargs)
{
	kobj->parent = parent;
	return kobject_add_internal(kobj);
}

static int kobject_add_internal(struct kobject *kobj)
{
	struct kobject *parent;
	parent = kobject_get(kobj->parent);
}
```

动态创建`kobject`对象的函数：`kobject_create()`。使用的`kobj_type`为`dynamic_kobj_ktype`，最终会用它来释放内存。

```c
static void dynamic_kobj_release(struct kobject *kobj)
{
	kfree(kobj);
}

static struct kobj_type dynamic_kobj_ktype = {
	.release	= dynamic_kobj_release,
};

struct kobject *kobject_create(void)
{
	struct kobject *kobj;

	kobj = kzalloc(sizeof(*kobj), GFP_KERNEL);
	kobject_init(kobj, &dynamic_kobj_ktype);

	return kobj;
}
```

### 6.2 释放`kobject`：减少引用计数，释放内存

我们调用`kobject_put()`函数来释放`kobject`内存。下面来分析源码，看看是怎么做到的：

`kobject_put -> kref_put -> kobject_release -> kobj->release(dynamic_kobj_release) -> kfree(kobj)`：最终调用`kfree(kobj)`释放内存

```c
void kobject_put(struct kobject *kobj)
{
	if (kobj) {
		kref_put(&kobj->kref, kobject_release); // 减少引用计数
	}
}

int kref_put(struct kref *kref, void (*kobject_release)(struct kref *kref))
{
	if (atomic_sub_and_test((int) 1, &kref->refcount)) {
		kobject_release(kref);  // 引用计数减到0时，调用释放内存函数
		return 1;
	}
	return 0;
}

static void kobject_release(struct kref *kref)
{
	struct kobject *kobj = container_of(kref, struct kobject, kref);
	kobj->release(kobj);    // 调用 dynamic_kobj_release 函数
}

static void dynamic_kobj_release(struct kobject *kobj)
{
	kfree(kobj);    // 调用kfree函数
}
```

### 6.3 总结

+ `kobject_create()`创建`kobj`：调用`kzalloc`申请内存给`kobj`
    ```c
    struct kobject *kobject_create(void)
    {
        struct kobject *kobj;

        kobj = kzalloc(sizeof(*kobj), GFP_KERNEL);
        kobject_init(kobj, &dynamic_kobj_ktype);

        return kobj;
    }
    ```

+ `kobject_put()`释放`kobj`：调用`kfree`释放`kobj`的内存
    ```c
    void kobject_put(struct kobject *kobj)
    {
        // kobj->release(kobj)  ->   dynamic_kobj_release(kobj)
        kfree(kobj);
    }
    ```

### 6.4 代码实测

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/kobject.h>

static void my_release(struct kobject *kobj);

static struct kobject *my_kobject;

static struct kobj_type my_kobj_type = {
    .release = my_release
};

static void my_release(struct kobject *kobj)
{
    printk(KERN_INFO "release kobj:%p\n", kobj);
    kfree(kobj);
}

static int __init my_init(void)
{
    my_kobject = kzalloc(sizeof(*my_kobject), GFP_KERNEL);
    kobject_init_and_add(my_kobject, &my_kobj_type, NULL, "%s", "my_kobject");

    printk(KERN_INFO "make_kobj init\n");

    return 0;
}

static void __exit my_exit(void)
{
    kobject_put(my_kobject);
    printk(KERN_INFO "make_kobj exit\n");
}

module_init(my_init);
module_exit(my_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ding");
```

测试结果：

![](./src/0016.jpg)

## 第7章 `container_of`宏定义

`container_of`是Linux内核中，通过结构体成员地址，反向获取整个结构体起始地址的核心宏，在链表、设备驱动等场景广泛应用。

### 7.1 参数详解

```c
#define container_of(ptr, type, member)
```

+ `ptr`: 成员指针. 指向结构体*成员变量*的地址
+ `type`: 结构体类型. 包含该成员的*结构体类型名*
+ `member`: 成员名称. 成员在结构体中的*字段名*
+ `返回值`: 指向结构体首地址的*结构体类型指针*

### 7.2 典型用法

#### 7.2.1 内核链表遍历(最常见场景)

作用：通过链表结点指针pos，反向定位包含它的sensor结构体

```c
struct list_head {
    struct list_head *next, *prev;
};

struct sensor {
    int id;
    struct list_head list;  // 嵌入链表节点
};

// 遍历链表获取 sensor 结构体
struct list_head *pos;
list_for_each(pos, &sensor_list) {
    struct sensor *s = container_of(pos, struct sensor, list);
    printk("Sensor ID: %d\n", s->id);
}
```

#### 7.2.2 设备驱动开发

作用：通过cdev成员指针获取自定义设备结构，实现多设备实例管理

```c
struct my_device {
    struct cdev cdev;  // 字符设备
    int irq;
};

static int dev_open(struct inode *inode, struct file *filp) {
    struct my_device *dev = container_of(inode->i_cdev, struct my_device, cdev);
    filp->private_data = dev;  // 存储设备结构体指针
}
```

## 第8章 `kobj`创建属性文件并实现读写功能

在第6章中，我们实现了自定义`kobj`的`release`释放函数，在创建`kobj`时设置生效。这样做的作用是，当`kobj`引用计数为0时，可以自动释放内存。

```c
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;
	struct attribute **default_attrs;
};

static struct kobj_type my_kobj_type = {
    .release = my_release
};

static void my_release(struct kobject *kobj)
{
    printk(KERN_INFO "release kobj:%p\n", kobj);
    kfree(kobj);
}

static int __init my_init(void)
{
    my_kobject = kzalloc(sizeof(*my_kobject), GFP_KERNEL);
    kobject_init_and_add(my_kobject, &my_kobj_type, NULL, "%s", "my_kobject");

    printk(KERN_INFO "make_kobj init\n");

    return 0;
}
```

实际上，`struct kobj_type`结构体除了`release`外，还有2个关键成员：`sysfs_ops`和`default_attrs`，这就是用来创建和读写`kobj`的属性文件的。接下来我们仔细分析！

### 8.1 代码实测

我们先写一个最简单的代码，来创建和读写属性文件。看看效果，下一节再去分析实现原理！

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/slab.h>
#include <linux/kobject.h>

struct my_kobj {
    struct kobject my_kobj;
    int value1;
    int value2;
};

static struct my_kobj *s_my_kobj;

static struct attribute value1 = {
    .name = "value1",
    .mode = 0666
};

static struct attribute value2 = {
    .name = "value2",
    .mode = 0666
};

static struct attribute *my_attrs[] = {
    &value1,
    &value2,
    NULL
};

static void my_release(struct kobject *kobj)
{
    struct my_kobj *p_my_kobj = container_of(kobj, struct my_kobj, my_kobj);
    printk(KERN_INFO "release attr_01\n");
    kfree(p_my_kobj);
}

static ssize_t my_attr_show(struct kobject *kobj, struct attribute *attr, char *buf)
{
    ssize_t count = 0;
    struct my_kobj *p_my_kobj = container_of(kobj, struct my_kobj, my_kobj);

    if (strcmp(attr->name, "value1") == 0) {
        count = sprintf(buf, "%d\n", p_my_kobj->value1);
    }
    else if (strcmp(attr->name, "value2") == 0) {
        count = sprintf(buf, "%d\n", p_my_kobj->value2);
    }

    return count;
}

static ssize_t my_attr_store(struct kobject *kobj, struct attribute *attr, const char *buf, size_t size)
{
    struct my_kobj *p_my_kobj = container_of(kobj, struct my_kobj, my_kobj);

    if (strcmp(attr->name, "value1") == 0) {
        sscanf(buf, "%d", &p_my_kobj->value1);
    }
    else if (strcmp(attr->name, "value2") == 0) {
        sscanf(buf, "%d", &p_my_kobj->value2);
    }
    
    return size;
}

static const struct sysfs_ops my_sysfs_ops = {
    .show  = my_attr_show,
	.store = my_attr_store,
};

static struct kobj_type my_kobj_type = {
    .release        = my_release,
    .sysfs_ops      = &my_sysfs_ops,
    .default_attrs  = my_attrs
};

static int __init my_init(void)
{
    s_my_kobj = kzalloc(sizeof(struct my_kobj), GFP_KERNEL);
    s_my_kobj->value1 = 1;
    s_my_kobj->value2 = 2;

    kobject_init_and_add(&s_my_kobj->my_kobj, &my_kobj_type, NULL, "%s", "attr_01");
    printk(KERN_INFO "attr_01 init\n");

    return 0;
}

static void __exit my_exit(void)
{
    kobject_put(&s_my_kobj->my_kobj);
    printk(KERN_INFO "attr_01 exit\n");
}

module_init(my_init);
module_exit(my_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ding");
```

测试结果：

![](./src/0017.jpg)

### 8.2 属性文件原理分析


