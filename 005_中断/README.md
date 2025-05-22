# 中断

## 第1章 `GIC`与`NVIC`

以前在stm32单片机中，用的是`NVIC`中断控制器。但是在linux系统中，用的是`GIC`控制器。为什么linux系统不用`NVIC`呢？二者有什么联系与区别。

### 1.1 硬件架构的差异

#### 1.1.1 目标处理器系列不同

+ `NVIC`：转为Cortex-M系列设计，如stm32、ESP32。面向低功耗、实时性强的单片机场景
+ `GIC`：转为Cortex-A系列设计，如Cortex-A系列，面向高性能、多核、复杂外设管理的Linux/Android系统

关键点：Cortex-A处理器的硬件架构中原生继承GIC，而非NVIC。

#### 1.1.2 中断管理规模

NVIC：

+ 最多支持240个外设中断，适合简单外设（`GPIO、UART、ADC`）
+ 中断类型单一，仅支持外设中断和内核异常（`Systick`）

GIC：

+ 支持SPI（共享外设中断）、PPI（私有外设中断）和SGI（软件生成中断）
+ GICv2支持最多1020个中断（SPI 32~1019），适配复杂外设（`PCIe、USB3、千兆网卡`）

现在Soc可能集成数百个外设，NVIC的240中断上限无法满足需求。

### 1.2 多核系统的支持

#### 1.2.1 GIC的多核特性

+ 中断路由与负载均衡：GIC的`Distributor`模块，可将SPI中断动态分发到不同CPU核心，优化负载（如网络中断绑定到空闲CPU）
+ 核间通信：通过SGI发送核间中断，触发其他CPU核心的动作（如任务调度）

#### 1.2.2 NVIC的局限性

NVIC只能适用于单核CPU。

### 1.3 中断类型与功能扩展

#### 1.3.1 GIC的细分中断类型

| 中断类型 | 功能 | Linux应用场景 |
| - | - |
| SPI | 共享外设中断（如网卡、磁盘） | 多核共享外设的中断处理 |
| PPI | CPU私有中断（如本地定时器） | 每核独立的时钟中断 |
| SGI | 软件生成中断（核间通信）） | 多核任务调度 |

#### 1.3.2 NVIC的单一中断类型

+ 所有外设中断统一管理，缺乏SPI、PPI/SGI的细分，无法支持多和系统的精细管理
+ 例如：CortexM的Systick定时器中断时全局的，无法绑定到特定的CPU核心

### 1.4 性能与实时性对比

#### 1.4.1 NVIC的实时性优势

+ 中断响应延时极低（通常<10周期），适合硬实时系统（如电机控制）
+ 原因：NVIC直接集成在Cortex-M内核中，无需通过总线访问外部控制器

#### 1.4.2 GIC的吞吐量优势

+ 通过多核分发和优先级抢占，支持高并发中断处理（如数据中心网卡的 100 万+ PPS）
+ 示例：GIC 可将多个网卡队列的中断分发到不同 CPU 核心，避免单核瓶颈

### 1.5 总结

ARM Linux为何必须使用GIC？

+ 硬件强制要求：Cortex-A处理器原生集成GIC，无法替换为NVIC
+ 多核支持：GIC的SPI-SGI中断机制，是Linux多核调度核负载均衡的基础
+ 扩展性需求：GIC支持上千个中断核虚拟化，适配复杂外设核云计算场景
+ Linux内核依赖：设备树、电源管理等特性深度绑定GIC功能

最终结论：

+ NVIC是为资源受限的单片机设计，适合实时操作系统（`FreeRTOS`）
+ 高性能多和系统设计的，是Linux在Cortex-A平台上的唯一选择。两者服务于不同的生态位，不可互换。

## 第2章 `GIC`与`CPU核心`

在ARM Linux SoC中，GIC（`Generic Interrupt Controller`）通常是集成在CPU核外部的独立模块，但与CPU核紧密集成在同一芯片上。外设的中断信号线会直接连到GIC，而非直接连到CPU核。

