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
| - | - | - |
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

## 第3章 `imx6ull`的GIC编程

在前面的章节，我们介绍了`GIC`的原理，感觉还是比较抽象，编程有点无从下手。有没有例程可以参考呢？

*有的，NXP官方提供了SDK。事实上，官方SDK提供了对外设操作的所有代码，我们拥有丰富的资料可以参考。*

### 3.1 官方SDK的中断操作

以最简单的GPIO中断为例，看看官方SDK是怎么写的。

#### 3.1.1 启动文件`startup_MCIMX6Y2.S`

启动文件`startup_MCIMX6Y2.S`，设置完栈指针后（C语言环境准备好了），就调用了`SystemInit`函数。

```S
/* Reset Handler */

Reset_Handler:
    cpsid   i               /* Mask interrupts */

    /* Reset SCTlr Settings */
    mrc     p15, 0, r0, c1, c0, 0     /* Read CP15 System Control register                  */
    bic     r0,  r0, #(0x1 << 12)     /* Clear I bit 12 to disable I Cache                  */
    bic     r0,  r0, #(0x1 <<  2)     /* Clear C bit  2 to disable D Cache                  */
    bic     r0,  r0, #0x2             /* Clear A bit  1 to disable strict alignment         */
    bic     r0,  r0, #(0x1 << 11)     /* Clear Z bit 11 to disable branch prediction        */
    bic     r0,  r0, #0x1             /* Clear M bit  0 to disable MMU                      */
    mcr     p15, 0, r0, c1, c0, 0     /* Write value back to CP15 System Control register   */

    /* Set up stack for IRQ, System/User and Supervisor Modes */
    cps     #0x12                /* Enter IRQ mode                */
    ldr     sp, =__IStackTop     /* Set up IRQ handler stack      */

    cps     #0x1F                /* Enter System mode             */
    ldr     sp, =__CStackTop     /* Set up System/User Mode stack */

    cps     #0x13                /* Enter Supervisor mode         */
    ldr     sp, =__CStackTop     /* Set up Supervisor Mode stack  */

    ldr     r0,=SystemInit
    blx     r0

    cpsie   i                    /* Unmask interrupts             */
```

#### 3.1.2 系统初始化函数`SystemInit`

`SystemInit`函数调用了`GIC_Init`函数，用来初始化GIC控制器。

```c
void SystemInit (void)
{
  uint32_t sctlr;
  uint32_t actlr;

  L1C_InvalidateInstructionCacheAll();
  L1C_InvalidateDataCacheAll();

  sctlr = __get_SCTLR();
  sctlr = (sctlr & ~(SCTLR_V_Msk       | /* Use low vector */
                     SCTLR_A_Msk       | /* Disable alignment fault checking */
                     SCTLR_M_Msk))       /* Disable MMU */
                 |  (SCTLR_I_Msk       | /* Enable ICache */
                     SCTLR_Z_Msk       | /* Enable Prediction */
                     SCTLR_CP15BEN_Msk | /* Enable CP15 barrier operations */
                     SCTLR_C_Msk);       /* Enable DCache */
  __set_SCTLR(sctlr);

  /* Set vector base address */
  GIC_Init();
  __set_VBAR((uint32_t)__VECTOR_TABLE);
}
```

#### 3.1.3 GIC初始化函数`GIC_Init`

这里实际上是在配置GIC的寄存器。`gic->D_xxx`指的是`Distributor`，`gic_C_xxx`指的是`CPU Interface`。我们先简单的看下这里做了些什么，后面再具体分析这些出现的寄存器。

1. `imx6ull`支持5位中断优先级，`2^5=32`，可设置的中断优先级为0~31
2. 获取`SoC`最大支持的GIC中断个数
3. 先禁用所有支持的GIC中断
4. 允许所有中断通过
5. 没有子优先级
6. 使能`Distributor`
7. 使能`CPU Interface`

