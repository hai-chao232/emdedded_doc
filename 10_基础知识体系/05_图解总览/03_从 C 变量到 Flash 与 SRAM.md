---
tags: [diagram, c, data, bss, stack]
---

# 从 C 变量到 Flash 与 SRAM

以下是“常见情况”，最终必须用 ELF 验证。

---

# 1. 非零初值全局变量

```c
uint32_t a = 10;
```

```text
SOURCE
a = 10
  │
  ▼
Object input .data
  │
  ▼
Linker
  ├─ Flash: initial image 10
  └─ SRAM : runtime address for a
  │
  ▼
Startup copy
  │
  ▼
main: a == 10
```

---

# 2. 未显式初始化全局变量

```c
uint32_t b;
```

```text
SOURCE
b
 │
 ▼
Object input .bss
 │
 ▼
Linker
 └─ SRAM reserve
 │
 ▼
Startup clear
 │
 ▼
main: b == 0
```

---

# 3. `const`

```c
const uint32_t c = 123;
```

常见：

```text
.rodata
→ Flash
```

如果不需要可写 runtime copy，就不必进 SRAM。

---

# 4. 普通局部变量

```c
void f(void)
{
    uint32_t x = 5;
}
```

不要直接写：

```text
x 一定在 stack
```

可能：

```text
register
stack
constant folded
optimized away
```

---

# 5. static local

```c
void f(void)
{
    static uint32_t count;
}
```

它不是 automatic storage duration。

常见：

```text
.bss
→ SRAM
```

虽然名字只在函数作用域可见。

---

# 6. 一张综合图

```text
C declaration
   │
   ├─ static duration + nonzero writable
   │        ↓
   │      .data
   │   Flash init + SRAM runtime
   │
   ├─ static duration + zero
   │        ↓
   │      .bss
   │      SRAM
   │
   ├─ readonly constant
   │        ↓
   │     .rodata
   │      Flash
   │
   └─ automatic local
            ↓
      register / stack / optimized
```

编译器有自由度，所以：

> 图是心智模型，ELF 才是当前构建事实。