### 2.1 典型SoC架构图

```
+-------------------+       +-------------------+
|      CPU Core      |       |      CPU Core      |
| (Cortex-A7/A53/A76)|       | (Cortex-A7/A53/A76)|
+---------+---------+       +---------+---------+
          |                           |
          |         +-------+         |
          +-------->|       |<--------+
                    |  GIC  |
                    |       |<--- [外设中断信号线]
                    +-------+
                      ^  |  ^
                      |  |  |
                      |  |  +--- 其他外设（如 GPU、NPU）
                      |  |
          +-----------+--+-----------+
          |           |              |
      [UART]       [GPIO]         [Ethernet]...
```

### 2.2 GIC的位置

+ 外部模块：GIC是SoC中断一个独立硬件组件，与CPU分离，不直接嵌入CPU内部。通过片上总线（AMBA/AXI）与CPU核紧密相连
+ 多核支持：GIC通常设计为支持多核架构，可集中管理多个CPU核的中断分发（例如分配中断到不同核心、设置优先级）

### 2.3 外设中断的连接方式

+ 中断信号接入GIC：所有外设（如 UART、GPIO、以太网控制器等）的中断请求线（IRQ）直接连接到 GIC 的输入端
+ GIC 的中断路由：
    + GIC的`Distributor`模块接收外设中断，根据优先级、目标 CPU 核配置（如亲和性）将中断分发给特定CPU核
    + CPU 核通过 CPU Interface 模块与 GIC 交互，接收中断信号（如 IRQ/FIQ）

### 2.4 与传统架构的区别

+ 传统中断控制器：在早期ARM架构（如ARM7/ARM9）中，外设中断可能直接连接到CPU核的专用中断引脚（如IRQ/FIQ），此时中断控制器可能集成在CPU核内部
+ 现在多核场景：GIC的出现是为了支持多核SoC核复杂中断管理（如负载均衡），因此必须作为独立模块存在

### 2.5 Linux内核的视角

+ 设备树配置：在设备树中，外设的中断属性会指定其连接的GIC和中断号（如`<interrupt-parent = <&gic>>`）
+ 中断处理流程：
    1. 外设触发中断，GIC接收并标记为挂起状态
    2. GIC根据优先级和目标核配置，向对应CPU发出中断信号
    3. CPU核通过读取`GICC_IAR`寄存器确认中断，执行中断服务程序（ISR）
    4. ISR完成后，CPU写入`GICC_EOIR`寄存器通知GIC结束中断

### 2.6 GIC的`Distributor`与`CPU Interface`

```
+---------------------+      +---------------------+
|    Peripheral 1     |      |    Peripheral N     |
| (SPI: UART/GPIO...) |      | (SPI: Ethernet...)  |
+----------+----------+      +----------+----------+
           | SPI                        | SPI
           v                            v
    +------+----------------------------+------+
    |              GIC Distributor               |
    | (全局中断管理：优先级仲裁、目标核分配、使能控制) |
    +------+----------------------------+------+
           | IRQ_OUT[0] ... IRQ_OUT[M]  |      | PPI/SGI (来自CPU核私有外设)
           v                            v      v
    +------+--------+            +------+--------+     +-----------------+
    | CPU Interface 0 |          | CPU Interface M |<----| SGI (核间中断)  |
    | (Core 0 专属)    |          | (Core M 专属)    |     | (软件触发，如IPI)|
    +------+--------+            +------+--------+     +-----------------+
           | IRQ/FIQ                  | IRQ/FIQ
           v                            v
    +------+--------+            +------+--------+
    |   CPU Core 0   |            |   CPU Core M   |
    | (Cortex-A系列核) |            | (Cortex-A系列核) |
    +---------------+            +---------------+
```

#### 2.6.1 关键模块与连接说明

1. 外设（`Peripheral`）

    + 所有外设的中断信号（SPI）直接连到`GIC Distributor`
    + 示例外设：UART、GPIO、以太网控制器

