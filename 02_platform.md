接下来我们正式进入：

```
platform/stm32f446ze/
```

而且从这里开始会明显更有“嵌入式开发”的感觉，因为我们将第一次真正解决：

```
STM32CubeF4 里那么多东西
        ↓
NUCLEO-F446ZE 到底需要哪些？
        ↓
startup 选哪个？
CMSIS 头文件怎么接？
CPU 编译参数是什么？
linker script 从哪里来？
SystemInit 怎么进入启动流程？
        ↓
最终先得到一个最小可链接的 STM32F446 ELF
```
首先：
把 `platform/stm32f446ze/` 从“空壳”变成**最小可用平台层**。

这一步我们只接 4 样东西：

```
Cortex-M4 CMSIS Core 头文件
STM32F446 Device 头文件
system_stm32f4xx.c
startup_stm32f446xx.s
```

ST 官方的 `startup_stm32f446xx.s` 和 `system_stm32f4xx.c` 正是 STM32F446 对应的启动与 CMSIS system 文件。
### ① 先确认文件确实存在

在仓库根目录执行：

```
ls third_party/STM32CubeF4/Drivers/CMSIS/Include/core_cm4.h

ls third_party/STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Include/stm32f446xx.h

ls third_party/STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/system_stm32f4xx.c

ls third_party/STM32CubeF4/Drivers/CMSIS/Device/ST/STM32F4xx/Source/Templates/gcc/startup_stm32f446xx.s
```

四条都应该能找到文件。

这四样分别可以先这样理解：

```
core_cm4.h
    ↓
ARM Cortex-M4 内核定义
NVIC / SysTick / SCB / CPU intrinsic...


stm32f446xx.h
    ↓
STM32F446 自己的定义
GPIO / RCC / USART / TIM / DMA...
寄存器地址、结构体、IRQ编号...


system_stm32f4xx.c
    ↓
提供 SystemInit()
SystemCoreClock
SystemCoreClockUpdate()


startup_stm32f446xx.s
    ↓
中断向量表
Reset_Handler
初始化 .data / .bss
调用 SystemInit()
最后进入 main()
```

特别注意：**这些文件继续留在 `third_party/`，不要复制到 `platform/`。**

`platform` 的职责是：

> “选择并使用哪些官方文件”，而不是再复制一份官方代码。

---

## ② 修改 `platform/stm32f446ze/CMakeLists.txt`

你现在这里应该还是：

```
add_library(platform_stm32f446ze INTERFACE)

target_include_directories(
    platform_stm32f446ze
    INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}
)
```

把它改成下面这样：

```CMake
# platform/stm32f446ze：NUCLEO-F446ZE 板级粘合

add_library(platform_stm32f446ze INTERFACE)

set(STM32CUBE_F4_DIR
    ${PROJECT_SOURCE_DIR}/third_party/STM32CubeF4
)

set(STM32F4_CMSIS_DEVICE_DIR
    ${STM32CUBE_F4_DIR}/Drivers/CMSIS/Device/ST/STM32F4xx
)

# ---------------------------------------------------------------------------
# CMSIS / device headers
# ---------------------------------------------------------------------------

target_include_directories(
    platform_stm32f446ze
    INTERFACE

    ${CMAKE_CURRENT_SOURCE_DIR}

    ${STM32CUBE_F4_DIR}/Drivers/CMSIS/Include
    ${STM32F4_CMSIS_DEVICE_DIR}/Include
)

# ---------------------------------------------------------------------------
# MCU selection / board clock facts
# ---------------------------------------------------------------------------

target_compile_definitions(
    platform_stm32f446ze
    INTERFACE

    STM32F446xx
    HSE_VALUE=8000000U
)

# ---------------------------------------------------------------------------
# Startup / CMSIS system source
# ---------------------------------------------------------------------------

target_sources(
    platform_stm32f446ze
    INTERFACE

    ${STM32F4_CMSIS_DEVICE_DIR}/Source/Templates/system_stm32f4xx.c
    ${STM32F4_CMSIS_DEVICE_DIR}/Source/Templates/gcc/startup_stm32f446xx.s
)

# ---------------------------------------------------------------------------
# Cortex-M4F architecture
# ---------------------------------------------------------------------------

target_compile_options(
    platform_stm32f446ze
    INTERFACE

    -mcpu=cortex-m4
    -mthumb
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard
)

target_link_options(
    platform_stm32f446ze
    INTERFACE

    -mcpu=cortex-m4
    -mthumb
    -mfpu=fpv4-sp-d16
    -mfloat-abi=hard
)
```

