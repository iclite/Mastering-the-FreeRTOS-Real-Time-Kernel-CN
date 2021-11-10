# **开发者支持**



## 引言与范围

- 提供对应用程序行为方式的深入了解。
- 突出强调优化的机会。

- 在错误发生的位置上进行捕获。



## configASSERT()

在C语言中，宏 `assert()` 用于验证程序所做的断言（假设）。断言被写成C语言表达式，如果表达式的值为false（0），那么断言就被视为失败。例如，清单163测试指针 `pxMyPointer` 不是 `NULL` 的断言。

```verilog
/* 测试pxMyPointer不为空 */
assert( pxMyPointer != NULL );
```

清单163 使用标准的C语言 `assert() `宏来检查`pxMyPointer`不是 `NULL` 。

应用程序编写者通过提供 `assert() `宏的实现，指定在断言失败时要采取的行动。

FreeRTOS的源代码没有调用 `assert()`，因为 `assert()` 在所有编译FreeRTOS的编译器中都不能使用。相反，FreeRTOS的源代码包含了对一个叫做 `configASSERT()` 的宏的大量调用，这个宏可以由应用程序编写者在`FreeRTOSConfig.h `中定义，其行为与标准C语言的 `assert() `完全一样。

一个失败的断言必须被视为一个致命的错误。不要试图执行超过断言失败的那一行。

使用 `configASSERT() `可以立即捕获和识别许多最常见的错误来源，从而提高生产力。我们强烈建议在开发或调试FreeRTOS应用程序时，定义 `configASSERT() `。

定义 `configASSERT()` 将大大有助于运行时的调试，但也会增加应用程序的代码大小，从而减慢其执行速度。如果没有提供 `configASSERT()` 的定义，那么将使用默认的空定义，所有对` configASSERT()` 的调用将被C预处理器完全删除。



### configASSERT()定义的实例

清单164中显示的 `configASSERT()` 的定义在一个应用程序在调试器的控制下执行时非常有用。它将在任何断言失败的行上停止执行，因此断言失败的行将是调试器在暂停调试会话时显示的行。

```verilog
/* 禁用中断，使tick中断停止执行，然后坐在一个循环中，使执行不超过断言失败的那一行。如果硬件支持调试中断指令，那么可以用调试中断指令来代替for()循环。 */
#define configASSERT( x ) if( ( x ) == 0 ) { taskDISABLE_INTERRUPTS(); for(;;); }
```

清单164 一个简单的 `configASSERT()` 定义，在调试器的控制下执行时非常有用 

清单165中显示的 `configASSERT()` 的定义在应用程序没有在调试器的控制下执行时很有用。它打印出，或以其他方式记录断言失败的源代码行。使用标准的 `C__FILE__` 宏来获取源文件的名称，使用标准的 `C__LINE__ `宏来获取源文件的行号，来识别断言失败的行。

```verilog
/* 这个函数必须在C源文件中定义，而不是在FreeRTOSConfig.h头文件中。 */
void vAssertCalled(const char *pcFile, uint32_t ulLine)
{
    /* 在这个函数中，pcFile持有包含检测到错误的行的源文件的名称，ulLine持有源文件中的行号。pcFile和ulLine的值可以被打印出来，或者以其他方式记录下来，然后再进入下面的无限循环。 */
    RecordErrorInformationHere(pcFile, ulLine);
    /* 禁用中断，使tick中断停止执行，然后坐在一个循环中，使执行不超过断言失败的那一行。 */
    taskDISABLE_INTERRUPTS();
    for (;;)
        ;
}
/*-----------------------------------------------------------*/
/* 下面这两行必须放在FreeRTOSConfig.h中。 */
extern void vAssertCalled(const char *pcFile, uint32_t ulLine);
#define configASSERT(x) \
    if ((x) == 0)       \
    vAssertCalled(__FILE__, __LINE__)
```

清单165 记录断言失败的源代码行的 `configASSERT()` 定义



## FreeRTOS+Trace

FreeRTOS+Trace是一个运行时诊断和优化工具，由我们的合作伙伴Percepio公司提供。

FreeRTOS+Trace可以捕获有价值的动态行为信息，然后在相互连接的图形视图中展示捕获的信息。

