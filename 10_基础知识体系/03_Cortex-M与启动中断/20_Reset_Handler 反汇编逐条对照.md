---
tags: [startup, reset-handler, disassembly, objdump, cortex-m]
stage: P0
---

# Reset_Handler 反汇编逐条对照

> [!abstract] 本页解决什么
> [[10_基础知识体系/03_Cortex-M与启动中断/07_Reset_Handler 逐段看]] 告诉你 `Reset_Handler`**为什么**有那几步。
> [[10_基础知识体系/03_Cortex-M与启动中断/18_读Startup所需的最小Thumb汇编]] 给你读它的**工具**（`ldr`/`str`/`b`/`bl`）。
> [[01_P0_仓库与构建基线/09_动手追踪一次启动]] 给你**实验流程**。
>
> 本页补中间缺的那一环：**真的把 `Reset_Handler` 反汇编打出来，一行一行读完。**
> 09 页说"在 copy loop 前后设置断点"——本页告诉你**那个 loop 长什么样、怎么找到它**。

> [!warning] 本页所有数字来自一次真实构建
> 工程：`embedded_lab`，实验 `001_gpio_output`。
> 换一个实验、改一次源码、升级一次工具链，**地址和大小都会变**。
> 要照着本页验证时，重新跑一遍第 0 节的命令，用你自己的输出对照。

## 0. 先拿到输出

```bash
cd embedded_lab
arm-none-eabi-objdump -d build/experiments/001_gpio_output/exp001_gpio_output.elf \
  | sed -n '/<Reset_Handler>:/,/<ADC_IRQHandler>:/p'
```

本页所有内容都来自这一条命令的真实输出。

---

## 1. 完整反汇编

```asm
08000278 <Reset_Handler>:
 8000278:  f8df d034   ldr.w   sp, [pc, #52]      @ 80002b0
 800027c:  f7ff ffec   bl      8000258 <SystemInit>
 8000280:  480c        ldr     r0, [pc, #48]      @ 80002b4
 8000282:  490d        ldr     r1, [pc, #52]      @ 80002b8
 8000284:  4a0d        ldr     r2, [pc, #52]      @ 80002bc
 8000286:  2300        movs    r3, #0
 8000288:  e002        b.n     8000290 <LoopCopyDataInit>

0800028a <CopyDataInit>:
 800028a:  58d4        ldr     r4, [r2, r3]
 800028c:  50c4        str     r4, [r0, r3]
 800028e:  3304        adds    r3, #4

08000290 <LoopCopyDataInit>:
 8000290:  18c4        adds    r4, r0, r3
 8000292:  428c        cmp     r4, r1
 8000294:  d3f9        bcc.n   800028a <CopyDataInit>
 8000296:  4a0a        ldr     r2, [pc, #40]      @ 80002c0
 8000298:  4c0a        ldr     r4, [pc, #40]      @ 80002c4
 800029a:  2300        movs    r3, #0
 800029c:  e001        b.n     80002a2 <LoopFillZerobss>

0800029e <FillZerobss>:
 800029e:  6013        str     r3, [r2, #0]
 80002a0:  3204        adds    r2, #4

080002a2 <LoopFillZerobss>:
 80002a2:  42a2        cmp     r2, r4
 80002a4:  d3fb        bcc.n   800029e <FillZerobss>
 80002a6:  f000 f811   bl      80002cc <__libc_init_array>
 80002aa:  f7ff ffd1   bl      8000250 <main>
 80002ae:  4770        bx      lr

 80002b0:  20020000    .word   0x20020000
 80002b4:  20000000    .word   0x20000000
 80002b8:  20000000    .word   0x20000000
 80002bc:  08000334    .word   0x08000334
 80002c0:  20000000    .word   0x20000000
 80002c4:  2000001c    .word   0x2000001c
```

**总共 27 条指令 + 6 个常量。**

> [!tip] 先看结构，别看细节
> ```text
> 8000278  设栈指针
> 800027c  调 SystemInit
> 8000280-8000294  .data 复制循环
> 8000296-80002a4  .bss 清零循环
> 80002a6  调 __libc_init_array
> 80002aa  调 main
> 80002ae  main 的返回出口（实际到不了）
> 80002b0-80002c4  常量池
> ```
> 骨架就这七块。接下来逐块看。
>
> [!warning] 有一个地方名字会骗你
> 第 3 节的 `SystemInit` **不是**配时钟的——它在开 FPU。
> 读完那节你会明白为什么"看名字猜行为"在这条链上一定会出错。

---

## 2. 第一块：重建栈指针（`8000278`）

```asm
8000278:  f8df d034   ldr.w   sp, [pc, #52]      @ 80002b0
```