2. `GIC Distributor`

    + 功能：
        + 接收所有外设的中断请求（SPI）
        + 管理中断优先级、使能状态、目标CPU核分配（亲和性）
        + 将中断路由到对应的`CPU Interface`

    + 关键信号：
        + `SPI[0-N]`：外设中断输入线
        + `IRQ OUT[0-M]`：路由到各CPU核的中断信号线

3. `CPU Interface`

    + 功能：
        + 每个CPU核对应一个独立的`CPU Interface`
        + 将`Distributor`路由的中断，转换为CPU核的IRQ信号
        + 处理CPU核的中断确认（`GICC_IAR`）和中断结束（`GICC_EOIR`）操作

    + 关键信号：
        + `IRQ/FIQ`：连接到CPU核的中断引脚
        + `PPI`：每个CPU核的私有中断
        + `SGI`：核间中断，由软件触发

4. `CPU`

    + 通过`IRQ/FIQ`引脚接收中断信号
    + 通过AMBA 总线（AXI/AHB）与GIC寄存器交互，完成中断处理流程

#### 2.6.2 详细信号流

1. 外设触发中断
    + 外设产生中断后，通过SPI信号线发送给`Distributor`

2. `Distributor`路由中断
    + `Distributor`根据优先级核目标CPU核配置，选择中断路由路径
    + 将中断通过`IRQ_OUT`信号线发送到目标`CPU Interface`

3. `CPU Interface`转发中断
    + `CPU Interface`将中断转换为IRQ或FIQ信号，触发CPU核进入中断处理模式

4. CPU 核处理中断
    + CPU 核读取`GICC_IAR`获取中断ID，执行中断服务程序（ISR）
    + ISR 完成后，写入`GICC_EOIR`通知GIC结束中断

**SPI信号流**

```
外设（SPI）--> GIC Distributor --> IRQ_OUT[0:M] --> CPU Interface --> IRQ/FIQ --> CPU 核
```

**PPI信号流**

```
CPU 核私有外设（PPI）--> CPU Interface --> IRQ/FIQ --> 同一 CPU 核
```

**SGI信号流**

```
CPU 核 A 写寄存器触发 SGI --> GIC Distributor --> IRQ_OUT[X] --> CPU Interface B --> IRQ --> CPU 核 B
```

## 2.7 `GICv2`状态机

下图是GICv2中断处理状态机，主要描述了中断从触发到完成的完整生命周期，涵盖`Distributor`和`CPU Interface`的协作流程。

```
    +-------------------+
    |      Idle         | <------------------+
    +-------------------+                    |
            |                             |
            | 外设触发中断                 |
            | (Distributor检测到中断)       |
            v                             |
    +-------------------+                    |
    |     Pending       |                    |
    +-------------------+                    |
            |                             |
            | Distributor仲裁并路由中断     |
            | (优先级、目标核亲和性)         |
            v                             |
    +-------------------+                    |
    |    Active         |--------------------+
    +-------------------+                    |
            |                             |
            | CPU核响应中断                 |
            | (读GICC_IAR获取中断ID)        |
            v                             |
    +-------------------+                    |
    | Active & Pending  |                    |
    +-------------------+                    |
            |                             |
            | 中断处理完成                 |
            | (写GICC_EOIR结束中断)        |
            v                             |
    +-------------------+                    |
    |     Idle         |--------------------+
    +-------------------+
```

### 2.7.1 状态机详细说明

1. `IDLE`（空闲状态）
    + 触发条件：
        + 初始状态，无中断挂起或处理中
        + 中断处理完成（CPU核写入`GICC_EOIR`）后返回此状态
    + 行为：
        + `Distributor`和`CPU Interface`等待新中断触发

2. `Pending`（挂起状态）
    + 触发条件：
        + 外设触发中断（SPI、PPI、SGI），`Distributor`检测到中断并标记为挂起
    + 行为：
        + `Distributor`更新中断状态寄存器`GICD_ISPENDRn`，标记中断为`Pending`
        + 若中断已使能（`GICD_ISENABLERn`置位），且优先级高于当前活动中断，进入仲裁流程
        ![](./src/0001.jpg)
        ![](./src/0002.jpg)
    + 关键操作：
        + 中断优先级仲裁，选择目标CPU核（通过`GICD_ITARGETSRn`配置亲和性）

