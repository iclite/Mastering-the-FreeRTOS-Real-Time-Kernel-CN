# FreeRTOS 的发行

## 引言与范围

FreeRTOS 作为单个 zip 文件存档分发，包含所有官方 FreeRTOS 移植和大量预配置的演示应用程序。

### 范围

本章旨在帮助用户通过以下方式使用 FreeRTOS 文件和目录来定位自己：

* 提供 FreeRTOS 目录结构的顶级视图。
* 描述任何特定 FreeRTOS 项目实际需要哪些文件。
* 介绍演示应用程序。
* 提供有关如何创建新项目的信息。

此处的描述仅涉及官方 FreeRTOS 的发行版。 本书附带的示例使用略有不同的组织。

## 了解 FreeRTOS 的分发

### 定义：FreeRTOS 移植

FreeRTOS 可以使用大约 20 种不同的编译器构建，并且可以运行在 30 多种不同的处理器架构上。 每个受支持的编译器和处理器组合都被视为一个单独的 FreeRTOS 移植。

### 编译 FreeRTOS

FreeRTOS 可以被认为是一个库，它可以提供多任务功能，而不是裸机应用程序。

FreeRTOS 是以一组 C 源文件提供给用户。 某些源文件对所有移植都是通用的，而某些源文件则特定于移植。 构建源文件作为项目的一部分，以使 FreeRTOS API 可用于您的应用程序。 为了方便您使用，每个官方 FreeRTOS 移植都配有演示应用程序。 演示应用程序已预先配置为构建正确的源文件，并包含正确的头文件。

演示应用程序应该 “开箱即用”，虽然有些演示比其他演示更旧，并且自演示程序发布以来构建工具的自身的更新可能会引起报错。 第 1.3 节描述了演示应用程序。

### FreeRTOSConfig.h

FreeRTOS 由名为 `FreeRTOSConfig.h` 的头文件配置。

`FreeRTOSConfig.h` 用于定制 FreeRTOS 以用于特定应用程序。 例如，`FreeRTOSConfig.h` 包含常量，如 `configUSE_PREEMPTION`，它的设置定义了将使用协作调度算法还是抢占调度算法。 由于 `FreeRTOSConfig.h` 包含特定于应用程序的定义，因此它应位于正在构建的应用程序部分的目录中，而不是位于包含 FreeRTOS 源代码的目录中。

为每个 FreeRTOS 移植提供演示应用程序，每个演示应用程序包含一个 `FreeRTOSConfig.h` 文件。 因此，永远不必从头开始创建 `FreeRTOSConfig.h` 文件。 相反，建议从 FreeRTOS 移植提供的演示应用程序中的 `FreeRTOSConfig.h` 开始，然后根据需求修改它。

### FreeRTOS 的官方发行版

FreeRTOS 发布在一个 zip 文件中。 该 zip 文件包含所有 FreeRTOS 移植的源代码，以及所有 FreeRTOS 演示应用程序的项目文件。 它还包含一系列 FreeRTOS+ 生态系统组件，以及一系列 FreeRTOS+ 生态系统演示应用程序。

不要被 FreeRTOS 发行版中的文件数量吓到！ 任何一个应用程序中只需要非常少量的文件。

### FreeRTOS 发行版的顶级目录

图 1 显示并描述了FreeRTOS发行版的第一级和第二级目录。

```text
FreeRTOS
 │  │
 │  ├─Source  包含 FreeRTOS 源文件的目录
 │  │
 │  └─Demo    包含预配置和移植特定 FreeRTOS 演示项目的目录
 │
FreeRTOS-Plus      
    │
    ├─Source  包含一些 FreeRTOS+ 生态系统组件的源代码的目录
    │
    └─Demo    包含 FreeRTOS+ 生态系统组件的演示项目的目录
```

图1. FreeRTOS 发行版中的顶级目录

zip 文件只包含 FreeRTOS 源文件的一个副本; 所有 FreeRTOS 演示项目和所有 FreeRTOS+ 演示项目都希望在 `FreeRTOS/Source` 目录中找到 FreeRTOS 源文件，如果目录结构发生变化，可能无法构建。