```c
#define __GIC_PRIO_BITS     5   /**< Number of Bits used for Priority Levels */

FORCEDINLINE __STATIC_INLINE void GIC_Init(void)
{
  uint32_t i;
  uint32_t irqRegs;
  GIC_Type *gic = (GIC_Type *)(__get_CBAR() & 0xFFFF0000UL);

  irqRegs = (gic->D_TYPER & 0x1FUL) + 1;

  /* On POR, all SPI is in group 0, level-sensitive and using 1-N model */

  /* Disable all PPI, SGI and SPI */
  for (i = 0; i < irqRegs; i++)
    gic->D_ICENABLER[i] = 0xFFFFFFFFUL;

  /* Make all interrupts have higher priority */
  gic->C_PMR = (0xFFUL << (8 - __GIC_PRIO_BITS)) & 0xFFUL;

  /* No subpriority, all priority level allows preemption */
  gic->C_BPR = 7 - __GIC_PRIO_BITS;

  /* Enable group0 distribution */
  gic->D_CTLR = 1UL;

  /* Enable group0 signaling */
  gic->C_CTLR = 1UL;
}
```

到目前为止，我们初始化完成了GIC控制器。接下来就可以用外设中断了。如果要启动外设中断，需要做两件事：

1. 设置外设中断的优先级
2. 使能外设中断

#### 3.1.4 设置外设中断优先级`GIC_SetPriority`

SDK中列出了`imx6ull`支持的全部中断号。我们要使用外设中断，就从这里面找对应的`中断IRQn`。

