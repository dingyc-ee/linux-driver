# 驱动基础

## 第1章 编写第一个驱动helloworld

### 1.1 最简单的Linux驱动结构

一个最简单的Linux驱动主要由以下几个部分组成。

1. 头文件(必须有)

    驱动需要包含内核响应头文件。必须包含<linux/module.h>和<linux/init.h>

2. 驱动加载函数(必须有)

    当加载驱动的时候，驱动加载函数会自动被内核执行

3. 驱动写在函数(必须有)

    当卸载驱动的时候，驱动希望在函数会自动被内核执行

4. 许可证声明(必须有)

    Linux驱动加载时，要最受开源协议。内核驱动中最常见的是"GPL v2"或"GPL"

5. 模块参数(可选)

    模块参数是，模块被加载时传递给内核模块的值

5. 作者和版本信息(可选)

    可以生命驱动的作者信息和代码的版本信息

### 1.2 VSCODE智能配置感知(c_cpp_properties.json)

做这个事情的目的是，为了让VSCODE正确识别内核头文件路径并消除代码警告。

#### 1.2.1 打开配置

    按下`Ctrl+Shift+P`，输入`C/C++: Edit Configurations`，进入配置界面。此时会自动生成`c_cpp_properties.json`文件

#### 1.2.2 通用模板

`c_cpp_properties.json`

```json
{
    "configurations": [
        {
            "name": "Linux ARM",
            "includePath": [
                "${workspaceFolder}/**",
                // 内核头文件路径（按实际路径修改）
                "/path/to/arm-linux-kernel/include/**",
                "/path/to/arm-linux-kernel/arch/arm/include/**",
                "/path/to/arm-linux-kernel/arch/arm/include/generated/**"
            ],
            "defines": [
                "__KERNEL__",
                "MODULE"
            ],
            "compilerPath": "/usr/bin/arm-linux-gnueabihf-gcc", // 交叉编译器路径
            "cStandard": "gnu11",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-gcc-arm"
        }
    ],
    "version": 4
}
```

接下来我们一一解释这些参数，并定制属于我们自己的json配置文件

1. `name`: VSCODE允许在`c_cpp_properties.json`中定义多套配置(如针对不同的Linux内核版本)，点击vscode右下角可以迅速切换