在分析、排除故障或简单地优化FreeRTOS应用程序时，所捕获的信息是非常宝贵的。

FreeRTOS+Trace可以与传统的调试器并排使用，并以更高层次的时间视角来补充调试器的观点。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636343690571-cbaa44ca-b585-493e-97be-3c13e04fe035.png)

图82 FreeRTOS+Trace包括20多个相互连接的视图

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636343747247-99193b55-dad4-434d-92d9-abd590bcdab8.png)

图83 FreeRTOS+Trace主跟踪视图——20多个相互连接的跟踪视图之一

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636342876673-4dbbc85e-6046-4871-b829-f90d8c05dc1d.png)

图84 FreeRTOS+Trace CPU负载视图——20多个相互连接的跟踪视图之一

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636342969529-efd3b559-3c39-4991-b5f5-ffaab7e91bf1.png)

图85 FreeRTOS+Trace响应时间视图——20多个相互连接的跟踪视图之一

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636343055008-54ae9435-ee01-40bd-b879-94ce2a0e4e4f.png)

图86 FreeRTOS+Trace用户事件图视图——20多个相互连接的跟踪视图之一

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636343819967-8a79f228-0196-4cab-9ad6-35bee51e5643.png)

图87 FreeRTOS+Trace内核对象历史视图 - 20多个相互连接的跟踪视图之一



## 调试相关的钩（回调）函数



### Malloc失败的钩

`malloc`失败的钩（或回调）在第2章，堆内存管理中有所描述。

定义一个`malloc`失败钩可以确保在试图创建任务、队列、信号或事件组失败时立即通知应用开发者。



### 堆栈溢出钩

堆栈溢出钩的细节在堆栈溢出那一节中提供。

定义一个堆栈溢出钩可以确保在一个任务使用的堆栈量超过分配给该任务的堆栈空间时，应用程序开发人员得到通知。



## 查看运行时和任务状态信息



### 任务运行时间统计

任务运行时间统计提供了关于每个任务所获得的处理时间的信息。一个任务的运行时间是该任务自应用程序启动以来一直处于运行状态的总时间。

运行时统计的目的是在项目的开发阶段作为剖析和调试的辅助工具使用。项目的开发阶段。它们提供的信息只在作为运行时统计时钟的计数器溢出之前有效。收集运行时统计将增加任务上下文切换时间。

要获得二进制运行时统计信息，请调用 `uxTaskGetSystemState()` API函数。要获得以人类可读的ASCII表形式的运行时统计信息，请调用 `vTaskGetRunTimeStats() `辅助函数。



### 运行时间统计时钟

运行时统计需要测量滴答周期的几分之一。因此，RTOS的tick计数不被用作运行时统计时钟，而是由应用程序代码提供时钟。建议使运行时统计时钟的频率比tick中断的频率快10到100倍。运行时统计时钟越快，统计就越准确，但时间值也会越早溢出。

理想情况下，时间值将由一个自由运行的32位外设定时器/计数器产生，其值可以在没有其他处理开销的情况下被读取。如果可用的外设和时钟速度不能使这种技术成为可能，那么替代性的但效率较低的技术包括：

1. 配置一个外设，使其在所需的运行时统计时钟频率下产生一个周期性中断，然后使用产生的中断数量的计数作为运行时统计时钟。

如果周期性中断只用于提供运行时的统计时钟，那么这种方法是非常低效的。然而，如果应用程序已经使用了频率合适的周期性中断，那么将产生的中断数量的计数加入到现有的中断服务例程中，就很简单和有效。

1. 用一个自由运行的16位外设定时器的当前值作为32位值中最没有意义的16位，用定时器溢出的次数作为32位值中最没有意义的16位，从而产生一个32位值。

通过适当的、有点复杂的操作，可以将RTOS的滴答声与ARM Cortex-M SysTick定时器的当前值结合起来，产生一个运行时统计时钟。FreeRTOS下载中的一些演示项目展示了如何实现这一目标。



### 配置应用程序以收集运行时统计信息