### 所有移植共有的 FreeRTOS 源文件

核心 FreeRTOS 源代码仅包含在所有 FreeRTOS 移植共有的两个 C 文件中。 这些被称为 `tasks.c` 和 `list.c`，它们直接位于 `FreeRTOS/Source` 目录中，如图 2 所示。除了这两个文件之外，以下源文件位于同一目录中：

* **queue.c**：`queue.c` 提供队列和信号量服务，如本书后面所述。 几乎总是需要 `queue.c`。
* **timers.c**：`timers.c` 提供软件计时器功能，如本书后面所述。 如果实际中将使用软件定时器，它只需要包含在构建中。
* **event\_groups.c**：`event_groups.c` 提供事件组功能，如本书后面所述。 如果实际要使用事件组，它只需要包含在构建中。
* **croutine.c**：`croutine.c` 实现了 FreeRTOS 协同例程功能。 如果实际要使用协同例程，它只需要包含在构建中。协同程序旨在用于非常小的微控制器，现在很少使用，因此不能保持与其他 FreeRTOS 功能相同的水平。本例中没有描述协同例程。

```text
FreeRTOS
    │
    └─Source
        │
        ├─tasks.c        FreeRTOS 源文件 -总是需要
        ├─list.c         FreeRTOS 源文件 -总是需要
        ├─queue.c        FreeRTOS 源文件 -几乎总是需要
        ├─timers.c       FreeRTOS 源文件 -可选
        ├─event_groups.c FreeRTOS 源文件 -可选
        └─croutine.c     FreeRTOS 源文件 -可选
```

图2. FreeRTOS 目录树中的核心 FreeRTOS 源文件

有的人会意识到文件名可能导致命名空间冲突，因为许多项目已经包含具有相同名称的文件。 然而，我们认为现在更改文件的名称会有问题，这样做会破坏与使用 FreeRTOS 的数千个项目以及自动化工具和 IDE 插件的兼容性。

### FreeRTOS 特定移植的源文件

特定 FreeRTOS 移植的源文件包含在 `FreeRTOS/Source/portable` 目录中。 可移植目录按层次结构排列，首先是编译器，然后是处理器体系结构。 层次结构如图 3 所示。

如果你在使用编译器 \[_compiler\]_ 的架构 \[_architecture\]_ 的处理器上运行 FreeRTOS，除了核心 FreeRTOS 源文件之外，你还必须构建位于 `FreeRTOS/Source/portable/[compiler]/[architecture]` 目录中的文件。

正如将在第 2 章堆内存管理中所描述的，FreeRTOS 还认为堆内存分配是便携式层的一部分。 使用早于V9.0.0 的 FreeRTOS 版本的项目必须包含堆内存管理器。 从 FreeRTOS V9.0.0 开始，只有在 `FreeRTOSConfig.h` 中将 `configSUPPORT_DYNAMIC_ALLOCATION` 设置为 1 或者未定义 `configSUPPORT_DYNAMIC_ALLOCATION` 时才需要堆内存管理器。

FreeRTOS 提供了五个示例堆分配方案。 这五个方案名为 heap\_1 到 heap\_5，分别由源文件 `heap_1.c` 到`heap_5.c` 实现。 示例堆分配方案包含在 `FreeRTOS/Source/portable/MemMang` 目录中。 如果已将 FreeRTOS 配置为使用动态内存分配，则必须在项目中构建这五个源文件中的一个，除非您的应用程序提供了替代实现。