> [!important] 先分清"事实"和"动机"
> 很多人以为复位后 CPU 已经把 MSP 设好了，`Reset_Handler` 里**不该再设一次**。
> 实际上它会再设一次。但是**为什么**要这样写，产物里没有答案。
>
> 先把能查的查清楚：
>
> | 说法 | 来源 | 能不能验证 |
> |---|---|---|
> | 这里有 `ldr.w sp` 这条指令 | `objdump -d` | ✅ 你的产物 |
> | 它读的常量是 `0x20020000` | `objdump` 注释 + `nm` 对照 = `_estack` | ✅ 你的产物 |
> | 复位后 MSP 已经等于这个值 | ARMv7-M 架构规定 | ⚠️ 一手资料，不在你 ELF 里 |
> | ST 为什么写这一句 | —— | ❌ **未知** |
>
> **最后一行是动机，不该替它编。**
>
> 能说的只是这条指令带来的**效果**（这是推论，不是 ST 的说法）：
> ```text
> Reset_Handler 自己建立 MSP
>   → 不依赖"硬件已经用过 vector[0]"这个前提
>   → 于是即使有人直接跳到 Reset_Handler，栈指针也是对的
> ```
>
> 这个效果**只在非复位入口才有意义**。
> 在真正的上电/复位路径上，`Reset_Handler` 之前没有任何代码运行，
> SP 不可能被谁"弄脏"——所以在这一路径上，这次加载写回的是同一个值。

### 这条指令怎么读

```text
ldr.w sp, [pc, #52]
   │     │   └─ 相对 PC 偏移 52 字节
   │     └─ 目标寄存器：sp
   └─ .w = 强制使用 32 位编码（普通 ldr 只有 16 位）
```

**计算地址时有个 Thumb 特性必须记住**：

> [!warning] PC 在读的时候 = 当前指令地址 + 4
> 这是 Cortex-M 流水线的历史遗留，不是笔误。
>
> ```text
> 当前指令地址        0x08000278
> 读出时 PC 的值      0x08000278 + 4 = 0x0800027C
> 加偏移 52 (0x34)    0x0800027C + 0x34 = 0x080002B0   ✓
> ```
>
> 所以反汇编注释里的 `@ 80002b0` 就是在告诉你**它实际读的是哪**。
> **看反汇编时优先信注释里的绝对地址**，不要自己心算（除非你在练习）。

### 常量池里是什么

```asm
80002b0:  20020000    .word   0x20020000
```

这就是 `_estack`。

用 `nm` 验证：

```bash
arm-none-eabi-nm build/.../exp001_gpio_output.elf | grep _estack
# 20020000 R _estack
```

**对上了。**

---

## 3. 第二块：`SystemInit`（`800027c`）

```asm
800027c:  f7ff ffec   bl      8000258 <SystemInit>
```

### `SystemInit` 里到底是什么

别猜，直接反汇编它：

```bash
arm-none-eabi-objdump -d build/.../exp001_gpio_output.elf \
  | sed -n '/<SystemInit>:/,/<Reset_Handler>:/p'
```

```asm
08000258 <SystemInit>:
 8000258:  b480       push  {r7}
 800025a:  af00       add   r7, sp, #0
 800025c:  4b05       ldr   r3, [pc, #20]   @ 8000274  ← 0xE000ED00
 800025e:  f8d3 3088  ldr.w r3, [r3, #136]  @ 0x88
 8000262:  4a04       ldr   r2, [pc, #16]   @ 8000274  ← 0xE000ED00
 8000264:  f443 0370  orr.w r3, r3, #15728640  @ 0xF00000
 8000268:  f8c2 3088  str.w r3, [r2, #136]  @ 0x88
 800026c:  bf00       nop
 800026e:  46bd       mov   sp, r7
 8000270:  bc80       pop   {r7}
 8000272:  4770       bx    lr

 8000274:  e000ed00   .word 0xE000ED00
```

**它不是空函数。** 把地址和偏移翻译一下：

```text
0xE000ED00  =  SCB 基地址（System Control Block）
      + 0x88 =  SCB->CPACR      （协处理器访问控制寄存器）
      ────────────────────────
0xE000ED88  =  CPACR 的绝对地址

orr #0xF00000  =  把 bit20-23 置 1
                  CP10 = 0b11
                  CP11 = 0b11
```

> [!important] 这三条指令在做的事：**打开 FPU**
> Cortex-M4F 的浮点单元默认是**关着**的。
> 不把 CPACR 的 CP10/CP11 置成全权限，任何浮点指令都会触发 **UsageFault**。
>
> ```text
> SystemInit  →  写 SCB->CPACR  →  FPU 可用
> ```
> 这是 `Reset_Handler` 里**唯一一处碰硬件的操作**，而且它必须在任何浮点代码之前完成。

> [!tip] 这正是"不要猜 `SystemInit` 做什么"的活教材
> 名字叫 `SystemInit`，很多教程会告诉你"它负责配时钟"。
> **但你的工程里它一行时钟代码都没有**——它只开了 FPU。
>
> 时钟那块，`platform/stm32f446ze/README.md` 第 12 行写着：
> ```text
> | clock | ST-LINK MCO 8 MHz → **HSE bypass** → PLL → SYSCLK 180 MHz（板载无独立 HSE 晶振）
>         | ⬜ 暂用 system_stm32f4xx.c 默认 HSI 16 MHz |
> ```
> 那个 ⬜ 表示**还没做**——所以现在跑的是默认 HSI 16 MHz，没有配 PLL。
>
> 结论只能从**当前链接进来的那一份**得出：
> ```text
> 名字         →  SystemInit
> 实际行为     →  开 FPU，不配时钟
> ```
> 名字和行为**不一致**，这在真实工程里是常态。

### `bl` 留下了什么

