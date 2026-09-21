# 理解这个仓库

> 这个 `embedded-lab` 不是一个普通的 “STM32 工程“
> 
> 

普通的 STM32 工程通常是：

```Plain Text
一个产品 / 实验
        ↓
       main.c
       HAL
       startup
       linker
       Drivers
       ...
```

而这个 `embedded-lab` 仓库的目标是：

```Plain Text
长期嵌入式实验平台
       │
       ├── 公共基础设施
       ├── 不同实验
       ├── 不同软件架构
       ├── 学习资料
       └── 最终综合项目
       
未来会有：
        实验001    GPIO
        实验002    EXTI
        实验003    NVIC
        ...
        FreeRTOS 实验
        ...
【它们共享一套底层基础设施】               
```

# 建立整个仓库 “地图“

整个仓库：

```Bash
embedded-lab/
│
├── cmake/
|
├── common/
|
├── docs/
|
├── experiments/
│
├── frameworks/
│
├── platform/
│
├── projects/
│
├── scripts/
│
├── third_party/
|
├── .gitignore                # 告诉Git忽略哪些文件，不纳入版本控制
├── .gitmodules               # 当前项目依赖另一个Git仓库（Git Submodule/子模块）
├── CMakeLists.txt
├── CMakePresets.json
├── LICENSE
├── README.md
├── ROADMAP.md
└── THIRD_PARTY_NOTICES.md
```

这样看很容易觉得 “十几个文件夹“ 没什么头绪

所以我们需要记成下面这个结构（整个仓库的骨架思想）：

```Plain Text
				  embedded-lab
                      │
      ┌───────────────┼────────────────┐
      │               │                │
    docs/          构建系统          代码体系
  学习资料           CMake               │
                                        │
                        ┌───────────────┼──────────────┐
                        │               │              │
                 third_party       platform        common
                  官方代码          板级层          通用层
                        │               │              │
                        └───────┬───────┴──────┬───────┘
                                │              │
                           frameworks       experiments
                            架构层            实验层
                                               │
                                            projects
                                             综合项目
```

# 根目录

> 相当于整个仓库的 “控制中心“
> 
> 

主要回答了：

```Plain Text
这个仓库是干什么的？       → README.md
我要按什么顺序学习？       → ROADMAP.md
整个工程怎么构建？         → CMakeLists.txt
怎么启动一次构建？         → CMakePresets.json
交叉编译器是什么？         → cmake/
自己的代码是什么许可？     → LICENSE
第三方代码是什么许可？     → THIRD_PARTY_NOTICES.md
什么东西不能提交 Git？     → .gitignore
该项目依赖的另一个仓库？    → .gitmodules
```

所以根目录实际上可以理解为：

> 整个实验平台的总指挥部。
> 
> 

## CMakeLists\.txt \-\-\- 整个代码世界的总入口

> 具体 CMake 可以看文档 《CMake》
> 
> ---
> 
> 

```CMake
cmake_minimum_required(VERSION 3.20)

project(embedded-lab C CXX ASM)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

add_subdirectory(common)
add_subdirectory(platform)

set(EXPERIMENTS
  # 001_gpio_output
  # 002_gpio_input
)

foreach(exp IN LISTS EXPERIMENTS)
  add_subdirectory(experiments/${exp})
endforeach()
```

（1）`cmake_minimum_required(VERSION 3.20)`

> **CMake** 版本至少是 *3\.20* 
> 
> 

（2）`project(embedded-lab C CXX ASM)`

> 第一个参数是 工程名字 ，后面生成的构建系统都会用到这个名字
> 
> 后面的参数 是指定的语言。表示 **C CXX ASM** 都是这个工程会用到的变成语言
> 
> - **C**     ：C语言源文件（`.C`）\-\-\-\> *主力开发语言*
> 
> - **CXX** ：C\+\+源文件（`.cpp、.cc`）\-\-\-\> *更复杂的抽象或框架*
> 
> - **ASM** ：汇编语言源文件（`.s`）\-\-\-\> *启动代码、中断向量表、或性能关键的底层实现*
> 
> ---
> 
> CMake 会根据这些语言，自动启用对应的编译器
> 
> 以便后续可以混合使用这些语言
> 
> 

（3）