### 这里最值得理解的是 `STM32F446xx`

你前面看过：

```
#include "stm32f4xx.h"
```

这个总入口头文件并不知道你到底是哪颗 STM32F4。

它内部会根据宏判断：

```
STM32F405xx
STM32F407xx
STM32F429xx
STM32F446xx
...
```

然后选择对应芯片头文件。

ST 官方源码也是按照这种预处理宏来选择具体 Device header 的。

所以：

```
STM32F446xx
```

实际上是在告诉 CMSIS：

> **我们现在编译的具体芯片就是 STM32F446。**

这不是随便写的工程宏。

---

### `HSE_VALUE=8000000U` 又是什么？

ST 提供的 `system_stm32f4xx.c` 默认：

```
HSE_VALUE = 25000000
```

如果工程没有自己定义它。

但我们的 NUCLEO-F446ZE 默认外部高速时钟来源是 **ST-LINK MCO 8 MHz**。

所以平台层应该明确告诉官方 system 文件：

```
这块板的 HSE 输入频率 = 8 MHz
```

因此：

```
HSE_VALUE=8000000U
```

注意它**并不意味着我们现在已经启用了 HSE**。

这是两个不同概念：

```
HSE_VALUE=8000000
        ↓
描述：
“如果使用 HSE，它是多少 Hz”


RCC 配置 HSE bypass
        ↓
动作：
“真正让 MCU 使用这个外部时钟”
```

后者我们现在**不做**。

因此目前 MCU 上电仍然可以保持默认 HSI 16 MHz。这样等我们真正学 RCC 时，再亲手把：

```
HSI 16 MHz

↓ 学习 RCC

HSE bypass 8 MHz
↓
PLL
↓
180 MHz
```

一步一步做出来，而不是平台提前帮你藏掉。

---

## ③ 为什么现在还没有 linker script？

这是故意的。

你可能已经发现：

```
startup      ✅
CMSIS        ✅
system       ✅
CPU 参数     ✅

linker       ❌
main.c       ❌
```

所以现在还不能产生真正的 `.elf`。

这是我们刻意分开的。

下一步才会处理：

```
platform/stm32f446ze/
└── linker/
    └── STM32F446ZE.ld
```

STM32CubeF4 官方的 NUCLEO-F446ZE linker script明确给出了：

```
Flash:
0x08000000
512 KB

RAM:
0x20000000
128 KB
```

以及 `.isr_vector`、`.text`、`.data`、`.bss`、stack/heap 等 section 布局。

这部分非常值得单独学，不应该和刚才 CMSIS/startup 一起一口气塞进去。
现在要再往深一层理解：

> **`platform/` 在整个代码依赖关系中，到底承担什么责任。**

你之前的笔记已经知道 `platform` 是“具体目标硬件的适配层”，也知道它和 `third_party/common/frameworks/experiments` 的边界。  
现在真正缺的是：**为什么一开始写代码，就必须先经过它。**

---

## 先暂时忘掉 CMake

假设我们现在马上开始：

```
experiments/001_gpio_output/main.c
```

你想写：

```
#include "stm32f446xx.h"

int main(void)
{
    RCC->AHB1ENR |= ...;
}
```

看起来只有一个 `main.c`。

但这个 `main.c` 真要变成 STM32F446ZE 能运行的固件，背后必须回答一大串问题：

