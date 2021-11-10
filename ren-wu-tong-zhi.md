# **任务通知**



## 引言与范围

我们已经看到，使用FreeRTOS的应用程序被构造成一组独立的任务，而且这些自主的任务很可能必须相互通信，这样它们才能共同提供有用的系统功能。



### 通过中介对象进行通信

这本书已经描述了任务之间相互通信的各种方式。到目前为止，所描述的方法都需要创建通信对象。通信对象的示例包括队列、事件组和各种不同类型的信号量。



当使用通信对象时，事件和数据不会直接发送给接收任务或接收ISR，而是发送给通信对象。同样，任务和ISR从通信对象接收事件和数据，而不是直接从发送事件或数据的任务或ISR接收事件和数据。如图76所示。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636334710668-2719753a-8da1-41d1-b629-77bf4ee588a9.png)

图76 用于将事件从一个任务发送到另一个任务的通信对象



### 任务通知-直接与任务通信

任务通知允许任务与其他任务交互，并与ISR同步，不需要单独的通信对象。通过使用任务通知，任务或ISR可以直接向接收任务发送事件。如图77所示。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636335089505-b72e3997-de68-4788-8626-f2939872cc4f.png)

图77 用于将事件从一个任务直接发送到另一个任务的任务通知

任务通知功能是可选的。要包含任务通知功能，请在`FreeRTOSConfig.h `中将`configUSE_TASK_NOTIFICATIONS` 设置为1。

当`configUSE_TASK_NOTIFICATIONS`被设置为1时，每个任务有一个 "通知状态"。可以是 "待定 "或 "未定"，还有一个 "通知值"，它是一个32位无符号整数。无符号整数。当一个任务收到一个通知时，它的通知状态被设置为待定。当一个任务读取其通知值时，其通知状态被设置为非待定。

任务可以在阻塞状态下等待其通知状态变为挂起（可选超时）。



### 范围

本章旨在让读者更好地理解：

- 任务的通知状态和通知值。
- 何时以及如何使用任务通知来代替通信对象(如信号量)。
- 使用任务通知代替通信对象的优点。



## 任务通知；优势和限制



### 任务通知的性能优势

使用任务通知向任务发送事件或数据要比使用队列、信号量或事件组执行等价操作快得多。



### 任务通知的RAM占用优势

同样，使用任务通知向任务发送事件或数据所需的RAM要比使用队列、信号量或事件组执行等价操作所需的RAM少得多。这是因为必须先创建每个通信对象(队列、信号量或事件组)，然后才能使用它，而启用任务通知功能的开销固定为每个任务8个字节的RAM。



### 任务通知的局限性

任务通知比通信对象更快，使用的RAM更少，但任务通知不能在所有场景中使用。介绍不支持任务通知的场景：

- 向ISR发送事件或数据

通信对象可用于将事件和数据从ISR发送到任务，或从任务发送到ISR。

任务通知可以用于从ISR向任务发送事件和数据，但不能用于从任务向ISR发送事件或数据。

- 启用多个接收任务

通信对象可以被任何知道其句柄(可能是队列句柄、信号量句柄或事件组句柄)的任务或ISR访问。任意数量的任务和isr都可以处理发送到任何给定通信对象的事件或数据。

任务通知直接发送到接收任务，因此只能由将通知发送到的任务。然而，这在实际情况中很少是一个限制因为，虽然有多个任务和isr向同一个通信对象发送消息是很常见的，但有多个任务和isr从同一个对象接收消息是很少见的沟通对象。

- 缓冲多个数据项

队列是一次可以保存多个数据项的通信对象。已发送到队列但尚未从队列接收的数据将在队列对象中进行缓冲。

任务通知通过更新接收任务的通知值向任务发送数据。任务的通知值一次只能保存一个值。

- 向多个任务广播

事件组是一种通信方式，可用于一次向多个任务发送事件。

任务通知直接发送给接收任务，只能由接收任务处理。

- 在阻塞状态下等待发送完成

如果沟通对象是暂时的状态,这意味着没有更多的数据或事件可以写(例如,当队列满是没有更多的数据可以被发送到队列),然后试图写入任务对象可以选择进入阻塞状态等待写操作完成。

如果任务尝试向已挂起通知的任务发送任务通知，则发送任务不可能在“阻塞”状态等待接收任务重置其通知状态。正如我们将看到的，在使用任务通知的实际情况中，这很少是一个限制。



## 使用任务通知



### 任务通知API选项

任务通知是一个非常强大的特性，经常可以用来代替二进制信号量、计数信号量、事件组，有时甚至是队列。这种广泛的使用场景可以通过使用`xTaskNotify()` API函数发送任务通知和使用`xTaskNotifyWait()` API函数接收任务通知来实现。

但是，在大多数情况下，不需要`xTaskNotify()`和`xTaskNotifyWait()` API函数提供的全部灵活性，简单的函数就足够了。因此，提供了`xTaskNotifyGive()` API函数作为xTaskNotify的一个更简单但更不灵活的替代方案，而`ulTaskNotifyTake()` API函数作为`xTaskNotifyWait()`的一个更简单但更不灵活的替代方案。



### xTaskNotifyGive() API函数

`xTaskNotifyGive()`直接向任务发送通知，并增加(向)接收任务的通知值。调用`xTaskNotifyGive()`将把接收任务的通知状态设置为挂起(如果它还没有挂起)。

提供了`xTaskNotifyGive()`的API函数，允许任务通知作为二进制或计数信号量的轻量级和更快的替代。

```groovy
BaseType_t xTaskNotifyGive( TaskHandle_t xTaskToNotify );
```

清单 145. `xTaskNotifyGive()` API函数原型

> `xTaskNotifyGive()`实际上是作为宏实现的，而不是一个函数。为了简单起见，本书将它称为一个函数。

表48. `xTaskNotifyGive()`参数和返回值

| 参数名称/返回的值 | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| `xTaskToNotify`   | 被发送通知的任务的句柄——有关获取任务句柄的信息，请参阅`xTaskCreate()` API函数的`pxCreatedTask`参数。 |
| 返回值            | `xTaskNotifyGive()`是一个调用`xTaskNotify()`的宏。宏传递给`xTaskNotify()`的参数被设置为pdPASS是唯一可能的返回值。`xTaskNotify()`将在本书后面进行描述。 |



### vTaskNotifyGiveFromISR() API函数

`vTaskNotifyGiveFromlSR()`是`xTaskNotifyGive()`的一个版本，可以在中断服务例程中使用。