```CMake
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
```

> 这些配置用来 精确控制CMake 编译 C语言 的标准和行为
> 
> - `set(CMAKE_C_STANDARD 11)` \-\-\-\> 指定 编译器 使用 C11标准 来编译 C源文件
> 
> - `set(CMAKE_C_STANDARD_REQUIRED ON)` \-\-\-\> 如果使用 C11 有问题就报错，而不是退回旧标准
> 
> - `set(CMAKE_C_EXTENSIONS OFF)` \-\-\-\> 不许使用编译器的扩展，使用 *纯标准 C11* 
> 
> ---
> 
> 1. 首先 **C11标准 **可以理解为是 C90、C99 的升级版*（增加了功能，更稳定）* 
> 
> 2. 所以我们一般直接使用 **C11** 就行，但是对于老项目来说可能会有两个问题：
> 
>     - **老项目使用了 C11 为了安全性移除的函数，如 ****`gets()`** 
> 
>         那么我们有两个解决方法：
> 
>         （1）替换掉 `gets()`使用更安全的 `fgets(buffer, size, stdin)`
> 
>         （2）如果必须保留，在 CMake 里面指定退回 旧版本C99
> 
>     - **老项目使用的 旧编译器 不支持 C11** 
> 
>         也有两个解决方法：
> 
>         （1）更新编译器
> 
>         （2）如果必须使用指定编译器，在 CMake 里面指定退回 旧版本C99
> 
> 

（4）

```CMake
add_subdirectory(common)
add_subdirectory(platform)
```

> `add_subdirectory(<dir>)`
> 
> 告诉 CMake 去指定的 目录 里面寻找 `CMakeLists.txt`
> 
> 如果找到，就会把这个 目录 当做一个 *子工程* 来处理
> 
> 该目录里面的 *目标（库、可执行文件等）* 会被加入到当前工程的构建树中：
> 
> ```CMake
> 根 CMakeLists.txt
> │
> ├── common/CMakeLists.txt
> │
> └── platform/CMakeLists.txt
>         │
>         └── stm32f446ze/CMakeLists.txt
> ```
> 
> 

---

> 好处：
> 
> - **模块化管理**：把不同功能拆分到不同目录（例如 `common` 放公共库，`platform` 放平台相关代码）。
> 
> - **层次化构建**：每个子目录有自己的 `CMakeLists.txt`，可以独立定义库、依赖、编译选项。
> 
> - **复用性**：`common` 目录可能编译成一个静态库/动态库，供主工程和其他模块使用。
> 
> 

（5）

```CMake
set(EXPERIMENTS
  001_gpio_output
  002_gpio_input
)

foreach(exp IN LISTS EXPERIMENTS)
  add_subdirectory(experiments/${exp})
endforeach()
```

> 1\. `set(EXPERIMENTS ...)`
> 
> - 定义一个变量 `EXPERIMENTS`，里面存放了多个子目录的名字：
> 
>     - `001_gpio_output`
> 
>     - `002_gpio_input`
> 
> 这相当于一个 **列表**。
> 
> ---
> 
> 2\. `foreach(exp IN LISTS EXPERIMENTS)`
> 
> - 循环遍历 `EXPERIMENTS` **列表 **里的每一项。
> 
> - 每次循环，变量 `exp` 就会取到一个值，比如第一次是 `001_gpio_output`，第二次是 `002_gpio_input`。
> 
> ---
> 
> 3\. `add_subdirectory(experiments/${exp})`
> 
> - 在循环体里，把 `experiments/${exp}` 作为子目录加入构建。
> 
> - `${exp}` 会被替换成当前循环的值：
> 
>     - 第一次：`add_subdirectory(experiments/001_gpio_output)`
> 
>     - 第二次：`add_subdirectory(experiments/002_gpio_input)`
> 
> ---
> 
> 整体效果
> 
> - 相当于自动批量调用多个 `add_subdirectory()`，而不是一行一行手写。
> 
> - 这样做的好处是：如果以后要增加实验，只需要在 `EXPERIMENTS` 列表里加名字，不用改循环逻辑。
> 
> ---
> 
> **基本概念** ：
> 
> `${...}` 表示取变量的值
> 
> 我们之前遇到的：`${CMAKE_CURRENT_SOURCE_DIR}` 是CMake 的一个内置变量
> 
> ---
> 
> 之后整个 **CMake树** （整个工程的构建骨架）：
> 
> ```CMake
> CMakeLists.txt
> │
> ├── common/
> │   └── CMakeLists.txt
> │
> ├── platform/
> │   └── CMakeLists.txt
> │       │
> │       └── stm32f446ze/
> │           └── CMakeLists.txt
> │
> └── experiments/
>     ├── 001_gpio_output/
>     │   └── CMakeLists.txt
>     │
>     ├── 002_gpio_input/
>     │   └── CMakeLists.txt
>     │
>     └── ...
> ```
> 
> 

