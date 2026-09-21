---
tags: [linker-script, SECTIONS, section]
---

# `SECTIONS` 命令怎样拼装最终镜像？

## 一句话先说清

`SECTIONS` 告诉 linker：把各个 object 的哪些 input section 收集起来，形成什么 output section，并把它放到哪个 memory region。

---

## 1. 最简单例子

```ld
.text :
{
    *(.text)
    *(.text*)
} >ROM
```

拆开：

```text
.text :
→ 创建/描述一个 output section 名叫 .text

*(.text)
*(.text*)
→ 从所有输入文件收集匹配 section

>ROM
→ 把最终 output .text 放进 MEMORY 中名为 ROM 的 region
```

---

## 2. `*()` 是什么意思？

概念：

```text
*(.text*)
```

表示：

> 任意 input file 中，section name 匹配 `.text*` 的内容。

例如可能收进：

```text
main.o:.text.main
system.o:.text.SystemInit
libc.a(...):.text.memcpy
```

具体命名取决于编译器选项。

---

## 3. 为什么常有 `.text*`？

启用：

```text
-ffunction-sections
```

后，每个函数可能进入独立 section：

```text
.text.main
.text.foo
.text.bar
```

所以 linker script 用：

```text
*(.text*)
```

一并收集。

---

## 4. 顺序为什么重要？

Output section 的排列会影响最终地址。

例如：

```ld
.isr_vector >ROM
.text       >ROM
.rodata     >ROM
```

通常意味着：

```text
Flash 起点
→ vector
→ code
→ rodata
```

如果你改变顺序，最终地址也会变。

---

## 5. `.data` 为什么特殊？

```ld
.data :
{
  ...
} >RAM AT>ROM
```

它同时涉及：

```text
运行地址 → RAM
加载地址 → ROM
```

详见：

[[10_基础知识体系/02_构建与链接/15_VMA LMA 与 data 初始化镜像]]

---

## 6. `.bss` 为什么没有 `AT>ROM`？

因为 `.bss` 不需要保存一整份零值 payload。

它只需要运行地址空间：

```ld
.bss :
{
  ...
} >RAM
```

Startup 运行时清零。

---

## 7. Wildcard 可能把你意想不到的内容收进去

所以分析最终镜像不要只看脚本“感觉”。

要用：

```text
MAP
objdump -h
readelf -S
```

确认实际输入和输出。

---

## 8. 关联

- [[10_基础知识体系/02_构建与链接/07_Section 到底是什么]]
- [[10_基础知识体系/02_构建与链接/16_KEEP gc-sections 与为什么向量表不能被删]]
- [[10_基础知识体系/02_构建与链接/17_ELF BIN HEX MAP 分别是什么]]
