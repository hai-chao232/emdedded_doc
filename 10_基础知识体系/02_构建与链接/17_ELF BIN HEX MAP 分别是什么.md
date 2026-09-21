---
tags: [build, ELF, BIN, HEX, MAP]
aliases: [ELF, BIN, HEX, MAP]
---

# ELF、BIN、HEX、MAP 分别是什么？

## 一句话先说清

它们不是“同一个固件换后缀”。ELF 保留最丰富的结构和调试信息；BIN 是原始数据镜像；HEX 是带地址的文本记录格式；MAP 是 linker 生成的人类可读布局报告。

---

## 1. ELF

适合：

```text
调试
反汇编
看 symbol
看 section
看 VMA/LMA
看 debug info
```

P0 最重要。

---

## 2. BIN

近似：

```text
连续原始 bytes
```

通常没有：

- symbol；
- section 名字；
- debug info。

适合某些烧录/打包流程。

---

## 3. Intel HEX

文本格式，每条记录包含：

```text
地址信息
数据
记录类型
校验
```

优点：

> 可以表达不连续地址区域。

---

## 4. MAP

Linker 生成的文本报告。

特别适合回答：

```text
这个 symbol 来自哪个 object？
这个 output section 收进了哪些 input section？
哪个 library member 被拉进来了？
地址是怎样一路排下来的？
```

因此 E1 自动生成 MAP 很有价值。

---

## 5. ELF → BIN/HEX

通常通过：

```bash
arm-none-eabi-objcopy
```

从已链接 ELF 提取。

说明：

> ELF 是更上游、更富信息的主产物。

---

## 6. 烧录器到底需要哪个？

取决于工具。

有的直接解析 ELF；
有的接受 HEX；
有的只接受 BIN 加起始地址。

不要把“能烧录”当作格式唯一价值。

---

## 7. 关联

- [[01_P0_仓库与构建基线/07_最小 Executable 与 ELF 体检]]
- [[10_基础知识体系/04_验证与调试/02_size nm objdump readelf 各自回答什么]]