![](./src//0001.jpg)

2. `includePath`: 添加内核源码的关键头文件路径

    + `include/`: 通用内核头文件
    + `arch/arm/include`: ARM架构相关头文件
    + 其他必要路径(如驱动子系统头文件)

3. `defines`: 用于给Vscode的C/C++插件传递预定义的宏，可以来控制代码的条件编译，或模拟编译环境
   
   + `__KERNEL__`: 内核中大量使用`#ifdef`条件编译指令。通过定义`__KERNEL__`，告诉 IntelliSense当前解析的是内核代码
  
        ```c
        #ifdef __KERNEL__
        // 内核空间代码
        #else
        // 用户空间代码
        #endif
        ```
    + 定义 MODULE 宏，表示当前代码是作为可加载模块(*.ko)编译的。这会启用内核头文件中与模块相关的代码路径

        ```c
        #include <linux/module.h>  // 依赖 MODULE 宏
        ```

4. `compilerPath`: 指定编译器路径。编译器会内置一些宏(如 __linux__、__arm__、__GNUC__)，这些宏会影响代码的条件编译逻辑(如 #ifdef __arm__)，IntelliSense 需要知道编译器的类型，才能正确解析这些宏

    + 在c_cpp_properties.json中，compilerPath需要指向交叉编译器的C编译器可执行文件(通常是arm-linux-gnueabihf-gcc)
    + 我的交叉编译器安装路径是`/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf`，因此`compilerPath`设为`/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabhf-gcc`

5. `cStandard和cppStandard`: Linux 内核代码大量依赖 GNU C 扩展语法。如果使用纯 ISO C 标准（如 c11），IntelliSense 会将这些 GNU 扩展语法标记为错误（红色波浪线）。
gnu11 表示使用 GNU 扩展的 C11 标准，使编辑器能够正确识别内核代码中的特殊语法

配置建议如下：

![](./src//0002.jpg)

#### 1.2.3 定制内容

根据前一节的介绍，我们开发设备驱动的通用配置如下：

```json
{
    "configurations": [
        {
            "name": "Linux ARM (4.1.15)",
            "includePath": [
                "${workspaceFolder}/**",
                // 内核头文件路径（按实际路径修改）
                "/home/ding/linux/imx/kernel/include/**",
                "/home/ding/linux/imx/kernel/arch/arm/include/**",
                "/home/ding/linux/imx/kernel/arch/arm/include/generated/**"
            ],
            "defines": [
                "__KERNEL__",
                "MODULE"
            ],
            "compilerPath": "/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc", // 交叉编译器路径
            "cStandard": "gnu11",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-gcc-arm"
        }
    ],
    "version": 4
}
```

### 1.3 helloworld程序

```c
#include <linux/module.h>
#include <linux/init.h>

static int helloworld_init(void)
{
	printk("helloworld_init\n");
	return 0;
}

static void helloworld_exit(void)
{
	printk("helloworld_exit\n");
}

module_init(helloworld_init);
module_exit(helloworld_exit);

MODULE_LICENSE("GPL v2");
MODULE_AUTHOR("ding");
MODULE_VERSION("V1.0");
```

这是最简单的驱动程序。但值得介绍的东西很多：

1. 包含必要的头文件
   
   + `<linux/module.h>`: 提供了`MODULE_LICENSE`、`MODULE_AUTHOR` ...
   + `<linux/init.h>`: 提供了`module_init`、`module_exit`
  
2. 驱动加载函数`_init`无入参有返回值，所以要写成`int xxxx_init(void)`的形式
3. 驱动卸载函数`_exit`无入参无返回值，所以要写成`void xxxx_init(void)`的形式
4. `MODULE_LICENSE`等声明通常放在最后面，靠近`module_exit`的位置，这是规范

### 1.4 编译Linux驱动程序

如何编译驱动程序？

第一种编译方法：将驱动放在Linux内核里面，然后编译Linux内核。

第二种编译方法：将驱动编译成内核模块，独立于Linux内核以外。

什么是Linux内核模块？

内核模块是Linux系统中一个特殊的机制，可以将一些使用频率很少或者暂时不用的功能编译成内核模块，在需要时再动态加载到内核里面。

使用内核模块可以较少内核体积，加快启动速度。内核模块的后缀是`.ko`

#### 1.4.1 驱动编译成内核模块

##### 1.4.1.1 通用模板

把驱动编译成Linux内核模块，需要一个简单的Makefile

```makefile
# 定义模块名称
obj-m += mydriver.o

# 定义内核源码路径
KDIR ?= /xxx/linux-kernel
PWD  ?= $(shell pwd)

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

Makefile解释如下：

1. `obj-m += mydriver.o`: `-m`表示编译成模块
2. `KDIR ?= /xxx/linux-kernel`: 使用绝对路径来指定内核源码的路径
3. `PWD ?= $(shell pwd)`: 获取Makefile和驱动源码文件所在的路径
4. `make -C $(KDIR) M=$(PWD) modules`: 进入KDIR目录，使用PWD路径下源码和Makefile文件编译驱动模块
5. `make -C $(KDIR) M=$(PWD) clean`: 清除编译文件

**注意：要编译驱动模块，必须先把Linux内核源码编译通过，否则无法编译内核模块**

##### 1.4.1.2 实际编译

还记得内核如何编译的吗？我们写了个脚本`build.sh`，然后在脚本里面指定了`ARCH`和`CROSS_COMPILE`

```sh
export ARCH=arm
export CROSS_COMPILE=/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-

# 其他编译
```

现在我们来编译helloworld驱动模块试试：

```makefile
obj-m += helloworld.o

KDIR ?= /home/ding/linux/imx/kernel
PWD  ?= $(shell pwd)

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

编译输出结果如下：编译失败了！

![](./src/0003.jpg)

似乎我们没有指定架构和编译器。在shell中通过export临时设置一下环境变量。再试试编译：

**可以看到，设置完ARCH和CROSS_COMPILE之后，就编译成功了**

![](./src/0004.jpg)

**注意：export设置的环境变量是临时的，而且只对当前的终端有效。比如我们关闭终端再重新打开，就有没了**

每次设置`ARCH`和`CROSS_COMPILE`都很麻烦啊，所以我们直接在内核的顶层Makefile中把这两个环境变量写死。

```Makefile
ARCH		?= arm
CROSS_COMPILE	?= /usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-
```

##### 1.4.1.3 模块操作的基本命令

1. 加载模块：`insmod helloworld.ko`或`modprobe helloworld.ko`
2. 卸载模块：`rmmod helloworld.ko`
3. 查看加载的模块：

    + `lsmod`命令：列出已经载入的内核模块
    + `cat /proc/modules`：查看模块是否加载成功

4. 查看模块信息：`modinfo helloworld.ko`

实测结果如下：

![](./src/0005.jpg)

还可以在ubuntu中查看模块信息：

![](./src/0006.jpg)

#### 1.4.2 menuconfig图形化配置

##### 1.4.2.1 打开图形化界面

1. 在终端中输入`export ARCH=arm`设置平台架构。平台结构是arm还是arm64，根据实际开发板来选择
   
   *为什么要先设置平台架构？因为linux支持多种平台架构，所以需要我们手动配置告诉他*

2. 在内核源码的顶层目录下，输入`make menuconfig`

输入完成即可打开配置页面：

![](./src/0007.jpg)

##### 1.4.2.2 图形化配置界面的操作

1. 移动

使用键盘的上、下、左、右按键，可以移动光标

2. 搜索功能

输入`/`即可弹出搜索界面

3. 配置驱动选项状态

    + 把驱动编译成模块，用M来表示
    + 把驱动编译到内核里面，用*来表示
    + 不编译
    + 使用`空格`来配置这三种不同的状态
  
比如我们想搜索LAN8720的驱动厂商SMSC，输入`/`，然后再输入SMSC，有以下界面：

![](./src/0008.jpg)

##### 1.4.2.3 与图形化有关的文件

Makefile、.config和Kconfig之间的关系：

**Kconfig文件**

Kconfig文件时图形化配置界面的源文件，图形化界面中的选项由Kconfig文件决定。当执行命令`make menuconfig`时，内核的配置工具会读取内核源码目录下的`arch/xxx/Kconfig`，`xxx`是命令`export ARCH=arm`中的`ARCH`值，然后再生成对应的配置界面。

如果我们要把`helloworld`编译进内核，那就可以修改Kconfig文件，在menuconfig的界面中增加一项。

**config文件和.config文件**

config文件和.config文件，都是Linux内核的配置文件。config位于`arch/$(ARCH)/configs`目录下，是Linux系统默认的配置文件。

.config位于内核源码的顶层目录下，编译Linux内核时会使用.config里面的配置来编译内核镜像。

若.config存在，`make menuconfig`界面的默认配置就是.config文件的配置，修改图形配置界面并保存时，会刷新.config文件。

若.config不存在，`make menuconfig`界面的默认配置，就是Kconfig文件中的默认配置。

使用命令`make xxx_defconfig`命令，会根据`arch/$(ARCH)configs`目录下默认配置文件，生成.config文件。

理解一下这幅图，沿着箭头的指向从上往下看。

1. 首先执行`make menuconfig`，打开图形化配置界面。图形化界面中的选项，由Kconfig决定
2. 打开图形化配置界面后，就可以在里面进行配置。比如让A驱动编译，B驱动不编译。设置完成保存，自动生成了.config文件
3. 执行`make`，就可以编译内核了

![](./src/0009.jpg)

*那么问题来了，似乎我们只用到了Kconfig和.config文件。config文件有什么用？*

Kconfig是固定不变的，menuconfig中的配置项成千上万。怎么知道哪些有用？我们需要一份默认配置(由厂商提供)，然后在默认配置上增加我们自己的内容了。

**所以，config文件就是厂商提供给我们的，可以生成定制化的.config文件，方便我们开发。**

![](./src/0010.jpg)

那么xxx_defconfig和.config的文件内容，是一致的吗？我们来看下。

可以看到，两个文件的大小明显不同。xxx_defconfig更像是一个精简的配置，而生成的.config文件则要大很多。

![](./src/0011.jpg)

从内容上看，两个文件的内容也不一样。那么.config文件是怎么来的？

![](./src/0012.jpg)

deepseek的回答是，xxx_defconfig只是覆盖了某些特定的配置项，而其他选项则保留为Kconfig中定义的默认值。

*也就是说，.config文件是defconfig和Kconfig默认值合并之后的结果，而且还自动处理了依赖关系。*

![](./src/0013.jpg)

##### 1.4.2.4 savedefconfig

*还有个问题，如果我们在图形界面中更改了某项值，那当然会生成.config文件对吧。但是只要执行`make distclean`全部重新编译时，.config就被删除了，这意味着下次我们还要重新打开图形界面再重复配置一次。*

如何解决？

`最理想的办法是，把更改图形界面配置项带来的增量变化，添加到xxx_defconfig这个默认的配置文件中。这样下次生成.config时，就会包含我们的更改了。`

能不能把.config直接覆盖xxx_defconfig？

`这样不好。因为前面提到，.config文件包含了默认Kconfig要大很多，而我们需要的只是修改的那几项，而不是整个的配置。所以要有个工具，能够去除.config中的合并引入的配置项，还原一个最精简的xxx_defconfig。这就是save_config`

![](./src/0014.jpg)

下面是一个简单的示例，添加博通的PHY驱动，看看saveconfig是怎么用的。

![](./src/0015.jpg)

当前目录下已经有了.config和Kconfig文件，我们执行`make savedefconfig`，来生成精简的`defconfig`配置文件。

![](./src/0017.jpg)

执行完成后得到了`defconfig`文件，与原配置文件`xxx_defconfig`比较发现，确实只增加了博通的PHY驱动，这是我们想要的。

![](./src/0018.jpg)

savedefconfig的工作原理：提取用户自定义的配置项，剥离默认值和隐式依赖项，生成最小化的配置模板

![](./src/0016.jpg)

#### 1.4.3 Makefile

Makefile里面包含了编译规则，告诉我们要如何编译Linux。

刚才我们再图形配置界面添加了博通的PHY驱动，对应的.config增加了`CONFIG_BROADCOM_PHY=y`

那么接下来Makefile就应该识别到这个配置项，来把博通的PHY驱动.c文件编译到内核

![](./src/0019.jpg)

#### 1.4.4 Kconfig语法

**mainmenu: 设置主菜单的标题**

![](./src/0020.jpg)

**menu/endmenu: 设置菜单结构**

可以用menu/endmenu来生成菜单，menu是菜单开始的标志，endmenu是菜单结束的标志。

![](./src/0021.jpg)

**配置选项**

使用关键字`config`来定义一个新的选项。每个选项都必须指定类型，类型包括bool、tristate、string、hex、int。

最常见的是bool tristate string这3种。bool类型有2种值(Y和N)，tristate有3种值(Y M N)

default提供了CONFIG_xxx的默认值。前面我们提到，当.config不存在时，使用的是Kconfig中的默认值，就是这个玩意了。

![](./src/0022.jpg)

**依赖关系**

Kconfig中的依赖关系，可以用depends on和select来描述

![](./src/0023.jpg)

这个在uboot中已经使用到了，举例如下：

![](./src/0024.jpg)

**注释**

Kconfig中使用comment用于注释，不过此注释非彼注释，这个注释是在图形化界面中显示一行注释。

![](./src/0025.jpg)

**source 读取另一个Kconfig文件**

![](./src/0026.jpg)

#### 1.4.5 实例: 把驱动编译进linux内核

现在我们尝试把驱动编译进linux内核。想想之前做了什么？

1. 要修改Kconfig文件，增加图形界面的配置项，最终在.config中得到`CONFIG_HELLOWORLD=y`
2. 要修改Makefile文件，增加helloworld.c的编译。既`obj-(CONFIG_HELLOWORLD) += helloworld.o`

接下来我们就尝试修改这些文件。加到哪里呢？放在字符设备中。`drivers/char`目录下

![](./src/0027.jpg)

这是当前字符设备的配置界面。我们可以有两种方式来添加helloworld。

1. 直接把helloworld加到`drivers/char`目录下，在这个Kconfig目录下增加helloworld
2. 在`drivers/char`目录下新增一个helloworld，在里面创建Kconfig，然后在上层通过source引用

先来试试第一种：很简单。

![](./src/0028.jpg)

实测结果如下：

![](./src/0029.jpg)

第二种方式更加复杂，我们来试试

##### 1.4.5.1 写Kconfig包含

1. 在`drivers/char`目录下创建`helloworld`目录
2. 在`hellowolrd`目录下创建Kconfig，内容跟前面第一种的内容一致
   
    ```Kconfig
    config HELLOWORLD
	bool "helloworld support"
	default y
	help
		build-in helloworld to kernel
    ```

3. 在`drivers/char`的Kconfig中使用`source`，把hellowolrd的Kconfig加进来

![](./src/0030.jpg)

验证结果没问题：

![](./src/0031.jpg)

##### 1.4.5.2 写驱动.c源文件

现在第一步完成了。我们要把之前写的helloworld.c复制到`helloworld`目录下

##### 1.4.5.3 写Makefile包含

`helloworld.c`复制好之后，我们要添加Makefile文件。这该怎么写？

在helloworld目录下，创建一个Makefile，内容如下：`obj-$(CONFIG_HELLOWORLD) += helloworld.o`

然后在上层Makefile中，包含子Makefile，添加一条：`obj-$(CONFIG_HELLOWORLD)	+= helloworld/`

![](./src/0032.jpg)

##### 1.4.5.3 编译linux内核

考虑以下，现在我们做了些什么，我们在Kconfig中增加了helloworld驱动编译。

之前编译Linux内核的步骤：

1. `make xxx_defconfig`

    执行make时，linux会把默认的defconfig和Kconfig合并，来生成一个.config文件。我想说的是，合并Kconfg这个过程应该已经把我们的`hellowolrd`驱动程序带进去了，不需要我们每次都执行`menuconfig`再保存一次

2. 执行`make menuconfig`修改配置界面，或者不要menuconfig，直接编译

3. 编译linux内核

编译完成后，在helloworld目录下生成了编译文件

![](./src/0033.jpg)

接下来启动一下linux内核，看内核启动时会不会自动加载helloworld驱动，可以看到，果然是有的

![](./src/0034.jpg)

如果日志太多不方便查看，我们还可以在`dmesg`中直接搜索，方式是`dmesg | grep "helloworld"`，看下输出结果

![](./src/0035.jpg)
