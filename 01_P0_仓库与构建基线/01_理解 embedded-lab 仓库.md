---
tags: [P0, repository, architecture]
aliases: [仓库骨架]
stage: P0
---

# 01｜理解 embedded-lab 仓库

> [!tip] 不懂就点进去
> 这一课主要讲“仓库边界”。如果底层术语还陌生，不用在这里硬扛：
>
> - [[10_基础知识体系/00_基础知识总地图]]
> - [[10_基础知识体系/02_构建与链接/01_从源码到 ELF 的完整流水线]]
> - [[10_基础知识体系/01_计算机与MCU基础/04_地址 地址空间 与内存映射]]

> [!abstract] 本节目标
> 不看具体 C 代码，先回答：**为什么仓库要这样拆？一段新内容到底应该放在哪里？**

## 1. 它不是一个普通 STM32 工程

普通入门工程常只追求“当前程序能跑”。`embedded_lab` 是长期实验平台，要沿着：

```text
GPIO → EXTI/NVIC → Timer → DMA/ADC → UART → I2C/SPI/CAN
→ 软件架构 → RTOS → 网络 → 综合项目
```

持续学习。因此它必须同时保证：实验足够独立、平台基础一致、第三方来源可追溯、资料有依据、后续可复现。

## 2. 目录地图

```text
embedded-lab/
├── docs/           资料与依据
├── third_party/    上游第三方代码
├── platform/       目标硬件公共构建事实
├── common/         经验证后提炼出的通用逻辑
├── frameworks/     软件架构 / RTOS 等框架
├── experiments/    单项实验
├── projects/       综合项目
├── scripts/        辅助流程
├── cmake/          构建辅助文件
└── CMakeLists.txt  顶层构建入口
```

真正要记的是“职责”，不是目录名本身。

## 3. 每个目录回答什么问题？

| 目录 | 核心问题 |
|---|---|
| `docs/` | 这个结论依据哪份官方资料？ |
| `third_party/` | 上游厂商/开源项目交付了什么代码？ |
| `platform/` | 所有当前 MCU 实验共同需要什么构建与运行基础？ |
| `common/` | 哪些逻辑已被证明值得跨实验复用？ |
| `frameworks/` | 软件怎样被不同框架/RTOS 组织？ |
| `experiments/` | 这一次具体验证哪个机制？ |
| `projects/` | 怎样组合已经验证过的能力？ |
| `scripts/` | 哪些稳定的人工流程值得自动化？ |

## 4. 用 GPIO 练习目录边界

假设要点亮板载 LED：

| 问题 | 去哪里 | 原因 |
|---|---|---|
| LED 接在哪个引脚？ | `docs/board/` | 板级硬件事实 |
| `GPIOB->MODER` 的结构定义在哪？ | `third_party/` | ST Device Header |
| 当前用哪颗 MCU、哪份 startup？ | `platform/` | 所有实验共享的平台事实 |
| 为什么要开 GPIOB 时钟？ | `experiments/001_gpio_output/` + RM | 本实验正在研究的机制 |
| 通用 ring buffer 后续放哪？ | 成熟后 `common/` | 与具体 GPIO 无关且可复用 |

如果把 GPIO 操作一开始就封装成 `board_led_on()`，虽然方便，却把本实验真正要学的寄存器机制藏掉了。

## 5. Platform 和 Experiment 的边界

一句话：

> **Platform 回答“程序运行在哪”，Experiment 回答“这次要让硬件做什么”。**

Platform 适合统一：

```text
MCU 宏 / CPU flags / CMSIS include / startup / system / linker script
```

001 实验自己研究：

```text
RCC_AHB1ENR → GPIOB_MODER → BSRR/ODR → 引脚电平 → LED
```

## 6. `common/` 的抽象原则

> [!important]
> **先写实验，再提炼公共件。**

考虑上提前至少问：

1. 是否已经在两个以上真实场景重复出现？
2. 是否与某颗具体 MCU / 当前实验机制解耦？
3. 抽象后是否会遮住正在学习的东西？
4. 接口边界是否已经稳定？

工程中的“过早抽象”和笔记中的“过早知识分类”其实是同一个问题。

## 7. Experiment 和 Project 不一样

```text
Experiment = 拆开学
Project    = 组合用
```

实验尽量只改变一个主要变量；项目允许 GPIO、Timer、DMA、UART、RTOS 等多个能力共同出现。

## 8. 一次实验怎样形成结论？

不是：

```text
写代码 → 能跑 → 完成
```

而是：

```text
提出问题
→ 查一手资料
→ 预测
→ 最小实现
→ 构建/烧录
→ 软件侧观察
→ 物理侧观察
→ 故障注入
→ 解释差异
→ 形成结论
```

例如 GPIO：寄存器读回正确只是软件证据；真正的物理输出还应由 LED、万用表或逻辑分析仪进一步验证。

## 9. 常见误区

- **Platform = 所有板级代码**：不对，当前学习机制不能被平台层提前隐藏。
- **`third_party/` = 我们自己的库**：不对，它强调上游来源与版本边界。
- **`common/` 越多越工程化**：不一定，过早抽象只会增加认知负担。
- **目录越细越专业**：没有真实职责边界时，只是在制造形式复杂度。

## 10. 离开本页前

闭卷回答：

1. `docs/` 与 `third_party/` 的根本区别是什么？
2. 为什么每个实验不各复制一份 startup/linker script？
3. GPIO 寄存器操作为什么暂时留在实验目录？
4. 一段代码什么时候才值得进入 `common/`？
5. `experiment` 和 `project` 的目标有何不同？
6. 新增 SPI Flash 实验时，你能判断原理图、Device Header、实验代码、通用 CRC 分别去哪吗？

下一步：[[02_资料分层与 STM32CubeF4]]。
