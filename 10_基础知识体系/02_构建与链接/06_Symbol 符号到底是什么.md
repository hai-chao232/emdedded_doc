---
tags: [build, symbol, linker]
aliases: [Symbol]
---

# Symbol：符号到底是什么？

## 一句话先说清

Symbol 是“名字与某种链接信息之间的关联”。它可以代表函数、对象、地址、linker 定义值等。Symbol 不是“那个物理内存单元本身”。

---

## 1. 一个函数为什么需要 symbol？

源码：

```c
void foo(void) {}
```

编译后 object 需要保留：

```text
名字：foo
类型：function
属于哪个 section
相对位置
binding
```

别的 object 调用：

```c
foo();
```

Linker 才能把引用解析到定义。

---

## 2. Undefined Symbol

某个 `.o` 里：

```text
U SystemInit
```

不一定是错误。

它只表示：

> 当前这个 object 引用了 SystemInit，但定义不在这个 object。

只要最终链接时能从另一个 object/library 找到定义，就没问题。

---

## 3. Global / Local / Weak

### Global

可参与跨 object 链接解析。

### Local

只在当前 object 的链接语义范围内。

### Weak

如果没有更强定义，可以使用；如果出现同名 strong definition，通常被覆盖。

Startup 默认 handler 常用 weak。

---

## 4. Linker Script 也可以创建 symbol

```ld
_sdata = .;
_estack = ORIGIN(RAM) + LENGTH(RAM);
```

这些名字不是 C 编译器生成的变量。

它们是 linker 创建的 symbol。

---

## 5. Symbol value 不一定是“内存里保存的值”

例如：

```text
_estack = 0x20020000
```

表示 symbol 的数值是这个地址/边界。

不等于：

```text
地址 0x20020000 里存着整数 0x20020000
```

---

## 6. `nm` 为什么有用？

```bash
arm-none-eabi-nm -n firmware.elf
```

让你看到：

```text
地址/值
symbol type
名字
```

因此特别适合验证：

```text
main
Reset_Handler
_sdata
_edata
_sbss
_ebss
_estack
```

---

## 7. Symbol 和 debug info 不是一个东西

即使 strip 掉很多调试信息，某些 symbol table 仍可能存在。

Debug info 还包含：

```text
源码行号
类型
变量位置
作用域
```

二者相关但不同。

---

## 8. 关联

- [[10_基础知识体系/02_构建与链接/05_Object File 目标文件是什么]]
- [[10_基础知识体系/02_构建与链接/08_Relocation 重定位到底是什么]]
- [[10_基础知识体系/02_构建与链接/13_Location Counter 点号是什么]]
