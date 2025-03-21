# 字符设备基础

## 第1章 申请字符设备号

### 1.1 理论讲解

**什么是设备号？**

linux给定每一个字符设备或者块设备，都必须有一个专属的设备号。一个设备号由主设备号和次设备号组成。

1. 主设备号用来表示某一类驱动，如键盘、鼠标都可以归类到USB驱动
2. 而次设备号用来表示这个驱动下的各个设备。比如第几个鼠标，第几个键盘等

所以，我们开发字符驱动程序，申请设备号是第一步，只有有了设备号，才可以向系统注册设备。

**设备号的类型**

Linux中使用`dev_t`的数据类型表示设备号。`dev_t`定义在`linux/types.h`里面，为`uint32`类型。

`linux/types.h`，设备号是一个32位的数据类型。

```c
typedef __u32 __kernel_dev_t;

typedef __kernel_dev_t		dev_t;  // u32类型
```

`linux/kdev_t.h`，高12位为主设备号`(MAJOR)`，低20位为次设备号`(MINOR)`。

```c
#define MINORBITS	20
#define MINORMASK	((1U << MINORBITS) - 1)

#define MAJOR(dev)	((unsigned int) ((dev) >> MINORBITS))
#define MAJOR(dev)	((unsigned int) ((dev) & MINORMASK))
#define MKDEV(ma,mi)	(((ma) << MINORBITS) | (mi))
```

`MAJOR`用于取高12位，`MAJOR`用于取第20位。`MKDEV`用于拼接设备号。

**设备号分配**

在编写字符设备驱动代码时，可以静态分配设备号，或动态分配设备号。

1. 查看系统已使用的设备号：`cat /proc/devices`
2. 静态分配设备号：开发者自己指定一个设备号，比如选择50作为主设备号。有些设备号已经被系统使用了，在指定设备号时就不能再使用了
3. 动态分配设备号：徐通自动给我们选择一个还没被使用的设备号

`linux/fs.h`，分配设备号的函数：

1. 静态分配设备号函数：`int register_chrdev_region(dev_t from, unsigned count, const char *name)`
   
   + `dev_t from`：起始设备号，通过MKDEV(major, minor)生成。`major`主设备号，`minor`起始次设备号(通常从0开始)
   + `unsigned count`：连续注册的次设备号数量
   + `const char *name`：设备名称，显示在`/proc/devices`，用于用户空间识别设备

2. 动态分配设备号函数：`int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)`

    + `dev_t *dev`：保存内核自动分配的设备号，通过指针返回
    + `unsigned baseminor`：起始次设备号：通常设为0
    + `unsigned count`：连续注册的次设备号数量
    + `const char *name`：设备名称，显示在`/proc/devices`，用于用户空间识别设备

3. 释放字符设备号函数：`void unregister_chrdev_region(dev_t from, unsigned count)`

    + `dev_t from`：起始设备号，需要与注册使用的`dev_t`完全一致
    + `unsigned count`：要释放的连续次设备号数量，需与注册时指定的数量一致

### 1.2 代码编写

我们写一段模块代码，来注册设备号。可以接受用户传参来设置设备号，如果没主动设置，就自动生成设备号。

`dev_t.c`

```c
#include <linux/init.h>			/* module_init, module_exit */
#include <linux/module.h>		/* MODULE_LISENCE, MODULE_AUTHOR */
#include <linux/moduleparam.h>	/* module_param */
#include <linux/types.h>		/* dev_t */
#include <linux/kdev_t.h>		/* MAJOR, MINOR, MKDEV */
#include <linux/fs.h>			/* alloc_chrdev_region, unregister_chrdev_region */

#define CHRDEV_NAME		"chr_test"

static int major = 0;
static int minor = 0;

module_param(major, int, S_IRUGO);
module_param(minor, int, S_IRUGO);

static __init int module_param_init(void)
{
	int ret;
	dev_t dev;

	if (major) {
		printk("major:%d, minor:%d\n", major, minor);
		dev = MKDEV(major, minor);
		ret = register_chrdev_region(dev, 1, CHRDEV_NAME);
		if (ret < 0) {
			printk("register_chrdev_region error\n");
			return ret;
		}
		else {
			printk("register_chrdev_region OK\n");
			printk("dev:%d\n", dev);
		}
	}
	else {
		ret = alloc_chrdev_region(&dev, 0, 1, CHRDEV_NAME);
		if (ret < 0) {
			printk("alloc_chrdev_region error\n");
			return ret;
		}
		else {
			printk("alloc_chrdev_region OK\n");
			major = MAJOR(dev);
			minor = MINOR(dev);
			printk("major:%d, minor:%d\n", major, minor);
			printk("dev:%d\n", dev);
		}
	}	

	return 0;
}

static __exit void module_param_exit(void)
{
	dev_t dev = MKDEV(major, minor);

	unregister_chrdev_region(dev, 1);
	printk("unregister_chrdev_region\n");
}

module_init(module_param_init);
module_exit(module_param_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ding");
```

### 1.3 实验测试

接下来我们要测试这个驱动程序。该怎么测试呢？

*回想一下，理论部分提到，我们注册的设备号会显示在`/proc/devices`，供给用户空间识别设备。*

OK，我们可以先查看`/proc/devices`，然后分别测试静态/动态注册，看`/proc/devices`的变化。

![](./src/0001.jpg)

可以看到，主设备号100是空闲的。我们就静态注册100来作为主设备号。

执行命令：`insmod dev_t.ko major=100`。测试结果注册成功

![](./src/0002.jpg)

接下来我们尝试自动分配设备号，测试结果成功

![](./src/0003.jpg)