```groovy
void vTaskNotifyGiveFromISR( TaskHandle_t xTaskToNotify, 
 							 BaseType_t *pxHigherPriorityTaskWoken );
```

清单146. `vTaskNotifyGiveFromISR()` API函数原型

表 49. `vTaskNotifyGiveFromISR()`参数和返回值

| 参数名称/返回的值           | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `xTaskToNotify`             | 被发送通知的任务的句柄——有关获取任务句柄的信息，请参阅`xTaskCreate()` API函数的pxCreatedTask参数。 |
| `pxHigherPriorityTaskWoken` | 如果正在发送通知的任务处于阻塞状态等待接收通知，则发送通知将导致任务离开阻塞状态。如果调用`vTaskNotifyGiveFromISR()`导致任务离开阻塞状态，并且未阻塞的任务的优先级高于当前正在执行的任务(被中断的任务)的优先级，那么，`vTaskNotifyGiveFromISR()`将在内部将`*pxhigherpriorityTaskWoken`设置为pdTRUE。如果`vTaskNotifyGiveFromlSR()`将这个值设置为pdTRUE，那么应该在中断退出之前执行上下文切换。这将确保中断直接返回到最高优先级的就绪状态任务。和所有中断安全API函数一样，`pxHigherPriorityTaskWoken`参数在使用之前必须设置为pdFALSE。 |



### ulTaskNotifyTake() API函数

`ulTaskNotifyTake()`允许任务在阻塞状态中等待它的通知值大于0，并且在返回之前减少(减去1)或清除任务的通知值。

提供了`ulTaskNotifyTake()` API函数，以允许任务通知作为二进制或计数信号量的轻量级和更快的替代方法。

```groovy
uint32_t ulTaskNotifyTake( BaseType_t xClearCountOnExit, TickType_t xTicksToWait );
```

清单147.  `ulTaskNotifyTake()` API函数原型

表 50. `ulTaskNotifyTake()`参数和返回值

| 参数名称/返回值     | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| `xClearCountOnExit` | 如果`xClearCountOnExit`设置为pdTRUE，那么在调用`ulTaskNotifyTake()`返回之前，调用任务的通知值将被清除为零。如果`xClearCountOnExit`设置为pdFALSE，并且调用任务的通知值大于零，那么调用任务的通知值将在调用`ulTaskNotifyTake()`返回之前递减。 |
| `xTicksToWait`      | 调用任务在等待其通知值大于零时应保持在状态阻塞的最大时间量。块时间以滴答周期指定，因此它所代表的绝对时间依赖于滴答频率。宏`pdMS_TO_TICKS()`可用于将以毫秒为单位指定的时间转换为以节拍为单位指定的时间。设置`xTicksToWait`为`portMAX_DELAY`将导致任务无限期等待(不会超时)在`FreeRTOSConfig.h`中将`INCLUDE_vTaskSuspend`设置为1。 |
| 返回值              | 返回值是调用任务的通知值，在它被清除为零或减少之前，由`xClearCountOnExit`参数的值指定。如果一块指定时间(`xTicksToWait`不是零),返回值不为零,那么它有可能调用任务是放置进入阻塞状态,等待其通知值大于零,和它的值更新阻塞时间到期前通知。如果指定了块时间(`xTicksToWait`不为零)，并且返回值为零，然后调用任务被置于阻塞状态，等待其通知值大于零，但指定的阻塞时间在此之前已过期。 |



### 示例24. 方法1使用任务通知来代替信号量

示例16使用了一个二进制信号量来解除中断服务程序中的任务阻塞——有效地将任务与中断同步。这个示例复制了示例16的功能，但是使用了一个直接到任务通知来代替二进制信号量。

清单148显示了与中断同步的任务的实现。在示例16中使用的对`xSemaphoreTake()`的调用已经被对`ulTaskNotifyTake()`的调用所取代。

`ulTaskNotifyTake() xClearCountOnExit`参数被设置为pdTRUE，这会导致接收任务的通知值在`ulTaskNotifyTake()`返回之前被清除为零。因此，有必要处理在每次调用`ulTaskNotifyTake()`之间已经可用的所有事件。在示例16中，因为使用了一个二进制信号量，所以挂起事件的数量必须从硬件中确定，这并不总是实际的。在示例24中，从`ulTaskNotifyTake()`返回挂起事件的数量。

在调用`ulTaskNotifyTake`之间发生的中断事件被锁定在任务的通知值中，如果调用的任务已经有挂起的通知，那么对`ulTaskNotifyTake()`的调用将立即返回。

```groovy
/* 周期性任务生成软件中断的速率。 */
const TickType_t xInterruptFrequency = pdMS_TO_TICKS(500UL);

static void vHandlerTask(void *pvParameters)
{
/* xMaxExpectedBlockTime设置为略长于事件之间的最大预期时间。 */
const TickType_t xMaxExpectedBlockTime = xInterruptFrequency + pdMS_TO_TICKS(10);
uint32_t ulEventsToProcess;
    
    /* 对于大多数任务来说，这个任务是在一个无限循环中实现的。 */
    for (;;)
    {
        /* 等待接收从中断服务程序直接发送到此任务的通知。 */
        ulEventsToProcess = ulTaskNotifyTake(pdTRUE, xMaxExpectedBlockTime);
        if (ulEventsToProcess != 0)
        {
            /* 要到达这里，必须至少发生一件事。在此循环，直到处理完所有挂起事件(在本例中，只需为每个事件打印一条消息)。*/
            while (ulEventsToProcess > 0)
            {
                vPrintString("Handler task - Processing event.\r\n");
                ulEventsToProcess--;
            }
        }
        else
        {
            /* 如果到达函数的这一部分，则中断没有在预期的时间内到达，并且(在真实的应用程序中)可能需要执行一些错误恢复操作。*/
        }
    }
}
```

清单148。中断处理所涉及的任务的实现 的实现（与中断同步的任务）。

用于生成软件中断的周期任务在中断生成之前打印一条消息，并在中断生成之后再次打印一条消息。这允许在生成的输出中观察执行顺序。

清单149显示了中断处理程序。除了直接向延迟中断处理的任务发送通知外，这几乎不做什么。

