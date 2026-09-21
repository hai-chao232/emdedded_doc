---
tags: [build, libc, newlib, syscall]
aliases: [Newlib, Newlib Nano, syscall]
---

# C Library、Newlib Nano 与裸机 syscall

## 一句话先说清

即使是裸机 C，你仍可能链接 C library。像 `printf/malloc/exit` 这样的库功能可能需要“底层系统服务”；裸机没有 Linux 内核，所以要由 stub、retarget 或你自己的实现补上。

---

## 1. C 语言和 C library 不是同一件事

语言特性：

```text
if
for
struct
function
pointer
```

不需要 OS。

而库函数：

```c
printf()
malloc()
memcpy()
strlen()
```

属于库。

其中一些纯计算函数很容易裸机使用，一些则需要底层 I/O/内存管理支持。

---

## 2. Newlib

ARM GNU Embedded 工具链常带 Newlib。

Newlib Nano 是更偏嵌入式体积优化的变体/配置。

工程中常见：

```text
--specs=nano.specs
```

---

## 3. 为什么 `printf` 需要 `_write`？

`printf` 最终需要：

> 把字符送到某个“文件描述符/输出设备”。

Linux 有内核 syscall。

裸机没有 stdout。

因此 Newlib 的底层钩子可能需要：

```text
_write
_read
_close
_fstat
_isatty
_lseek
_sbrk
_exit
...
```

具体由你实际使用的功能决定。

---

## 4. `nosys.specs`

常见：

```text
--specs=nosys.specs
```

提供一部分不连接真实 OS 的 stub。

它可以帮助链接通过，但不意味着：

> printf 自动从 UART 出来了。

---

## 5. Retarget 是什么？

例如后面 UART 实验中，你可以让：

```text
_write()
```

把 bytes 发到 USART。

于是：

```c
printf("hello");
```

最终通过 UART 输出。

这叫把 C library I/O “重定向/retarget”到你的硬件。

---

## 6. 为什么 P0 不急着做 printf？

因为这样会同时引入：

```text
UART
clock
GPIO alternate function
baud rate
libc I/O
retarget
```

变量太多。

所以课程把 UART/printf 放到合适实验阶段。

---

## 7. `malloc` 与 `_sbrk`

Newlib 的 heap 管理可能用 `_sbrk` 向底层请求扩大 heap。

裸机必须定义 heap 边界和冲突策略，否则动态分配不可控。

见：

[[10_基础知识体系/01_计算机与MCU基础/12_堆 Heap 是什么]]

---

## 8. 关联

- [[10_基础知识体系/02_构建与链接/17_ELF BIN HEX MAP 分别是什么]]
- [[10_基础知识体系/01_计算机与MCU基础/12_堆 Heap 是什么]]