## CMakePresets\.json \-\-\- CMake 的 启动配置

> 简单但十分有用：
> 
> - **减少命令行复杂度**：不用记一堆 `-D` 参数。
> 
> - **支持多配置**：可以写多个* preset*，比如 `debug`、`release`、`cross-arm`，然后切换很方便。
> 
> - 传参数给 **CMakeLists\.txt** 
> 
> 

```Bash
{
  "version": 3,
  "configurePresets": [
    {
      "name": "stm32f446ze",
      "displayName": "NUCLEO-F446ZE (arm-none-eabi)",
      "description": "交叉编译 preset：VS Code CMake Tools 与命令行共用同一套构建逻辑",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build",
      "toolchainFile": "${sourceDir}/cmake/arm-none-eabi.cmake"
    }
  ]
}
```

> 真正影响构建行为的是这些字段：
> 
> - `name`：preset 的唯一标识符，命令行里用 `cmake --preset stm32f446ze` 就是靠这个名字。
> 
> - `generator`：指定构建系统，比如 Ninja、Makefiles 等。
> 
> - `binaryDir`：构建输出目录，告诉 CMake 把生成的文件放到哪里。
> 
> - `toolchainFile`：交叉编译时的工具链文件路径，决定用哪个编译器和编译规则。
> 
> 这些是“硬参数”，直接决定 CMake 怎么跑。
> 
> ---
> 
> `displayName`、`description`
> 
> 可以简单理解为：说明性的文字输出，便于人观察
> 
> ---
> 
> 使用时直接 ：`cmake --preset stm32f446ze`
> 
> CMake 就知道 你再 CMakePresets\.json 里面的配置了
> 
> 

---

有了 **CMakePresets\.json** 我们可以创建多个 `preset` 切换使用：

```Bash
{
  "version": 3,
  "configurePresets": [
    {
      "name": "stm32f446ze",
      "displayName": "NUCLEO-F446ZE (arm-none-eabi)",
      "description": "交叉编译 preset：VS Code CMake Tools 与命令行共用同一套构建逻辑",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build",
      "toolchainFile": "${sourceDir}/cmake/arm-none-eabi.cmake"
    },
    {
      "name": "release",
      ...
    },
    {
      "name": "release",
      ...
    },
  ]
}
```

## cmake/arm\-none\-eabi\.cmake \-\-\- 告诉 CMake：“我不是在编电脑程序”