```groovy
static uint32_t ulExampleInterruptHandler(void)
{
BaseType_t xHigherPriorityTaskWoken;
    
    /* xHigherPriorityTaskWoken参数必须初始化为pdFALSE，因为如果需要进行上下文切换，它将在中断安全API函数中被设置为pdTRUE。*/
    xHigherPriorityTaskWoken = pdFALSE;
    
    /* 直接向延迟中断处理的任务发送通知。 */
    vTaskNotifyGiveFromISR(/* 正在发送通知的任务的句柄。句柄是在创建任务时保存的。*/
                           xHandlerTask,
        
                           /* xHigherPriorityTaskWoken以通常的方式使用。 */
                           &xHigherPriorityTaskWoken);

    /*将xHigherPriorityTaskWoken值传递给portYIELD_FROM_ISR()。如果在vTaskNotifyGiveFromISR()中xHigherPriorityTaskWoken被设置为pdTRUE，那么调用portYIELD_FROM_ISR()将请求一个上下文切换。如果xHigherPriorityTaskWoken仍然是pdFALSE，那么调用portYIELD_FROM_ISR()将没有效果。Windows端口使用的portYIELD_FROM_ISR()的实现包含一个return语句，这就是为什么该函数没有显式返回值。*/
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单149. 例24中使用的中断服务例程的实现

执行示例24时产生的输出如图78所示。正如预期的那样，它与执行示例16时产生的结果相同。`vHandlerTask()`在生成中断后立即进入运行状态，因此任务的输出将分割周期性任务产生的输出。图79提供了进一步的解释。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636340508634-f4ae301a-d216-4a7a-ac50-4045d007b87a.png)

图78  执行示例16时产生的输出



![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636340626206-22890559-016b-4f87-8268-e7af8d15f79f.png)

图79  执行示例24时的执行顺序



### 示例25. 方法2使用任务通知来代替信号量

在示例24中，`ulTaskNotifyTake() xClearOnExit`参数被设置为pdTRUE。示例25稍微修改了示例24，以演示将`ulTaskNotifyTake() xClearOnExit`参数改为pdFALSE时的行为。

当`xClearOnExit`为pdFALSE时，调用`ulTaskNotifyTake()`只会减少(减少1)调用任务的通知值，而不是将其清除为零。因此，通知计数是已发生事件的数量与已处理事件的数量之间的差值。这允许`vHandlerTask()`的结构以两种方式简化：

1. 等待处理的事件数量保存在通知值中，因此不需要保存在局部变量中。
2. 每次调用`ulTaskNotifyTake()`之间只需要处理一个事件。

示例25中使用的`vHandlerTask()`的实现如清单150所示。

```groovy
static void vHandlerTask(void *pvParameters)
{
/*xMaxExpectedBlockTime设置为比事件之间的最大预期时间稍长一点。*/
const TickType_t xMaxExpectedBlockTime = xInterruptFrequency + pdMS_TO_TICKS(10);
    
    /* 对于大多数任务来说，这个任务是在一个无限循环中实现的。*/
    for (;;)
    {
        /*等待接收从中断服务程序直接发送到此任务的通知。xClearCountOnExit参数现在是pdFALSE，因此任务的通知值将由ulTaskNotifyTake()递减，而不是清除为零。*/
        if (ulTaskNotifyTake(pdFALSE, xMaxExpectedBlockTime) != 0)
        {
            /*要到达这里，必须有事件发生。处理事件(在本例中只打印一条消息)。*/
            vPrintString("Handler task - Processing event.\r\n");
        }
        else
        {
            /*如果到达了函数的这一部分，那么在预期的时间内中断没有到达，并且(在真实的应用程序中)可能需要执行一些错误恢复操作。*/
        }
    }
}
```

清单150. 示例25中中断处理延迟到的任务的实现(与中断同步的任务)。

出于演示的目的，中断服务程序也被修改为每个中断发送多个任务通知，这样做，模拟在高频率下发生的多个中断。示例25中使用的中断服务例程的实现如清单151所示。

```groovy
static uint32_t ulExampleInterruptHandler(void)
{
BaseType_t xHigherPriorityTaskWoken;
    
    xHigherPriorityTaskWoken = pdFALSE;
    
    /* 多次向处理程序任务发送通知。第一个“给出”将解除对任务的阻止，下面的“给出”将演示接收任务的通知值正在用于计数（锁定）事件-允许任务依次处理每个事件。*/
    vTaskNotifyGiveFromISR(xHandlerTask, &xHigherPriorityTaskWoken);
    vTaskNotifyGiveFromISR(xHandlerTask, &xHigherPriorityTaskWoken);
    vTaskNotifyGiveFromISR(xHandlerTask, &xHigherPriorityTaskWoken);
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单151. 例25中使用的中断服务例程的实现

执行示例25时产生的输出如图80所示。可以看到，`vHandlerTask()`在每次生成中断时处理所有这三个事件。

![img](https://cdn.nlark.com/yuque/0/2021/jpeg/23129846/1636341690974-60c2b176-1dc3-48b8-90f4-aed271d6d082.jpeg)

图80  执行例25时产生的输出



### xTaskNotify()和xTaskNotifyFromlSR() API函数

`xTaskNotify()`是`xTaskNotifyGive()`的一个更强大的版本，可以用以下任何一种方式来更新接收任务的通知值：

- 增加接收任务的通知值（向其中添加一个），在这种情况下，`xTaskNotify()`相当于`xTaskNotifyGive()`。
- 在接收任务的通知值中设置一个或多个位。这允许将任务的通知值用作事件组的更轻的权重和更快的替代。

- 向接收任务的通知值中写入一个全新的数字，但只有当接收任务在上次更新后已经读取了它的通知值时才可以。这允许任务的通知值提供类似于长度为1的队列所提供的功能。
- 向接收任务的通知值中写入一个全新的数字，即使接收任务自上次更新后还没有读到它的通知值。这允许任务的通知值提供与`xQueueOverwrite()` API函数类似的功能。结果行为有时被称为“邮箱”。

`xTaskNotify()`比`xTaskNotifyGive()`更灵活和强大，而且由于这种额外的灵活性和强大功能，它使用起来也有点复杂。

`xTaskNotifyFromISR()`是`xTaskNotify()`的一个版本，可以在中断服务例程中使用，因此有一个额外的`pxHigherPriorityTaskWoken`参数。

调用`xTaskNotify()`将始终将接收任务的通知状态设置为挂起(如果它还没有挂起)。

```groovy
BaseType_t xTaskNotify(TaskHandle_t xTaskToNotify,
                       uint32_t ulValue,
                       eNotifyAction eAction);

BaseType_t xTaskNotifyFromISR(TaskHandle_t xTaskToNotify,
                              uint32_t ulValue,
                              eNotifyAction eAction,
                              BaseType_t *pxHigherPriorityTaskWoken);
```

清单152.  `xTaskNotify()`和`xTaskNotifyFromISR()` API函数的原型

表 51.  `xTaskNotify()`参数和返回值

| 参数名称/返回值 | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `xTaskToNotify` | 被发送通知的任务的句柄——有关获取任务句柄的信息，请参阅`xTaskCreate()` API函数的`pxCreatedTask`参数。 |
| `ulValue`       | `ulValue`的使用方式取决于`eNotifyAction`值。见表52。         |
| `eNotifyAction` | 指定如何更新接收任务的通知值的枚举类型返回值                 |
| 返回值          | `xTaskNotify()`将返回pdPASS，除了表52中提到的一种情况。      |

表 52. 有效的`xTaskNotify() eNotifyAction`参数值，以及它们对接收任务的通知值的结果影响

| 参数名称/返回值             | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `eNoAction`                 | 接收任务的通知状态被设置为挂起，而它的通知值不被更新。没有使用`xTaskNotify() uIValue`参数。`eNoAction`操作允许任务通知作为二进制信号量的更快和更轻的替代。 |
| `eSetBits`                  | 接收任务的通知值是按位或通过`xTaskNotify()`的`ulValue`参数传递的值。例如，`ulValue`设置为0x01，则接收任务的通知值中设置0位。另一个例子，如果`ulValue`是0x06(二进制0110)，那么第1位和第2位将被设置在接收任务的通知值中。`eSetBits`动作允许任务通知被用作事件组的更快和更轻的替代。 |
| `eIncrement`                | 接收任务的通知值增加。没有使用`xTaskNotify() uIValue`参数。`elncrement`操作允许任务通知作为二进制或计数信号量的更快和更轻的替代方法，它相当于更简单的`xTaskNotifyGive()` API函数。 |
| `eSetValueWithoutOverwrite` | 如果接收任务在调用`xTaskNotify()`之前有一个通知挂起，则不会采取任何操作，`xTaskNotify()`将返回pdFAlL。如果在调用`xTaskNotify()`之前接收任务没有挂起通知，那么接收任务的通知值将被设置为在`xTaskNotify() uNValue`参数中传递的值。 |
| `eSetValueWithOverwrite`    | 接收任务的通知值被设置为在`xTaskNotify() ulValue`参数中传递的值，不管接收任务在调用`xTaskNotify()`之前是否有一个通知挂起。 |



### xTaskNotifyWait() API函数

`xTaskNotifyWait()`是`ulTaskNotifyTake()`的一个更强大的版本。它允许任务等待(带有可选超时)，以便调用任务的通知状态变为挂起状态(如果它还没有挂起)。`xTaskNotifyWait()`提供了在进入函数和退出函数时要在调用任务的通知值中清除的位的选项。

```groovy
BaseType_t xTaskNotifyWait( uint32_t ulBitsToClearOnEntry,
 							uint32_t ulBitsToClearOnExit,
 							uint32_t *pulNotificationValue,
 							TickType_t xTicksToWait );
```

清单153.  `xTaskNotifyWait()` API函数原型

表 53.  `xTaskNotifyWait()`参数和返回值

| 参数名称/返回值        | 描述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `ulBitsToClearOnEntry` | 如果调用任务在调用`xTaskNotifyWait()`之前没有一个通知挂起，那么`ulBitsToClearOnEntry`中设置的任何位都将在进入函数时在任务的通知值中被清除。例如，如果`ulBitsToClearOnEntry`为0x01，则任务通知值的位0将被清除。另一个例子是，将`ulBitsToClearOnEntry`设置为0xFFFFFF（`ULONG_MAX`）将清除任务通知值中的所有位，有效地将该值清除为0。 |
| `ulBitsToClearOnExit`  | 如果调用任务退出`xTaskNotifyWait()`因为它收到一个通知,或因为它已经通知等待当`xTaskNotifyWait()`被称为,那么任何部分设置在`ulBitsToClearOnExit`将被清除在任务前的通知价值任务退出`xTaskNotifyWait()`函数。任务的通知值保存在`*pulNotificationValue`中后，这些位将被清除（请参见下面的`pulNotificationValue`说明）。例如，如果`ulBitsToClearOnExit`为0x03，那么在函数退出之前，任务通知值的0位和1位将被清除。将`ulBitsToClearOnExit`设置为0xfff (`ULONG_MAX`)将清除任务通知值中的所有位，有效地将该值清除为0。 |
| `pulNotificationValue` | 用于传递任务的通知值。复制到`*pulNotificationValue`的值是任务的通知值，与由于`ulBitsToClearOnExit`设置而清除任何位之前的值相同。`pulNotificationValue`是一个可选参数，如果不需要，可以设置为NULL。 |
| `xTicksToWait`         | 调用任务应保持在阻止状态以等待其通知状态变为挂起的最长时间。块时间以滴答周期为单位指定，因此它表示的绝对时间取决于滴答频率。宏`pdMS_TO_TICKS()`可用于将以毫秒为单位指定的时间转换为以TICKS为单位指定的时间。如果`FreeRTOSConfig.h`中将`INCLUDE_vTaskSuspend`设置为1，则将`xTicksToWait`设置为`portMAX_DELAY`将导致任务无限期等待（不超时）。 |
| 返回值                 | 有两个可能的返回值：<br/>pdTRUE<br/>这表明返回`xTaskNotifyWait()`是因为收到了通知，或者是因为调用`xTaskNotifyWait()`时，调用任务已经有一个通知挂起。如果指定了阻止时间（`xTicksToWait`不是零），则调用任务可能被置于阻止状态，以等待其通知状态变为挂起，但其通知状态在阻止时间到期之前被设置为挂起。<br/>pdFALSE<br/>这表明`xTaskNotifyWait()`在调用任务没有收到任务通知的情况下返回。如果`xTicksToWait`不是零，则调用任务将一直处于阻止状态，以等待其通知状态变为挂起，但指定的阻止时间在此之前已过期。 |



### 外围设备驱动程序中使用的任务通知：UART示例

外设驱动程序库提供了在硬件接口上执行常用操作的函数。通常提供此类库的外设示例包括通用异步接收器和发射器(UARTs)、串行外设接口(SPI)端口、模拟到数字转换器(adc)和以太网端口。这些库通常提供的函数示例包括初始化外设、向外设发送数据和从外设接收数据的函数。

外围设备上的一些操作需要较长的时间才能完成。这类操作的例子包括高精度ADC转换，以及在UART上传输大数据包。在这些情况下，驱动程序库函数可以实现轮询反复读取外设的状态寄存器，以确定操作何时完成。然而，以这种方式轮询几乎总是浪费时间，因为它利用了处理器100%的时间，而没有执行有效的处理。在多任务系统中，这种浪费尤其昂贵，在该系统中，轮询外围设备的任务可能会阻止执行具有生产性处理的低优先级任务。

为了避免浪费处理时间的可能性，高效的RTOS感知设备驱动程序应该是中断驱动的，并为启动长时间操作的任务提供在阻塞状态下等待操作完成的选项。这样，低优先级任务可以在执行长时间操作的任务处于阻塞状态时执行，并且没有任务会使用处理时间，除非它们能够有效地使用处理时间。

RTOS驱动程序库通常使用二进制信号量将任务置于阻塞状态。清单154中所示的伪代码演示了该技术，它提供了在UART端口上传输数据的支持RTOS的库函数的概要。在清单154中：

- xUART是描述UART外围设备并保存状态信息的结构。该结构的`xTxSemaphore`成员是`semaphhorehandle_t`类型的变量。它假设信号量已经被创建。
- `xUART_Send()`函数不包含任何互斥逻辑。如果有多个任务要使用`xUART_Send()`函数，那么应用程序编写人员必须在应用程序本身中管理互斥。例如，任务可能需要在调用`xUART_Send()`之前获取互斥锁。

- `xSemaphoreTake()` API函数用于在启动UART传输之后将调用任务置于阻塞状态。
- `xSemaphoreGiveFromISR()` API函数用于在传输完成后将任务从阻塞状态中删除，此时UART外设的传输端中断服务例程执行。

```groovy
/*驱动程序库函数，用于向UART发送数据。*/
BaseType_t xUART_Send(xUART *pxUARTInstance, uint8_t *pucDataSource, size_t uxLength)
{
BaseType_t xReturn;
    
    /*通过尝试在没有超时的情况下获取信号量，确保UART的传输信号量不可用。*/
    xSemaphoreTake(pxUARTInstance->xTxSemaphore, 0);
    
    /* Start the transmission. */
    UART_low_level_send(pxUARTInstance, pucDataSource, uxLength);

    /*阻塞信号灯以等待传输完成。如果获得信号量，那么xReturn将设置为pdPASS。如果信号量take操作超时，	 那么xReturn将设置为pdFAIL。注意，如果中断发生在被调用的UART_low_level_send()和被调用的xSemaphoreTake()之间，那么事件将被锁存在二进制信号量中，对xSemaphoreTake()的调用将立即返回。*/
    xReturn = xSemaphoreTake(pxUARTInstance->xTxSemaphore, pxUARTInstance->xTxTimeout);

    return xReturn;
}
/*-----------------------------------------------------------*/

/*UART传输结束中断的服务例程，在最后一个字节发送到UART后执行。 */
void xUART_TransmitEndISR(xUART *pxUARTInstance)
{
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    /* 清除中断。 */
    UART_low_level_interrupt_clear(pxUARTInstance);

    /* 发送Tx信号量以指示传输结束。如果任务在等待信号量时被阻塞，则该任务将从阻塞状态中删除。*/
    xSemaphoreGiveFromISR(pxUARTInstance->xTxSemaphore, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单154。演示如何在驱动程序库传输函数中使用二进制信号量的伪代码

清单154中演示的技术是完全可行的，确实是常见的做法，但它有一些缺点：

- 库使用多个信号量，这增加了内存占用。
- 信号量只有在创建之后才能使用，所以使用信号量的库只有在显式初始化之后才能使用。

- 信号量是适用于广泛用例的通用对象；它们包括允许任意数量的任务在阻塞状态下等待信号量变为可用的逻辑，以及在信号量变为可用时选择（以确定性方式）从阻塞状态中移除哪个任务的逻辑。执行该逻辑需要有限的时间，在清单154所示的场景中，处理开销是不必要的，在该场景中，在任何给定时间都不能有多个任务等待信号量。

清单155演示了如何使用任务通知代替二进制信号量来避免这些缺点。

> 注意:如果库使用任务通知，那么库的文档必须清楚地说明调用库函数可以更改调用任务的通知状态和通知值。

在清单155中：

- `xUART`结构的`xTxSemaphore`成员已被`xTaskToNotify`成员替换。`xTaskToNotify`是TaskHandle_t类型的变量，用于保存等待UART操作完成的任务的句柄。
- `xTaskGetCurrentTaskHandle() `FreeRTOS API函数用于获取运行状态任务的句柄。

- 库不创建任何FreeRTOS对象，因此不会引起内存开销，也不需要显式初始化。
- 任务通知将直接发送到等待UART操作完成的任务，因此不会执行不必要的逻辑。

`xTaskToNotify`结构的`xTaskToNotify`成员可以从任务和中断服务程序中访问，需要考虑处理器将如何更新它的值：

- 如果`xTaskToNotify`通过一个内存写操作更新，那么它可以在临界区外更新，如清单155所示。如果`xTaskToNotify`是一个32位变量(TaskHandle_t是一个32位类型)，并且运行FreeRTOS的处理器是一个32位处理器，就会出现这种情况。
- 如果更新`xTaskToNotify`需要一个以上的内存写入操作，则`xTaskToNotify`只能从关键部分中更新，否则中断服务例程可能会在`xTaskToNotify`处于不一致状态时访问`xTaskToNotify`。如果`xTaskToNotify`是32位变量，并且FreeRTOS运行的处理器是16位处理器，则会出现这种情况，因为它需要两个16位内存写入操作来更新所有32位。

在FreeRTOS实现内部，TaskHandle_t是一个指针，所以`sizeof(TaskHandle_t)`总是等于`sizeof(void *)`。

```groovy
/*驱动程序库函数，用于向UART发送数据。*/
BaseType_t xUART_Send(xUART *pxUARTInstance, uint8_t *pucDataSource, size_t uxLength)
{
BaseType_t xReturn;
    
    /*保存调用此函数的任务的句柄。这本书的正文包含了关于以下行是否需要受关键部分保护的注释。*/
    pxUARTInstance->xTaskToNotify = xTaskGetCurrentTaskHandle();

    /*通过调用ulTaskNotifyTake()，将xClearCountOnExit参数设置为pdTRUE，并将阻塞时间设置为0（不阻塞），确保调用任务尚未挂起通知。*/
    ulTaskNotifyTake(pdTRUE, 0);
    
    /*启动变速器。*/
    UART_low_level_send(pxUARTInstance, pucDataSource, uxLength);
    
    /*阻塞，直到通知传输完成。如果收到通知，则xReturn将设置为1，因为ISR已将此任务的通知值增加为1（pdTRUE）。如果操作超时，则xReturn将为0（pdFALSE），因为此任务的通知值在上面清除为0后不会更		改。请注意，如果ISR在对UART_low_level_send()的调用和对ulTaskNotifyTake()的调用之间执行，则事件将锁定在任务的通知值中，对ulTaskNotifyTake()的调用将立即返回。*/
    xReturn = (BaseType_t)ulTaskNotifyTake(pdTRUE, pxUARTInstance->xTxTimeout);
    
    return xReturn;
}
/*-----------------------------------------------------------*/

/*最后一个字节发送到UART后执行的ISR。*/
void xUART_TransmitEndISR(xUART *pxUARTInstance)
{
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    /*除非有任务等待通知，否则不应执行此函数。使用断言测试此条件。严格来说，此步骤不是必需的，但有助于调试。第11.2节介绍了configASSERT()。*/
    configASSERT(pxUARTInstance->xTaskToNotify != NULL);

    /*清除中断。*/
    UART_low_level_interrupt_clear(pxUARTInstance);
    
    /*直接向名为xUART_Send()任务发送通知。如果任务在等待通知时被阻止，则该任务将从阻止状态中删除。*/
    vTaskNotifyGiveFromISR(pxUARTInstance->xTaskToNotify, &xHigherPriorityTaskWoken);

    /*现在没有任务等待通知。将xUART结构的xTaskToNotify成员设置回NULL。此步骤并非绝对必要，但有助于调试。*/
    pxUARTInstance->xTaskToNotify = NULL;
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单155. 演示如何在驱动程序库传输函数中使用任务通知的伪代码

任务通知还可以替换接收函数中的信号量，如伪代码清单156所示，它提供了一个RTOS感知库函数的概要，该库函数在UART端口上接收数据。参见清单156：

- `xUART_Receive()`函数不包含任何互斥逻辑。如果有多个任务要使用`xUART_Receive()`函数，那么应用程序编写器必须管理应用程序本身中的互斥。例如，在调用`xUART_Receive()`之前，可能需要一个任务来获取互斥体。
- UART的接收中断服务程序将UART接收到的字符放到RAM缓冲区中。函数的作用是:从RAM缓冲区返回字符。

- `xUART_Receive() uxWantedBytes`参数用于指定要接收的字符数。如果RAM缓冲区尚未包含请求的数字字符，则调用任务将处于阻止状态，以等待通知缓冲区中的字符数已增加。`while()`循环用于重复此序列，直到接收缓冲区包含请求的字符数，或者发生超时。
- 调用任务可能多次进入阻塞状态。因此，将块时间调整为考虑自调用`xUART_Receive()`以来已经经过的时间。这些调整确保在`xUART_ Receive()`内花费的总时间不会超过由xUART结构的`xRxTimeout`成员指定的块时间。块时间是使用`FreeRTOS VTaskSetTimeOutState()`和`xTaskCheckForTimeOut()`辅助函数调整的。

```groovy
/*用于从UART接收数据的驱动程序库函数。*/
size_t xUART_Receive(xUART *pxUARTInstance, uint8_t *pucBuffer, size_t uxWantedBytes)
{
size_t uxReceived = 0;
TickType_t xTicksToWait;
TimeOut_t xTimeOut;
    
    /*记录输入此功能的时间。*/
    vTaskSetTimeOutState(&xTimeOut);
    
    /*xTicksToWait是超时值-它最初设置为此UART实例的最大接收超时。*/
    xTicksToWait = pxUARTInstance->xRxTimeout;
    
    /*保存调用此函数的任务的句柄。这本书的正文包含了关于以下行是否需要受关键部分保护的注释。*/
    pxUARTInstance->xTaskToNotify = xTaskGetCurrentTaskHandle();
    
    /*循环，直到缓冲区包含所需的字节数，或者发生超时。*/
    while (UART_bytes_in_rx_buffer(pxUARTInstance) < uxWantedBytes)
    {
        /*查找超时，调整xTicksToWait以说明到目前为止在该函数中花费的时间。*/
        if (xTaskCheckForTimeOut(&xTimeOut, &xTicksToWait) != pdFALSE)
        {
            /*在所需字节数可用之前超时，请退出循环。*/
            break;
        }
        
        /*接收缓冲区尚未包含所需的字节数。等待通知接收中断服务例程已将更多数据放入缓冲区的xTicksToWait ticks的最大值。调用此函数时，调用任务是否已挂起通知并不重要，如果已挂起，则只需围绕此循环进行一次额外的循环。*/
        ulTaskNotifyTake(pdTRUE, xTicksToWait);
    }
    
    /*没有任务等待接收通知，因此将xTaskToNotify设置回NULL。这本书的正文包含了关于以下行是否需要受关	 键部分保护的注释。*/
    pxUARTInstance->xTaskToNotify = NULL;
    
    /*尝试将uxWantedBytes从接收缓冲区读取到pucBuffer中。返回读取的实际字节数（可能小于uxWantedBytes）。*/
    uxReceived = UART_read_from_receive_buffer(pxUARTInstance, pucBuffer, uxWantedBytes);
    
    return uxReceived;
}
/*-----------------------------------------------------------*/

/*UART接收中断的中断服务程序。*/
void xUART_ReceiveISR(xUART *pxUARTInstance)
{
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    /*将接收到的数据复制到此UART的接收缓冲区并清除中断。*/
    UART_low_level_receive(pxUARTInstance);
    
    /*如果任务正在等待新数据的通知，请立即通知它。*/
    if (pxUARTInstance->xTaskToNotify != NULL)
    {
        vTaskNotifyGiveFromISR(pxUARTInstance->xTaskToNotify, &xHigherPriorityTaskWoken);
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
```

清单156. 演示如何在驱动程序库接收函数中使用任务通知的伪代码



### 外围设备驱动程序中使用的任务通知：ADC示例

上一节演示了如何使用`vTaskNotifyGiveFromISR()`将任务通知从中断发送到任务。`vTaskNotifyGiveFromISR()`是一个简单的函数，但其功能有限；它只能将任务通知作为无值事件发送，不能发送数据。本节演示如何使用`xTaskNotifyFromISR()`发送带有任务通知事件的数据。清单157中所示的伪代码演示了该技术，它提供了用于模数转换器（ADC）的RTOS感知中断服务例程的概要。在清单157中：

- 假设ADC转换至少每50毫秒启动一次。
- `ADC_ConversionEndISR()`是ADC转换结束中断的中断服务程序，它是每次有新的ADC值可用时执行的中断。

- 由`vADCTask()`实现的任务处理ADC生成的每个值。假设任务的句柄在创建任务时存储在`xADCTaskToNotify`中。
- `ADC_ConversionDisr()`使用`xTaskNotifyFromISR()`，并将eAction参数设置为`EsetValueWithoutOverwrite`，向`vADCTask()`任务发送任务通知，并将ADC转换的结果写入任务的通知值。

- `VADCTask()`任务使用`xTaskNotifyWait()`来等待收到新ADC值可用的通知，并从其通知值检索ADC转换的结果。

```groovy
/*使用ADC的任务。 */
void vADCTask(void *pvParameters)
{
uint32_t ulADCValue;
BaseType_t xResult;
    
    /*触发ADC转换的速率。*/
    const TickType_t xADCConversionFrequency = pdMS_TO_TICKS(50);
    
    for (;;)
    {
        /*等待下一个ADC转换结果。*/
        xResult = xTaskNotifyWait(
            /*新的ADC值将覆盖旧值，因此在等待新的通知值之前无需清除任何位。*/
            0,
            /*未来的ADC值将覆盖现有值，因此在退出xTaskNotifyWait()之前无需清除任何位。*/
            0,
            /*将任务的通知值（保存最新ADC转换结果）复制到的变量的地址。*/
            &ulADCValue,
            /*应在每个XADCConversionFrequenct滴答声中接收新的ADC值。*/
            xADCConversionFrequency * 2);

        if (xResult == pdPASS)
        {
            /*接收到新的ADC值。现在就处理它。*/
            ProcessADCResult(ulADCValue);
        }
        else
        {
            /*对xTaskNotifyWait()的调用未在预期时间内返回，触发ADC转换的输入或ADC本身一定有问题。在这里处理错误。*/
        }
    }
}
/*-----------------------------------------------------------*/

/*每次ADC转换完成时执行的中断服务例行程序。*/
void ADC_ConversionEndISR(xADC *pxADCInstance)
{
uint32_t ulConversionResult;
BaseType_t xHigherPriorityTaskWoken = pdFALSE, xResult;
    
    /*读取新的ADC值并清除中断。*/
    ulConversionResult = ADC_low_level_read(pxADCInstance);
    
    /*将通知和ADC转换结果直接发送到vADCTask()。*/
    xResult = xTaskNotifyFromISR(xADCTaskToNotify,          /* xTaskToNotify参数。*/
                                 ulConversionResult,        /* ulValue参数。*/
                                 eSetValueWithoutOverwrite, /* eAction参数。*/
                                 &xHigherPriorityTaskWoken);
    /*如果对xTaskNotifyFromISR()的调用返回pdFAIL，则该任务无法跟上生成ADC值的速率。第11.2节介绍了configASSERT()。*/
    configASSERT(xResult == pdPASS);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单157. 演示如何使用任务通知将值传递给任务的伪代码



### 直接在应用程序中使用的任务通知

本节通过在一个假设的应用程序中演示任务通知的使用来增强任务通知的功能，该应用程序包含以下功能：

1. 应用程序通过缓慢的互联网连接进行通信，向远程数据服务器发送数据，并从远程数据服务器请求数据。从这里开始，远程数据服务器被称为云服务器。
2. 从云服务器请求数据后，请求任务必须在阻塞状态下等待请求的数据被接收。

3. 发送任务向云服务器发送数据后，必须处于阻塞状态等待云服务器正确接收数据的确认。

软件设计原理图如图81所示。图81：

- 处理到云服务器的多个互联网连接的复杂性被封装在一个FreeRTOS任务中。该任务充当FreeRTOS应用程序中的代理服务器，被称为服务器任务。
- 应用程序任务通过调用`CloudRead()`从云服务器读取数据。`CloudRead()`不直接与云服务器通信，而是将读取请求发送到队列上的服务器任务，并从服务器任务接收请求的数据作为任务通知。

- 应用程序任务通过调用`CloudWrite()`将日期写入云服务器。`CloudWrite()`不直接与云服务器通信，而是将写请求发送到队列上的服务器任务，并从服务器任务接收写操作的结果作为任务通知。

由`ClouddRead()`和`CloudWrite()`函数发送到服务器任务的结构如清单158所示。

![img](https://cdn.nlark.com/yuque/0/2021/jpeg/23129846/1636352261318-a237622a-4f80-4a4b-8b49-99a0e9809e0b.jpeg)

图81  从应用程序任务到云服务器的通信路径

```groovy
typedef enum CloudOperations
{
    eRead, 						/*将数据发送到云服务器。*/
    eWrite 						/*从云服务器接收数据。*/
} Operation_t;
typedef struct CloudCommand
{
    Operation_t eOperation;     /*要执行的操作（读或写）。*/
    uint32_t ulDataID;          /*标识正在读取或写入的数据。*/
    uint32_t ulDataValue;       /*仅在将数据写入云服务器时使用。*/
    TaskHandle_t xTaskToNotify; /*执行操作的任务的句柄。*/
} CloudCommand_t;
```

清单158. 队列上发送到服务器任务的结构和数据类型

`CloudRead()`的伪代码如清单159所示。该函数将其请求发送到服务器任务，然后调用`xTaskNotifyWait()`以在阻塞状态下等待，直到通知请求的数据可用为止。

显示服务器任务如何管理读取请求的伪代码如清单160所示。当从云服务器接收到数据时，服务器任务将取消阻止应用程序任务，并通过调用`xTaskNotify()`将接收到的数据发送到应用程序任务，并将eAction参数设置为`eSetValueWithOverwrite`。

清单160显示了一个简化的场景，因为它假设`GetCloudData()`不必等待从云服务器获取值。

```groovy
/*ulDataID标识要读取的数据。pulValue保存从云服务器接收的数据要写入的变量的地址。*/
BaseType_t CloudRead(uint32_t ulDataID, uint32_t *pulValue)
{
CloudCommand_t xRequest;
BaseType_t xReturn;
    
    /*将CloudCommand_t结构成员设置为此读取请求正确。*/
    xRequest.eOperation = eRead;                          /*这是一个读取数据的请求。*/
    xRequest.ulDataID = ulDataID;                         /*标识要读取的数据的代码。*/
    xRequest.xTaskToNotify = xTaskGetCurrentTaskHandle(); /*调用任务的句柄。*/
    
    /*通过读取块时间为0的通知值，确保没有已挂起的通知，然后将结构发送到服务器任务。*/
    xTaskNotifyWait(0, 0, NULL, 0);
    xQueueSend(xServerTaskQueue, &xRequest, portMAX_DELAY);
    
    /*等待来自服务器任务的通知。服务器任务将从云服务器接收的值直接写入该任务的通知值，因此在进入或退出xTaskNotifyWait()函数时，无需清除通知值中的任何位。接收到的值被写入*pulValue，因此pulValue作为通知值写入的地址被传递。*/
    xReturn = xTaskNotifyWait(0,                   /*输入时没有清除位。*/
                              0,                   /*退出时没有要清除的位。*/
                              pulValue,            /*通知值转换为*pulValue。*/
                              pdMS_TO_TICKS(250)); /*最多等待250毫秒。*/

    /*如果xReturn是pdPASS，则获得该值。如果xReturn为pdFAIL，则请求超时。*/
    return xReturn;
}
```

清单159. 云读取API函数的实现

```groovy
void ServerTask(void *pvParameters)
{
    CloudCommand_t xCommand;
    uint32_t ulReceivedValue;
    for (;;)
    {
        /*等待从任务接收下一个CloudCommand\t结构。*/
        xQueueReceive(xServerTaskQueue, &xCommand, portMAX_DELAY);
        switch (xCommand.eOperation) /*是读还是写请求？*/
        {
        case eRead:
            /*从远程云服务器获取请求的数据项。*/
            ulReceivedValue = GetCloudData(xCommand.ulDataID);
            /*调用xTaskNotify()将通知和从云服务器接收的值发送给发出请求的任务。任务的句柄是从CloudCommand_t结构获得的。*/
            xTaskNotify(xCommand.xTaskToNotify, /*任务的句柄位于结构中。*/
                        ulReceivedValue,        /*作为通知值发送的云数据。*/
                        eSetValueWithOverwrite);
            break;

            /*其他开关箱在这里。*/
        }
    }
}
```

清单160. 处理读取请求的服务器任务

`CloudWrite()`的伪代码如清单161所示。出于演示的目的，`CloudWrite()`返回一个按位状态代码，其中状态代码中的每一位都被赋予一个唯一的含义。清单161顶部的`#define`语句显示了四个示例状态位。

任务清除四个状态位，将其请求发送到服务器任务，然后调用`xTaskNotifyWait()`以在阻塞状态下等待状态通知。

```groovy
/*云写入操作使用的状态位。*/
#define SEND_SUCCESSFUL_BIT 			( 0x01 << 0 )
#define OPERATION_TIMED_OUT_BIT 		( 0x01 << 1 )
#define NO_INTERNET_CONNECTION_BIT 		( 0x01 << 2 )
#define CANNOT_LOCATE_CLOUD_SERVER_BIT 	        ( 0x01 << 3 )

/*设置了四个状态位的掩码。*/
#define CLOUD_WRITE_STATUS_BIT_MASK 	( SEND_SUCCESSFUL_BIT |
 			                  OPERATION_TIMED_OUT_BIT |
			                  NO_INTERNET_CONNECTION_BIT | 
 			                  CANNOT_LOCATE_CLOUD_SERVER_BIT ) 

uint32_t CloudWrite( uint32_t ulDataID, uint32_t ulDataValue )
 {
 CloudCommand_t xRequest;
 uint32_t ulNotificationValue;
     
     /*将CloudCommand_t结构成员设置为此写入请求正确。*/
     xRequest.eOperation = eWrite;                         /*这是一个写入数据的请求。*/
     xRequest.ulDataID = ulDataID;                         /*标识正在写入的数据的代码。*/
     xRequest.ulDataValue = ulDataValue;                   /*写入云服务器的数据的值。*/
     xRequest.xTaskToNotify = xTaskGetCurrentTaskHandle(); /*调用任务的句柄。*/
     
     /*通过调用xTaskNotifyWait()，将ulBitsToClearOnExit参数设置为CLOUD_write_status_BIT_MASK，并将块时间设置为0，清除与写入操作相关的三个状态位。当前通知值不是必需的，因此					  pulNotificationValue参数设置为NULL。*/
     xTaskNotifyWait(0, CLOUD_WRITE_STATUS_BIT_MASK, NULL, 0);
     
     /*将请求发送到服务器任务。*/
     xQueueSend(xServerTaskQueue, &xRequest, portMAX_DELAY);
     
     /*等待来自服务器任务的通知。服务器任务将按位状态代码写入此任务的通知值，该值将写入`ulNotificationValue`。*/
     xTaskNotifyWait(0,                           /*输入时没有清除位。*/
                     CLOUD_WRITE_STATUS_BIT_MASK, /*退出时将相关位清除为0。*/
                     &ulNotificationValue,        /*通知值。*/
                     pdMS_TO_TICKS(250));         /*最多等待250毫秒。*/

     /*将状态代码返回给调用任务。*/
     return (ulNotificationValue & CLOUD_WRITE_STATUS_BIT_MASK);
```

清单161. Cloud Write API函数的实现

清单162显示了演示服务器任务如何管理写入请求的伪代码。将数据发送到云服务器后，服务器任务将取消阻止应用程序任务，并通过调用`xTaskNotify()`将按位状态代码发送到应用程序任务，并将eAction参数设置为eSetBits。只有`CLOUD_WRITE_STATUS_BIT_MASK`常量定义的位才能在接收任务的通知值中更改，因此接收任务可以将其通知值中的其他位用于其他目的。

清单162显示了一个简化的场景，因为它假设`SetCloudData()`不必等待从远程云服务器获得确认。

```groovy
void ServerTask(void *pvParameters)
{
CloudCommand_t xCommand;
uint32_t ulBitwiseStatusCode;
    
    for (;;)
    {
        /*等待下一条消息。*/
        xQueueReceive(xServerTaskQueue, &xCommand, portMAX_DELAY);
        
        /*是读还是写请求？*/
        switch (xCommand.eOperation)
        {
        case eWrite:
            
            /*将数据发送到远程云服务器。SetCloudData()返回一个按位状态代码，该代码只使用`CLOUD_WRITE_STATUS_BIT_MASK`定义定义的位（如清单161所示）。*/
            ulBitwiseStatusCode = SetCloudData(xCommand.ulDataID, xCommand.ulDataValue);
            
            /*向发出写入请求的任务发送通知。使用该操作时，ulBitwiseStatusCode中设置的任何状态位都将在被通知任务的通知值中设置。所有其他位保持不变。任务的句柄是从CloudCommand_t结构获得的。*/
            xTaskNotify(xCommand.xTaskToNotify, /*任务的句柄位于结构中。*/
                        ulBitwiseStatusCode,    /*作为通知值发送的云数据。*/
                        eSetBits);
            break;

            /*其他开关箱在这里。*/
        }
    }
}
```

清单162.  处理发送请求的服务器任务