---
tags: [P0, startup, reset-handler, cortex-m]
stage: P0
---

# 06｜Startup 与 Reset_Handler

> [!important] 这一课的底层入口
> 你不需要在本页把所有词一次吃透。任何一个卡住就点进去：
>
> - [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/05_Cortex-M Reset 到底发生了什么]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/02_SP MSP PSP 是什么]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/06_Startup 文件到底是什么]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/07_Reset_Handler 逐段看]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/08_data copy 与 bss clear 为什么由 Startup 做]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/09_SystemInit 到底处在什么位置]]
>
> 总图：[[10_基础知识体系/05_图解总览/06_从上电到 main 时间线]]

> [!abstract] 本节目标
> 回答：**MCU 刚复位、还没有进入 `main()` 时，CPU 怎样一步一步建立 C 程序需要的运行环境？**

## 1. 最重要的纠正：Reset_Handler 不是第一件事

Cortex-M 复位时，硬件先从向量表读取：

```text
vector[0] → initial MSP
vector[1] → Reset vector / Reset_Handler
```

所以：**CPU 在进入 `Reset_Handler` 前已经获得初始 MSP。**

Startup 中即使再次出现 `ldr sp, =_estack`，也是显式重装 SP，不能理解为此前完全没有栈。

## 2. 向量表先怎么理解？

概念：

```text
0x08000000  initial MSP
0x08000004  Reset_Handler
0x08000008  NMI_Handler
0x0800000C  HardFault_Handler
...
```

第 0 项不是函数地址，而是初始 MSP；第 1 项才是复位入口。

## 3. Startup 提供什么？

当前 ST startup 通常包含：

```text
Vector Table
Reset_Handler
默认异常/中断 Handler
weak symbol
进入 C runtime 前的低级初始化
```

它不是普通工具库，而是“CPU 如何进入你的 C 程序”的核心契约。

## 4. Reset_Handler 主线

按当前课程所用 ST startup 概括：

```text
Reset
  ↓
硬件加载 MSP
  ↓
硬件取得 Reset_Handler
  ↓
Reset_Handler
  ↓
可能显式重装 SP = _estack
  ↓
SystemInit()
  ↓
复制 .data：Flash → RAM
  ↓
清零 .bss
  ↓
__libc_init_array()
  ↓
main()
```

实际工程升级第三方代码后，应重新核对真实 startup 文件。

## 5. `SystemInit()` 的时机很重要

当前顺序中它位于 `.data/.bss` 初始化之前。

因此如果以后自己修改 `SystemInit()`：

> 不要随意依赖普通全局变量已经完成 C runtime 初始化。

`SystemInit()` 的具体行为必须看当前 `system_stm32f4xx.c`；不能仅凭函数名假设它把系统配置到 168/180 MHz。

## 6. `.data` 复制循环在做什么？

典型先取得：

```asm
ldr r0, =_sdata
ldr r1, =_edata
ldr r2, =_sidata
```

概念：

```text
r0 = RAM destination start
r1 = RAM destination end
r2 = Flash source
```

然后循环把 Flash 初始镜像复制到 RAM，直到目标地址达到 `_edata`。

所以通常处理半开区间：

```text
[_sdata, _edata)
```

## 7. 当前最小程序为什么 `.data` 不需要真正复制？

之前真实 symbol：

```text
_sdata = 0x20000000
_edata = 0x20000000
```

所以：

```text
.data length = 0
```

这说明当前最小 `main()` 没有需要非零初值的可写静态对象。Startup 复制逻辑仍在，只是这次长度为 0。

这也是 [[09_动手追踪一次启动]] 要人为加入：

```c
volatile uint32_t counter = 10;
```

的原因。

## 8. `.bss` 为什么必须清零？

例如：

```c
uint32_t flag;
```

C 语言要求静态存储期未显式初始化对象以 0 开始，但 RAM 上电值不能靠猜。

所以 Startup 对：

```text
[_sbss, _ebss)
```

写 0。

之前真实：

```text
_sbss = 0x20000000
_ebss = 0x2000001c
```

代表这组 symbol 标识的区间为 28 B；`size` 的 bss 汇总为什么更大，见 [[05_Linker Script 与内存布局]]。

## 9. `__libc_init_array()`

它处理 C/C++ runtime 的初始化数组，C++ 全局构造函数是最典型例子。

当前纯 C 最小实验不要求深入 Newlib 实现，但要记住：

> “现在 main 还能跑”不足以证明随意删除 runtime 初始化是安全的。

## 10. 然后才进入 `main()`

到这里通常已经满足：

```text
Stack 已建立
.data 初值正确
.bss 已清零
必要 runtime 初始化完成
```

随后 branch/call 到 `main()`。

嵌入式 `main()` 通常不返回，因此最小程序写无限循环。

## 11. Weak Handler

Startup 中很多默认 Handler 是 weak，这样后续用户可提供同名强定义覆盖。

所以 `nm` 中：

```text
W Reset_Handler
```

的 `W` 表示 weak symbol，不是 warning 或错误。

## 12. ELF ENTRY 与硬件 Reset 不要混淆

`ENTRY(Reset_Handler)` 属于 linker/ELF 层；真实 Cortex-M reset 读取的是向量表前两项。

因此验证启动链时，应检查：

```text
.isr_vector 的位置和内容
Reset_Handler symbol
最终反汇编控制流
```

而不是只看 ELF entry。

## 13. 怎样验证真的进入 `main()`？

ELF 中存在 `main` 和 `Reset_Handler` 只能证明它们被链接。

更强验证是烧录后：

```text
reset
break main
continue
```

观察：

```text
PC 是否到 main
SP 是否在 SRAM 合理范围
全局变量是否为预期初值
```

## 14. 故障注入思路

教学实验可故意：

- 跳过 `.data` copy → 非零初值静态变量不再有保证。
- 跳过 `.bss` clear → 未初始化静态对象不再保证为 0。
- 破坏向量表位置 → CPU 可能在进入 Reset_Handler 前就失败。

故障注入的价值是验证因果链，不是为了“把工程搞坏”。

## 15. 离开本页前

闭卷讲清：

```text
Reset → vector[0] → MSP → vector[1] → Reset_Handler
→ SystemInit → .data copy → .bss clear → runtime init → main
```

并回答：

1. CPU 是先得到 MSP 还是先执行 Reset_Handler？
2. Linker 与 Startup 在 `.data` 初始化中分别负责什么？
3. `_sdata == _edata` 表示什么？
4. 为什么 `.bss` 必须由运行时代码清零？
5. 为什么 ELF 正确仍不能证明 Startup 真执行成功？

下一步：[[07_最小 Executable 与 ELF 体检]]。