```text
bl SystemInit
    ↓
LR ← 0x08000280        （返回地址 = 下一条指令）
PC ← 0x08000258        （跳过去）
```

所以 `SystemInit` 执行完能回到 `0x08000280`。

这就是 [[10_基础知识体系/03_Cortex-M与启动中断/19_C与汇编如何互相调用_AAPCS最小理解]] 讲的那件事——**汇编之所以能调用一个 C 函数，是因为双方遵守同一套 ABI**，而不是因为"名字对上了"。

---

## 4. 第三块：`.data` 复制循环（`8000280`-`8000294`）

### 准备阶段

```asm
8000280:  480c   ldr  r0, [pc, #48]   @ 80002b4   → r0 = 0x20000000
8000282:  490d   ldr  r1, [pc, #52]   @ 80002b8   → r1 = 0x20000000
8000284:  4a0d   ldr  r2, [pc, #52]   @ 80002bc   → r2 = 0x08000334
8000286:  2300   movs r3, #0                        → r3 = 0
```

对照 `nm` 输出，把三个寄存器的角色填上：

| 寄存器 | 值 | 符号 | 角色 |
|---|---|---|---|
| `r0` | `0x20000000` | `_sdata` | 目标起点（RAM） |
| `r1` | `0x20000000` | `_edata` | 目标终点（RAM） |
| `r2` | `0x08000334` | `_sidata` | 源起点（Flash） |
| `r3` | `0` | — | 偏移量 |

```bash
arm-none-eabi-nm build/.../exp001_gpio_output.elf | grep -E "_sidata|_sdata|_edata"
# 08000334 A _sidata
# 20000000 D _sdata
# 20000000 D _edata
```

**三个常量全对上了。**

### 循环体

```asm
0800028a <CopyDataInit>:
800028a:  58d4   ldr  r4, [r2, r3]   ← 从 Flash 读一个字到 r4
800028c:  50c4   str  r4, [r0, r3]   ← 写到 RAM
800028e:  3304   adds r3, #4         ← 偏移 +4（一个字）

08000290 <LoopCopyDataInit>:
8000290:  18c4   adds r4, r0, r3     ← r4 = 当前目标地址
8000292:  428c   cmp  r4, r1         ← 和 _edata 比
8000294:  d3f9   bcc.n 800028a       ← 如果 r4 < r1，回去继续
```

**这就是那个"copy loop"。** 09 页让你找的东西，就是 `800028a`。

> [!tip] 循环的四个角色一眼认出
> ```text
> ldr  ← 源读取
> str  ← 目标写入
> adds ← 地址推进
> cmp + bcc ← 边界判断
> ```
> 这是**所有**内存复制循环的通用骨架。以后看 `memcpy` 实现、DMA 描述符填充、帧缓冲搬运，都是这个形状。

### ⚠️ 关键观察：**这个循环在你的工程里一次都没执行**

```text
r0 = _sdata = 0x20000000
r1 = _edata = 0x20000000
```

第一次进入 `LoopCopyDataInit`（`8000290`）：

```text
r4 = r0 + r3 = 0x20000000 + 0 = 0x20000000
cmp r4, r1   →  0x20000000 vs 0x20000000
bcc          →  不成立（bcc 是"低于"，相等不算低于）
```

**直接掉出去了。**

这完全对应 `size` 里的 `data = 0`：

```text
没有"带初值的可写全局变量"
    ↓
.data 段长度为 0
    ↓
_sdata == _edata
    ↓
循环条件第一次就不成立
```

> [!success] 这是一个真正的"验证闭环"
> ```text
> size 说 data = 0             ← 静态证据
> nm 说 _sdata == _edata       ← 静态证据
> 反汇编说循环第一次就跳出      ← 静态证据
> 三者互相印证
> ```
> 但**它们都还没证明"CPU 真的执行过这段代码"**。
> 要证明那个，你得在 `8000290` 下断点，看 `r4` 和 `r1` 的实际值。这就是 09 页要做的。

---

## 5. 第四块：`.bss` 清零循环（`8000296`-`80002a4`）

```asm
8000296:  4a0a   ldr  r2, [pc, #40]   @ 80002c0   → r2 = 0x20000000
8000298:  4c0a   ldr  r4, [pc, #40]   @ 80002c4   → r4 = 0x2000001c
800029a:  2300   movs r3, #0                        → r3 = 0

0800029e <FillZerobss>:
800029e:  6013   str  r3, [r2, #0]   ← 把 0 写进去
80002a0:  3204   adds r2, #4         ← 指针 +4

080002a2 <LoopFillZerobss>:
80002a2:  42a2   cmp  r2, r4         ← 到 _ebss 了吗
80002a4:  d3fb   bcc.n 800029e       ← 没到就继续
```

| 寄存器 | 值 | 符号 | 角色 |
|---|---|---|---|
| `r2` | `0x20000000` | `_sbss` | 起点 |
| `r4` | `0x2000001c` | `_ebss` | 终点 |
| `r3` | `0` | — | 要写的值 |

### 和 copy 循环的三处差别

```text
① 没有源地址——源永远是常量 0
② r3 是"要写的值"（0），不是偏移量
③ 指针 r2 自己往前挪（adds r2, #4），不用基址+偏移
```

