---
tags: [P0, executable, ELF, gcc, objdump, nm, readelf, size]
stage: P0
---

# 07｜最小 Executable 与 ELF 体检

> [!tip] 深挖入口
> - [[10_基础知识体系/02_构建与链接/23_真实链接脚本逐段注解]]
> - [[10_基础知识体系/02_构建与链接/24_从症状到诊断_数字对不上时怎么办]]
> - [[10_基础知识体系/02_构建与链接/22_ELF Header Section Header Program Header]]
> - [[10_基础知识体系/02_构建与链接/19_ELF Section 与 Segment 的区别]]
> - [[10_基础知识体系/02_构建与链接/20_NOBITS NOLOAD 与为什么RAM占用不等于Flash占用]]
> - [[10_基础知识体系/02_构建与链接/17_ELF BIN HEX MAP 分别是什么]]
> - [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]
> - [[10_基础知识体系/02_构建与链接/07_Section 到底是什么]]
> - [[10_基础知识体系/03_Cortex-M与启动中断/18_读Startup所需的最小Thumb汇编]]
> - [[10_基础知识体系/04_验证与调试/02_size nm objdump readelf 各自回答什么]]
> - [[10_基础知识体系/04_验证与调试/07_怎样验证向量表]]

> [!abstract] 本节目标
> 学会把“构建成功”拆成可以检查的证据：**到底编进去了什么？被放到什么地址？关键 symbol 是什么？**
> 同时明确：ELF 静态证据不能代替运行时验证。

## 1. 为什么第一版 `main()` 故意什么也不做？

```c
int main(void)
{
    while (1) {
    }
}
```

P0 先单独验证地基：

```text
CMake + Toolchain + Platform + CMSIS + startup + system + linker script
                           ↓
                      STM32F446ZE ELF
```

如果第一步就加入 GPIO/UART/DMA，一旦失败，变量太多。

## 2. 构建时先看两层证据

### 第一层：详细编译命令

```bash
cmake --build build --target exp001_gpio_output -v
```

确认：

```text
arm-none-eabi-gcc
-DSTM32F446xx
-mcpu=cortex-m4
-mthumb
FPU / float ABI
CMSIS / Device include
```

### 第二层：详细链接命令

确认：

```text
startup object
system object
main object
-T xxx.ld
必要 library / link options
```

只有输入正确，分析最终 ELF 才有意义。

## 3. ELF 不是“更高级的 bin”

ELF 是有结构的容器，典型包含：

```text
ELF Header
Program Header Table
Section Header Table
.text
.rodata
.data
.bss
.isr_vector
.symtab / .strtab
.debug_*
...
```

`.bin` 更接近“准备写入地址空间的一串原始字节”；ELF 还保留地址、符号、section、调试等大量结构信息。

深入：
- [[10_基础知识体系/02_构建与链接/22_ELF Header Section Header Program Header]]

## 4. Section 与 Segment 不要混为一谈

链接/调试视角常看 section：

```text
.text
.data
.bss
.isr_vector
```

装载视角常看 program header 描述的 LOAD segment。

深入：
- [[10_基础知识体系/02_构建与链接/19_ELF Section 与 Segment 的区别]]

因此：

> 烧录器处理 ELF 时，不应简单理解成“把 ELF 文件从第 0 字节原封不动复制到 Flash”。

## 5. ELF、BIN、HEX 的角色

### ELF

包含丰富结构：

```text
机器码 / 数据 / section / symbol / 地址 / debug info（若有）
```

适合调试与分析。

### BIN

更接近一串原始 bytes，结构信息少。

### Intel HEX

文本形式保存地址、数据和校验信息。

自动生成 `.map/.bin/.hex` 已安排到 E1；P0 先理解它们为什么存在。

## 6. `arm-none-eabi-size`：先看规模

```bash
arm-none-eabi-size firmware.elf
```

粗略理解：

```text
text → 代码 + 只读内容为主
data → 有初值的可写静态数据
bss  → 零初始化/NOBITS 类 RAM 占用汇总
```

常用近似：

```text
Flash ≈ text + data
RAM static ≈ data + bss
```

这是“快速体检”，不是完整内存安全证明。

不要把列名机械等同于某一个 ELF section。

## 7. 当前一次真实 `size` 记录

之前实际得到：

```text
text    data    bss    dec    hex
820        0   1568   2388    954
```

### 7.1 为什么 `data = 0`？

同一次构建：

```text
_sdata = 0x20000000
_edata = 0x20000000
```

因此：

```text
_edata - _sdata = 0
```

