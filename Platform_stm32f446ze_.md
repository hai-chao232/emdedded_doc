# Platform/stm32f446ze/

## 先理解C语言

> C 语言代码 → 内存 → CPU → 芯片里的硬件 → 开发板上的真实器件
> 
> `.h` \-\-\> 描述文件、`.c` \-\-\> 实现文件
> 
> 

总结：

- **C语言 \-\-\> 对内存的读写 和 逻辑** 

- **而芯片把某些特殊的内存地址映射成了硬件寄存器** 

- **因此对这些地址进行读写就等于控制真实硬件**

**为什么 *****MCU\(芯片\)***** **** 可以让 "内存操作" 控制硬件** 

> 假设 STM32F446 有这样一个寄存器：
> 
> `GPIOA ODR`，地址：*0x40020014* 
> 
> 于是这个地址就不再是 *普通 RAM* ，变成了 *特殊的内存地址* 
> 
> ---
> 
> 芯片内部规定：
> 
> ```Plain Text
> 0x40020014
>       ↓
> GPIOA 的输出数据寄存器
> ```
> 
> 于是我们可以这样写：
> 
> `*(volatile uint32_t *)0x40020014 |= (1 << 5);`
> 
> 也就是向这个地址写数据。
> 
> ---
> 
> 然后 STM32 内部的硬件会看到：
> 
> ```Plain Text
> 有人访问 0x40020014
>         ↓
> 这是 GPIOA ODR
>         ↓
> 把数据交给 GPIOA 外设
>         ↓
> 改变 PA5 输出状态
> ```
> 
> 所以这里有一个非常漂亮的设计：
> 
> ```Plain Text
> CPU
>                │
>                │ 读/写地址
>                ▼
>      地址总线/数据总线/(总线系统)
>                │
>       ┌────────┼─────────┐
>       │        │         │
>       ▼        ▼         ▼
>      RAM      Flash     外设寄存器
>                         │
>                  ┌──────┼──────┐
>                  ▼      ▼      ▼
>                 GPIO   UART    TIM
> ```
> 
> ---
> 
> 例如：
> 
> 1. 程序执行 `GPIOA->ODR |= (1<<5);`
> 
> 2. CPU 把地址 *0x40020014* 放到地址总线上，把 *数据* 放到数据总线上。
> 
> 3. 总线系统识别这个地址属于 GPIOA 外设寄存器。
> 
> 4. GPIOA 外设硬件更新寄存器，并驱动 PA5 引脚输出高电平。
> 
> 这就是：*软件 \-\-\> 内存地址 \-\-\> 硬件* 
> 
> 

一个芯片里面是有非常多的寄存器

它们本质上都是一组组 *特殊的内存位置* 

## 然后理解为什么需要 ST 的头文件

> 假设没有官方文件，我们要想控制 GPIOA：
> 
> - 我们得先查手册找寄存器对应的地址与功能，如：
> 
>     ```Plain Text
>     GPIOA base address = 0x40020000
>     
>     MODER offset = 0x00
>     OTYPER offset = 0x04
>     OSPEEDR offset = 0x08
>     ...
>     ```
> 
> - 知道了这些信息后，我们再用 C语言 描述：
> 
>     ```C
>     #define GPIOA_BASE 0x40020000
>     
>     #define GPIOA_MODER    (*(volatile uint32_t*)(GPIOA_BASE + 0x00))
>     #define GPIOA_OTYPER   (*(volatile uint32_t*)(GPIOA_BASE + 0x04))
>     #define GPIOA_OSPEEDR   (*(volatile uint32_t*)(GPIOA_BASE + 0x08))
>     ...
>     ```
> 
> 

> 这还只是一个引脚的部分寄存器，而这些信息都是固定的
> 
> 所以 ST官方 帮我们做了这些 固定重复的 描述性工作，提供：`stm32f446xx.h`
> 
> ```C
> Reference Manual
>         |
>         | 告诉我们硬件规则
>         👇
>    stm32f446xx.h
>         |
>         | 帮我们用 C 表达描述这些硬件
>         👇
>     我们的 C 代码
> ```
> 
> 