> - **Toolchain\.cmake** 是 CMake 交叉编译配置文件的通用泛称（或推荐命名规则）
> 
> - **arm\-none\-eabi\.cmake** 则是针对 ARM Bare\-metal（无操作系统）交叉编译器（`arm-none-eabi-gcc`）的一个**具体实现文件**。
> 
> ---
> 
> ```CMake
> set(CMAKE_SYSTEM_NAME Generic)      # 裸机，无 OS
> set(CMAKE_SYSTEM_PROCESSOR arm)
> ```
> 
> - `CMAKE_SYSTEM_NAME Generic`：
> 
>     告诉 CMake 正在进行交叉编译，目标环境是一个无操作系统（Bare\-metal）的裸机环境。CMake 会据此自动屏蔽诸如 Linux/Windows 平台专属的系统库链接（如 `-lpthread`、`-ldl`）。
> 
> - `CMAKE_SYSTEM_PROCESSOR arm`：
> 
>     指定目标 CPU 的架构类别为 `arm`。
> 
> ---
> 
> ```CMake
> set(TOOLCHAIN_PREFIX arm-none-eabi-)
> 
> # 编译/汇编共用 GCC 前端
> set(CMAKE_C_COMPILER   ${TOOLCHAIN_PREFIX}gcc)
> set(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}g++)
> set(CMAKE_ASM_COMPILER ${TOOLCHAIN_PREFIX}gcc)
> ```
> 
> - `TOOLCHAIN_PREFIX`：
> 
>     定义前缀变量 `arm-none-eabi-`（其中 `none` 表示无 OS，`eabi` 表示嵌入式应用二进制接口）。
> 
> - `CMAKE_C_COMPILER` / `CMAKE_CXX_COMPILER`：
> 
>     将 C 和 C\+\+ 编译器分别绑定到 `arm-none-eabi-gcc` 和 `arm-none-eabi-g++`。
> 
> - `CMAKE_ASM_COMPILER`：
> 
>     特意指定为 `arm-none-eabi-gcc` 而非 `arm-none-eabi-as`。这是嵌入式开发的最佳实践，因为通过 GCC 前端来处理 `.S` 汇编文件，可以支持 C 语言预处理器指令（如 `#include`、`#define`、`#ifdef`）。
> 
> ---
> 
> ```CMake
> # 二进制工具
> set(CMAKE_OBJCOPY ${TOOLCHAIN_PREFIX}objcopy CACHE FILEPATH "objcopy")
> set(CMAKE_OBJDUMP ${TOOLCHAIN_PREFIX}objdump CACHE FILEPATH "objdump")
> set(CMAKE_SIZE    ${TOOLCHAIN_PREFIX}size    CACHE FILEPATH "size")
> set(CMAKE_GDB     ${TOOLCHAIN_PREFIX}gdb     CACHE FILEPATH "gdb")
> ```
> 
> 这部分指定了嵌入式开发常用的 处理和调试 工具：
> 
> - **`objcopy`**：用于将编译出的 `.elf` 文件转换为烧录所需的 `.bin` 或 `.hex` 格式。
> 
> - **`objdump`**：用于反汇编和分析目标文件。
> 
> - **`size`**：用于输出程序占用的 Flash（text/data）和 RAM（bss/data）大小。
> 
> - **`gdb`**：在线调试器。
> 
> **`CACHE FILEPATH "..."`**：将这些变量写入 CMake 的缓存（`CMakeCache.txt`），方便其他自定义构建命令或外部脚本在需要时直接引用这些工具
> 
> ---
> 
> ```CMake
> # 交叉编译时禁止运行探测程序（目标机跑不了 host 程序）
> set(CMAKE_TRY_COMPILE_TARGET_TYPE STATIC_LIBRARY)
> ```
> 
> - 默认情况下，CMake 在 `project()` 初始化阶段会尝试**编译并链接一个简单的测试程序**，以验证编译器是否正常工作。
> 
> - 但在交叉编译场景下，目标机（比如 `Cortex‑M MCU`）根本不能在宿主机上运行
> 
> - 所以这个的作用就是：
> 
>     编译一个 `.a` 静态库，只验证编译器和链接器是否能正常工作，不运行。
> 
> ---
> 
> ```CMake
> # 只在目标系统根目录里找库/头文件；host 工具（如 cmake 自身）不受影响
> set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER) 
> set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY) 
> set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY) 
> set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
> ```
> 
> 这四行是交叉编译的安全隔离防护罩
> 
> 严格区分了“开发主机（Host）”和“目标芯片（Target）”的搜索资源
> 
> 

## 构建启动链

```CMake
cmake --preset stm32f446ze
        │
        ▼
CMakePresets.json
        │
        ├── generator = Ninja
        ├── build = build/
        │
        └── toolchain =
            cmake/arm-none-eabi.cmake
                    │
                    ▼
             arm-none-eabi-gcc
                    │
                    ▼
              CMakeLists.txt
                    │
           ┌────────┴─────────┐
           ▼                  ▼
        common             platform
                              │
                              ▼
                        stm32f446ze
                              │
                              ▼
                        experiments
```

然后：

`ninja -C build exp001_gpio_output`

才是真正指定：

> 构建实验 001。
> 
> 

# third\_party/ \-\-\- "别人的代码，只从这进"