3. `Active`活动状态
    + 触发条件：
        + `Distributor`完成仲裁，将中断路由到目标CPU核的`CPU Interface`
    + 行为：
        + `Distributor`将中断状态标记为`Active`（`GICD_ISACTIVERn`置位）
        + `CPU Interface`向目标CPU核发送IRQ或FIQ信号        
    + 关键操作：
        + CPU核响应中断，读取`GICC_IAR`寄存器获取中断ID，状态转换为`Active & Pending`
        ![](./src/0003.jpg)

4. `Active & Pending`活动并挂起状态
    + 触发条件：
        + 在 Active 状态下，同一中断源再次触发中断（例如高优先级中断抢占）
        + 中断处理过程中，新中断到达并被仲裁为更高优先级
    + 行为：
        + `Distributor`标记中断为`Active & Pending`（`GICD_ISPENDRn`和`GICD_ISACTIVERn`同时置位）
        + CPU 核可能暂停当前低优先级中断，处理高优先级中断（抢占机制）
    + 关键操作：
        + 若发生抢占，当前中断状态保持`Active`，新中断进入`Pending`队列

    典型使用场景：

    ![](./src/0004.jpg)

## 第3章 `imx6ull`芯片的GIC控制器

### 3.1 中断ID分类

前一章介绍`GICC_IAR`寄存器时，`bit [0~9]`表示`Interrupt ID`，也就是说最多支持`2^10=1024`个中断ID。按范围核用途分成三类：

| 中断类型 | 中断ID范围 | 说明 |
| - | - | - |
| SGI | 0-15 | Software Gennerated Interrupts（软件生成中断），用于核间通信 |
| PPI | 16-31 | Private Peripheral Interrupts（私有外设中断），每个CPU核心独享的中断 |
| SPI | 32-1019 | Shared Peripheral Interrupts（共享外设中断），全局外设中断，可路由到任意核心 |

ID 1020-1023：保留给GIC内部使用。保留中断ID的特殊用途：

+ ID 1023：虚拟化扩展中的维护中断
+ ID 1020-1022：可能用于调试或未来扩展

### 3.2 `imx6ull`的芯片中断

`imx6ull`芯片支持128个外设SPI中断。

![](./src/0005.jpg)

### 3.3 `Distributor`寄存器描述

`GICv2 Distributor`是中断控制的核心模块，负责全局中断管理，如优先级仲裁、目标核分配、中断使能。

#### 3.3.1 `GICD_CTRL (Distributor Control Register)`

+ 功能：全局控制`Distributor`的行为
+ 位域：
    + Bit 0(Enable)：置1使能`Distributor`
    + 示例：
    ```c
    // 使能 Distributor
    *((volatile uint32_t*)GICD_CTRL) |= 0x1;
    ```

#### 3.3.2 `GICD_TYPER (Distributor Type Register)`

+ 功能：描述`Distributor`的硬件实现特性（只读）
+ 位域：
    + Bits 4:0 (ITLinesNumber): 支持的SPI中断数量
    + Bit 10 (CPUNumber): 支持的 CPU 接口数量

#### 3.3.3 中断使能与禁用`GICD_ISENABLERn / GICD_ICENABLERn (Interrupt Set/Clear Enable Registers)`

+ 功能：使能或禁用特定中断
+ 地址计算：每个寄存器控制32个中断，偏移由中断号决定
```c
reg_offset = 0x100 + (interrupt_id / 32) * 4;  // ISENABLERn
reg_offset = 0x180 + (interrupt_id / 32) * 4;  // ICENABLERn
```
+ 位操作：写1到某一位使能/禁用对应中断
```c
// 使能中断号 50（SPI）
uint32_t reg = GICD_ISENABLERn + (50 / 32) * 4;
*((volatile uint32_t*)reg) |= (1 << (50 % 32));
```

