---
tags: [linker, VMA, LMA, data]
aliases: [VMA, LMA]
---

# VMA、LMA 与 `.data` 初始化镜像

## 一句话先说清

VMA 是 section 运行时使用的地址；LMA 是它在加载镜像中的地址。`.data` 因为“运行时要在 SRAM，但初值必须随固件保存在 Flash”，所以常出现 VMA 和 LMA 不同。

---

## 1. 先看 `.text`

代码通常：

```text
保存在哪里？Flash
运行/取指在哪里？Flash
```

所以：

```text
VMA ≈ LMA ≈ Flash address
```

---

## 2. 再看 `.data`

```c
uint32_t counter = 10;
```

运行时：

```text
counter 必须可写
→ SRAM
```

断电后：

```text
初值 10 不能丢
→ Flash image
```

于是：

```text
LMA = Flash
VMA = SRAM
```

---

## 3. 图

```text
ELF / Flash image

0x0800....
┌───────────────────┐
│ .data load image  │
│ counter = 10      │
└────────┬──────────┘
         │
         │ Reset_Handler copies
         ▼

SRAM

0x2000....
┌───────────────────┐
│ .data runtime     │
│ counter = 10      │
└───────────────────┘
```

---

## 4. `>RAM AT>ROM`

```ld
.data :
{
   ...
} >RAM AT>ROM
```

读成：

```text
VMA region = RAM
LMA region = ROM
```

---

## 5. `_sidata`

```ld
_sidata = LOADADDR(.data);
```

得到：

```text
.data load image 的起始地址
```

Startup 用：

```text
source = _sidata
destination = _sdata
end = _edata
```

完成 copy。

---

## 6. 为什么不是烧录工具直接写 RAM？

因为 SRAM 是易失的。

即使某次下载工具顺便写了 RAM，断电后也没有持久性。

固件必须在每次 reset/startup 时建立 C runtime 需要的状态。

---

## 7. `.bss` 为什么没有 LMA payload？

它的逻辑初始值就是 0。

保存一个：

```text
长度 N，需要清零
```

的信息足够，不需要在 Flash 放 N 个 0 byte。

---

## 8. 怎样验证？

```bash
arm-none-eabi-objdump -h firmware.elf
```

找到 `.data`：

```text
VMA → 0x200...
LMA → 0x080...
```

再用：

```bash
nm
```

看：

```text
_sidata
_sdata
_edata
```

---

## 9. 关联

- [[10_基础知识体系/02_构建与链接/14_data 与 bss 到底是什么]]
- [[10_基础知识体系/03_Cortex-M与启动中断/08_data copy 与 bss clear 为什么由 Startup 做]]