表54详细介绍了收集任务运行时统计数据所需的宏。原本打算将这些宏包含在RTOS端口层中，这就是为什么这些宏的前缀是 "port"，但事实证明在 `FreeRTOSConfig.h` 中定义它们更为实用。

表 54. 用于收集运行时统计数据的宏程序

| 宏观                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `configGENERATE_RUN_TIME_STATS`                              | 这个宏必须在`FreeRTOSConfig.h`中设置为1。当这个宏被设置为1时，调度器将在适当的时候调用本表中详述的其他宏。 |
| `portCONFIGURE_TIMER_FOR_RUN_TIME_STATS()`                   | 必须提供这个宏来初始化用于提供运行时统计时钟的任何一个外设。 |
| `portGET_RUN_TIME_COUNTER_VALUE()`, or `portALT_GET_RUN_TIME_COUNTER_VALUE(Time)` | 必须提供这两个宏中的一个来返回当前的运行时统计时钟值。这是自应用程序首次启动以来，应用程序运行的总时间，以运行时统计时钟为单位。如果使用第一个宏，它必须被定义为评估为当前的时钟值。如果使用第二个宏，它必须被定义为将其 “时间”参数设置为当前时钟值。 |



### uxTaskGetSystemState() API函数

`uxTaskGetSystemState()` 为FreeRTOS调度器控制下的每个任务提供了一个状态信息快照。这些信息是以TaskStatus_t结构数组的形式提供的，每个任务在数组中都有一个索引。TaskStatus_t由清单167和表56描述。

```verilog
UBaseType_t uxTaskGetSystemState(TaskStatus_t *const pxTaskStatusArray,
                                 const UBaseType_t uxArraySize,
                                 uint32_t *const pulTotalRunTime);
```

清单 166. `uxTaskGetSystemState()` API函数原型

表 55 `uxTaskGetSystemState()` 参数和返回值

| 参数名称            | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| `pxTaskStatusArray` | 一个指向TaskStatus_t结构数组的指针。该数组必须为每个任务至少包含一个TaskStatus_t结构。任务的数量可以用`uxTaskGetNumberOfTasks()` API函数来确定。TaskStatus_t结构在清单167中显示，TaskStatus_t结构成员在表56中描述。 |
| `uxArraySize`       | `pxTaskStatusArray`参数所指向的数组的大小。该大小被指定为数组中的索引数（数组中包含的TaskStatus_t结构数），而不是数组中的字节数。 |
| `pulTotalRunTime`   | 如果`configGENERATE_RUN_TIME_STATS`被设置为1，在 中设置为1，那么`*pulTotalRunTime`就会被 `uxTaskGetSystemState()` 设置为目标启动后的总运行时间（由应用程序提供的运行时间统计时钟定义）。`pulTotalRunTime`是可选的，如果不需要总运行时间，可以设置为`NULL`。 |
| 返回值              | 返回由 `uxTaskGetSystemState()` 填充的TaskStatus_t结构的数量。返回的值应该等于 `uxTaskGetNumberOfTasks()` API函数返回的数字，但如果在`uxArraySize`参数中传递的值太小，则会为零。 |

```verilog
typedef struct xTASK_STATUS
{
    TaskHandle_t xHandle;
    const char *pcTaskName;
    UBaseType_t xTaskNumber;
    eTaskState eCurrentState;
    UBaseType_t uxCurrentPriority;
    UBaseType_t uxBasePriority;
    uint32_t ulRunTimeCounter;
    uint16_t usStackHighWaterMark;
} TaskStatus_t;
```

清单 167. TaskStatus_t结构

表 56. TaskStatus_t结构成员

