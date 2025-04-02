# 并发与竞争

## 第1章 原子操作

### 1.1 什么是原子操作

*原子操作指在执行过程中不会被中断的操作，要么完全执行，要么完全不执行，中间不会插入其他操作。这对于避免多核竞态条件至关重要。*

### 1.2 原子操作的接口

#### 1.2.1 原子操作的数据结构`atomic_t`

`linux/types.h`，定义了原子操作的数据结构`atomic_t`。其实就是一个int整数。

```c
typedef struct {
	int counter;
} atomic_t;
```

#### 1.2.2 原子变量初始化 `ATOMIC_INIT(i)`

`linux/asm/atomic.h`

由于原子变量的类型是一个结构体，所以不能直接赋值，而是要使用`ATOMIC_INIT`。例如：

```c
// 把一个原子变量的初始值设为1
atomic_t count = ATOMIC_INIT(1);
```

这个宏定义的实现方式也很简单，如下所示。其实就是给结构体的成员赋值。

```c
#define ATOMIC_INIT(i)	{ (i) }
```

#### 1.2.3 原子变量的直接赋值函数

| 函数原型 | 作用描述 | 示例 |
| ------- | ------- | ---- |
| `atomic_read(atomic_t *v)` | 读取原子变量的值 | `int val = atomic_read(&counter);` |
| `atomic_set(atomic_t *v, int i)` | 设置原子变量的值 | `atomic_set(&counter, 10);` |
| `atomic_add(int val, atomic_t *v)` | 原子地加上 val | `atomic_add(5, &counter);` |
| `atomic_sub(int val, atomic_t *v)` | 原子地减去 val | `atomic_sub(3, &counter);` |
| `atomic_inc(atomic_t *v)` | 原子地加 1 | `atomic_inc(&counter);` |
| `atomic_dec(atomic_t *v)` | 原子地减 1 | `atomic_dec(&counter);` |

#### 1.2.4 原子变量的test函数

| 函数原型 | 作用描述 | 示例 |
| ------- | ------- | ---- |
| `atomic_inc_and_test(atomic_t *v)` | 加 1 并检查是否为 0 | `if (atomic_inc_and_test(&counter)) {...}` |
| `atomic_dec_and_test(atomic_t *v)` | 减 1 并检查是否为 0 | `if (atomic_dec_and_test(&counter)) {...}` |

### 1.3 原子操作的测试

我们想通过原子操作，实现这样一个事情：`一个设备驱动，同时智能被一个应用程序打开。`

实现方式是，在open函数中判断原子变量。

```c
static int module_cdev_open(struct inode *inode, struct file *filp)
{
	if (!atomic_dec_and_test(&v)) {
		atomic_inc(&v);
		return -EBUSY;
	}
	filp->private_data = &dev;
	return 0;
}
```

直接看完整的驱动代码：

`atomic驱动.c`

```c
#include <linux/init.h>			/* module_init, module_exit */
#include <linux/module.h>		/* MODULE_LISENCE, MODULE_AUTHOR */
#include <linux/moduleparam.h>	/* module_cdev */
#include <linux/types.h>		/* dev_t */
#include <linux/kdev_t.h>		/* MAJOR, MINOR, MKDEV */
#include <linux/fs.h>			/* alloc_chrdev_region, unregister_chrdev_region */
#include <linux/cdev.h>			/* struct cdev, cdev_init, cdev_add */
#include <linux/device.h>		/* class_create, device_create */
#include <linux/uaccess.h>		/* copy_to_user, copy_from_user */
#include <linux/errno.h>		/* IS_ERR, PTR_ERR */
#include <linux/io.h>			/* ioremap, iounmap */
#include <linux/atomic.h>		/* atomic_t, atomic_add */
#include <asm/atomic.h>			

#define CLASS_NAME			"cdev_test"
#define CDEV_NAME			"cdev_test"
#define PRINTK(fmt, ...)	printk("[KERN] " fmt, ##__VA_ARGS__)

struct cdev_dev {
	dev_t  dev;
	struct class *class;
	struct cdev cdev;
	struct device *device;
};

static struct cdev_dev dev;
static atomic_t v = ATOMIC64_INIT(1);

static int module_cdev_open(struct inode *inode, struct file *filp)
{
	if (!atomic_dec_and_test(&v)) {
		atomic_inc(&v);
		return -EBUSY;
	}
	filp->private_data = &dev;
	PRINTK("Char device open\n");
	return 0;
}

ssize_t module_cdev_read(struct file *filp, char __user *buf, size_t size, loff_t *ppos)
{
	return 0;
}

static ssize_t module_cdev_write(struct file *filp, const char __user *buf, size_t size, loff_t *ppos)
{
	return 0;
}

static int module_cdev_release(struct inode *inode, struct file *filp)
{
	atomic_inc(&v);
	PRINTK("Char device close\n");
	return 0;
}

static struct file_operations cdev_ops = {
	.owner		= THIS_MODULE,
	.open		= module_cdev_open,
	.read		= module_cdev_read,
	.write		= module_cdev_write,
	.release	= module_cdev_release,
};

static __init int module_cdev_init(void)
{
	int ret;

	ret = alloc_chrdev_region(&dev.dev, 0, 1, CDEV_NAME);
	if (ret < 0) {
		goto err_alloc_chrdev;
	}
	dev.class = class_create(THIS_MODULE, CLASS_NAME);
	if (IS_ERR(dev.class)) {
		ret = PTR_ERR(dev.class);
		goto err_class_create;
	}
	cdev_init(&dev.cdev, &cdev_ops);
	dev.cdev.owner = THIS_MODULE;
	ret = cdev_add(&dev.cdev, dev.dev, 1);
	if (ret < 0) {
		goto err_cdev_add;
	}
	dev.device = device_create(dev.class, NULL, dev.dev, NULL, CDEV_NAME);
	if (IS_ERR(dev.device)) {
		ret = PTR_ERR(dev.device);
		goto err_device_create;
	}
	
	PRINTK("Char device initializing...\n");

	return 0;

err_device_create:
	cdev_del(&dev.cdev);
err_cdev_add:
	class_destroy(dev.class);
err_class_create:
	unregister_chrdev_region(dev.dev, 1);
err_alloc_chrdev:
	return ret;
}

static __exit void module_cdev_exit(void)
{
	device_destroy(dev.class, dev.dev);
	cdev_del(&dev.cdev);
	class_destroy(dev.class);
	unregister_chrdev_region(dev.dev, 1);

	PRINTK("Char device removed\n");
}

module_init(module_cdev_init);
module_exit(module_cdev_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ding");
```

应用代码中，我们打开一个设备文件后Sleep占用5秒，然后关闭设备文件。我们要测试的是，这5秒内其他应用程序能否打开设备文件。

`应用代码.c`

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>		/* atoi函数 */
#include <unistd.h>		/* close函数 */
#include <sys/types.h>	/* open函数要使用以下3个头文件 */
#include <sys/stat.h>
#include <fcntl.h>

int main(int argc, char *argv[])
{
	int fd, val;

	fd = open(argv[1], O_RDWR);
	if (fd < 0) {
		printf("open %s error\n", argv[1]);
		return fd;
	}
	sleep(5);

	close(fd);
	return 0;
}
```

测试结果跟预期一致。5秒内的访问会直接报错，5秒之后第一个应用程序退出，设备文件又可以继续访问了。

![](./src/0001.jpg)

