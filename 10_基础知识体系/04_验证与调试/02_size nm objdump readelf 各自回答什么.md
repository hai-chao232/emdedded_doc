---
tags: [debug, size, nm, objdump, readelf]
---

# `size`、`nm`、`objdump`、`readelf` 各自回答什么？

## 一句话先说清

它们都在读 ELF，但视角不同：`size` 看容量概览；`nm` 看 symbol；`objdump` 适合 section/content/disassembly；`readelf` 更直接展示 ELF 结构。

---

# 1. `size`

问题：

> 这个固件大概用了多少 Flash / 静态 RAM？

```bash
arm-none-eabi-size firmware.elf
```

不要用它独自回答：

```text
所有 section 到底在哪？
stack 峰值是多少？
```

---

# 2. `nm`

问题：

> 某个 symbol 是否存在？最终值/地址是什么？weak 还是 strong？

```bash
arm-none-eabi-nm -n firmware.elf
```

特别适合：

```text
main
Reset_Handler
_estack
_sdata/_edata
counter
IRQHandler
```

---

# 3. `objdump -h`

问题：

> 最终有哪些 section？VMA/LMA/size 是多少？

```bash
arm-none-eabi-objdump -h firmware.elf
```

---

# 4. `objdump -s`

问题：

> 某个 section 的原始 bytes 是什么？

```bash
arm-none-eabi-objdump -s -j .isr_vector firmware.elf
```

---

# 5. `objdump -d`

问题：

> 最终机器码反汇编后控制流是什么？

```bash
arm-none-eabi-objdump -d firmware.elf
```

适合验证：

```text
Reset_Handler 是否 call SystemInit/main
GPIO 操作最后生成了什么
```

---

# 6. `readelf -h`

问题：

> ELF header 说目标架构、entry 等是什么？

---

# 7. `readelf -S`

问题：

> section table 详细属性？

---

# 8. `readelf -s`

问题：

> symbol table 的 type/bind/section index？

比 nm 信息更结构化。

---

# 9. `readelf -r`

问题：

> relocatable object 里有哪些 relocation？

对 `.o` 特别有教学价值。

---

# 10. 不要混用“工具名”和“问题”

正确习惯：

```text
问题：
Reset_Handler 最终地址是多少？

工具：
nm
```

而不是：

```text
今天学 nm，所以跑 nm 看看。
```

---

## 11. 关联

- [[10_基础知识体系/02_构建与链接/17_ELF BIN HEX MAP 分别是什么]]
- [[10_基础知识体系/04_验证与调试/07_怎样验证向量表]]