| 参数名称/返回值        | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `xHandle`              | 该结构中的信息所涉及的任务的句柄。                           |
| `pcTaskName`           | 任务的可读性文本名称。                                       |
| `xTaskNumber`          | 每个任务都有唯一的`xTaskNumber`值。如果一个应用程序在运行时创建和删除任务，那么一个任务有可能与之前被删除的任务有相同的句柄。`xTaskNumber`被提供给应用程序代码和内核感知调试器，以区分一个仍然有效的任务和一个与有效任务有相同句柄的被删除任务。 |
| `eCurrentState`        | 一个保持任务状态的枚举类型。`eCurrentState`可以是以下值之一：`eRunning`, `eReady`, `eBlocked`, `eSuspended`, `eDeleted`。一个任务只有在调用 `vTaskDelete() `删除任务后，到Idle任务释放分配给被删除任务的内部数据结构和堆栈的内存这段时间内，才会被报告为处于`eDeleted`状态。在这段时间之后，该任务将不再以任何方式存在，试图使用其句柄是无效的。 |
| `uxCurrentPriority`    | 在调用 `uxTaskGetSystemState() `时，任务运行的优先级。只有当任务按照7.3节 "互斥（和二进制半挂）"中描述的优先级继承机制暂时被分配了更高的优先级时，uxCurrentPriority才会高于应用编写者分配给任务的优先级。 |
| `uxBasePriority`       | 应用程序编写者分配给任务的优先级。只有在`FreeRTOSConfig.h` 中` configUSE_MUTEXES` 被设置为1时，`uxBasePriority`才有效。 |
| `ulRunTimeCounter`     | 自任务创建以来，任务使用的总运行时间。总运行时间是以绝对时间的形式提供的，它使用应用程序编写者提供的时钟来收集运行时间的统计数据。`ulRunTimeCounter` 只有在 `FreeRTOSConfig.h `中`configGENERATE_RUN_TIME_STATS `被设置为1时才有效。 |
| `usStackHighWaterMark` | 任务的堆积高水位。这是自任务创建以来，任务所剩的最小堆栈空间。它是任务离溢出堆栈有多远的指示；这个值越接近零，任务就越接近溢出堆栈。 `usStackHighWaterMark `的单位是字节。 |



### vTaskList() 辅助函数

`vTaskList()` 提供的任务状态信息与 `uxTaskGetSystemState()` 提供的信息类似，但它以人类可读的ASCII表而不是二进制值数组的形式呈现信息。

`vTaskList()` 是一个非常耗费处理器的函数，并且会让调度器在很长一段时间内悬浮。因此，建议该函数仅用于调试目的，而不是在生产性实时系统中使用。

如果 `FreeRTOSConfig.h `中的` configUSE_TRACE_FACILITY `和` configUSE_STATS_FORMATTING_FUNCTIONS `都被设置为1，则 `vTaskList() `可用。

```verilog
void vTaskList( signed char *pcWriteBuffer );
```

清单 168. `vTaskList()` API函数原型

表 57.` vTaskList() `参数

| 参数名称        | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `pcWriteBuffer` | 一个指向字符缓冲区的指针，格式化和人类可读的表被写入其中。该缓冲区必须大到足以容纳整个表，因为不进行边界检查。 |

图88显示了` vTaskList() `生成的输出的一个例子。 在输出中：

- 每一行都提供了一个单一的任务信息。
- 第一列是任务名称。

- 第二列是任务的状态，其中'R'表示就绪，'B'表示阻塞，'S'表示暂停，'D'表示任务已被删除。一个任务只有在调用` vTaskDelete() `删除任务后，到Idle任务释放分配给被删除任务的内部数据结构和堆栈的内存这段时间内，才会被报告为处于删除状态。在这段时间之后，该任务将不再以任何方式存在，试图使用其句柄是无效的。
- 第三列是任务的优先级。

- 第四列是任务的堆栈高水位。见表56中对 `usStackHighWaterMark` 的描述。
- 第五列是分配给该任务的唯一编号。见表56中对`xTaskNumber`的描述。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636353441516-6a3312ed-11d9-435c-a274-b2a9ca36ade5.png)

图88 由`vTaskList()`生成的输出示例



### vTaskGetRunTimeStats() 辅助函数

`vTaskGetRunTimeStats() `将收集到的运行时统计数据格式化为人类可读的ASCII表格。

`vTaskGetRunTimeStats() `是一个非常耗费处理器的函数，并且会让调度器暂停很长一段时间。因此，建议该函数仅用于调试目的，而不是在生产型实时系统中使用。

当 `FreeRTOSConfig.h `中 `configGENERATE_RUN_TIME_STATS` 和 `configUSE_STATS_FORMATTING_FUNCTIONS `都被设置为1时，`vTaskGetRunTimeStats()` 可用。

