---
tags: [linker-script, MEMORY]
---

# `MEMORY` 命令到底在描述什么？

## 一句话先说清

`MEMORY` 给 linker 一张“允许使用的内存区域表”。它不创造 Flash/SRAM，只描述 linker 认为哪些地址范围可供 output section 使用。

---

## 1. 例子

```ld
MEMORY
{
    RAM (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
    ROM (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
}
```

---

## 2. `ORIGIN`

```text
这个 memory region 从哪个地址开始
```

例如：

```text
RAM origin = 0x20000000
```

---

## 3. `LENGTH`

```text
这个 region 有多大
```

128 KiB：

```text
128 × 1024
= 131072
= 0x20000
```

所以边界：

```text
0x20000000 + 0x20000
= 0x20020000
```

---

## 4. `(rx)` `(xrw)`

这些是 region attribute。

粗略：

```text
r = readable
w = writable
x = executable
```

它们可帮助 linker 匹配/检查 section 属性，但不要把它们误认为 MPU 硬件权限配置。

它们不是在运行时真的修改 STM32 总线权限。

---

## 5. `ROM` 只是名字

你可以写：

```ld
FLASH
```

也可以写：

```ld
ROM
```

只要后面一致。

这个名字不是硬件自动识别关键字。

比如：

```ld
.data >RAM AT>ROM
```

中的 `RAM`、`ROM` 是你在 MEMORY 里定义的 region name。

---

## 6. `ORIGIN(RAM)` 与 `LENGTH(RAM)`

Linker script 内置表达式可以读取 region 参数：

```ld
_estack = ORIGIN(RAM) + LENGTH(RAM);
```

于是当前：

```text
0x20000000 + 128K
= 0x20020000
```

---

## 7. Linker 怎样检查溢出？

如果某 output section 被放进：

```text
RAM
```

但最终超出 region 长度，linker 可以报：

```text
region `RAM' overflowed
```

这是非常有价值的静态检查。

---

## 8. 但它不能检查运行时 stack 峰值

如果 linker 只预留了某个最小 stack 区域，运行中深度调用/中断嵌套仍可能超出。

所以：

> memory region 容量检查 ≠ 完整运行时内存安全。

---

## 9. 关联

- [[10_基础知识体系/02_构建与链接/10_Linker Script 为什么存在]]
- [[10_基础知识体系/02_构建与链接/12_SECTIONS 命令怎样拼装最终镜像]]
- [[10_基础知识体系/01_计算机与MCU基础/06_Flash SRAM ROM RAM]]
