---
tags: [reference, arm-gnu, commands]
---

# ARM GNU 工具链命令

假设：

```bash
ELF=build/experiments/001_gpio_output/exp001_gpio_output.elf
```

每次运行命令前先问：**我现在具体想回答哪个问题？**

## 1. 快速看体积：`size`

```bash
arm-none-eabi-size "$ELF"
```

关注：`text / data / bss`。这只是汇总视图，不是完整内存地图。

## 2. 看 Section：`objdump -h`

```bash
arm-none-eabi-objdump -h "$ELF"
```

关注：

```text
.isr_vector / .text / .rodata / .data / .bss
Size / VMA / LMA / Alignment
```

## 3. 看符号：`nm -n`

```bash
arm-none-eabi-nm -n "$ELF"
```

P0 常用：

```bash
arm-none-eabi-nm -n "$ELF" \
  | grep -E 'Reset_Handler|main|_estack|_sidata|_sdata|_edata|_sbss|_ebss'
```

自定义变量：

```bash
arm-none-eabi-nm -n "$ELF" | grep -E 'counter|flag'
```

### 常见 symbol type

| 字母 | 常见含义 |
|---|---|
| `T/t` | text/code |
| `D/d` | initialized data |
| `B/b` | bss |
| `R/r` | read-only data（具体结合上下文） |
| `W/w` | weak symbol |
| `A/a` | absolute symbol |
| `U` | undefined |

## 4. 反汇编：`objdump -d`

```bash
arm-none-eabi-objdump -d "$ELF" | less
```

可搜索：

```text
<Reset_Handler>
<main>
```

如果有调试信息：

```bash
arm-none-eabi-objdump -S "$ELF" | less
```

可混合源码与反汇编。

## 5. 查看某个 Section 的 bytes

向量表：

```bash
arm-none-eabi-objdump -s -j .isr_vector "$ELF"
```

`.data`：

```bash
arm-none-eabi-objdump -s -j .data "$ELF"
```

## 6. `readelf`

```bash
arm-none-eabi-readelf -h "$ELF"   # ELF header
arm-none-eabi-readelf -S "$ELF"   # section table
arm-none-eabi-readelf -s "$ELF"   # symbol table
```

适合确认目标架构、入口、section 属性和 symbol 详细信息。

## 7. ELF → BIN

```bash
arm-none-eabi-objcopy -O binary "$ELF" firmware.bin
```

## 8. ELF → Intel HEX

```bash
arm-none-eabi-objcopy -O ihex "$ELF" firmware.hex
```

自动生成 `.bin/.hex/.map` 属于既定 E1 工程增强；P0 先理解作用。

## 9. 查看工具链版本

```bash
arm-none-eabi-gcc --version
arm-none-eabi-gcc -dumpfullversion
```

当前项目具体基线以工程 README / ROADMAP 为准；自动强制版本检查放在 E1。

## 10. 看最终构建命令

```bash
cmake --build build --target exp001_gpio_output -v
```

重点找：

```text
编译器完整路径
-DSTM32F446xx
-I...
-mcpu / -mthumb / FPU flags
-T linker_script.ld
```

## 11. P0 推荐体检顺序

```text
size
→ objdump -h
→ nm -n
→ objdump -s -j .isr_vector
→ objdump -d
```

对应：

```text
多大？
→ 各 section 在哪？
→ 关键 symbol 在哪？
→ 向量表实际内容是什么？
→ 最终控制流是什么？
```