## 理解 ARM 的 CMSIS

首先确定一点：**CMSIS 是一个更大的标准体系** 

现在这个阶段我们只是看 **CMSIS\-Core** 

这个体系还有：

- **CMSIS\-DSP**：DSP 算法库

- **CMSIS\-RTOS**：RTOS 统一接口

- **CMSIS\-NN**：神经网络相关

> STM32F446 芯片里面真的有一个：`Cortex-M4 CPU` 硬件，里面有：
> 
> ```Plain Text
> CPU寄存器
> R0 ~ R15
> xPSR
> MSP
> PSP
> 
> NVIC
> SysTick
> SCB
> MPU（具体型号视配置）
> FPU
> ```
> 
> 这些是真实的硬件模块。
> 
> ---
> 
> **CMSIS\-Core 主要就是描述 Cortex\-M 内核，并提供访问 Cortex\-M 内核功能的标准 C 接口。**
> 
> 比如 CMSIS\-Core 给你提供：
> 
> ```Plain Text
> NVIC_EnableIRQ(...);
> 
> NVIC_DisableIRQ(...);
> 
> NVIC_SetPriority(...);
> 
> SysTick_Config(...);
> ```
> 
> 以及各种寄存器结构体：
> 
> ```Plain Text
> NVIC->ISER
> SCB->VTOR
> SysTick->CTRL
> ```
> 
> 所以可以看到其实这和 ST 提高的差不多，都是为了 方便我们使用。
> 
> ---
> 
> 比如：
> 
> 你想打开某个中断，本质上就是：向某个特定地址写某个 bit，也就是：
> 
> ```Plain Text
> CPU
>  ↓
> 访问 NVIC 某个寄存器
>  ↓
> NVIC 硬件发生变化
> ```
> 
> 如果没有 CMSIS 时：
> 
> ```Plain Text
> #define NVIC_ISER0 (*(volatile unsigned int *)0xE000E100)
> 
> NVIC_ISER0 |= (1 << 5);
> ```
> 
> 这非常原始，于是 ARM 提供 CMSIS\-Core：
> 
> ```Plain Text
> NVIC_EnableIRQ(...);
> ```
> 
> 我们直接使用就可以，方便高效
> 
> 

## CMSIS\-Core 和 STM32F446xx\.h

> ```CMake
> STM32F446
>                      │
>           ┌──────────┴──────────┐
>           │                     │
>           ▼                     ▼
>       Cortex-M4             STM32外设
>           │                     │
>           │                     │
>     CMSIS-Core            STM32 Device Header
>           │                     │
>           ▼                     ▼
>     NVIC/SysTick/SCB       GPIO/RCC/USART/TIM
> ```
> 
> 

CMSIS\-Core 负责：

> ```Plain Text
> Cortex-M4 内核
> ```
> 
> 例如：
> 
> ```Plain Text
> NVIC
> SysTick
> SCB
> CPU寄存器
> 中断
> ```
> 
> 

---

STM32F446xx\.h 负责：

> ```Plain Text
> STM32F446 芯片特有的东西
> ```
> 
> 例如：
> 
> ```Plain Text
> GPIO
> RCC
> USART
> SPI
> TIM
> ADC
> DMA
> ```
> 
> 

有了 ARM 和 ST 给我们提供的这些官方文件

我们就可以专注于 控制板子上的器件 和 代码逻辑

## `platform/` 在整个代码依赖关系中的位置、责任

