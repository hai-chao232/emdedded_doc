---
tags: [P0, cmake, toolchain, build]
stage: P0
---

# 03｜CMake 与交叉编译

> [!tip] 深挖入口
> 如果 `compile / assemble / link / .o / ELF` 还没有形成直觉，先点：
>
> - [[10_基础知识体系/02_构建与链接/00_构建与链接地图]]
> - [[10_基础知识体系/02_构建与链接/01_从源码到 ELF 的完整流水线]]
> - [[10_基础知识体系/02_构建与链接/05_Object File 目标文件是什么]]
> - [[10_基础知识体系/02_构建与链接/09_Linker 到底做了什么]]

> [!abstract] 本节目标
> 能看懂一次完整构建经过哪些程序，并能判断错误发生在 Configure、Compile、Assemble 还是 Link。

## 1. 先分清角色

```text
CMake
→ 读取工程描述，生成构建规则

Ninja
→ 根据规则调度实际命令

ARM GNU Toolchain
→ 真正编译、汇编、链接和分析 ARM 裸机二进制
```

所以：**CMake ≠ 编译器，Ninja ≠ 编译器。**

## 2. 完整构建链

```text
cmake --preset stm32f446ze
        ↓
Preset / Toolchain / CMakeLists
        ↓
生成 Ninja rules

cmake --build build --target exp001_gpio_output -v
        ↓
Ninja 执行规则
        ↓
C / ASM → .o
        ↓
Linker + linker script
        ↓
ELF
```

Configure/Generate 和 Build 是两个阶段。报错时首先判断卡在哪个阶段。

## 3. 为什么需要交叉编译？

```text
Host：x86-64 Linux
         │ arm-none-eabi-gcc
         ▼
Target：ARM Cortex-M4 / bare metal
```

宿主机 `/usr/bin/gcc` 默认生成给本机架构运行的程序，STM32 无法执行。

`arm-none-eabi` 可先实用地理解为：面向 ARM 裸机嵌入式目标的一套 GNU 工具。

## 4. Toolchain file 解决什么？

它提前告诉 CMake：

```text
目标系统不是桌面 Linux，而是 Generic/bare metal
C/C++/ASM 用哪套交叉编译器
objcopy/objdump/size/gdb 等工具来自哪里
配置阶段不要按宿主程序的方式验证运行结果
```

### 一个关键排错信号

如果构建日志突然出现：

```text
/usr/bin/cc
```

而不是：

```text
arm-none-eabi-gcc
```

优先检查 preset、toolchain file 和旧 build cache，而不是去查 GPIO 或 linker script。

## 5. Preset 的价值

Preset 把稳定配置固化，例如：

```text
Generator = Ninja
binaryDir = build/
toolchainFile = cmake/...
```

它解决“大家怎样进入同一种构建配置”，减少每次手写长命令导致的差异。

## 6. 根 `CMakeLists.txt`

典型职责：

```cmake
project(... C CXX ASM)
set(CMAKE_C_STANDARD 11)
add_subdirectory(common)
add_subdirectory(platform)
add_subdirectory(experiments/...)
```

`add_subdirectory()` 只是进入子目录执行它的 `CMakeLists.txt`，并不会自动把该目录所有 `.c` 编进固件。

真正构建对象来自：

```cmake
add_executable(...)
add_library(...)
target_sources(...)
```

## 7. CMake 的核心：Target

现代 CMake 不应只看“文件列表”，而要看 target 关系。

例如：

```text
exp001_gpio_output
        ↓ depends on
platform_stm32f446ze
```

Platform target 可以传播：

```text
-DSTM32F446xx
include directories
-mcpu/-mthumb/FPU flags
startup/system sources
link options / linker script
```

## 8. INTERFACE target

概念示意：

```cmake
add_library(platform_stm32f446ze INTERFACE)
```

INTERFACE target 可以自己不产生 `.a`，主要传播“使用该平台的人必须具备的要求”。

### PRIVATE / PUBLIC / INTERFACE

实用记法：

```text
PRIVATE   → 当前 target 自己需要
PUBLIC    → 自己需要，使用者也需要
INTERFACE → 自己不编译使用，但使用者需要
```

## 9. Compile 和 Link 需要不同信息

### Compile 常见

```text
-DSTM32F446xx
-I...
-mcpu=cortex-m4
-mthumb
-mfpu=...
-mfloat-abi=...
```

### Link 常见

```text
-T xxx.ld
--gc-sections
各个 .o / library
```

所以“C 文件编译成功”不代表最终一定能链接成功。

## 10. `.c → .o → .elf`

```text
main.c              → main.c.o
system_stm32f4xx.c  → system...o
startup...s          → startup...o

所有 .o
    ↓ linker
firmware.elf
```

`.o` 是可重定位目标文件，最终 MCU 地址仍待链接阶段确定。

## 11. 从真实命令反查 CMake 来源

```bash
cmake --build build --target exp001_gpio_output -v
```

挑一条 `main.c` 编译命令，逐项追：

```text
-DSTM32F446xx ← 哪个 target_compile_definitions？
-I...         ← 哪个 target_include_directories？
-mcpu=...     ← 哪个 target_compile_options？
```

再看链接命令：

```text
-Txxx.ld      ← 哪个 target_link_options？
```

这比只读 CMake 文件更容易建立“配置 → 最终命令”的因果链。

## 12. GCC driver 不等于 linker

最终可能看到：

```bash
arm-none-eabi-gcc ... -o firmware.elf
```

这里 GCC 充当 driver，会调用 assembler、linker 和相应 libraries；并不表示 `gcc` 这个名字本身就是 GNU linker。

## 13. 为什么启用 ASM？

Startup 是汇编文件，因此即使当前实验业务代码只写 C，工程通常仍需启用：

```text
C CXX ASM
```

否则 startup 可能不能按预期参与构建。

## 14. 分层诊断表

| 失败位置 | 优先检查 |
|---|---|
| Configure | CMake 语法、路径、preset、toolchain、submodule |
| C Compile | 头文件、宏、include、C 语法、CPU flags |
| ASM | ASM 工具、CPU 架构、startup 语法 |
| Link | undefined/multiple definition、`.ld`、region overflow、库 |
| ELF 正常但板上不跑 | 向量表、startup、烧录、调试、真实硬件 |

## 15. 动手检查

在工程中运行：

```bash
cmake --preset stm32f446ze
cmake --build build --target exp001_gpio_output -v
```

然后回答：

1. C 编译器完整路径是什么？
2. `-DSTM32F446xx` 是否出现？
3. Cortex-M4F 参数是否出现？
4. startup 是否被汇编？
5. 最终链接命令是否真的使用 `.ld`？
6. ELF 最终生成在哪里？

## 16. 离开本页前

能闭卷讲：

```text
Preset → Toolchain → CMake targets → Ninja
→ ARM GNU compile/assemble → .o → link → ELF
```

并解释为什么看到 `/usr/bin/cc` 时应该先查 Toolchain，而不是查 GPIO。

下一步：[[04_platform_stm32f446ze 平台层]]。