说明当前最小程序没有需要 `.data` 运行副本的非零初值可写静态对象。

### 7.2 为什么 `bss = 1568`，但 `_ebss - _sbss` 只有 28 B？

真实 symbol：

```text
_sbss = 0x20000000
_ebss = 0x2000001c
```

差值：

```text
0x1c = 28 B
```

这说明 `size` 汇总的 bss 还包含其他 NOBITS / RAM 占用 section，例如 linker script 预留的 heap/stack 区域。

因此不能简单写：

```text
size.bss == _ebss - _sbss
```

要继续看 section table。

### 7.3 这 1540 字节到底是谁

不是"其他杂七杂八的东西"，是一个**具体的段**：

```ld
/* 在 platform/stm32f446ze/linker/STM32F446ZE.ld 里 */
._user_heap_stack :
{
  . = ALIGN(8);
  PROVIDE ( end = . );
  PROVIDE ( _end = . );
  . = . + _Min_Heap_Size;      /* 512 */
  . = . + _Min_Stack_Size;     /* 1024 */
  . = ALIGN(8);
} >RAM
```

它不含 "bss" 字样，却完全符合 `size` 对 `bss` 列的统计定义，所以被算进去了。

```text
28 (.bss) + 1540 (._user_heap_stack，含对齐补齐) = 1568
```

**完整诊断过程**（怎么自己查出来，而不只是背结论）：

- [[10_基础知识体系/02_构建与链接/24_从症状到诊断_数字对不上时怎么办]]
- [[10_基础知识体系/02_构建与链接/23_真实链接脚本逐段注解]]

深入：
- [[10_基础知识体系/02_构建与链接/20_NOBITS NOLOAD 与为什么RAM占用不等于Flash占用]]

## 8. `objdump -h`：看 Section Table

```bash
arm-none-eabi-objdump -h firmware.elf
```

重点：

```text
.isr_vector
.text
.rodata
.data
.bss
其他 NOBITS / heap / stack reserve
```

以及：

```text
Size / VMA / LMA / Alignment
```

其中 `.data` 最值得看：

```text
VMA → RAM
LMA → Flash
```

这与 Linker Script 中 `>RAM AT>ROM` 应互相对应。

问自己：

```text
它的 Size 是多少？
VMA 是多少？
LMA 是多少？
```

## 9. `nm -n`：看关键 Symbol

```bash
arm-none-eabi-nm -n firmware.elf
```

之前实际得到：

```text
08000250 T main
08000278 W Reset_Handler
08000334 A _sidata
20000000 D _edata
20000000 B _sbss
20000000 D _sdata
2000001c B _ebss
20020000 R _estack
```

这是 P0 最有价值的一组真实证据之一。

不要只看名字。至少同时读：

```text
symbol value/address
symbol type 字母
symbol name
```

典型字母：

```text
T/t → text/code
D/d → initialized data
B/b → bss
A   → absolute
W/w → weak
R/r → read-only data
```

注意：这些字母是 `nm` 的显示约定，具体语义仍应结合 ELF section/binding 一起确认。

深入：
- [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]

## 10. `main` 地址更小，为什么仍然后执行？

```text
main          0x08000250
Reset_Handler 0x08000278
```

地址排序不等于执行顺序。

复位时 CPU 从向量表取得 Reset_Handler；Reset_Handler 最后显式 branch/call 到 `main()`。

> **控制流决定执行顺序，不是 symbol 地址从小到大自动执行。**

## 11. `W Reset_Handler` 的 W 是什么？

`W` 常表示 weak symbol。

不是：

```text
Warning
Wrong
```

Weak symbol 允许后续同名强定义覆盖；Startup 中默认 Handler 广泛使用这种机制。

## 12. `_sidata = 0x08000334`

它表示 `.data` 初始镜像的加载地址。

这次虽然 `.data size = 0`，Linker 仍然可以定义 `_sidata`。加入真正的非零初值全局变量后，就可以观察 `.data` 是否出现。

## 13. `_estack = 0x20020000`

当前 SRAM：

```text
0x20000000 + 128 KiB
= 0x20000000 + 0x20000
= 0x20020000
```

因此 `_estack` 与 SRAM 顶部边界一致，是 Linker Script 和目标内存事实匹配的证据点。

## 14. 这组符号应该怎样连起来

```text
_sidata   0x08000334
_sdata    0x20000000
_edata    0x20000000
_sbss     0x20000000
_ebss     0x2000001c
_estack   0x20020000
```

你应能解释：