> 假设我们现在写一个 `main.c`：
> 
> ```C
> #include "stm32f446xx.h"
> 
> int main(void){
>     RCC->AHB1ENR |= ...;
> }
> ```
> 
> 如果让这个 `main.c` 变成 **STM32F446ZE** 能运行的固件，背后必须回答一大串问题：
> 
> ```Markdown
> 我是哪颗 MCU？
>         ↓
> STM32F446ZE
> -------------------------------------------
> 它是什么 CPU？
>         ↓
> Cortex-M4F
> -------------------------------------------
> 寄存器定义从哪来？
>         ↓
> stm32f446xx.h
> -------------------------------------------
> NVIC / SysTick / CPU 定义从哪来？
>         ↓
> CMSIS Core
> -------------------------------------------
> CPU 上电以后从哪开始执行？
>         ↓
> startup_stm32f446xx.s
> -------------------------------------------
> 谁初始化 .data 和 .bss？
>         ↓
> startup
> -------------------------------------------
> 谁提供 SystemInit()？
>         ↓
> system_stm32f4xx.c
> -------------------------------------------
> 用什么 CPU 指令集编译？
>         ↓
> -mcpu=cortex-m4 -mthumb ...
> -------------------------------------------
> Flash/RAM 在哪里？
>         ↓
> linker script
> ```
> 
> 而且这些问题 不是 “某个实验” 的研究内容 
> 
> 那么这些 **每个实验都必须回答，但又不是每个实验研究对象** 的东西都放在：`platform/`
> 
> 

---

**再回头看 ****`third_party`****，区别也会更清楚**

> `third_party/STM32CubeF4` 就像一个巨大的官方仓库：
> 
> ```Plain Text
> STM32CubeF4
> ├── F401
> ├── F405
> ├── F407
> ├── F411
> ├── F429
> ├── F446
> ├── HAL
> ├── CMSIS
> ├── BSP
> ├── startup xxxxx
> └── ...
> ```
> 
> 它只是 **原材料仓库，**它不会主动告诉我们的工程：
> 
> 

> ```Plain Text
> 你应该使用 STM32F446xx
> 你应该选 startup_stm32f446xx.s
> 你应该用 Cortex-M4F
> 你的 HSE 是多少
> 你的 linker 是哪个
> ```
> 
> 谁负责从这一大堆原材料中作出选择？就是：
> 
> ```Plain Text
> platform/stm32f446ze
> ```
> 
> 所以可以形成一个非常好记的关系：
> 
> ```Plain Text
> third_party
>     =
> 仓库里有什么原材料？
> 
> 
> platform
>     =
> 针对“我的硬件”，到底选哪些原材料，
> 并怎么组合？
> ```
> 
> 

# 下面把 `platform/stm32f446ze/` 变成最小可用平台

> 首先我们只接 4 个文件
> 
> ```C
> Cortex-M4 CMSIS Core 头文件
> STM32F446 Device 头文件
> system_stm32f4xx.c
> startup_stm32f446xx.s
> ```
> 
> 

*CMSIS Core 头文件* 和 *STM32F446 Device 头文件* 

我们在之前已经理解了。

现在主要理解 *启动文件*  和 *系统初始化文件* 

> 🟢 `startup_stm32f446xx.s`
> 
> 它会做的事情：
> 
> - **建立向量表**：把所有中断入口地址（Reset、NMI、HardFault、SysTick、外设中断等）放在一个表里，告诉 CPU 发生中断时跳到哪里执行。
> 
> - **设置堆栈指针**：在复位时初始化主堆栈指针 \(MSP\)，保证 C 语言环境能正常运行。
> 
> - **执行 Reset\_Handler**：这是 MCU 上电后第一段代码，它会：
> 
>     1. 调用 `SystemInit()`（在 system 文件里）做时钟初始化。
> 
>     2. 初始化数据段和 BSS 段（拷贝全局变量初值、清零未初始化变量）。
> 
>     3. 跳到 `main()`，开始执行用户代码。
> 
> 👉 没有它：MCU 上电后没有入口，程序根本跑不起来，中断也无法响应。
> 
> ---
> 
> 🟢 `system_stm32f4xx.c`
> 
> 它会做的事情：
> 
> - **SystemInit\(\)**：配置系统时钟源（比如用外部晶振 HSE，经 PLL 倍频得到 168MHz）。
> 
> - **更新 SystemCoreClock**：把当前 CPU 主频保存到一个全局变量，方便后续库函数和用户代码使用。
> 
> - **可选初始化**：可能还会启用 FPU、Cache 或其他系统级设置。
> 
> 👉 没有它：MCU会跑在默认的内部 HSI 8MHz 时钟，很多外设时钟不对，串口波特率、延时函数都会出错。
> 
> 

