---
tags: [P0, platform, CMSIS, STM32F446ZE]
stage: P0
---

# 04｜platform/stm32f446ze 平台层

> [!tip] 深挖入口
> - [[10_基础知识体系/03_Cortex-M与启动中断/01_Cortex-M4 与 STM32F446 的关系]]
> - [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]
> - [[10_基础知识体系/01_计算机与MCU基础/06_Flash SRAM ROM RAM]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/09_SystemInit 到底处在什么位置]]

> [!abstract] 本节目标
> 理解 Platform 不是“万能 BSP”，而是一组所有实验共享的**目标硬件构建事实**。

## 1. 为什么要有 Platform？

如果每个实验都自己维护：

```text
startup
system_stm32f4xx.c
CMSIS include
STM32F446xx
CPU flags
linker script
```

很快就会出现 001/002/003 的基础配置彼此不一致。届时实验现象不同，无法判断究竟来自外设机制还是地基差异。

所以 Platform 的任务是：

> **让所有实验站在同一套已知基础上。**

## 2. Platform 不是什么？

它不是“所有板级功能的集合”。当前阶段不应自动完成：

```text
LED 初始化
GPIOB 时钟使能
UART 初始化
DMA 配置
```

因为这些正是后续实验要学习的对象。

一句话：

> **Platform 回答“程序运行在哪”，Experiment 回答“这次让硬件做什么”。**

## 3. 当前平台至少统一五类事实

| 问题 | 平台提供 / 选择什么 |
|---|---|
| 编译哪颗芯片？ | `STM32F446xx` |
| 生成什么指令？ | Cortex-M4 / Thumb / FPU / ABI 参数 |
| 寄存器名字从哪来？ | CMSIS-Core + STM32F446 Device Header include |
| 复位后执行什么？ | startup + `system_stm32f4xx.c` |
| 最终地址怎么安排？ | STM32F446ZE 对应 linker script |

## 4. `STM32F446xx` 为什么重要？

它不是普通标签，而会参与 ST 头文件的条件编译，影响：

```text
具体设备头文件
外设与寄存器定义
IRQ 定义
部分芯片能力条件编译
```

宏选错可能直接编译失败，也可能更危险：编译成功但硬件定义不匹配。

## 5. Cortex-M4F 参数为什么属于 Platform？

编译器需要知道：

```text
目标 CPU
Thumb 状态
FPU 类型
浮点 ABI
```

因此常见参数包括：

```text
-mcpu=cortex-m4
-mthumb
-mfpu=fpv4-sp-d16
-mfloat-abi=...
```

它们是目标硬件事实，不应由每个实验各写一遍。

## 6. CMSIS 与 Device include

实验可能只写：

```c
#include "stm32f446xx.h"
```

但编译器要能找到 CMSIS Core 与 ST Device Header。Platform 统一传播 include path，避免实验关心第三方目录内部结构。

这和 [[02_资料分层与 STM32CubeF4]] 中的“资料职责”相呼应：实验使用名字，但机制仍回 RM 核对。

## 7. Startup 与 System 为什么属于平台地基？

Startup 和目标 MCU 的向量表、IRQ 数量、复位入口紧密相关；`system_stm32f4xx.c` 提供 `SystemInit()`、`SystemCoreClock` 等系统层逻辑。

它们不是 GPIO/UART 某一个实验的专属实现，所以应由平台统一选择。

Startup 具体机制见 [[06_Startup 与 Reset_Handler]]。

## 8. Linker Script 为什么由 Platform 选择？

当前目标的主存储区：

```text
Flash：0x08000000 起，512 KiB
SRAM ：0x20000000 起，128 KiB
```

拿错另一颗 MCU 的 `.ld`，可能链接失败，也可能链接成功却使用不存在的地址。

所以“当前目标用哪份 linker script”属于平台事实；“linker script 自己怎样工作”则见 [[05_Linker Script 与内存布局]]。

## 9. Platform 怎样作用到实验？

核心不是目录名，而是 CMake target 依赖：

```text
exp001_gpio_output
        ↓ target_link_libraries(...)
platform_stm32f446ze
        ↓
compile definitions / include dirs / CPU flags
startup / system / link options
```

因此“实验依赖 Platform”最终必须能在详细构建命令里找到证据。

## 10. `HSE_VALUE=8000000U` 只是一项频率声明

它表达：

> **如果软件使用 HSE，按 8 MHz 输入频率计算。**

它不会自动：

```text
打开 HSE
等待 Ready
配置 PLL
设置 Flash latency
切换 SYSCLK
设置 AHB/APB prescaler
```

所以：

```text
HSE_VALUE = 8 MHz
```

不能推出“CPU 当前运行在 8 MHz”，更不能推出“已经运行在 180 MHz”。

## 11. 为什么当前早期实验仍按 HSI 16 MHz？

这是课程节奏设计。

001～004 先建立 GPIO / EXTI / NVIC 基本因果链，不提前引入：

```text
HSE bypass
PLL
Flash latency
APB prescaler
Timer clock ×2
```

这些会在既定 Clock checkpoint 系统学习。

## 12. `SystemInit()` 不能靠函数名猜

看到：

```c
SystemInit();
```

不能直接脑补“它会配置最终 180 MHz”。必须打开**当前真正参与构建**的 `system_stm32f4xx.c`，结合 RCC 寄存器和 `SystemCoreClock` 判断。

第三方代码版本变化后，同名函数实现也可能变化。

## 13. Platform 与 Experiment 的边界练习

### 适合 Platform

```text
目标 MCU 宏
CPU compile flags
CMSIS / Device include
startup
system
linker script
公共 board clock constant
```

### 001 暂时留在 Experiment

```text
GPIOB clock enable
PB0 MODER
BSRR/ODR
delay loop
LED on/off
```

因为这些是 001 正在验证的机制。

## 14. 实际核对清单

对某个实验 target 的 `-v` 构建输出，检查：

- `STM32F446xx` 是否真的出现
- Cortex-M4F 参数是否真的出现
- CMSIS / Device include 来自哪里
- startup / system 是否参与构建
- `.ld` 是否真的传给最终链接命令

## 15. 常见误区

- Platform 不是 HAL。
- INTERFACE platform target 可能根本不产生 `.a`。
- 写了 `HSE_VALUE` 不代表切到 HSE。
- “板上 LED 方便封装”不等于当前学习阶段应该封装。

## 16. 离开本页前

1. Platform 为什么存在？
2. 哪五类信息属于当前平台地基？
3. `STM32F446xx` 错了会怎样？
4. 为什么 linker script 应由平台选择？
5. `HSE_VALUE=8000000U` 能证明什么、不能证明什么？
6. 为什么 GPIOB 时钟使能暂时不进 Platform？

下一步：[[05_Linker Script 与内存布局]]。