```text
FreeRTOS 
  │
  └─Source
     │
     └─portable 包含所有特定移植源文件的目录
        │   
        ├─MemMang 包含 5 个备用堆分配源文件的目录
        │   
        ├─[compiler 1] 包含特定移植编译器 1 文件的目录
        │   │
        │   ├─[architecture 1] 包含移植编译器 1 架构 1 的文件
        │   ├─[architecture 2] 包含移植编译器 1 架构 2 的文件
        │   └─[architecture 3] 包含移植编译器 1 架构 3 的文件
        │
        └─[compiler 2] 包含特定移植编译器 2 文件的目录
            │
            ├─[architecture 1] 包含移植编译器 2 架构 1 的文件
            ├─[architecture 2] 包含移植编译器 2 架构 2 的文件
            └─[etc.]
```

图3. FreeRTOS 目录树中的特定移植源文件

### 包含路径

FreeRTOS 要求在编译器的 include 路径中包含三个目录。 如下：

1. FreeRTOS 核心头文件的路径，它始终是 `FreeRTOS / Source / include`。
2. 正在使用特定移植的 FreeRTOS 的源文件的路径。 如上所述，这是 `FreeRTOS/Source/portable/[compiler]/[architecture]` 
3. `FreeRTOSConfig.h` 头文件的路径。

### 头文件

使用 FreeRTOS API 的源文件必须包含 `FreeRTOS.h`，然后是包含正在使用的 API 函数原型的头文件：`task.h`，`queue.h`，`semphr.h`，`timers.h` 或 `event_groups.h`。

## 演示应用程序

每个 FreeRTOS 移植都附带至少一个可以构建的演示应用程序，不会生成任何错误或警告，尽管某些演示比其他演示更旧，并且自演示程序发布以来构建工具的自身的更新可能会引起报错。