注意：**这些文件继续留在 *****third\_party/**** *

**platform** 的职责：

> *选择并使用哪些官方文件* 
> 
> 

---

**使用 CMake 将 platform 描述成 INTERFACE库，并给予它配置。让 platform 拥有回答那些问题能力** 

```CMake
# platform/stm32f446ze：NUCLEO-F446ZE 板级粘合

# ---------------------------------------------------------------------------
# Library
# 语法：add_Library(<name> [STATIC|SHARED|MODULE|INTERFACE])
# 创建一个叫 platform_stm32f446ze 的库，类型是 INTERFACE：
#                                          1.不会编译生成.a或.so文件。 
#                                          2.只是一个属性合集，包含include路径、宏定义、编译选项等。 
#                                          3.任何依赖它的target都会继承这些属性
#
# 语法：set(<variable> <value> ...)
# 用来定义变量，后面可以通过 ${<variable>} 引用该变量
# ---------------------------------------------------------------------------
add_library(platform_stm32f446ze INTERFACE)
...
```

---

结合之前的，我们就达到了：

```CMake
# 执行：cmake --preset stm32f446ze

CMakePresets.json
        ↓
toolchain
        ↓
根 CMakeLists.txt
        ↓
platform/stm32f446ze/CMakeLists.txt
        ↓
CMake 成功生成 Ninja 构建文件
```

但要注意：

> `Configuring done / Generating done` **≠ STM32 程序已经编译成功**。
> 
> 

因为目前还没有真正的固件 executable target，所以 CMake 只是把“规则”读懂了，还没让 GCC 编译、汇编和链接。

---

# Linker script 和 startup\-Reset\_Handler

把刚才已经读懂的 `STM32F446ZE.ld` 正式挂到 `platform_stm32f446ze` 上。

打开：

```Plain Text
platform/stm32f446ze/CMakeLists.txt
```

在前面已有内容基础上，增加 linker script 这一段：

```Plain Text
# ---------------------------------------------------------------------------
# Linker script
# ---------------------------------------------------------------------------

set(STM32F446ZE_LINKER_SCRIPT
    ${CMAKE_CURRENT_SOURCE_DIR}/linker/STM32F446ZE.ld
)

target_link_options(
    platform_stm32f446ze
    INTERFACE

    -T${STM32F446ZE_LINKER_SCRIPT}
)
```

如果把我们目前已经做的内容串起来，现在这个 target 的职责就变成：

```Plain Text
platform_stm32f446ze
│
├── STM32F446xx
│      → 告诉 CMSIS：具体是哪颗 MCU
│
├── CMSIS include 路径
│      → Cortex-M4 + STM32F446 寄存器定义
│
├── startup_stm32f446xx.s
│      → Reset_Handler / 向量表
│
├── system_stm32f4xx.c
│      → SystemInit()
│
├── Cortex-M4F 编译参数
│      → -mcpu=cortex-m4 ...
│
└── STM32F446ZE.ld
       → Flash / RAM / section 布局
```

也就是说我们学的 linker script 不再只是“放在目录里的一份文件”，而是正式成为：

> **所有依赖 ****`platform_stm32f446ze`**** 的最终固件，都必须使用的链接规则。**
> 
> 

这里的：

```Plain Text
-T${STM32F446ZE_LINKER_SCRIPT}
```

最终会变成 GCC 链接命令里的：

```Plain Text
-T/path/to/STM32F446ZE.ld    
```

`-T` 的意思就是：

> 使用这份 linker script 来进行链接。
> 
> 



然后再执行一次：

> ```Plain Text
> cmake --preset stm32f446ze
> ```
> 
> 现在大概率仍然会看到：
> 
> ```Plain Text
> -- Configuring done
> -- Generating done
> ```
> 
> 这是正常的，因为：
> 
> ```Plain Text
> -T STM32F446ZE.ld
> ```
> 
> 属于**链接阶段参数**。
> 
> 而我们现在还没有真正的 executable target，所以 linker 依旧还没真正运行。
> 
> 这一步的意义只是：
> 
> ```Plain Text
> platform_stm32f446ze
>         ↓
> 现在已经拥有完整 linker 规则
> ```
> 
> 