> 这是整个代码分层的第一层
> 
> （只管引进，集中管理）
> 
> 

```CMake
third_party/
└── STM32CubeF4/
```

以后 ST 官方的：

> CMSIS
> 
> HAL
> 
> LL
> 
> startup
> 
> FreeRTOS
> 
> 

来源都统一来自：`STM32CubeF4`

### **STM32CubeF4 的 *****Git submodule / tag 管理**** *

> *Submodule* 与 *Tag* 结合使用，做第三方库的 **版本管理**
> 
> - *Submodule* ：解决 ”依赖怎么来” 的问题（轻量引入，不破坏自身仓库的干净度）
> 
> - *Tag* ：解决 “依赖怎么控制“ 的问题（死锁版本，保证工程 100% 可复现）
> 
> 

如果不这样做的两个痛点：

> 1. 直接把官方代码解压添加到项目：
> 
>     - Git 仓库体积极速膨胀
> 
>     - 如果我们顺手改了代码，无法区分哪些是官方的、哪些是自己改的
> 
> 2. 如果只保留 官方Github 链接：
> 
>     - 如果官方修改了某个 库函数、改了某些结构体名字，我们就无法复现了
> 
> 

**Submodule** 

> `git submodule add http://github...`
> 
> 其实就是和克隆一样，也是克隆 目标仓库
> 
> 不一样的是 *Submodule* 是将目标仓库克隆到 当前仓库（*clone* 是克隆到当前文件夹）
> 
> 目标仓库仍是独立管理
> 
> 

**Tag** 

> 在嵌入式和硬件底层开发过程中绝对不要追最新版本（追 *master* ）
> 
> 1. *master* 不等于稳定版，是开发者的工作台，充斥着 ””未完成的新特新’
> 
> 2. 如果底层库天天跟着 *master* 变，就很难确定是 硬件问题、应用层问题还是三方库问题
> 
> 3. 经常需要工业级复现，如果你追了 `master`，一年后的编译器和底层库全部变了样，重新编译出来的固件可能导致设备直接瘫痪。
> 
> 

`submodule`拉取官方的 master，切换到官方发布稳定的`tag`上面，然后再添加到我们的仓库里面

---

这些主要都是 *Git* 相关操作，具体怎么做，可以研究一下 *Git* 。或者直接让AI来做

# platform/ \-\-\- "very 重要"

```CMake
platform/
├── CMakeLists.txt
└── stm32f446ze/
    ├── CMakeLists.txt
    └── README.md
```