### 这次循环**会**执行

```text
0x20000000 → 0x2000001c
(0x1c - 0x00) / 4 = 7 次
```

7 次 × 4 字节 = **28 字节**。

对照 `objdump -h`：

```text
9 .bss   0000001c  20000000  ...   ALLOC
         └─ 0x1c = 28
```

**对上了。**

> [!warning] 注意循环条件是 `<` 不是 `<=`
> ```asm
> cmp r2, r4
> bcc ...        ← bcc = "低于"（unsigned <）
> ```
> `_ebss` 是"结束后的**第一个**地址"，**不属于 `.bss`**。
> 所以必须用 `<`，用 `<=` 会多写 4 字节，**踩到 `._user_heap_stack` 的头部**。
>
> 这类差一错误在链接脚本边界符号上是设计好的——**边界符号总是指向"范围外第一个字节"**，跟 C++ 的 `end()` 迭代器同一个约定。

---

## 6. 第五块：`__libc_init_array`（`80002a6`）

```asm
80002a6:  f000 f811   bl   80002cc <__libc_init_array>
```

这一条直接把 `06_Startup` 第 8 节讲的机制摆在你面前：

```text
bl __libc_init_array     ← 汇编调用一个 C 函数 ✓
```

去看看它内部（`80002cc`）：

```asm
080002cc <__libc_init_array>:
 8002cc:  b570       push  {r4, r5, r6, lr}
 8002ce:  4b0d       ldr   r3, [pc, #52]   @ 8000304  ← 0x0800032c
 8002d0:  4c0d       ldr   r4, [pc, #52]   @ 8000308  ← 0x0800032c
 8002d2:  2600       movs  r6, #0
 8002d4:  1b1d       subs  r5, r3, r4
 8002d6:  ebb6 0fa5  cmp.w r6, r5, asr #2
 8002da:  d109       bne.n 80002f0
 8002dc:  f000 f81a  bl    8000314 <_init>
 8002e0:  4c0a       ldr   r4, [pc, #40]   @ 800030c  ← 0x0800032c
 8002e2:  4b0b       ldr   r3, [pc, #44]   @ 8000310  ← 0x08000330
 8002e4:  2600       movs  r6, #0
 8002e6:  1b1d       subs  r5, r3, r4
 8002e8:  ebb6 0fa5  cmp.w r6, r5, asr #2
 8002ec:  d105       bne.n 80002fa
 8002ee:  bd70       pop   {r4, r5, r6, pc}

 80002f0:  f854 3b04  ldr.w r3, [r4], #4     ← 取下个指针，同时 r4 += 4
 80002f4:  4798       blx   r3               ← 间接调用
 80002f6:  3601       adds  r6, #1
 80002f8:  e7ed       b.n   80002d6

 80002fa:  f854 3b04  ldr.w r3, [r4], #4
 80002fe:  4798       blx   r3
 8000300:  3601       adds  r6, #1
 8000302:  e7f1       b.n   80002e8

 8000304:  0800032c   .word 0x0800032c
 8000308:  0800032c   .word 0x0800032c
 800030c:  0800032c   .word 0x0800032c
 8000310:  08000330   .word 0x08000330
```

> [!important] 注意：**是两个循环**，不是一个
> ```text
> 8002ce-80002da   遍历 .preinit_array
> 80002dc          调 _init
> 8002e0-80002ec   遍历 .init_array
> ```
> 两段的常量不同，正好说明各自的表有多长：
> ```text
> 前半段  0x0800032c / 0x0800032c   差 0 字节  →  0 个元素
> 后半段  0x0800032c / 0x08000330   差 4 字节  →  1 个元素
> ```
> 这也解释了 `objdump -h` 里为什么会有个 `.preinit_array`。

### 循环体更直白

```asm
ldr.w r3, [r4], #4    ← 后索引：读出指针，同时 r4 自动 +4
blx   r3              ← 跳到 r3 指向的地址（间接调用）
adds  r6, #1          ← 计数
```

> [!success] 这三条指令就是"`bl` 只能调代码"的证据
> | 指令 | 目标地址从哪来 |
> |---|---|
> | `bl 8000314 <_init>` | **写死在指令里**，编译期就确定 |
> | `blx r3` | **从内存读出来**，运行期才知道 |
>
> `.init_array` 是**数据**——所以只能先把指针 `ldr` 进寄存器，再 `blx` 跳过去。
> 如果它是代码，这里就该一条 `bl` 直接调过去，根本不需要这个循环。
>
> 这就是 `06_Startup` 第 8 节说的：
> > `.init_array` 是**数据**（函数指针表），`__libc_init_array` 是**代码**（遍历它的 C 函数）。

### 你的工程里它做了多少事

```bash
arm-none-eabi-size -A build/.../exp001_gpio_output.elf | grep -E "init_array|preinit"
```

```text
.preinit_array         0   134218540     ← 空
.init_array            4   134218540     ← 1 个指针
```

对照上面那四个常量，结论完全一致：

```text
preinit  差 0 字节  →  0 个元素
init     差 4 字节  →  1 个元素
```

**两个证据互相印证。**

### 那唯一的 1 个指针，到底指向谁

三层命令，一层层看下去。