```c
typedef enum IRQn {
  /* Auxiliary constants */
  NotAvail_IRQn                = -128,             /**< Not available device specific interrupt */

  /* Core interrupts */
  Software0_IRQn               = 0,                /**< Cortex-A7 Software Generated Interrupt 0 */
  Software1_IRQn               = 1,                /**< Cortex-A7 Software Generated Interrupt 1 */
  Software2_IRQn               = 2,                /**< Cortex-A7 Software Generated Interrupt 2 */
  Software3_IRQn               = 3,                /**< Cortex-A7 Software Generated Interrupt 3 */
  Software4_IRQn               = 4,                /**< Cortex-A7 Software Generated Interrupt 4 */
  Software5_IRQn               = 5,                /**< Cortex-A7 Software Generated Interrupt 5 */
  Software6_IRQn               = 6,                /**< Cortex-A7 Software Generated Interrupt 6 */
  Software7_IRQn               = 7,                /**< Cortex-A7 Software Generated Interrupt 7 */
  Software8_IRQn               = 8,                /**< Cortex-A7 Software Generated Interrupt 8 */
  Software9_IRQn               = 9,                /**< Cortex-A7 Software Generated Interrupt 9 */
  Software10_IRQn              = 10,               /**< Cortex-A7 Software Generated Interrupt 10 */
  Software11_IRQn              = 11,               /**< Cortex-A7 Software Generated Interrupt 11 */
  Software12_IRQn              = 12,               /**< Cortex-A7 Software Generated Interrupt 12 */
  Software13_IRQn              = 13,               /**< Cortex-A7 Software Generated Interrupt 13 */
  Software14_IRQn              = 14,               /**< Cortex-A7 Software Generated Interrupt 14 */
  Software15_IRQn              = 15,               /**< Cortex-A7 Software Generated Interrupt 15 */
  VirtualMaintenance_IRQn      = 25,               /**< Cortex-A7 Virtual Maintenance Interrupt */
  HypervisorTimer_IRQn         = 26,               /**< Cortex-A7 Hypervisor Timer Interrupt */
  VirtualTimer_IRQn            = 27,               /**< Cortex-A7 Virtual Timer Interrupt */
  LegacyFastInt_IRQn           = 28,               /**< Cortex-A7 Legacy nFIQ signal Interrupt */
  SecurePhyTimer_IRQn          = 29,               /**< Cortex-A7 Secure Physical Timer Interrupt */
  NonSecurePhyTimer_IRQn       = 30,               /**< Cortex-A7 Non-secure Physical Timer Interrupt */
  LegacyIRQ_IRQn               = 31,               /**< Cortex-A7 Legacy nIRQ Interrupt */

  /* Device specific interrupts */
  IOMUXC_IRQn                  = 32,               /**< General Purpose Register 1 from IOMUXC. Used to notify cores on exception condition while boot. */
  DAP_IRQn                     = 33,               /**< Debug Access Port interrupt request. */
  SDMA_IRQn                    = 34,               /**< SDMA interrupt request from all channels. */
  TSC_IRQn                     = 35,               /**< TSC interrupt. */
  SNVS_IRQn                    = 36,               /**< Logic OR of SNVS_LP and SNVS_HP interrupts. */
  LCDIF_IRQn                   = 37,               /**< LCDIF sync interrupt. */
  RNGB_IRQn                    = 38,               /**< RNGB interrupt. */
  CSI_IRQn                     = 39,               /**< CMOS Sensor Interface interrupt request. */
  PXP_IRQ0_IRQn                = 40,               /**< PXP interrupt pxp_irq_0. */
  SCTR_IRQ0_IRQn               = 41,               /**< SCTR compare interrupt ipi_int[0]. */
  SCTR_IRQ1_IRQn               = 42,               /**< SCTR compare interrupt ipi_int[1]. */
  WDOG3_IRQn                   = 43,               /**< WDOG3 timer reset interrupt request. */
  Reserved44_IRQn              = 44,               /**< Reserved */
  APBH_IRQn                    = 45,               /**< DMA Logical OR of APBH DMA channels 0-3 completion and error interrupts. */
  WEIM_IRQn                    = 46,               /**< WEIM interrupt request. */
  RAWNAND_BCH_IRQn             = 47,               /**< BCH operation complete interrupt. */
  RAWNAND_GPMI_IRQn            = 48,               /**< GPMI operation timeout error interrupt. */
  UART6_IRQn                   = 49,               /**< UART6 interrupt request. */
  PXP_IRQ1_IRQn                = 50,               /**< PXP interrupt pxp_irq_1. */
  SNVS_Consolidated_IRQn       = 51,               /**< SNVS consolidated interrupt. */
  SNVS_Security_IRQn           = 52,               /**< SNVS security interrupt. */
  CSU_IRQn                     = 53,               /**< CSU interrupt request 1. Indicates to the processor that one or more alarm inputs were asserted. */
  USDHC1_IRQn                  = 54,               /**< USDHC1 (Enhanced SDHC) interrupt request. */
  USDHC2_IRQn                  = 55,               /**< USDHC2 (Enhanced SDHC) interrupt request. */
  SAI3_RX_IRQn                 = 56,               /**< SAI3 interrupt ipi_int_sai_rx. */
  SAI3_TX_IRQn                 = 57,               /**< SAI3 interrupt ipi_int_sai_tx. */
  UART1_IRQn                   = 58,               /**< UART1 interrupt request. */
  UART2_IRQn                   = 59,               /**< UART2 interrupt request. */
  UART3_IRQn                   = 60,               /**< UART3 interrupt request. */
  UART4_IRQn                   = 61,               /**< UART4 interrupt request. */
  UART5_IRQn                   = 62,               /**< UART5 interrupt request. */
  eCSPI1_IRQn                  = 63,               /**< eCSPI1 interrupt request. */
  eCSPI2_IRQn                  = 64,               /**< eCSPI2 interrupt request. */
  eCSPI3_IRQn                  = 65,               /**< eCSPI3 interrupt request. */
  eCSPI4_IRQn                  = 66,               /**< eCSPI4 interrupt request. */
  I2C4_IRQn                    = 67,               /**< I2C4 interrupt request. */
  I2C1_IRQn                    = 68,               /**< I2C1 interrupt request. */
  I2C2_IRQn                    = 69,               /**< I2C2 interrupt request. */
  I2C3_IRQn                    = 70,               /**< I2C3 interrupt request. */
  UART7_IRQn                   = 71,               /**< UART-7 ORed interrupt. */
  UART8_IRQn                   = 72,               /**< UART-8 ORed interrupt. */
  Reserved73_IRQn              = 73,               /**< Reserved */
  USB_OTG2_IRQn                = 74,               /**< USBO2 USB OTG2 */
  USB_OTG1_IRQn                = 75,               /**< USBO2 USB OTG1 */
  USB_PHY1_IRQn                = 76,               /**< UTMI0 interrupt request. */
  USB_PHY2_IRQn                = 77,               /**< UTMI1 interrupt request. */
  DCP_IRQ_IRQn                 = 78,               /**< DCP interrupt request dcp_irq. */
  DCP_VMI_IRQ_IRQn             = 79,               /**< DCP interrupt request dcp_vmi_irq. */
  DCP_SEC_IRQ_IRQn             = 80,               /**< DCP interrupt request secure_irq. */
  TEMPMON_IRQn                 = 81,               /**< Temperature Monitor Temperature Sensor (temperature greater than threshold) interrupt request. */
  ASRC_IRQn                    = 82,               /**< ASRC interrupt request. */
  ESAI_IRQn                    = 83,               /**< ESAI interrupt request. */
  SPDIF_IRQn                   = 84,               /**< SPDIF interrupt. */
  Reserved85_IRQn              = 85,               /**< Reserved */
  PMU_IRQ1_IRQn                = 86,               /**< Brown-out event on either the 1.1, 2.5 or 3.0 regulators. */
  GPT1_IRQn                    = 87,               /**< Logical OR of GPT1 rollover interrupt line, input capture 1 and 2 lines, output compare 1, 2, and 3 interrupt lines. */
  EPIT1_IRQn                   = 88,               /**< EPIT1 output compare interrupt. */
  EPIT2_IRQn                   = 89,               /**< EPIT2 output compare interrupt. */
  GPIO1_INT7_IRQn              = 90,               /**< INT7 interrupt request. */
  GPIO1_INT6_IRQn              = 91,               /**< INT6 interrupt request. */
  GPIO1_INT5_IRQn              = 92,               /**< INT5 interrupt request. */
  GPIO1_INT4_IRQn              = 93,               /**< INT4 interrupt request. */
  GPIO1_INT3_IRQn              = 94,               /**< INT3 interrupt request. */
  GPIO1_INT2_IRQn              = 95,               /**< INT2 interrupt request. */
  GPIO1_INT1_IRQn              = 96,               /**< INT1 interrupt request. */
  GPIO1_INT0_IRQn              = 97,               /**< INT0 interrupt request. */
  GPIO1_Combined_0_15_IRQn     = 98,               /**< Combined interrupt indication for GPIO1 signals 0 - 15. */
  GPIO1_Combined_16_31_IRQn    = 99,               /**< Combined interrupt indication for GPIO1 signals 16 - 31. */
  GPIO2_Combined_0_15_IRQn     = 100,              /**< Combined interrupt indication for GPIO2 signals 0 - 15. */
  GPIO2_Combined_16_31_IRQn    = 101,              /**< Combined interrupt indication for GPIO2 signals 16 - 31. */
  GPIO3_Combined_0_15_IRQn     = 102,              /**< Combined interrupt indication for GPIO3 signals 0 - 15. */
  GPIO3_Combined_16_31_IRQn    = 103,              /**< Combined interrupt indication for GPIO3 signals 16 - 31. */
  GPIO4_Combined_0_15_IRQn     = 104,              /**< Combined interrupt indication for GPIO4 signals 0 - 15. */
  GPIO4_Combined_16_31_IRQn    = 105,              /**< Combined interrupt indication for GPIO4 signals 16 - 31. */
  GPIO5_Combined_0_15_IRQn     = 106,              /**< Combined interrupt indication for GPIO5 signals 0 - 15. */
  GPIO5_Combined_16_31_IRQn    = 107,              /**< Combined interrupt indication for GPIO5 signals 16 - 31. */
  Reserved108_IRQn             = 108,              /**< Reserved */
  Reserved109_IRQn             = 109,              /**< Reserved */
  Reserved110_IRQn             = 110,              /**< Reserved */
  Reserved111_IRQn             = 111,              /**< Reserved */
  WDOG1_IRQn                   = 112,              /**< WDOG1 timer reset interrupt request. */
  WDOG2_IRQn                   = 113,              /**< WDOG2 timer reset interrupt request. */
  KPP_IRQn                     = 114,              /**< Key Pad interrupt request. */
  PWM1_IRQn                    = 115,              /**< hasRegInstance(`PWM1`)?`Cumulative interrupt line for PWM1. Logical OR of rollover, compare, and FIFO waterlevel crossing interrupts.`:`Reserved`) */
  PWM2_IRQn                    = 116,              /**< hasRegInstance(`PWM2`)?`Cumulative interrupt line for PWM2. Logical OR of rollover, compare, and FIFO waterlevel crossing interrupts.`:`Reserved`) */
  PWM3_IRQn                    = 117,              /**< hasRegInstance(`PWM3`)?`Cumulative interrupt line for PWM3. Logical OR of rollover, compare, and FIFO waterlevel crossing interrupts.`:`Reserved`) */
  PWM4_IRQn                    = 118,              /**< hasRegInstance(`PWM4`)?`Cumulative interrupt line for PWM4. Logical OR of rollover, compare, and FIFO waterlevel crossing interrupts.`:`Reserved`) */
  CCM_IRQ1_IRQn                = 119,              /**< CCM interrupt request ipi_int_1. */
  CCM_IRQ2_IRQn                = 120,              /**< CCM interrupt request ipi_int_2. */
  GPC_IRQn                     = 121,              /**< GPC interrupt request 1. */
  Reserved122_IRQn             = 122,              /**< Reserved */
  SRC_IRQn                     = 123,              /**< SRC interrupt request src_ipi_int_1. */
  Reserved124_IRQn             = 124,              /**< Reserved */
  Reserved125_IRQn             = 125,              /**< Reserved */
  CPU_PerformanceUnit_IRQn     = 126,              /**< Performance Unit interrupt ~ipi_pmu_irq_b. */
  CPU_CTI_Trigger_IRQn         = 127,              /**< CTI trigger outputs interrupt ~ipi_cti_irq_b. */
  SRC_Combined_IRQn            = 128,              /**< Combined CPU wdog interrupts (4x) out of SRC. */
  SAI1_IRQn                    = 129,              /**< SAI1 interrupt request. */
  SAI2_IRQn                    = 130,              /**< SAI2 interrupt request. */
  Reserved131_IRQn             = 131,              /**< Reserved */
  ADC1_IRQn                    = 132,              /**< ADC1 interrupt request. */
  ADC_5HC_IRQn                 = 133,              /**< ADC_5HC interrupt request. */
  Reserved134_IRQn             = 134,              /**< Reserved */
  Reserved135_IRQn             = 135,              /**< Reserved */
  SJC_IRQn                     = 136,              /**< SJC interrupt from General Purpose register. */
  CAAM_Job_Ring0_IRQn          = 137,              /**< CAAM job ring 0 interrupt ipi_caam_irq0. */
  CAAM_Job_Ring1_IRQn          = 138,              /**< CAAM job ring 1 interrupt ipi_caam_irq1. */
  QSPI_IRQn                    = 139,              /**< QSPI1 interrupt request ipi_int_ored. */
  TZASC_IRQn                   = 140,              /**< TZASC (PL380) interrupt request. */
  GPT2_IRQn                    = 141,              /**< Logical OR of GPT2 rollover interrupt line, input capture 1 and 2 lines, output compare 1, 2 and 3 interrupt lines. */
  CAN1_IRQn                    = 142,              /**< Combined interrupt of ini_int_busoff,ini_int_error,ipi_int_mbor,ipi_int_txwarning and ipi_int_waken */
  CAN2_IRQn                    = 143,              /**< Combined interrupt of ini_int_busoff,ini_int_error,ipi_int_mbor,ipi_int_txwarning and ipi_int_waken */
  Reserved144_IRQn             = 144,              /**< Reserved */
  Reserved145_IRQn             = 145,              /**< Reserved */
  PWM5_IRQn                    = 146,              /**< Cumulative interrupt line. OR of Rollover Interrupt line, Compare Interrupt line and FIFO Waterlevel crossing interrupt line */
  PWM6_IRQn                    = 147,              /**< Cumulative interrupt line. OR of Rollover Interrupt line, Compare Interrupt line and FIFO Waterlevel crossing interrupt line */
  PWM7_IRQn                    = 148,              /**< Cumulative interrupt line. OR of Rollover Interrupt line, Compare Interrupt line and FIFO Waterlevel crossing interrupt line */
  PWM8_IRQn                    = 149,              /**< Cumulative interrupt line. OR of Rollover Interrupt line, Compare Interrupt line and FIFO Waterlevel crossing interrupt line */
  ENET1_IRQn                   = 150,              /**< ENET1 interrupt */
  ENET1_1588_IRQn              = 151,              /**< ENET1 1588 Timer interrupt [synchronous] request. */
  ENET2_IRQn                   = 152,              /**< ENET2 interrupt */
  ENET2_1588_IRQn              = 153,              /**< MAC 0 1588 Timer interrupt [synchronous] request. */
  Reserved154_IRQn             = 154,              /**< Reserved */
  Reserved155_IRQn             = 155,              /**< Reserved */
  Reserved156_IRQn             = 156,              /**< Reserved */
  Reserved157_IRQn             = 157,              /**< Reserved */
  Reserved158_IRQn             = 158,              /**< Reserved */
  PMU_IRQ2_IRQn                = 159               /**< Brown-out event on either core, gpu or soc regulators. */
} IRQn_Type;
```