```
我是哪颗 MCU？
        ↓
STM32F446ZE

它是什么 CPU？
        ↓
Cortex-M4F

寄存器定义从哪来？
        ↓
stm32f446xx.h

NVIC / SysTick / CPU 定义从哪来？
        ↓
CMSIS Core

CPU 上电以后从哪开始执行？
        ↓
startup_stm32f446xx.s

谁初始化 .data 和 .bss？
        ↓
startup

谁提供 SystemInit()？
        ↓
system_stm32f4xx.c

用什么 CPU 指令集编译？
        ↓
-mcpu=cortex-m4 -mthumb ...

Flash/RAM 在哪里？
        ↓
linker script
```

注意：

**这些问题一个都不是“GPIO 输出实验”的研究内容。**

GPIO 实验真正想研究的是：

```
RCC 为什么要开 GPIO 时钟？
MODER 怎么工作？
ODR / BSRR 有什么区别？
GPIO 输出波形是什么样？
```

于是问题就出现了：

> 那些“每个实验都必须回答，但又不是每个实验研究对象”的东西，到底放哪里？

答案就是：

# `platform/`

这就是你现在需要补上的核心理解。

---

# `platform` 不是“放硬件代码的文件夹”

它更准确的角色是：

> **定义：在这个仓库中，“STM32F446ZE 这个目标平台”到底意味着什么。**

所以看到：

```
platform/stm32f446ze/
```

你脑子里以后不要只想到：

> “这里放板级代码。”

而应该想到：

```
只要某个程序声称：

“我要运行在 STM32F446ZE 上”

那么它就必须继承这一整套事实：

CPU        = Cortex-M4F
MCU        = STM32F446ZE
CMSIS      = STM32F4 CMSIS
startup    = startup_stm32f446xx.s
system     = system_stm32f4xx.c
Flash/RAM  = STM32F446ZE 的存储布局
HSE        = 这块板对应的外部时钟事实
...
```

这才是 `platform`。

---

## 所以为什么上来先改 `platform/stm32f446ze/CMakeLists.txt`？

因为 CMake 需要一种办法把上面这一整包东西表达出来。

于是我们创建：

```
add_library(platform_stm32f446ze INTERFACE)
```

这个名字其实可以用中文读：

> **“STM32F446ZE 平台能力包”**

它暂时不是传统意义上的 `.a` 库。

而是一个 CMake target，用来告诉其他 target：

> “如果你要运行在 STM32F446ZE 上，你需要这些东西。”

然后开始往这个“平台能力包”上挂属性。

---

### 第一件事：告诉你头文件在哪

```
target_include_directories(
    platform_stm32f446ze
    INTERFACE
    ...
)
```

意思就是：

```
谁依赖 platform_stm32f446ze
        ↓
谁就自动知道：

CMSIS Core 头文件在哪
STM32F4 Device 头文件在哪
platform 自己的头文件在哪
```

以后实验不需要写：

```
target_include_directories(exp001 ...一大堆路径...)
```

---

### 第二件事：告诉代码“你是哪颗 MCU”

```
target_compile_definitions(
    platform_stm32f446ze
    INTERFACE
    STM32F446xx
)
```

意思就是：

```
谁依赖这个 platform
        ↓
编译时自动拥有 STM32F446xx
```

所以 CMSIS 才能正确选择：

```
stm32f446xx.h
```

这个定义不是 GPIO 实验自己的东西。

因为以后：

```
001_gpio_output
002_gpio_input
003_exti
004_nvic
005_timer
...
```

全都是 STM32F446ZE。

所以它当然应该属于：

```
platform/stm32f446ze
```

---

### 第三件事：把启动文件挂进去

```
target_sources(
    platform_stm32f446ze
    INTERFACE
    system_stm32f4xx.c
    startup_stm32f446xx.s
)
```

意思也是：

```
只要某个最终固件使用 STM32F446ZE 平台
        ↓
它就需要 STM32F446 的 startup
        ↓
也需要 STM32F4 system 文件
```

而不是：

```
001 自己找 startup
002 再找一次 startup
003 再找一次
...
```