```text
_sidata 在 Flash
→ 因为它描述 .data 初始化镜像

_sdata/_edata 在 RAM
→ 因为 C 代码运行时在 RAM 访问这些变量

_sbss/_ebss 在 RAM
→ 因为 .bss 只有运行时实体

_estack 在 RAM 顶边界
→ 因为初始 MSP 要放在栈区高端
```

## 15. 直接看向量表内容

```bash
arm-none-eabi-objdump -s -j .isr_vector firmware.elf
```

可以进一步核对：

```text
第 0 项 → 初始 MSP
第 1 项 → Reset_Handler
```

具体 byte/word 解释还要考虑小端与 Cortex-M Thumb 地址语义。

## 16. `readelf`：看 ELF 原始结构

```bash
arm-none-eabi-readelf -h firmware.elf
arm-none-eabi-readelf -S firmware.elf
arm-none-eabi-readelf -l firmware.elf
arm-none-eabi-readelf -s firmware.elf
```

分别看：

```text
-h → ELF Header
-S → Section Header Table
-l → Program Header Table / Segment
-s → Symbol Table
```

命令速查见 [[90_知识卡片/ARM GNU 工具链命令]]。

## 17. `objdump -d`：看真正生成的机器代码

```bash
arm-none-eabi-objdump -d firmware.elf
```

可以找：

```text
<Reset_Handler>
<main>
```

目标不是现在读懂全部汇编，而是验证：

- `Reset_Handler` 是否真的存在代码。
- 是否出现向 `.data`/`.bss` 边界循环的指令。
- 是否调用 `SystemInit/main`。
- C 源码最终到底被翻译成了什么。

如果带调试信息，还可：

```bash
arm-none-eabi-objdump -S firmware.elf
```

混合源码与反汇编。

深入：
- [[10_基础知识体系/03_Cortex-M与启动中断/18_读Startup所需的最小Thumb汇编]]

## 18. ELF 能证明什么？

它能强力证明：

```text
构建系统最终生成了什么
symbol 最终地址是什么
section 怎样布局
机器代码是什么
向量表静态字节是什么
linker script 的很多结果是否生效
```

## 19. ELF 不能证明什么？

它不能单独证明：

```text
板子已经正确烧录
CPU 已执行 Reset_Handler
CPU 真从向量表启动了
.data 已真的复制到 RAM
.bss 已真的清零
main 已运行
GPIO 已改变物理电平
```

这些属于运行时事实，需要 Debugger/硬件证据。

## 20. 证据层级

```text
源码
  ↓
CMake target
  ↓
详细 compile/link command
  ↓
ELF section/symbol/disassembly
  ↓
烧录
  ↓
Debugger PC/register/memory
  ↓
真实引脚/仪器
```

越往下越接近物理现实。

## 21. 裸机与 C library

如果链接出现：

```text
_write / _read / _sbrk / _exit
```

等 undefined reference，要先判断这是否来自 C library 对系统服务的需求。

裸机没有 Linux syscall，因此可能需要 Newlib Nano / nosys / 自己的 retarget，具体取决于实际使用的库功能。

UART printf 会留到后续 UART 实验再系统引入。

## 22. 为什么 map 文件值得以后加？

Map 特别适合回答：

```text
某 symbol 来自哪个 .o？
某 output section 收了哪些 input section？
哪个 library object 被拉进来？
内存具体怎样被占用？
```

所以自动生成 `.map` 已安排到 E1，而不是回头扩建 P0。

## 23. P0 ELF 体检清单

```text
[ ] 目标架构为 ARM
[ ] .isr_vector 存在
[ ] .isr_vector 位于预期 Flash 区域
[ ] Reset_Handler 存在
[ ] main 存在
[ ] _estack 对应 RAM top
[ ] .data VMA/LMA 关系合理
[ ] .bss 位于 RAM
[ ] 反汇编能找到 Reset_Handler → main 的控制流
```

## 24. 离开本页前

1. ELF 比 BIN 多了哪些信息？
2. `size/nm/objdump/readelf` 各自最适合回答什么？
3. 当前为什么 `data=0`？
4. 为什么 `_ebss-_sbss=28`，`size` 却显示 `bss=1568`？
5. `W Reset_Handler` 表示什么？
6. `main` 地址更小为什么不代表先执行？
7. Section 和 Segment 是同一个概念吗？
8. 怎样从 ELF 静态证据推导 `.data` 的搬运关系？
9. 什么证据才能进一步说明 CPU 真进入了 `main()`？

下一步：[[08_P0 完整启动链]]，随后做 [[09_动手追踪一次启动]]。