设置中断优先级，其实就是设置`中断IRQn`对应的这个`D_IPRIORITYR`寄存器的值。

```c
FORCEDINLINE __STATIC_INLINE void GIC_SetPriority(IRQn_Type IRQn, uint32_t priority)
{
  GIC_Type *gic = (GIC_Type *)(__get_CBAR() & 0xFFFF0000UL);

  gic->D_IPRIORITYR[((uint32_t)(int32_t)IRQn)] = (uint8_t)((priority << (8UL - __GIC_PRIO_BITS)) & (uint32_t)0xFFUL);
}
```

### 3.1.5 使能外设中断`GIC_EnableIRQ`

使能中断，就是设置`D_ISENABLER`寄存器。

```c
FORCEDINLINE __STATIC_INLINE void GIC_EnableIRQ(IRQn_Type IRQn)
{
  GIC_Type *gic = (GIC_Type *)(__get_CBAR() & 0xFFFF0000UL);

  gic->D_ISENABLER[((uint32_t)(int32_t)IRQn) >> 5] = (uint32_t)(1UL << (((uint32_t)(int32_t)IRQn) & 0x1FUL));
}
```

#### 3.1.5 总结

在`imx6ull`芯片中，我们要启动某个硬件外设的中断，调用以下3个函数：

1. 初始化GIC控制器：`void GIC_Init(void)`
2. 设置外设的中断优先级：`void GIC_SetPriority(IRQn_Type IRQn, uint32_t priority)`
3. 使能外设中断：`void GIC_EnableIRQ(IRQn_Type IRQn)`