Linux _\*\*_用户须知：FreeRTOS 是在 Windows 主机上开发和测试的。 在 Linux 主机上构建演示项目时，偶尔会导致构建错误。 构建错误几乎总是与引用文件名时使用的字母大小写或文件路径中使用的斜杠字符的方向有关。 请使用 FreeRTOS 联系表（[http://www.FreeRTOS.org/contact](http://www.FreeRTOS.org/contact)）提醒我们任何此类错误。

演示应用程序有几个目的：

* 提供一个工作和预配置项目的示例，包含正确的文件，并设置正确的编译器选项。
* 允许以最小的设置或先验知识进行 “开箱即用” 的实验。
* 作为如何可以使用 FreeRTOS API 的演示。
* 作为可以创建真实应用程序的基础。

每个演示项目都位于 `FreeRTOS/Demo` 目录下的唯一子目录中。 子目录的名称表示演示项目所涉及的移植。

每个演示应用程序也由 FreeRTOS.org 网站上的网页描述。网页包含以下信息：

* 如何在 FreeRTOS 目录结构中找到演示的项目文件。
* 项目配置使用的硬件。
* 如何设置运行演示的硬件。
* 如何构建演示。
* 如何演示预期行为。

所有演示项目都创建了常见演示任务的子集，其实现包含在 `FreeRTOS/Demo/Common/Minimal` 目录中。 常见的演示任务纯粹是为了演示如何使用 FreeRTOS API —— 它们没有实现任何特定的有用功能。

最近的演示项目也可以构建一个初学者的 'blinky' 项目。 Blinky 项目是非常基本的。 通常，他们只创建两个任务和一个队列。

每个演示项目都包含一个名为 `main.c` 的文件。 它包含 `main()` 函数，从中创建所有演示应用程序任务。 有关该演示的特定信息，请参阅各个 `main.c` 文件中的注释。

`FreeRTOS/Demo` 目录层次结构如图 4 所示。

```text
FreeRTOS
    │
    └─Demo        包含所有演示项目的目录
       │
       ├─[Demo x] 包含构建演示'x'的项目文件
       │
       ├─[Demo y] 包含构建演示'y'的项目文件
       │
       ├─[Demo z] 包含构建演示'z'的项目文件
       │
       └─Common   包含由所有演示应用程序构建的文件
```

图 4. 演示目录层次结构

## 创建一个 FreeRTOS 项目

### 适配其中一个提供的演示项目

每个 FreeRTOS 移植都附带至少一个预配置的演示应用程序，该应用程序应该构建时不会出现错误或警告。 建议通过适配其中一个现有项目来创建新项目; 这将允许项目包含正确的文件，安装正确的中断处理程序以及正确的编译器选项集。

要从现有演示项目开始新的应用程序：

1. 打开提供的演示项目，确保按预期构建和执行。 
2. 删除定义演示任务的源文件。 可以从项目中删除位于 `Demo/Common` 目录中的任何文件。
3. 除了 `prvSetupHardware()` 和 `vTaskStartScheduler()` 之外，删除 `main()` 中的所有函数调用，如清单1.4 所示。
4. 检查项目是否仍可构建。

按照这些步骤创建一个包含正确 FreeRTOS 源文件的项目，但不定义任何功能。

```c
int main( void )
{
    /* 执行必要的任何硬件设置。 */
    prvSetupHardware();

    /* ---可以在这里创建应用程序任务 ---*/
    /* 启动已创建的任务。 */
    vTaskStartScheduler();

    /* 只有当没有足够的堆来启动调度程序时，执行才会到达这里。 */
    for( ;; );
    return 0;
}
```

清单 1. 新 `main()` 函数的模板

### 从头开始创建一个新项目

如前所述，建议从现有的演示项目创建新项目。 如果不希望这样，则可以使用以下过程创建新项目：

1. 使用您选择的工具链，创建一个尚未包含任何 FreeRTOS 源文件的新项目。
2. 确保可以构建新项目，下载到目标硬件并执行。
3. 只有当您确定已经有一个工作项目，将表 1 中详述的 FreeRTOS 源文件添加到项目中。
4. 将为移植提供的演示项目使用的 `FreeRTOSConfig.h` 头文件复制到项目目录中。
5. 将以下目录添加到路径中，项目将搜索以查找头文件：
   1. `FreeRTOS/Source/include`
   2. `FreeRTOS/Source/portable/[compiler]/[architecture]` 其中 `[compiler]` 和 `[architecture]` 适合您选择的移植
   3. 包含 `FreeRTOSConfig.h` 头文件的目录
6. 从相关演示项目中复制编译器设置。
7. 安装可能需要的任何 FreeRTOS 中断处理程序。 使用描述正在使用移植的网页以及为使用移植提供的演示项目作为参考。

表 1. 要包含在项目中的 FreeRTOS 源文件

| 文件 | 位置 |
| :--- | :--- |
| tasks.c | FreeRTOS/Source |
| queue.c | FreeRTOS/Source |
| list.c | FreeRTOS/Source |
| timers.c | FreeRTOS/Source |
| event\_groups.c | FreeRTOS/Source |
| 所有 C 和汇编文件 | FreeRTOS/Source/portable/\[compiler\]/\[architecture\] |
| heap\_n.c | FreeRTOS/Source/portable/MemMang，n 可以是 1，2，3，4，5。这个文件在 FreeRTOS V9.0.0 中是可选的。 |

使用早于 V9.0.0 的 FreeRTOS 版本的项目必须构建一个 `heap_n.c` 文件。从 FreeRTOS V9.0.0 开始，只有在 `FreeRTOSConfig.h` 中将 `configSUPPORT_DYNAMIC_ALLOCATION` 设置为 1 或未定义 `configSUPPORT_DYNAMIC_ALLOCATION`时才需要 `heap_n.c` 文件。 有关更多信息，请参阅第 2 章堆内存管理。

## 数据类型和编码风格指南

### 数据类型

FreeRTOS 的每个端口都有一个唯一的 `portmacro.h` 头文件，其中包含两个端口特定数据类型的定义（其中包括）：`TickType_t` 和 `BaseType_t`。 表 2 中描述了这些数据类型。

表 2. FreeRTOS使用的端口特定数据类型

| 使用的宏或 typedef | 实际类型 |
| :--- | :--- |
| TickType\_t | FreeRTOS 配置一个称为节拍中断的周期性中断。 自 FreeRTOS 应用程序启动以来发生的节拍中断数称为节拍计数。 刻度计数用作时间的度量。 两次滴答中断之间的时间称为滴答期。 时间被指定为滴答期的倍数。 `TickType_t` 是用于保存刻度计数值和指定时间的数据类型。 `TickType_t` 可以是无符号 16 位类型，也可以是无符号 32 位类型，具体取决于`FreeRTOSConfig.h` 中 `configUSE_16_BIT_TICKS` 的设置。如果 `configUSE_16_BIT_TICKS` 设置为 1，则 `TickType_t` 定义为 `uint16_t`。 如果 `configUSE_16_BIT_TICKS` 设置为 0，则 `TickType_t` 定义为 `uint32_t`。使用 16 位类型可以极大地提高 8 位和 16 位体系结构的效率，但严重限制了可以指定的最大块周期。 没有理由在 32 位架构上使用 16 位类型。 |
| BaseType\_t | 这始终被定义为架构的最有效数据类型。 通常，这是 32 位架构上的 32 位类型，16 位架构上的 16 位类型和 8 位架构上的8位类型。 `BaseType_t` 通常用于只能采用非常有限的值范围的返回类型，以及 `pdTRUE`/`pdFALSE` 类型的布尔值。 |

有些编译器会使所有不合格的 `char` 变量无符号，而其他编译器会对它们进行有符号处理。 出于这个原因，FreeRTOS 源代码代码显式限定 `char` 的每个用户都使用 `signed` 或 `unsigned`，除非 char 用于保存 ASCII 字符，或者指向 `char` 的指针用于指向字符串。

不使用普通 int 类型。

### 变量名

变量的前缀是它们的类型：

* `c` 表示 `char`
* `s` 表示 `int16_t`（`short`）
* `l`表示 `int32_t`（`long`）
* `x` 表示 `BaseType_t` 和任何其他非标准类型（结构，任务句柄，队列句柄等）

如果变量是无符号的，则它也带有 `u` 前缀。 如果变量是指针，则它也带有 `p` 前缀。 例如，`uint8_t` 类型的变量将以 `uc` 为前缀，而指向 `char` 的类型指针的变量将以 `pc` 为前缀。

### 函数名

函数以它们返回的类型和它们在其中定义的文件为前缀。 例如：

* _v_**Task**PrioritySet\(\) 返回一个 `void`，并定义在 **task**.c 中。
* _x_**Queue**Receive\(\) 返回一个 `BaseType_t`类型的变量，并定义在 **queue**.c 中。
* _pv_**Timer**GetTimerID\(\) 返回一个 `void`类型的指针，并定义在 **timers**.c 中。

文件范围（私有）函数以 `prv` 为前缀。

### 格式化

1 个 Tab 总是等于 4 个空格。

### 宏名

大多数宏以大写字母书写，并以小写字母为前缀，表示宏的定义位置。 表 3 提供了前缀列表。

表 3. 宏的前缀

| 前缀 | 宏定义的位置 | 例子 |
| :--- | :--- | :--- |
| port | portable.h 或 portmacro.h | **port**MAX\_DELAY |
| task | task.h | **task**ENTER\_CRITICAL\(\) |
| pd | projdefs.h | **pd**TRUE |
| config | FreeRTOSConfig.h | **config**USE\_PREEMPTION |
| err | projdefs.h | **err**QUEUE\_FULL |

请注意，信号量 API 几乎完全是作为一组宏编写的，但遵循函数命名约定，而不是宏命名约定。表 4 中定义的宏在整个 FreeRTOS 源代码中使用。

表 4. 常见的宏定义

| 宏 | 值 |
| :--- | :--- |
| pdTRUE | 1 |
| pdFALSE | 0 |
| pdPASS | 1 |
| pdFAIL | 0 |

### 构造中间过度类型的理由

FreeRTOS 源代码可以使用不同的编译器进行编译，所有编译器的编写方式和生成警告的方式都有所不同。 特别是，不同的编译器希望以不同的方式使用转换。 因此，FreeRTOS 源代码包含比通常保证更多的类型转换。