**第一步：读表里的值**

```bash
arm-none-eabi-objdump -s -j .init_array build/.../exp001_gpio_output.elf
```

```text
Contents of section .init_array:
 800032c 2d020008                             -...
```

小端解释 `2d 02 00 08` → **`0x0800022D`**。

> [!tip] 又是奇数
> `0x0800022D` 最低位是 1 —— 函数指针的 Thumb bit。
> 这正是 [[10_基础知识体系/03_Cortex-M与启动中断/15_Thumb状态与函数地址最低位]] 说的规则，
> 在 `.init_array` 里**又出现了一次**。
>
> 所以 `blx r3` 跳过去时，硬件会先把最低位摘掉，实际跳到 `0x0800022C`。

**第二步：`nm` 看表项的符号名**

```bash
arm-none-eabi-nm -n build/.../exp001_gpio_output.elf | awk '$1>="08000300" && $1<="08000340"'
```

```text
0800032c r __frame_dummy_init_array_entry
0800032c r __init_array_start
0800032c d __preinit_array_end
0800032c d __preinit_array_start
08000330 r __init_array_end
```

注意 `__init_array_start` 和 `__preinit_array_end` **同一个地址** —— 因为前一个表是空的。

**第三步：看那个函数是什么**

```text
0x0800022D  →  摘掉 Thumb bit  →  0x0800022C  →  frame_dummy
```

```asm
0800022c <frame_dummy>:
 800022c:  b508       push   {r3, lr}
 800022e:  4b05       ldr    r3, [pc, #20]   @ 8000244  ← 0x00000000
 8000230:  b11b       cbz    r3, 800023a     ← r3 == 0，跳过
 8000232:  ...        （这段被跳过了）
 800023a:  e8bd 4008  ldmia.w sp!, {r3, lr}
 800023e:  f7ff bfcf  b.w    80001e0 <register_tm_clones>

 8000244:  00000000   .word 0x00000000
```

`frame_dummy` 是 GCC 运行时（`crtbegin.o`）提供的函数，正常用来注册异常栈帧信息。

**但这里它什么也没做**：`r3` 从常量池读到 `0x00000000`，`cbz r3` 判断为零就直接跳过整段调用。

```text
frame_dummy  →  r3 == 0  →  跳过注册  →  尾调用 register_tm_clones
```

> [!success] 一个完整的"从表到代码"的追踪
> ```text
> .init_array 里 4 个字节
>   → 0x0800022D（带 Thumb bit）
>   → 0x0800022C（frame_dummy）
>   → 读到常量 0
>   → 直接跳过
> ```
> **整条链上没有任何"显然是空的"标记。** 你得一路查下去才知道它是空转。
>
> 这就是为什么 `__libc_init_array` 不能凭"我用不到 C++"就删掉——
> 它是 runtime 契约的一部分，删了以后你的构造函数、`__attribute__((constructor))` 会静默失效。

你没写任何 C++ 全局对象，也没用 `__attribute__((constructor))`，所以表里只有 GCC 自带的这一项。

**机制是完整的**，等你以后写了构造函数，不用改任何链接脚本它就能工作。

---

## 7. 第六块：`main`（`80002aa`）

```asm
80002aa:  f7ff ffd1   bl   8000250 <main>
```

### 一个反直觉事实

```text
main           0x08000250
Reset_Handler  0x08000278
```

**`main` 的地址比 `Reset_Handler` 小，但它后执行。**

为什么 `main` 反而排在前面？因为同一段的输入顺序由**链接命令行里 `.o` 的顺序**决定。你的 `main.o` 在 `startup_stm32f446xx.o` 之前被传给链接器，所以 `main` 的函数落在前面。

**这纯粹是文件顺序的副产品，没有任何语义。** 执行顺序由控制流决定——`Reset_Handler` 里那条 `bl main`。

这条正是 [[01_P0_仓库与构建基线/08_P0 完整启动链]] 第 5 节强调的反直觉点，现在有了真实地址做证据。

### 然后呢？——`bx lr` 和那条"到不了的路"

```asm
80002ae:  4770   bx lr
```

`main` 如果真的**返回**了，就会执行这条 `bx lr`，回到 `LR` 保存的地址。

但实际上：

```c
int main(void) {
    while (1)
    {
    }
}
```

**`main` 永远不会返回。** 所以 `80002ae` 这条 `bx lr` 是**死代码**。

> [!warning] 这里藏着一个坑
> 如果你的 `main` 某天真的返回了，`bx lr` 会跳到一个**没有意义**的地方——
> 因为这是在裸机上，"调用者"是 startup 汇编，返回地址 `LR` 指向 `80002ae` 之后，而那里没有代码。
>
> 结果通常是跑飞、HardFault，或进入某个未知状态。
>
> **裸机 `main` 必须有 `while(1)` 兜底**，这不是风格问题，是正确性问题。

---

## 8. 第七块：常量池（`80002b0`-`80002c4`）

```asm
80002b0:  20020000   .word  0x20020000    ← _estack
80002b4:  20000000   .word  0x20000000    ← _sdata
80002b8:  20000000   .word  0x20000000    ← _edata
80002bc:  08000334   .word  0x08000334    ← _sidata
80002c0:  20000000   .word  0x20000000    ← _sbss
80002c4:  2000001c   .word  0x2000001c    ← _ebss
```

