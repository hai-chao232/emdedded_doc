---
tags: [AAPCS, ABI, C, assembly, calling-convention]
stage: P0
---

# C 与汇编如何互相调用：AAPCS 最小理解

## 1. 为什么 Startup 汇编能直接 `bl main`

因为汇编和 C 编译器约定了一套共同接口。

如果没有约定：

- 参数放哪里？
- 返回值放哪里？
- 哪些寄存器调用后还能保持？
- 栈怎样对齐？
- 返回地址怎样处理？

双方就无法可靠合作。

这类约定属于 ABI（Application Binary Interface）的一部分；ARM 常见调用约定由 AAPCS 描述。

## 2. 最小参数规则

当前阶段先记最常用直觉：

```text
r0-r3 → 常用于前几个参数
r0    → 常用于返回值
```

复杂类型、更多参数、浮点 ABI 等以后再学。

## 3. LR 与返回

`bl foo` 会保存返回相关信息到 LR，并跳到 `foo`。

函数结束时按约定把控制流送回调用者。

因此：

```text
Reset_Handler
  bl SystemInit
  ...
```

执行完 `SystemInit` 后还能回来继续。

## 4. caller-saved / callee-saved

某些寄存器允许被被调用函数随意改，调用者如果需要应自己保存。

另一些寄存器如果被函数使用，函数需要按约定恢复。

这让不同编译单元、不同语言之间可以独立编译后仍正确互调。

## 5. 为什么这和 Startup 有关

Startup 可能是汇编写的：

```text
assembly Reset_Handler
```

而：

```text
SystemInit
__libc_init_array
main
```

可能是 C 编译器产生的。

它们能连接起来，不只靠“函数名字一样”，还靠：

```text
linker symbol resolution
+
ABI calling convention
```

## 6. P0 不需要深入什么

暂时不要求：

- 完整 AAPCS32 文档；
- VFP 参数规则；
- variadic function 细节；
- structure return ABI；
- unwind tables。

只要知道：

> 汇编和 C 能互调，是因为它们遵守共同二进制接口约定。