```verilog
void vTaskGetRunTimeStats( signed char *pcWriteBuffer );
```

清单169. `vTaskGetRunTimeStats() `API函数原型

表 58. `vTaskGetRunTimeStats() `参数

| 参数值          | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `pcWriteBuffer` | 一个指向字符缓冲区的指针，格式化和人类可读的表被写入其中。该缓冲区必须大到足以容纳整个表，因为不进行边界检查。 |

图89显示了` vTaskGetRunTimeStats() `产生的一个输出例子。在输出中：

- 每一行都提供了一个单一的任务信息。
- 第一列是任务名称。

- 第二列是任务在运行状态下花费的时间，是一个绝对值。见表56中 `ulRunTimeCounter` 的描述。
- 第三列是任务在运行状态下花费的时间，占目标启动后总时间的百分比。显示的百分比时间的总和通常会小于预期的100%，因为统计数据的收集和计算是使用整数计算，四舍五入到最近的整数值。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636354332626-801684d7-47e1-4065-aebc-b8326dc379d6.png)

图89 由` vTaskGetRunTimeStats() `生成的输出示例



### 生成和显示运行时的统计数据，一个工作实例

这个例子使用一个假设的16位定时器来产生一个32位的运行时统计时钟。该计数器被配置为在每次16位值达到最大值时产生一个中断，有效地产生一个溢出中断。中断服务程序对溢出发生的次数进行计数。

32位值是通过使用溢出发生次数作为32位值中最有意义的两个字节，以及使用当前16位计数器值作为32位值中最没有意义的两个字节来创建的。中断服务例程的伪代码显示在清单170中。

```verilog
void TimerOverflowInterruptHandler(void)
{
    /* 只需计算中断的数量。 */
    ulOverflowCount++;
    /* 清除中断。 */
    ClearTimerInterrupt();
}
```

清单 170. 用于计算定时器溢出的16位定时器溢出中断处理程序

清单171显示了添加到`FreeRTOSConfig.h`中的行，以实现对运行时统计数据的收集。

```verilog
/* 将 configGENERATE_RUN_TIME_STATS 设置为 1 以启用运行时统计数据的收集。这样做时， 必须同时定义 portCONFIGURE_TIMER_FOR_RUN_TIME_STATS() 和 portGET_RUN_TIME_COUNTER_VALUE() 或 portALT_GET_RUN_TIME_COUNTER_VALUE(x)。 */
#define configGENERATE_RUN_TIME_STATS 1
/* portCONFIGURE_TIMER_FOR_RUN_TIME_STATS()被定义为调用设置假设的16位定时器的函数(该函数的实现没有显示)。 */
void vSetupTimerForRunTimeStats(void);
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS() vSetupTimerForRunTimeStats()
/* portALT_GET_RUN_TIME_COUNTER_VALUE()的定义是将其参数设置为当前的运行时间计数器/时间值。返回的时间值是32位的，是通过将16位定时器溢出的计数移到32位数字的前两个字节中，然后将结果与当前的16位计数器值进行位法OR形成的。 */
#define portALT_GET_RUN_TIME_COUNTER_VALUE(ulCountValue)             \
    {                                                                \
        extern volatile unsigned long ulOverflowCount;               \
                                                                     \
        /* 断开时钟与计数器的连接，使其不发生变化而其值正在被使用。 */    
        PauseTimer();                                                \
                                                                     \
        /* 溢出的数量被转移到最重要的返回的32位数值的两个字节。 */                   			   
        ulCountValue = (ulOverflowCount << 16UL);                    \
                                                                     \
        /* 当前的计数器值被用作最小有效值。返回的32位数值的两个字节。 */                    			   
        ulCountValue |= (unsigned long)ReadTimerCount();             \
                                                                     \
        /* 重新将时钟连接到计数器。 */                   			   \
        ResumeTimer();                                               \
    }
```

清单 171. 添加到` FreeRTOSConfig.h` 中的宏，以实现对运行时统计数据的收集

清单172中显示的任务每5秒打印出收集到的运行时统计数据。