### 3.2 `imx6ull`的`GIC`寄存器

#### 3.2.1 **中断优先级寄存器`GICD_IPRIORITYR`**

![](./src/0005.jpg)

`GICD_IPRIORITYR`是`Distributor`的核心寄存器，用于设置每个中断的优先级。

+ 寄存器结构：多中断共享的32位寄存器组。

    GICv2的终端数量，由`GICD_TYPER`寄存器的`ITLinesNumber`字段决定，最多支持1020个中断。为了管理这些优先级，每个寄存器是32位，包含4个8位的优先级字段（每个字段控制1个中断）。

+ 位域定义：有效位与RAZ/WI位

    每个8位优先级字段的有效位数由芯片设计决定（ARM规范允许1~8位），未使用的位为RAZ/WI（读返回0，写忽略）。

+ 有效位：有效位是优先级字段中，实际用于表示优先级的位，决定了中断的优先级数值范围。

    + 5位有效位(`imx6ull、am335x`)：`Bit[7:3]`，对应32级优先级（0~31）
    + 4位有效位(`部分Cortex-A9芯片`)：`Bit[7:4]`，对应16级优先级（0~15）
    + 8位有效位(`高端芯片`)：`Bit[7:0]`，对应256级优先级

+ RAZ/WI位：保留位

    未被有效位占据的低位为RAZ/WI位，软件写入时会被忽略，读取时返回0。

