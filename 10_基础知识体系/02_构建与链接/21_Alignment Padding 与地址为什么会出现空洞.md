---
tags: [alignment, padding, linker, memory]
stage: P0
---

# Alignment、Padding 与地址为什么会出现空洞

## 1. 对齐是什么

很多数据/指令希望或要求从特定倍数地址开始。

例如 4 字节对齐：

```text
合法起点：
...00
...04
...08
...0C
```

这里说的是地址能被 4 整除的概念。

## 2. 为什么会出现 padding

如果前一个对象结束后，当前位置不满足下一个对象的对齐要求：

```text
对象 A
↓
当前位置不合适
↓
插入若干 padding
↓
对象 B 从合法边界开始
```

因此：

```text
两个 symbol 地址之间的差
```

不一定全部都是“有意义的变量数据”。

## 3. Linker Script 中的 ALIGN

示例：

```ld
. = ALIGN(4);
```

可以理解为：

> 把当前位置向上推进到下一个满足 4 字节对齐的地址。

## 4. 为什么 MCU 学习早期就要知道

你后面会在很多地方再次遇到：

- section 布局；
- 结构体 padding；
- 栈对齐；
- DMA 缓冲；
- cache line；
- 向量表对齐。

因此现在先形成“地址空洞可能是对齐产生的”这个直觉即可。

## 5. 怎样验证

用：

```bash
nm -n firmware.elf
objdump -h firmware.elf
readelf -S firmware.elf
```

观察相邻 section/symbol 地址。

再回看 linker script 是否出现：

```text
ALIGN(...)
```

不要看到间隙就立即认为链接器浪费或丢失了内容。
