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

#### 3.2.1 关系

// TODO

#### 3.2.2 空`kset`初始状态

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

#### 3.2.3 添加一个`kobject`后的状态

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

#### 3.2.4 添加二个`kobject`后的状态

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

#### 3.2.5 添加三个`kobject`后的状态

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