+ 优先级编码规则：数值越小，优先级越高

+ 与其他寄存器的协同工作

    + `GICC_PMR（CPU 接口优先级屏蔽寄存器）`
        + 功能：设置CPU可响应的最低优先级阈值
        + 协同：仅当`中断优先级 < GICC_PMR值`时，中断才会被传递给CPU

    + `GICC_BPR（二进制点寄存器）`
        + 功能：划分主优先级和子优先级
        + 协同：若启用子优先级(`BPR < 7`)，`GICD_IPRIORITYR`的有效位会被分成主优先级和子优先级，影响中断的抢占顺序
        
+ SDK代码的定义：MICIX6Y2.h`头文件，写了只支持5位中断优先级，范围0~31

    ```c
    #define __GIC_PRIO_BITS 5         /**< Number of Bits used for Priority Levels */
    ```

+ SDK设置优先级的函数

    ```c
    void GIC_SetPriority(IRQn_Type IRQn, uint32_t priority)
    {
    GIC_Type *gic = (GIC_Type *)(__get_CBAR() & 0xFFFF0000UL);

    gic->D_IPRIORITYR[((uint32_t)(int32_t)IRQn)] = (uint8_t)((priority << (8UL - __GIC_PRIO_BITS)) & (uint32_t)0xFFUL);
    }
    ```

#### 3.2.2 **中断主优先级/子优先级划分寄存器`GICC_BPR`**

![](./src/0006.jpg)

`GICC_BPR`寄存器，用于划分中断的主优先级和子优先级。*建议值：通常我们把中断优先级的有效位数都设为主优先级，不使用子优先级。*

+ 寄存器结构：32位寄存器，但仅低3位(`Bit[2:0]`)有效。`2^3=8`，正好对应了8位优先级。

    + 主优先级位数：`7 - GICC_BPR`
    + 子优先级位数：有效优先级位数(`imx6ull为5`) - 主优先级位数

+ 优先级抢占规则：主优先级决定抢占，子优先级决定顺序

    + 主优先级数值越小，抢占能力越强。高主优先级的中断可以`立即抢占`当前运行的低主优先级中断(无论子优先级如何)
    + 若连个中断的主优先级相同，子优先级数值越小的中断先被处理(不抢占、按顺序执行)

+ 注意事项：通常建议系统`禁用子优先级`，简化配置并减少仲裁逻辑复杂度。

+ SDK中`GIC_Init`函数，`禁用了子优先级`：`GICC_BPR = 7 - 5 = 2`，此时主优先级位数为5，这就是全部的有效位数了。

    ```c
    #define __GIC_PRIO_BITS 5

    void GIC_Init(void)
    {
        /* No subpriority, all priority level allows preemption */
        gic->C_BPR = 7 - __GIC_PRIO_BITS;
    }
    ```

#### 3.2.3 **中断优先级屏蔽寄存器`GICC_PMR`**

![](./src/0007.jpg)

`GICC_PMR`是`Distributor`的核心寄存器，用于设置CPU可响应的最低优先级阈值。仅当`中断优先级高于(数值小于)该阈值`时，CPU才会接收并处理该中断。

+ 寄存器结构：32位寄存器，低8位有效

    `GICC_PMR`是32位寄存器，但仅低8位(`Bit[7:0]`)用于存储优先级屏蔽值。低8位的有效位数由芯片`SoC`实现决定（ARM规范允许1~8位）

+ 位域定义：有效位与RAZ/WI位

    `GICC_PMR`的低8位中，仅部分为有效，其余位为RAZ/WI位。有效位数与GIC支持的优先级有效位数一致（如imx6ull的中原优先级支持5位有效位，GICC_PMR的有效位也为5位）。

    `imx6ull`芯片的sdk中，只有5位有效位，`Bit[7:3]=0b11111`（十进制31），此时仅优先级<31的中断会被响应，优先级31的中断被屏蔽。

    ```c
    void GIC_Init(void)
    {
        /* Make all interrupts have higher priority */
        gic->C_PMR = (0xFFUL << (8 - __GIC_PRIO_BITS)) & 0xFFUL;
    }
    ```

+ 优先级比较规则：数值越小、优先级越高

    仅当`中断优先级 < GICC_PMR值`时，中断才会被传递给CPU。

#### 3.2.4 **外设中断使能寄存器`GICD_ISENABLER`**

![](./src/0008.jpg)

`GICD_ISENABLER`是`Distributor`的核心寄存器，核心功能是启用特定中断，使其能够被分发到CPU接口并触发CPU响应。

+ 功能定位：中断的`启动开关`，仅当`GICD_ISENABLER`中对应位为1时，中断才可能被分发到CPU。

+ SDK使能GIC外设中断的函数中，`IRQn >> 5`实际上就等价于`IRQn / 32`，因为这个寄存器可以控制32个中断源。

```c
void GIC_EnableIRQ(IRQn_Type IRQn)
{
  GIC_Type *gic = (GIC_Type *)(__get_CBAR() & 0xFFFF0000UL);

  gic->D_ISENABLER[((uint32_t)(int32_t)IRQn) >> 5] = (uint32_t)(1UL << (((uint32_t)(int32_t)IRQn) & 0x1FUL));
}
```

#### 3.2.5 **外设中断禁止寄存器`GICD_ICENABLER`**

![](./src/0009.jpg)

`GICD_ICENABLER`是`Distributor`的核心寄存器，核心功能是禁用特定中断，写1屏蔽该中断源的请求。

+ SDK禁用GIC外设中断的函数中，`IRQn >> 5`实际上就等价于`IRQn / 32`，因为这个寄存器可以控制32个中断源。

```c
void GIC_DisableIRQ(IRQn_Type IRQn)
{
  GIC_Type *gic = (GIC_Type *)(__get_CBAR() & 0xFFFF0000UL);

  gic->D_ICENABLER[((uint32_t)(int32_t)IRQn) >> 5] = (uint32_t)(1UL << (((uint32_t)(int32_t)IRQn) & 0x1FUL));
}
```

### 3.3 中断号

Linux内核中的中断管理采用双重标识机制(`IRQ number`和`HW interrupt ID`)，来标识一个来自外设的中断。

1. IRQ Number：CPU需要为每一个外设中断编号，我们称为IRQ Number。这个IRQ Number是一个虚拟的`interrupt ID`，和硬件无关，仅仅是被CPU用来标识一个外设中断
2. HW interrupt ID：对于GIC中断控制器而言，它收集了多个外设的`interrupt request line`并向上传递，因此，GIC控制器需要对外设中断进行编码。GIC控制器用HW interrupt ID来标识外设的中断。入宫只有一个GIC中断控制器，那IRQ number和HW interrupt ID是可以一一对应的。

![](./src/0010.jpg)

#### 3.3.1 为什么需要双重中断号

如果是在GIC控制器级联的情况下，仅仅用HW interrupt ID就不能唯一标识一个外设中断，还需要知道该HW interrupt ID所属的中断控制器(因为HW interrupt ID在不同的中断控制器上是会重复编码的)

![](./src/0011.jpg)

对于驱动工程师而言，我们只希望得到一个IRQ number，而不关心具体是哪个GIC中断控制器上的哪个HW interrupt ID。这样一个好处是，中断相关的硬件发生改变时，驱动软件不需要修改。因此，linux 内核的中断子系统需要提供一个将HW interrupt ID映射到IRQ Number的机制，也就是IRQ domain

驱动无需感知HW ID，只需调用`platform_get_irq()`或`of_irq_get()`获取IRQ number

#### 3.3.2 单片机为何只有一种中断号

1. 硬件环境单一
    + 固定中断控制器：单片机(如stm32)通常集成单一中断控制器(NVIC)，中断源数量有限且物理连接固定
    + 无动态扩展需求：外设与中断线绑定在芯片设计时确定，无需多平台适配
2. 资源限制：IRQ domain映射表需占用内存(约数KB) 在资源紧张的单片机中不必要

