---
tags: [P0, docs, STM32CubeF4, CMSIS]
stage: P0
---

# 02｜资料分层与 STM32CubeF4

> [!tip] 深挖入口
> - [[10_基础知识体系/03_Cortex-M与启动中断/01_Cortex-M4 与 STM32F446 的关系]]
> - [[10_基础知识体系/01_计算机与MCU基础/05_MMIO 为什么寄存器像内存]]
> - [[10_基础知识体系/01_计算机与MCU基础/07_指针到底是什么]]

> [!abstract] 本节目标
> 建立“遇到问题先判断属于哪一层”的习惯，并理解官方文档与官方代码包各自解决什么问题。

## 1. 四层资料模型

```text
Board      → 开发板怎样连接
MCU        → 具体 STM32 外设怎样工作
Cortex-M   → CPU 内核机制
Protocol   → 换一颗 MCU 仍成立的通信规则
```

### Board：NUCLEO-F446ZE

主要查：

- 板载 LED / 按钮接到哪个 pin
- ST-LINK 怎样连接目标 MCU
- MCO/HSE 路径
- 跳线与接口
- 原理图上的电阻、有效电平

典型资料：Nucleo-144 用户手册、板卡原理图。

### MCU：STM32F446

主要查：GPIO、RCC、USART、TIM、DMA、ADC 等 STM32F446 外设。

- Datasheet 更偏能力、引脚、容量、电气特性。
- RM0390 更偏外设机制、寄存器、配置顺序与状态。

### Cortex-M4

主要查：NVIC、SysTick、异常、MSP/PSP、SCB、PRIMASK/BASEPRI、FPU 等内核机制。

### Protocol

主要查：UART 帧、SPI CPOL/CPHA、I2C ACK/NACK、CAN 仲裁等跨 MCU 仍成立的规则。

## 2. 实际例子：LED 为什么会亮？

不要只说“GPIO 输出高电平”。应分层：

```text
Board
→ LED 实际接到哪个 MCU pin？高电平还是低电平有效？

MCU / RM
→ 该 GPIO port 的时钟怎样使能？MODER/BSRR 怎样工作？

Device Header
→ RCC->AHB1ENR、GPIOB->MODER 这些 C 名字怎样映射到地址？

实物
→ 寄存器值、引脚电平和 LED 状态是否一致？
```

一份资料只回答它擅长的问题，不要求“一本手册包办全部”。

## 3. Datasheet、RM、Header 不要混为一谈

| 来源 | 主要回答 |
|---|---|
| Datasheet | 芯片有什么、引脚/容量/限制是什么 |
| Reference Manual | 外设寄存器和机制怎样工作 |
| Board Manual/Schematic | 板子实际上怎样连接 |
| Device Header | 手册中的地址/位怎样用 C 名字表达 |
| HAL/LL | ST 怎样进一步封装操作接口 |

Header 告诉你“名字映射到哪里”，RM 告诉你“为什么这样写会产生这个行为”。

## 4. 为什么接入 STM32CubeF4？

`third_party/STM32CubeF4` 是统一的 ST 官方代码原材料来源，后续会用到：

```text
CMSIS-Core
STM32F4 Device Header
startup_stm32f446xx.s
system_stm32f4xx.c
HAL / LL（后续对比）
```

如果每个实验都手工复制所需文件，会逐渐失去版本来源与可复现性。

## 5. Submodule 的三个层次

```text
.gitmodules
→ 记录路径与远程 URL

主仓库某个 commit
→ 记录子模块应停在哪个具体 commit

STM32CubeF4 自己
→ 有自己的 Git history / tag / branch
```

真正锁定版本的是主仓库记录的 submodule commit，而不是“URL 没变”。

首次 clone 后常需要：

```bash
git submodule update --init --recursive
```

排查复现问题时最好同时记录：

```text
embedded_lab commit
STM32CubeF4 commit
```

## 6. CMSIS-Core 与 Device Header

```text
ARM Cortex-M4
   ↓
CMSIS-Core
   ↓
NVIC / SysTick / SCB / core registers

STM32F446
   ↓
STM32 Device Header
   ↓
RCC / GPIO / USART / TIM / DMA / IRQn
```

### CMSIS-Core

把 Cortex-M 内核访问方式标准化，例如后续可能看到：

```c
NVIC_EnableIRQ(...);
__disable_irq();
```

### Device Header

例如 `stm32f446xx.h`，描述具体芯片外设寄存器、基地址、IRQ 等。

> Device Header 主要是“描述硬件”，不是“替你完成驱动”。

## 7. 官方资料怎样归档？

推荐流程：

```text
完整官方资料
  ↓
先归档
  ↓
做实验时真正去查
  ↓
Obsidian 只记录自己的理解、定位路径和结论
```

不要在还没用到时给几千页 RM 制造大量二手摘要。

## 8. 实用的资料定位算法

遇到问题依次问：

1. 是板上连接吗？→ Board。
2. 是 STM32 外设机制吗？→ RM / MCU。
3. 是 Cortex-M 内核机制吗？→ Cortex-M。
4. 是协议本身吗？→ Protocol。
5. 想找 C 里的寄存器名字？→ Device Header。
6. 想比较 ST 封装？→ LL/HAL，但仍要回 RM 理解机制。

## 9. 常见误区

- Datasheet 不是 Reference Manual 的替代品。
- 看 Header 不能替代看 RM。
- 官方 example 能跑，也可能针对另一块板、另一时钟或另一引脚。
- `.gitmodules` 的 URL 固定，不代表代码版本天然固定。

## 10. 离开本页前

1. Board / MCU / Cortex-M / Protocol 各回答什么？
2. Datasheet 与 RM 的职责有什么差异？
3. Device Header 与 HAL 有什么区别？
4. CMSIS-Core 主要描述什么？
5. 主仓库是怎样锁住 STM32CubeF4 具体版本的？
6. 如果 LED 不亮，你能从板级资料一路追到寄存器和实物吗？

下一步：[[03_CMake 与交叉编译]]。