---

### 第四件事：CPU 编译参数

```
-mcpu=cortex-m4
-mthumb
-mfpu=fpv4-sp-d16
-mfloat-abi=hard
```

这些参数属于谁？

显然不是：

```
GPIO
UART
DMA
```

因为换任何一个实验，它们都不变。

它们实际上是在描述：

> **STM32F446ZE 这颗 MCU 使用什么 CPU。**

所以依然属于：

```
platform
```

---

# 这样你再看未来的实验 CMake，就会突然很清楚

未来可能有：

```
add_executable(exp001_gpio_output
    main.c
)

target_link_libraries(
    exp001_gpio_output
    PRIVATE
    platform_stm32f446ze
)
```

表面只有一句：

```
platform_stm32f446ze
```

但背后实际上继承的是：

```
exp001_gpio_output
        │
        │ 我要运行在 STM32F446ZE
        ▼
platform_stm32f446ze
        │
        ├── Cortex-M4F 编译参数
        ├── STM32F446xx 宏
        ├── CMSIS Core include
        ├── CMSIS Device include
        ├── startup
        ├── system
        ├── linker script       ← 后面加
        └── 板级基础事实        ← 以后逐渐加
```

于是：

```
experiment
```

只需要关心：

```
“我这次到底实验什么？”
```
# 这和我们之前纠正过的“platform 不屏蔽硬件”完全不矛盾

这是非常关键的一点。

比如 GPIO 实验：

```
platform
    │
    ├── 告诉你 MCU 是 STM32F446
    ├── 给你 CMSIS
    ├── 帮你启动 CPU
    ├── 告诉编译器这是 Cortex-M4
    └── 以后告诉 linker Flash/RAM 在哪
            ↓

experiment
    │
    ├── 自己打开 RCC GPIO 时钟
    ├── 自己配置 MODER
    ├── 自己操作 BSRR
    ├── 自己制造“不开时钟”的故障
    └── 自己观察 GPIO 波形
```

所以：

> **platform 替你解决“我怎样成为一个能在 STM32F446ZE 上运行的程序”。**

而：

> **experiment 解决“这次我要研究 STM32F446ZE 的什么机制”。**

这两句话非常重要。

---

# 再回头看 `third_party`，区别也会更清楚

`third_party/STM32CubeF4` 就像一个巨大的官方仓库：

```
STM32CubeF4
├── F401
├── F405
├── F407
├── F411
├── F429
├── F446
├── HAL
├── CMSIS
├── BSP
├── startup xxxxx
└── ...
```

它只是：

> **原材料仓库。**

它不会主动告诉我们的工程：

```
你应该使用 STM32F446xx
你应该选 startup_stm32f446xx.s
你应该用 Cortex-M4F
你的 HSE 是多少
你的 linker 是哪个
```

谁负责从这一大堆原材料中作出选择？

就是：

```
platform/stm32f446ze
```

所以可以形成一个非常好记的关系：

```
third_party
    =
仓库里有什么原材料？


platform
    =
针对“我的硬件”，到底选哪些原材料，
并怎么组合？


experiment
    =
利用这套硬件环境，我这次研究什么？
```

---

# 我觉得你真正缺的就是这个“依赖视角”

你之前是按**目录视角**理解仓库：

```
third_party 放官方代码
platform 放平台代码
common 放通用代码
experiments 放实验
```

这个没有错。

现在要升级成**依赖视角**：

```
                   experiment
                       │
                       │ 我需要运行在某个平台
                       ▼
                    platform
                       │
             ┌─────────┴─────────┐
             │                   │
       选择官方代码          描述目标硬件
             │                   │
             ▼                   │
        third_party              │
                                 │
                           CPU / MCU / Board
```

这样 `platform/stm32f446ze/CMakeLists.txt` 就不再是一坨突然冒出来的 CMake 代码。

它其实是在做一句非常简单的事情：

> **“用 CMake 把 STM32F446ZE 这个平台描述出来。”**