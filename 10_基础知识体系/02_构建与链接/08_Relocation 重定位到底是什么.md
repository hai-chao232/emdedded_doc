---
tags: [build, relocation, linker]
aliases: [Relocation]
---

# Relocation：重定位到底是什么？

## 一句话先说清

编译/汇编阶段很多引用还不知道最终地址。Relocation 就是“这里以后需要 linker 根据最终布局修正”的记录和机制。

---

## 1. 最简单的例子

`a.c`：

```c
extern int value;

int read_value(void)
{
    return value;
}
```

编译 `a.c` 时：

> `value` 最终放哪里？

不知道。

因为定义可能在：

```text
b.o
library
linker script
```

所以 object 里保留：

```text
对 symbol value 的引用
+
需要修正的位置
+
relocation type
```

---

## 2. 函数调用也一样

```c
foo();
```

如果 foo 在另一个 object，assembler 可能无法确定 branch offset。

Linker 最终知道：

```text
caller address
foo address
```

才能修正相关机器码/字面量。

---

## 3. 为什么叫“重定位”？

因为 object file 在链接前没有固定最终位置。

Linker 把：

```text
input section A
```

放到某个最终地址后，原来相对/未知的引用就需要重新计算。

---

## 4. Linker 做的不是简单“文件拼接”

如果只是把 `.o` byte 串起来：

```text
main.o bytes + startup.o bytes
```

外部函数调用和地址引用根本不会自动正确。

Linker 必须：

```text
选择布局
解析 symbol
应用 relocation
生成最终地址
```

---

## 5. 怎样看 relocation？

对 `.o`：

```bash
arm-none-eabi-readelf -r main.c.o
```

你可能看到 relocation entries。

最终可执行 ELF 中很多静态 relocation 已被处理，因此表现不同。

---

## 6. 关联

- [[10_基础知识体系/02_构建与链接/05_Object File 目标文件是什么]]
- [[10_基础知识体系/02_构建与链接/06_Symbol 符号到底是什么]]
- [[10_基础知识体系/02_构建与链接/09_Linker 到底做了什么]]