> [!important] 这六个值就是 05 页和 06 页说的"接口"
> [[01_P0_仓库与构建基线/05_Linker Script 与内存布局]] 列过这组符号，说它们是**链接脚本给 Startup 的接口**。
> 现在你看到接口的**物理形态**了：**就是六个放在 Flash 里的 32 位数。**
>
> ```text
> 链接脚本：_estack = ORIGIN(RAM) + LENGTH(RAM);
>      ↓ 链接器算出这个值
>      ↓ 写成 .text 里的一个常量
> Startup：ldr.w sp, [pc, #52]
>      ↓ 运行时把它读进寄存器
> ```
>
> **没有任何"变量"被创建，也没有任何 RAM 被占用。** 六个常量，躺在 Flash 里被读一次。

### 为什么 ARM 要这么绕

Cortex-M 没有"把 32 位立即数直接装进寄存器"的指令（指令宽度不够）。

所以编译器用**常量池 + PC 相对加载**：

```text
把常量放在函数末尾（常量池）
    ↓
用 ldr rX, [pc, #offset] 读进来
```

这就是为什么你会在一段代码后面看到一串 `.word`——**它们不是数据，是被拆散的立即数。**

---

## 9. 附带发现：`ADC_IRQHandler`

紧跟 `Reset_Handler` 之后：

```asm
080002c8 <ADC_IRQHandler>:
80002c8:  e7fe   b.n   80002c8 <ADC_IRQHandler>
```

**这条指令跳到自己。**

`b.n 80002c8` 的机器码是 `e7fe`。想确认它真的是自跳转，可以自己汇编一句 `b .`：

```bash
printf '\t.thumb\n\t.syntax unified\n\t.text\nloop:\n\tb .\n' > btest.s
arm-none-eabi-as -mthumb -o btest.o btest.s
arm-none-eabi-objdump -d btest.o
```

```text
00000000 <loop>:
   0:  e7fe    b.n   0 <loop>
```

**机器码一模一样，目标地址等于自己。**

> [!note] 为什么是 `e7fe` 而不是"偏移 0"
> Thumb 的 `b` 是 16 位指令，格式为 `11100` + 11 位立即数：
> ```text
> e7fe  =  1110 0111 1111 1110
>          └─11100─┘ └──imm11──┘
>                      = 0x7FE = 2046
> ```
> 分支偏移 = `imm11 × 2` = 4092，按 **12 位有符号**解释 = **−4**。
>
> 再算上 Thumb 的 PC 约定（读 PC 时 = 当前地址 + 4）：
> ```text
> 目标 = (当前地址 + 4) + (−4) = 当前地址
> ```
> **偏移 −4 和 PC+4 正好抵消，落回自己。** 这就是"偏移 0 却写成 e7fe"的原因。

这就是 Startup 里**所有未实现的 weak handler** 的样子。你在向量表 dump 里看到的：

```text
8000000 00000220 79020008 c9020008 c9020008
                        └─ 0x080002C9
```

`0xC9` 结尾（奇数）= `0x080002C8` + Thumb bit。这些全是 `ADC_IRQHandler` 这类**空 handler**。

```text
如果 ADC 中断意外触发：
    → CPU 跳进 0x080002C8
    → 死循环卡在那里
```

> [!tip] 这是一个很好的"意外中断"诊断线索
> 调试时如果程序莫名其妙"卡住不走了"，看一眼 PC：
> 如果停在这种 `b .` 自跳转指令上，说明**有个你没实现的中断被触发了**。
> 顺着这个 PC 反查它属于哪个 Handler，就能定位是哪个外设。

---

## 10. 两个会绊倒你的真实产物

这一节记录的是**同一份 ELF 里两个反直觉的现象**。不写下来，你早晚会撞上并被误导。

### 10.1 `readelf -h` 的入口地址是奇数

```bash
arm-none-eabi-readelf -h build/.../exp001_gpio_output.elf | grep Entry
```

```text
Entry point address:               0x8000279
```

但 `nm` 明明说：

```text
08000278 W Reset_Handler
```

**差 1。**

> [!important] 这不是 bug，是 Thumb bit
> ```text
> nm 报的         0x08000278   偶数，真实指令地址
> ELF e_entry     0x08000279   奇数，最低位置 1
> ```
> 链接器在写 `e_entry` 时**主动把最低位设成了 1**，表示"这是一个 Thumb 函数"。
>
> 对照向量表 dump 里那一列全是 `...C9` 的值——都是奇数，同一个原因。

**两个数都对。** 但这里必须把“代码地址”和“带 Thumb 状态的入口表示”分开：

```text
Reset_Handler 的代码 symbol value → 0x08000278
ELF e_entry 的入口表示            → 0x08000279
```

> [!important]
> `e_entry` 是 ELF 文件格式中的字段，**Cortex-M 硬件 Reset 不会读取 ELF Header 来决定启动位置**。
> 真正的复位入口仍来自向量表 `vector[1]`。
>
> 因此这里的重点只是：同一个 Thumb 函数在不同表示场景中，可能看到偶数代码地址和 bit0=1 的入口值；不要把 `e_entry` 当成硬件复位向量。