```verilog
/* 为了清楚起见，本代码清单中省略了对fflush()的调用。 */
static void prvStatsTask(void *pvParameters)
{
    TickType_t xLastExecutionTime;
    /* 用来保存格式化的运行时统计文本的缓冲区需要相当大。因此，它被声明为静态的，以确保它不被分配在任务栈上。这使得这个函数不能重入。 */
    static signed char cStringBuffer[512];
    /* 该任务将每5秒运行一次。 */
    const TickType_t xBlockPeriod = pdMS_TO_TICKS(5000);
    /* 将xLastExecutionTime初始化为当前时间。这是该变量唯一需要明确写入的时间。之后，它将在vTaskDelayUntil()API函数中进行内部更新。 */
    xLastExecutionTime = xTaskGetTickCount();
    /* 和大多数任务一样，这个任务是在一个无限循环中实现的。 */
    for (;;)
    {
        /* 等到再次运行这项任务的时候。 */
        vTaskDelayUntil(&xLastExecutionTime, xBlockPeriod);
        /* 从运行时统计信息中生成一个文本表。这必须适合于cStringBuffer数组。 */
        vTaskGetRunTimeStats(cStringBuffer);

        /* 打印运行时统计表的列标题。 */
        printf("\nTask\t\tAbs\t\t\t%%\n");
        printf("-------------------------------------------------------------\n");

        /* 打印出运行时统计数据本身。数据表格包含多行，所以调用vPrintMultipleLines()函数，而不是直接调用printf()。vPrintMultipleLines()只是对每一行单独调用printf()，以确保行缓冲按预期工作。 */
        vPrintMultipleLines(cStringBuffer);
    }
}
```

清单172. 打印出收集到的运行时统计数据的任务



## 追踪钩宏

跟踪宏是被放置在FreeRTOS源代码的关键点上的宏。默认情况下，这些宏是空的，因此不产生任何代码，也没有运行时间的开销。通过覆盖默认的空实现，应用程序编写者可以：

- 在不修改FreeRTOS源文件的情况下将代码插入FreeRTOS。
- 通过目标硬件上的任何手段输出详细的执行顺序信息。跟踪宏出现在FreeRTOS源代码中的足够多的地方，允许它们被用来创建一个完整和详细的调度器活动跟踪和分析日志。



### 可用的追踪钩宏

如果在这里详细介绍每一个宏，会占用太多的篇幅。表59详细列出了被认为对应用程序编写者最有用的宏子集。

表59中的许多描述提到了一个叫做`pxCurrentTCB`的变量。一个FreeRTOS的私有变量，它持有运行状态下任务的句柄，并且对任何从 `FreeRTOS/Source/tasks.c `源文件中调用的宏是可用的。

表 59. 最常用的跟踪钩宏的选择