这里容易和 **third\_party/** 搞混，区分：



```CMake
third_party
    =
ST 给的通用 STM32F4 代码                【STM32F446 芯片】


platform
    =
我们为了“这一块具体开发板”写的适配代码    【NUCLEO 开发板】
```

> `platform/` 就像是为你所有的实验代码搭建了一堵“隔离墙”：
> 
> - 墙的**下面**（硬件、引脚、时钟、芯片寄存器、HAL 库）：全是复杂的脏活累活，由 `platform/` 一次性搞定。
> 
> - 墙的**上面**（`experiments/`）：环境极其纯净。你不用关心任何硬件细节，**只专注验证你想测试的算法、协议或业务逻辑**。
> 
> 

### **具体装什么：**

#### ***clock：***

> 系统时钟配置，比如这里的时钟来自 **ST\-Link 提供的 数字时钟** 
> 
> 没有独立的 **HSE 晶振** 
> 
> ---
> 
> MCU 运行肯定是需要时钟的
> 
> MCU 内部有一个 *时钟树\(Clock Tree\)* ，负责把主时钟分配到各个外设【由RCC模块负责】
> 
> ---
> 
> MCU 的时钟源可来自 
> 
> - **HSI** ：MCU自带的 *RC振荡器* 
> 
> - **HSE** ：接 *外部晶振*  或  *外部时钟信号* 
> 
> 

#### ***startup：**** *

> `startup_stm32f446xx.s`（可能来自 *third\_party/STM32CubeF4* ）
> 
> - **注意：**文件本体属于 *third\_party* 
> 
> - **而：** 这一块板子应该选哪个 startup 文件，属于 **platform** 的责任
> 
> 

#### ***linker script：***

> 链接脚本需要知道：
> 
> ```CMake
> Flash 多大
> RAM 多大
> 地址从哪里开始
> ...
> ------------------------------------------------
> 这都属于具体硬件平台
> ```
> 
> 

#### ***board：***** **

> 以后可能定义：
> 
> ```Plain Text
> LED1 = PB0
> LED2 = PB7
> LED3 = PB14
> 
> USER_BUTTON = PC13
> ```
> 
> 实现：**板级事实集中管理。**
> 
> 

#### ***printf 重定向：***

> 例如：
> 
> ```Plain Text
> printf
>   ↓
> USART3
>   ↓
> PD8 / PD9
>   ↓
> ST-Link Virtual COM Port
>   ↓
> PC
> ```
> 
> 以后日志输出也可以成为统一的平台能力。
> 
> 

### **platform 的重要边界：** 

> 只负责硬件平台，不知道应用用哪个 RTOS。
> 
> ```CMake
> 硬件相关
>         ↓
> platform
> 
> 
> RTOS / framework 相关
>         ↓
> frameworks
> ```
> 
> 

### **platform 的 CMakeLists\.txt：** 

> **层次化组织**
> 
> - `platform/CMakeLists.txt` 本身并不直接定义库或可执行文件，而是继续往下拆分。
> 
> - 它通过 `add_subdirectory(stm32f446ze)` 把具体平台（比如 `stm32f446ze`）纳入构建树。
> 
> 这样做的好处是：
> 
> 可以在 `platform` 目录下同时管理多个平台（stm32f4、stm32h7、linux…），而不是把所有逻辑写在一个文件里。
> 
> ---
> 
> **INTERFACE 库的作用**
> 
> - 在 `platform/stm32f446ze/CMakeLists.txt` 里定义了一个 **INTERFACE library**：
> 
>     ```CMake
>     add_library(platform_stm32f446ze INTERFACE)
>     ```
> 
> - **INTERFACE 库 **不会生成任何目标文件（比如 `.a` 或 `.so`），也不会编译 `.c` 文件。
> 
> - 你可以往这个库里添加 **编译选项**、**宏定义**、**公共头文件路径**，甚至是依赖关系。
> 
> **为什么要这样设计**
> 
> ---
> 
> - **解耦**：主工程只关心“依赖哪个平台”，而不关心具体文件。
> 
> - **可扩展**：以后你要支持别的 MCU，只需再定义一个新的 **INTERFACE 库**，挂上对应的属性。
> 
> - **复用**：公共代码 可以通过 **INTERFACE 库 **统一挂载，而不是每个工程都复制一份。
> 
> 

# common/ \-\-\- 与硬件无关的公共逻辑

> 它 既不认识 MCU，也不认识 RTOS
> 
> - 这就是 *common* 的判断标准
> 
> 比如：
> 
> ```CMake
> ring buffer
> CRC
> 字节序转换
> 数据帧解析
> 通用断言
> 纯逻辑日志格式化
> ```
> 
> 

> 因为这些东西理论上：
> 
> - STM32 能用
> 
> - ESP32 能用
> 
> - Linux 程序甚至也能使用
> 
> 

---

**先写实验，再提炼公共件。**

> 实验里第二次出现同样的代码时才上提。
> 
> 

这条规则能防止一个常见问题：

```Plain Text
实验一个都没写
    ↓
先设计一个超大型 utils/
    ↓
再设计 middleware/
    ↓
再设计 abstraction/
    ↓
半年以后发现根本没用
```

我们的路线是：

```Plain Text
001 先写
002 再写
...
发现某段代码真正重复
        ↓
再抽到 common/
```

这叫：

> **从实际需求中提炼抽象，**而不是提前幻想抽象。
> 
> 

# 以 UART 举例 \-\-\- 来理解：

```CMake
STM32 官方 UART 寄存器/HAL 定义
             ↓
third_party/


NUCLEO-F446ZE USART3 → ST-Link VCP
             ↓
platform/


UART 数据接收 ring buffer
             ↓
common/


我用 UART 做收发实验
             ↓
experiments/
```

# frameworks/ \-\-\- 软件架构

以 *freeRTOS* 举例：

```CMake
frameworks/
└── freertos/
    ├── config/
    │   └── FreeRTOSConfig.h
    │
    └── port/
        └── ...
```

重点 三个边界：

```CMake
common/
    不认识 MCU
    不认识 RTOS


platform/
    认识硬件
    不认识 RTOS


frameworks/
    认识某个 framework / RTOS
```

# experiments/ \-\-\- 真正的主战场

> 第一个实验开始后出现：
> 
> ```CMake
> experiments/
> └── 001_gpio_output/
>     ├── README.md
>     ├── CMakeLists.txt
>     ├── main.c
>     └── artifacts/
> ```
> 
> 

---

这里应该只保留 \-\-\-\> **真正属于 这个实验的东西**

```CMake
main.c
README.md
实验特殊源代码
CMakeLists.txt
实测结果
```

**一个完整的实验：** 

> 包含：*代码 \+ 接线 \+ 故障 \+ 结论 \+ 波形* /数据
> 
> ---
> 
> README 里记录：
> 
> ```Plain Text
> 为什么做
> 怎么接线
> 关注什么寄存器
> 怎么测试
> 怎么故障注入
> 得出了什么结论
> ```
> 
> `artifacts/` 保存：
> 
> ```Plain Text
> 逻辑分析仪截图
> 示波结果
> 串口日志
> 测试数据
> ```
> 
> ---
> 
> 这样未来再回来看就不是 *一个不知道为什么能跑的 main\.c* 
> 
> 而是：
> 
> **一次完整、可复现的实验。**
> 
> 

# frameworks 与 experiments

```CMake
frameworks/freertos
                           │
                           │ 提供框架能力
                           ▼
experiments/xxx_freertos_xxx
                           │
                           │ 应用层
                           ▼
                       main.c
```

也就是说：

> `frameworks/` 提供能力，`experiments/` 使用能力。
> 
> 

# docs/ \-\-\- “知识”与“代码”彻底分开

> 保存学习过程中需要长期查阅的资料与知识笔记。
> 
> 它又分四个技术层次：
> 
> ```Plain Text
> docs/
> ├── board/
> ├── mcu/
> ├── cortex_m/
> └── protocols/
> ```
> 
> 再加：
> 
> ```Plain Text
> planning/
> experiment_template.md
> ```
> 
> ---
> 
> 

未来创建：`experiments/001_gpio_output/`

> 第一步不是先写：`int main(void)`
> 
> 而是先：
> 
> ```Plain Text
> docs/experiment_template.md
>         ↓ 复制
> experiments/001_gpio_output/README.md
> ```
> 
> 模板要求填写：
> 
> ```Plain Text
> 目的
> Level
> 前置依赖
> 硬件连接
> 寄存器 / 外设关注点
> 构建与烧录
> 仪器验证
> 故障注入
> 实验记录
> 关联知识
> ```
> 
> 

### **docs/board/ \-\-\-\> 开发板** 

> 如：
> 
> ```CMake
> docs/board/
> └── NUCLEO-F446ZE/
>     ├── schematic/
>     ├── user_manual/
>     └── notes/
> ```
> 
> 

> 这里保存的是：
> 
> **NUCLEO\-F446ZE 这块具体板子的资料** 
> 
> ```CMake
> 原理图
> 用户手册
> 板上 LED 怎么接
> ST-Link 怎么接
> 跳线帽
> Arduino header
> Morpho connector
> 虚拟串口
> ```
> 
> 

### **docs/mcu/ \-\-\-\> 芯片** 

> 如：
> 
> ```CMake
> docs/mcu/
> └── STM32F446/
>     ├── datasheet/            // 这颗具体芯片有什么、引脚是什么、电气特性是什么。
>     ├── reference_manual/     // STM32F446 外设到底怎么工作、寄存器怎么配置。
>     └── notes/
> ```
> 
> 

> 这里保存的是：
> 
> **STM32F446 芯片资料** 
> 
> ```CMake
> GPIO
> RCC
> USART
> DMA
> ADC
> SPI
> I2C
> Timer
> ```
> 
> 

### **docs/cortex\_m/ \-\-\-\> 内核** 

> 如：
> 
> ```CMake
> docs/cortex_m/
> └── Cortex-M4/
>     ├── generic_user_guide/
>     ├── programming_manual/
>     └── notes/
> ```
> 
> 

> 因为 **STM32F446** 只是采用 **ARM Cortex\-M4** ，作为 CPU 内核：
> 
> ```CMake
> STM32F446
> ┌─────────────────────────────┐
> │                             │
> │       Cortex-M4 core        │
> │                             │
> │   NVIC    SysTick   CPU     │
> │                             │
> ├─────────────────────────────┤
> │ RCC GPIO UART SPI ADC DMA   │
> │         ST 外设              │
> └─────────────────────────────┘
> ```
> 
> 所以说：
> 
> - `NVIC`、`SysTick`、`CPU 寄存器`、`异常模型`、`指令集`
> 
> 这些都是属于内核
> 
> ---
> 
> - `GPIO`、`UART`、`TIM`、`DMA`、`ADC`
> 
> 这些属于某个具体的MCU
> 
> 

**NVIC** 例子：

> 以后做：`004_nvic`
> 
> 需要同时查：
> 
> - **docs/mcu/reference\_manual** 
> 
> - **docs/cortex\_m/Cortex\-M4/** 
> 
> 因为：
> 
> ```CMake
> 某个 USART 中断源怎么产生
>         ↓
> STM32 外设问题
> 
> 
> 中断优先级怎么抢占
> 异常怎么进入 CPU
> NVIC 怎么工作
>         ↓
> Cortex-M 内核问题
> ```
> 
> 

### **docs/protocols/ \-\-\-\> 和某个芯片无关的通信知识** 

> 例如：
> 
> ```CMake
> START          STOP
> ACK            NACK
> address        clock stretching
> 400 kHz
> ```
> 
> 

> 这些不是 *STM32F446 专属知识* ，换芯片 I2C协议 依然存在，所以放在：
> 
> `docs/protocols/i2c/`
> 
> 

# scripts/ \-\-\- 开发流程自动化（脚本）

> 如：
> 
> ```CMake
> scripts/
> ├── flash.sh
> ├── GDB 初始化
> ├── 逻辑分析仪辅助
> └── ...
> ```
> 
> 

> 这些东西：
> 
> - 不是 `firmware` 本身
> 
> - 是实验复现流程的一部分
> 
> ---
> 
> 

# projects/ \-\-\- 综合项目

> 未来项目比如：
> 
> - 环境监测终端
> 
> 里面可能同时有：
> 
> ```Plain Text
> ADC
> DMA
> UART
> SPI Flash
> FreeRTOS
> MQTT
> CLI
> 日志
> 故障恢复
> ```
> 
> 这时候不能再以：
> 
> - “我要证明 I2C 工作” 为中心。
> 
> 而要以：
> 
> - “我要把整个设备做好” 为中心。
> 
> 独立进入：projects/
> 
> 

# “代码依赖链”

```CMake
STMicroelectronics
                  │
                  ▼
        third_party/STM32CubeF4
                  │
                  │ 官方 CMSIS / HAL / LL / startup
                  ▼
       platform/stm32f446ze
                  │
                  │ 时钟 / linker / board / printf
                  │
          ┌───────┴─────────┐
          │                 │
          ▼                 ▼
       common/         frameworks/
   FIFO / CRC /...    FreeRTOS / QP...
          │                 │
          └────────┬────────┘
                   ▼
              experiments/
                   │
              001 / 002 ...
                   │
                   ▼
               projects/
```

# “资料链”

```CMake
docs/board/
    ↓
LED 到底连在哪个引脚？


docs/mcu/
    ↓
GPIO 寄存器怎么工作？


docs/cortex_m/
    ↓
CPU / 内核相关机制是什么？


docs/protocols/
    ↓
如果是 I2C/SPI，再学习协议本身
```

# 总结

> `docs` 告诉我知识从哪里来；
> 
> `third_party` 提供官方代码原材料；
> 
> `platform` 把这些原材料适配到 NUCLEO\-F446ZE；
> 
> `common` 保存真正跨硬件可复用的逻辑；
> 
> `frameworks` 提供不同的软件架构；
> 
> `experiments` 用这些能力逐项验证知识；
> 
> `artifacts` 保存实验依据；
> 
> `projects` 最终把所有能力组合成完整设备；
> 
> CMake 把整个代码体系连接起来。
> 
> 