完整原理见 [[10_基础知识体系/03_Cortex-M与启动中断/15_Thumb状态与函数地址最低位]]。

### 10.2 `objdump -h` 给 `.bss` 显示了一个 Flash 的 LMA

```bash
arm-none-eabi-objdump -h build/.../exp001_gpio_output.elf
```

看这三行：

```text
Idx Name              Size      VMA       LMA
  8 .data            00000000  20000000  20000000   ← 同一个坑
  9 .bss             0000001c  20000000  08000334
 11 ._user_heap_stack 00000604  2000001c  08000334
```

**`.bss` 和 `._user_heap_stack` 的 LMA 是 `0x08000334`——一个 Flash 地址。**

> [!warning] 这里会让人得出错误结论
> 你可能刚学完 "`.bss` 没有 `AT> ROM`，所以它不该有 LMA"，然后就看到这个 Flash LMA，以为哪里错了。
>
> **这个 LMA 没有意义。** 原因：
> ```text
> LMA = "这个 section 的内容在文件里的加载位置"
> .bss 和 ._user_heap_stack 的文件内容是 0 字节
>   ↓
> 没有内容需要加载
>   ↓
> 链接器只是把它们排在某个 LOAD segment 里
> 就顺手填了个该 segment 的地址
> ```
> `0x08000334` 正好就是 `_sidata`——它只是"上一个 LOAD segment 结束的位置"，被借用了。

#### 更麻烦的一行：`.data` 的 LMA 自相矛盾

```text
objdump -h 说   .data LMA = 0x20000000   ← RAM 地址
nm 说           _sidata  = 0x08000334   ← Flash 地址
```

链接脚本明明写了 `>RAM AT> ROM`，`.data` 的 LMA 该在 Flash 才对。

> [!note] 原因：`Size = 0` 时 binutils 把 LMA 折叠成了 VMA
> 当前 `.data` 长度是 0（没有非零初值的全局变量），**没有内容需要"从哪加载"**，
> 于是 `objdump -h` 干脆把 LMA 显示成和 VMA 一样。
>
> `_sidata` 不受影响——它是链接脚本算出来的符号，仍然是 `0x08000334`。

**教训：`objdump -h` 的 LMA 列在空段/无内容段上不可靠。**
`Size = 0` 的段（`.rodata`、`.preinit_array`、`.data`）都有这个现象。

> [!tip] 什么时候该怀疑 LMA 列
> ```text
> 段的 Size = 0        → LMA 列看看就好
> 段没有 CONTENTS      → LMA 列看看就好
> ```
> 回到 `nm` 看链接脚本符号（`_sidata`/`_sdata`），或看 `readelf -l` 的 `PhysAddr`。

### 正确的读法是 `readelf -l`

```bash
arm-none-eabi-readelf -l build/.../exp001_gpio_output.elf
```

```text
LOAD 0x001000 0x08000000 0x08000000 0x00334 0x00334 R E 0x1000
                  └ VMA       └ LMA     └FileSiz └MemSiz
LOAD 0x001000 0x20000000 0x08000334 0x00000 0x0001c RW  0x1000
                                        └ 0     └ 28
LOAD 0x00001c 0x2000001c 0x08000334 0x00000 0x00604 RW  0x1000
                                        └ 0     └ 1540
```

**决定性的是这两列：**

```text
.bss 段：          FileSiz = 0        MemSiz = 0x1c   = 28
._user_heap_stack：FileSiz = 0        MemSiz = 0x604  = 1540
```

> [!success] 这就是"占 RAM 不占 Flash"最干净的证据
> ```text
> FileSiz = 0    → 文件里一个字节都没有 → 不占 Flash ✓
> MemSiz  = 28   → 运行时占 28 字节 RAM  ✓
> ```
> **比 `objdump -h` 好得多**，因为 `objdump -h` 根本不显示 FileSiz/MemSiz，
> 只给你一个会误导人的 LMA。

顺带，这两行还顺手证实了 [[01_P0_仓库与构建基线/07_最小 Executable 与 ELF 体检]] 第 7.3 节的算术：

```text
28 (.bss) + 1540 (._user_heap_stack) = 1568 = size 报的 bss
```

**三个工具（`size -A`、`readelf -l`、链接脚本）互相印证。**

---

## 11. 把整页串起来

```text
08000278  ldr.w sp, =_estack          ← 从常量池重建 MSP
0800027c  bl SystemInit               ← 当前工程中开启 FPU（CP10/CP11），不配置系统时钟
08000280  ldr r0, =_sdata   ┐
08000282  ldr r1, =_edata   │ 装三个地址
08000284  ldr r2, =_sidata  ┘
08000288  b LoopCopyDataInit
0800028a  ┌ ldr r4,[r2,r3]            ← .data copy
          │ str r4,[r0,r3]
          │ adds r3,#4                ← 本次没执行（_sdata == _edata）
08000290  └ cmp / bcc
08000296  ldr r2, =_sbss    ┐
08000298  ldr r4, =_ebss    ┘
0800029e  ┌ str r3,[r2]               ← .bss clear
          │ adds r2,#4
08002a2  └ cmp / bcc                  ← 执行 7 次 = 28 字节
08002a6  bl __libc_init_array         ← 遍历 .init_array（只有 1 项）
08002aa  bl main                      ← 进入 C 世界
08002ae  bx lr                        ← 到不了
```

