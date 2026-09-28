---
tags: [assembly, thumb, startup, cortex-m]
stage: P0
---

# 读 Startup 所需的最小 Thumb 汇编

> 目标不是学会手写复杂汇编，而是能读懂 Reset_Handler 中“加载地址、复制、清零、调用函数、循环”的骨架。

## 1. 先认寄存器名字

常见：

```text
r0-r12  通用寄存器
sp      stack pointer
lr      link register
pc      program counter
```

Startup 中常把：

```text
源地址
目标地址
结束地址
临时数据
```

放在 r0/r1/r2/r3 等寄存器。

## 2. `ldr`

两种常见直觉：

```asm
ldr r0, [r1]
```

→ 从 `r1` 指向的内存取值到 `r0`。

```asm
ldr r0, =_sdata
```

→ 汇编器伪指令语境下，把 `_sdata` 对应的地址/值装入 `r0`（具体编码可能变成 literal load 等）。

读反汇编时重点问：

> 这是在加载“地址”，还是在加载“地址指向的数据”？

## 3. `str`

```asm
str r0, [r1]
```

把 r0 中的值写到 r1 指向的内存。

`.data` copy 本质上就是：

```text
ldr source
str destination
source += word
destination += word
repeat
```

## 4. `mov / movs`

把一个值放进寄存器。

`.bss` clear 常先得到：

```text
0
```

再反复：

```text
str 0 → RAM
```

## 5. `cmp`

比较两个值，为后面的条件分支准备状态。

例如：

```text
当前目标地址
vs
结束地址
```

## 6. `b / bxx`

无条件或条件跳转。

循环常表现为：

```text
loop:
  ...
  cmp ...
  bne loop
```

## 7. `bl`

Branch with Link。

直觉：

```text
LR ← 返回相关地址
PC ← 目标函数入口
```

因此你会看到：

```asm
bl SystemInit
bl main
```

真正的参数/返回约定由 AAPCS 规定：

[[10_基础知识体系/03_Cortex-M与启动中断/19_C与汇编如何互相调用_AAPCS最小理解]]

## 8. 自增寻址

Startup 中可能出现：

```asm
ldr r3, [r0], #4
str r3, [r1], #4
```

可以先读成：

```text
使用当前地址访问
然后地址向前移动 4 字节
```

这非常适合按 word 复制 `.data`。

## 9. 怎样读一段陌生 Startup

不要逐条翻译成中文。

先标注角色：

```text
r0 = source?
r1 = destination?
r2 = end?
r3 = temp?
```

再识别结构：

```text
初始化地址
↓
loop
  load
  store
  地址推进
  compare
  branch
↓
下一个阶段
```

## 10. P0 的完成标准

你能在反汇编中指出：

- `.data` copy loop；
- `.bss` clear loop；
- `SystemInit` call；
- `main` call；

就足够进入 GPIO。