#### 3.3.4 中断优先级配置`GICD_IPRIORITYRn (Interrupt Priority Registers)`

+ 功能：设置每个中断的优先级（8位/中断），0–255，值越小优先级越高
+ 优先级值：数值越小优先级越高（0为最高，255为最低）

#### 3.3.5 中断目标CPU配置`GICD_ITARGETSRn (Interrupt Target CPU Registers)` 

+ 功能：指定SPI中断的路由目标CPU（每个中断占8位，每个bit对应一个CPU，如bit0=CPU0）
+ 示例：将中断ID 32路由到CPU0和CPU1
```c
uint32_t reg_offset = 0x800 + (32 / 4) * 4;
write32(GICD_BASE + reg_offset, 0x3 << (8 * (32 % 4))); // 0x3 = CPU0 | CPU1
```

#### 3.3.6 中断触发类型配置`GICD_ICFGRn (Interrupt Configuration Registers)`

+ 功能：配置中断触发方式（电平触发或边沿触发）
+ 地址计算：每个寄存器控制 16 个中断（每个中断占用 2 位）
+ 功能: 配置中断触发类型
    + 00：保留
    + 01：边沿触发
    + 10：保留
    + 11：电平触发
+ 示例：配置中断号 50 为边沿触发
```c
uint32_t reg = GICD_ICFGRn + (50 / 16) * 4;
uint32_t shift = (50 % 16) * 2;
*((volatile uint32_t*)reg) = (*((volatile uint32_t*)reg) & ~(0x3 << shift)) | (0x3 << shift);
```

### 3.4 `CPU Interface`寄存器描述

#### 3.4.1 `GICC_CTLR (CPU Interface Control Register)`

+ 功能：控制`CPU Interface`的全局行为
+ 位域：
    + Bit 0 (Enable): 置1使能`CPU Interface`（必须使能才能接收中断）
    + Bit 1 (FIQEn): 置1允许`FIQ`中断（否则所有中断通过`IRQ`触发）
+ 示例：使能`CPU Interface`并允许FIQ
```c
*((volatile uint32_t*)GICC_CTLR) |= (1 << 0) | (1 << 1);
```

#### 3.4.2 `GICC_BPR (Binary Point Register)`

+ 功能：设置优先级分组点，决定抢占级别和子优先级
+ 位域：二进制点值（0~7），将优先级字段分为两部分
    + 抢占级别（Group）：高位部分，用于中断抢占决策
    + 子优先级（Subpriority）：低位部分，仅在同组内排序
+ 示例：设置优先级分组点为 3（抢占级别占高4位，子优先级占低4位）
```c
// 设置优先级分组点为 3（抢占级别占高4位，子优先级占低4位）
*((volatile uint32_t*)GICC_BPR) = 2;  // BPR=2 → 分组点=3
```

#### 3.4.3 `GICC_IAR (Interrupt Acknowledge Register)`

+ 功能：CPU核读取此寄存器以获取当前中断ID，并标记中断为活动状态
+ 返回值：
    + Bits 9:0 (Interrupt ID): 中断编号（0~1023）
    + Bits 12:10 (CPU ID): 发送 SGI 的源 CPU 核编号（仅对 SGI 有效）
+ 操作：读取后，GIC将中断状态从`Pending`转为`Active`
+ 示例：读取当前中断ID
```c
// 读取当前中断 ID
uint32_t iar = *((volatile uint32_t*)GICC_IAR);
uint32_t int_id = iar & 0x3FF;  // 提取中断 ID
```

#### 3.4.4 `GICC_EOIR (End of Interrupt Register)`

+ 功能：CPU核写入中断ID，通知GIC中断处理完成，清除活动状态
+ 注意事项：写入的ID必须与之前`GICC_IAR`读取的值一致
+ 示例：结束中断处理（假设 int_id 为之前读取的中断 ID）
```c
// 结束中断处理（假设 int_id 为之前读取的中断 ID）
*((volatile uint32_t*)GICC_EOIR) = int_id;
```