**二十七条指令，把 C 语言的运行环境建立起来了。**

---

## 12. 现在去做 09 页的实验

有了本页的地址，[[01_P0_仓库与构建基线/09_动手追踪一次启动]] 的实验可以填得更具体：

```text
[ ] 在 8000290 (LoopCopyDataInit) 下断点
        → 看 r0、r1 是否都是 0x20000000
        → 单步一次，确认直接跳出（因为 _sdata == _edata）

[ ] 在 80002a2 (LoopFillZerobss) 下断点
        → 看 r2 从 0x20000000 开始
        → 单步 7 次，看 r2 到达 0x2000001c

[ ] 在 80002a6 (bl __libc_init_array) 前看
        → 0x0800032c 处的值（第 6 节已推出是 0x0800022D）
        → 动态确认：x/1xw 0x0800032c  →  应该是 0x0800022d

[ ] 在 80002aa (bl main) 下断点
        → 这是"C 世界"的入口

[ ] 在 800027c (bl SystemInit) 前后各看一次 CPACR
        → 单步过 SystemInit，然后  x/1xw 0xE000ED88
        → 调用前 bit20-23 应该是 0，调用后应该是 0xF00000
        → 这是整条启动链里唯一一次碰硬件寄存器的操作
```

**静态已经推出来的东西，动态再验一遍** —— 两边的结论对上了，才算真的证实。

第 6 节给的是静态推断（读 ELF 得出的），第 3 节给的也是静态推断（`SystemInit` 写 CPACR）。
下面这些断点就是用运行时证据去核对它们。

> [!tip] 造一个真的有 `.data` 的场景
> 现在 `.data` 是空的，copy 循环观察不到任何东西。
> 按 09 页第 1 节，在 `main.c` 里加：
> ```c
> volatile uint32_t counter = 10;
> volatile uint32_t flag;
> ```
> 重新构建后：
> ```text
> counter → .data → copy 循环真的会转
> flag    → .bss  → clear 循环多 4 字节
> ```
> **那时候再看 `800028a` 那个循环，才是它真正工作的样子。**

---

## 13. 离开本页前

1. `Reset_Handler` 第一句为什么还要设一次 SP？CPU 复位时不是已经设过了吗？
2. `ldr rX, [pc, #offset]` 算地址时，PC 的值为什么是"当前地址 + 4"？
3. 常量池里的六个值分别对应哪些符号？它们占 RAM 吗？
4. 为什么 `.data` copy 循环在你当前工程里一次都没执行？
5. `.bss` clear 循环为什么用 `bcc`（`<`）而不是 `<=`？用 `<=` 会怎样？
6. `bl __libc_init_array` 之后，它内部怎么知道要遍历多少个函数？
7. `main` 地址比 `Reset_Handler` 小，为什么反而后执行？
8. `bx lr` 那条指令为什么是死代码？如果 `main` 真返回了会怎样？
9. 看到 PC 停在 `b .` 自跳转上，应该怀疑什么？
10. `readelf -h` 报的入口地址是奇数，`nm` 报的是偶数，哪个对？
11. `objdump -h` 给 `.bss` 显示了一个 Flash 的 LMA，这个值有意义吗？应该换哪个命令看？
12. `SystemInit` 在你的工程里具体做了什么？为什么它必须在浮点代码之前执行？
13. `.init_array` 里那一个指针指向哪个函数、带不带 Thumb bit？它实际做了什么？
14. `__libc_init_array` 为什么用 `blx r3` 而不是 `bl`？

> [!tip] 答不上来的，回去看对应小节
> ```text
> 12  →  第 3 节（SCB->CPACR）
> 13  →  第 6 节（frame_dummy）
> 14  →  第 6 节（blx 只能跳数据里读出来的地址）
> ```

---

## 14. 关联

- [[10_基础知识体系/03_Cortex-M与启动中断/07_Reset_Handler 逐段看]]（本页的骨架版：为什么有这几步）
- [[10_基础知识体系/03_Cortex-M与启动中断/18_读Startup所需的最小Thumb汇编]]（本页用到的指令）
- [[10_基础知识体系/03_Cortex-M与启动中断/19_C与汇编如何互相调用_AAPCS最小理解]]（`bl` 为什么能调 C 函数）
- [[10_基础知识体系/03_Cortex-M与启动中断/15_Thumb状态与函数地址最低位]]（入口地址为什么是奇数）
- [[10_基础知识体系/02_构建与链接/20_NOBITS NOLOAD 与为什么RAM占用不等于Flash占用]]（FileSiz / MemSiz 的含义）
- [[10_基础知识体系/02_构建与链接/23_真实链接脚本逐段注解]]（常量池里那六个值的来源）
- [[10_基础知识体系/03_Cortex-M与启动中断/04_向量表 Vector Table]]（`ADC_IRQHandler` 在表里的位置）
- [[01_P0_仓库与构建基线/09_动手追踪一次启动]]（用本页地址做实验）
- [[10_基础知识体系/03_Cortex-M与启动中断/06_Startup 文件到底是什么]]