| 宏观                                          | 描述                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| `traceTASK_INCREMENT_TICK(xTickCount)`        | 在tick中断期间，在tick计数被增加后调用。`xTickCount`参数将新的tick计数值传递给宏。 |
| `traceTASK_SWITCHED_OUT()`                    | 在一个新的任务被选中运行之前被调用。此时，`pxCurrentTCB`包含即将离开运行状态的任务的句柄。 |
| `traceTASK_SWITCHED_IN()`                     | 在一个任务被选中运行后被调用。此时，`pxCurrentTCB`包含即将进入运行状态的任务的句柄。 |
| `traceBLOCKING_ON_QUEUE_RECEIVE(pxQueue)`     | 在当前执行的任务试图从空队列中读取数据，或者试图 "夺取 "空信号或突发事件后进入阻塞状态前立即调用。`pxQueue`参数将目标队列或信号量的句柄传递给宏。 |
| `traceBLOCKING_ON_QUEUE_SEND(pxQueue)`        | 在试图向已满的队列写入后，当前执行的任务进入阻塞状态前立即调用。`pxQueue`参数将目标队列的句柄传入宏。 |
| `traceQUEUE_SEND(pxQueue)`                    | 在`xQueueSend()`、`xQueueSendToFront()`、`xQueueSendToBack()`或任何信号量"give"函数中调用，当队列发送或信号量"give"成功时，`pxQueue`参数将目标队列或 信号量的句柄传入该宏。 |
| `traceQUEUE_SEND_FAILED(pxQueue)`             | 从`xQueueSend()`、`xQueueSendToFront()`、`xQueueSendToBack() `或任何信号量"give"函数中调用，当队列发送或信号量"give"操作失败时。如果队列已满，并且在指定的任何块时间内保持满的状态，那么队列发送或信号量"give" 将失败。`pxQueue`参数将目标队列或信号灯的句柄传递给宏。 |
| `traceQUEUE_RECEIVE(pxQueue)`                 | 在 `xQueueReceive() `或任何信号量"take" 函数中调用，当队列接收或信号量"take "成功时。`pxQueue`参数将目标队列或信号灯的句柄传递给宏。 |
| `traceQUEUE_RECEIVE_FAILED(pxQueue)`          | 当队列或信号灯接收操作失败时，从`xQueueReceive() `或任何信号灯 "take" 函数中调用。如果队列或信号灯是空的，并且在指定的时间段内保持空的状态，那么队列接收或信号灯 "take" 的操作将失败。`pxQueue`参数将目标队列或信号灯的句柄传递给宏。 |
| `traceQUEUE_SEND_FROM_ISR(pxQueue)`           | 当发送操作成功时，从 `xQueueSendFromISR()` 中调用。`pxQueue`参数将目标队列的句柄传入宏。 |
| `traceQUEUE_SEND_FROM_ISR_FAILED(pxQueue)`    | 当发送操作失败时，从` xQueueSendFromISR()` 中调用。如果队列已经满了，发送操作将失败。`pxQueue`参数将目标队列的句柄传入宏。 |
| `traceQUEUE_RECEIVE_FROM_ISR(pxQueue)`        | 当接收操作成功时，从 `xQueueReceiveFromISR()` 中调用。`pxQueue`参数将目标队列的句柄传给宏。 |
| `traceQUEUE_RECEIVE_FROM_ISR_FAILED(pxQueue)` | 当接收操作由于队列已经为空而失败时，在`xQueueReceiveFromISR() `中调用。`pxQueue`参数将目标队列的句柄传入宏。 |
| `traceTASK_DELAY_UNTIL()`                     | 从内部调用 `vTaskDelayUntil()` 中，在调用任务进入阻塞状态前立即调用。 |
| `traceTASK_DELAY()`                           | 在调用任务进入阻塞状态之前，从 `vTaskDelay()` 中调用。       |



### 定义跟踪钩宏

每个跟踪宏都有一个默认的空定义。默认定义可以通过在 `FreeRTOSConfig.h` 中提供一个新的宏定义来覆盖。如果跟踪宏的定义变得很长或很复杂，那么它们可以在一个新的头文件中实现，然后本身包含在`FreeRTOSConfig.h`中。

根据软件工程的最佳实践，FreeRTOS坚持严格的数据隐藏政策。追踪宏允许用户代码被添加到FreeRTOS的源文件中，因此追踪宏所看到的数据类型将与应用代码所看到的数据类型不同：

- 在 `FreeRTOS/Source/tasks.c` 源文件中，任务句柄是一个指向描述任务的数据结构（任务的任务控制块，或TCB）的指针。在 `FreeRTOS/Source/tasks.c `源文件之外，任务句柄是一个指向`void`的指针。
- 在 `FreeRTOS/Source/queue.c `源文件中，队列句柄是一个指向描述队列的数据结构的指针。在`FreeRTOS/Source/queue.c `源文件之外，一个队列句柄是一个指向`void`的指针。

如果一个通常是私有的FreeRTOS数据结构被跟踪宏直接访问，需要特别小心，因为私有数据结构在FreeRTOS版本之间可能会发生变化。



### FreeRTOS感知调试器插件

提供一些FreeRTOS意识的插件可用于以下IDE。 这个列表可能并不详尽:

- Eclipse (StateViewer)
- Eclipse (ThreadSpy)

- IAR
- ARM DS-5 

- Atollic TrueStudio
- Microchip MPLAB

- iSYSTEM WinIDEA

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636357790972-483d19f5-328f-478b-929d-713b284d0f07.png)

图90 FreeRTOS ThreadSpy Eclipse 插件，来自Code Confidence Ltd