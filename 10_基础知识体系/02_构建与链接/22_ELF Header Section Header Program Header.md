---
tags: [ELF, header, readelf]
stage: P0
---

# ELF Header、Section Header、Program Header

## 1. 先把 ELF 看成一个结构化文件

```text
ELF
├─ ELF Header
├─ Program Header Table
├─ 各种 section 内容
└─ Section Header Table
```

实际文件排列可以更复杂，但 P0 先建立这个逻辑模型。

## 2. ELF Header

用：

```bash
readelf -h firmware.elf
```

可以看到 ELF 的总体身份信息，例如：

- 32/64 位；
- endian；
- machine architecture；
- entry point；
- program header/section header 的位置和数量。

它回答：

> “这到底是一个什么 ELF？”

## 3. Section Header Table

用：

```bash
readelf -S firmware.elf
```

看到每个 section 的：

```text
name
type
address
offset
size
flags
alignment
...
```

它回答：

> “ELF 内部有哪些逻辑分类区域？”

## 4. Program Header Table

用：

```bash
readelf -l firmware.elf
```

看到：

```text
LOAD
filesz
memsz
flags
alignment
...
```

它回答：

> “如果从运行/装载角度看，要建立哪些内存区域？”

## 5. `filesz` 与 `memsz` 为什么可能不同

这是理解 `.bss` 的好入口。

某个运行时区域：

```text
memsz > filesz
```

可能意味着：

> 运行时需要的内存空间，比文件中真实携带的数据字节更多。

其中一部分可能是启动时清零建立的。

## 6. Entry point 不等于 Cortex-M 复位机制全部真相

ELF Header 可能有 entry point 字段。

但 Cortex-M 硬件复位时依照架构从向量表获得初始 MSP 和 reset vector。

因此不要把：

```text
ELF entry point
```

机械理解成：

```text
芯片上电后硬件唯一直接读取的位置
```

需要把“文件格式入口信息”与“Cortex-M reset vector 规则”区分开。

### 一个真实的值

```bash
arm-none-eabi-readelf -h build/.../exp001_gpio_output.elf | grep Entry
```

```text
Entry point address:               0x8000279
```

两个观察：

```text
① 它是奇数   → 最低位是 Thumb 状态位，不是笔误
② 它确实是 Reset_Handler（0x08000278 + 1）
   → 说明当前工程的 ENTRY(Reset_Handler) 是对的
```

但 **即使它是错的，芯片上电也照样从 vector[1] 启动**——因为 Cortex-M 硬件复位根本不看 `e_entry`。

```text
e_entry        → 给调试器/加载器看的"建议入口"
vector[1]      → 硬件实际使用的复位入口
```

两者恰好一致是**因为链接脚本写了 `ENTRY(Reset_Handler)`**，不是硬件要求的。

见 [[10_基础知识体系/03_Cortex-M与启动中断/20_Reset_Handler 反汇编逐条对照]] 第 10.1 节。

