---
tags: [linker-script, location-counter]
aliases: [Location Counter, 点号]
---

# Location Counter：Linker Script 里的 `.` 是什么？

## 一句话先说清

在 `SECTIONS` 布局过程中，`.` 表示 linker 当前输出位置。随着内容被放进去，它会向后移动；给 symbol 赋值时，就能记录某个 section 的起点或终点。

---

## 1. 例子

```ld
.data :
{
    _sdata = .;
    *(.data*)
    _edata = .;
} >RAM
```

假设进入 `.data` 前：

```text
. = 0x20000000
```

于是：

```text
_sdata = 0x20000000
```

---

## 2. 收集内容后 `.` 会向前走

如果 `.data` 总共 12 bytes：

```text
0x20000000
+ 12
= 0x2000000C
```

所以：

```text
_edata = 0x2000000C
```

因此：

```text
[_sdata, _edata)
```

就是 12-byte 半开区间。

---

## 3. 可以手动移动 `.`

例如：

```ld
. = ALIGN(4);
```

把当前位置推进到满足 4-byte 对齐的位置。

或者：

```ld
. = . + _Min_Stack_Size;
```

人为预留空间。

---

## 4. 为什么 `_sdata = .` 不是写内存？

这是链接时表达式。

它创建：

```text
symbol _sdata
value = 当前地址
```

不会让 MCU 在该位置写一个数。

---

## 5. 这和普通 C 的 `.` 完全不同

C：

```c
obj.member
```

是结构体成员运算符。

Linker script：

```ld
.
```

是 location counter。

只因为符号长得一样，没有关系。

---

## 6. 关联

- [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]
- [[10_基础知识体系/02_构建与链接/12_SECTIONS 命令怎样拼装最终镜像]]
- [[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]
