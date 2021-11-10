# **事件组**



## 引言与范围

人们已经注意到，实时嵌入式系统必须对事件采取行动。前几章描述了FreeRTOS允许事件与任务通信的特性。这类特性的例子包括信号量和队列，它们都具有以下属性:

- 它们允许任务在阻塞状态中等待单个事件的发生。
- 当事件发生时，它们解除阻塞单个任务——解除阻塞的任务是等待事件的最高优先级的任务。

事件组是FreeRTOS的另一个特性，它允许事件与任务通信。与队列和信号量不同：

- 事件组允许任务在阻塞状态下等待多个事件的组合发生。
- 事件组在事件发生时解除等待同一事件或事件组合的所有任务的阻塞。

事件组的这些独特的性质使其有用的多个任务同步广播事件的多个任务,允许任务阻塞状态等待任何发生的一系列事件之一,并允许一个任务在阻塞状态等多个操作完成。

事件组还提供了减少应用程序使用的RAM的机会，因为通常可以用单个事件组替换许多二进制信号量。

事件组功能是可选的。要包含事件组功能，构建FreeRTOS源文件`event_groups.c`作为项目的一部分。



### 范围

本章旨在让读者更好地理解:

- 事件组的实际用途。
- 事件组相对于其他FreeRTOS特性的优点和缺点。

- 如何设置事件组中的位。
- 如何在阻塞状态下等待事件组中的位被设置。

- 如何使用事件组同步一组任务。



## 事件组的特征

### 事件组、事件标志和事件位

事件 "标志 "是一个布尔值（1或0），用于指示一个事件是否发生。事件 "组 "是一组事件标志。

事件标志只能为1或0，允许事件标志的状态存储在单个位中，事件组中所有事件标志的状态存储在单个变量中；事件组中每个事件标志的状态由类型为EventBits_t的变量中的单个位表示。因此，事件标志也被称为事件“位”。如果EventBits_t变量中的一位被设为1，则该位表示的事件已经发生。如果在EventBits_t变量中一个位被设置为0。那么由该位表示的事件没有发生。

图71显示了如何将单个事件标志映射到类型变量中的单个位EventBits_t。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636082654638-c4484f4f-5c5b-4348-bcbd-3d96004c940b.png)

图71 EventBits_t类型变量中的事件标志到位号的映射

例如，如果一个事件组的值是0x92(二进制1001 0010)，那么只有事件位1、4和7被设置，因此只有由1、4和7表示的事件发生。图72显示了EventBits_t类型的变量，设置了事件位1、4和7，并清除了所有其他事件位，使事件组的值为0x92。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636082893303-94a007a8-cf5e-46b1-a4f5-caab3caaed2c.png)

图72 一个事件组，其中只设置了1、4和7位，并且清除了所有其他事件标志，使事件组的值为0x92

由应用程序编写器为事件组中的单个位分配意义。例如，应用程序编写器可以创建一个事件组，然后：

- 将事件组中的第0位定义为“已从网络接收到消息”。
- 将事件组中的第1位定义为“消息已准备好发送到网络上”。

- 将事件组中的第2位定义为“终止当前网络连接”。

关于EventBits_t数据类型的更多信息

事件组中的事件位数依赖于`FreeRTOSConfig.h`中的`configUSE_16_BIT_ TICKS`编译时配置常量：

- 如果`configUSE_16_BIT_TICKS`为1，则每个事件组包含8个可用的事件位。
- 如果`configUSE_16_BIT_TICKS`为0，则每个事件组包含24个可用的事件位。

> FreeRTOSConfig.h：configUSE_16_BIT_TICKS配置用于保存RTOS滴答数的类型，因此似乎与事件组特性无关。它对EventBits_t类型的影响是FreeRTOS内部实现的结果，当FreeRTOS在一个能够比32位tvpes更有效地处理16位类型的架构上执行时，configUSE_16_BIT_TICKS应该只设置为1。



### 多任务访问

事件组是具有自身权限的对象，可以被任何知道其存在的任务或ISR访问。任意数量的任务可以在同一事件组中设置位，任意数量的任务可以从同一事件组中读取位。



### 一个使用事件组的实例

FreeRTOS+TCP TCP/IP栈的实现提供了一个实例，说明如何使用事件组同时简化设计，并最小化资源使用。

一个TCP套接字必须响应许多不同的事件。事件的例子包括接受事件、绑定事件、读取事件和关闭事件。套接字在任何给定时间所期望的事件取决于套接字的状态。例如，如果套接字已经创建，但还没有绑定到地址，那么它可以预期接收绑定事件，但不会预期接收读事件(如果没有地址，它就无法读取数据)。

一个FreeRTOS+TCP套接字的状态保存在一个叫做FreeRTOS_Socket_t的结构中。该结构包含一个事件组，该事件组为套接字必须处理的每个事件定义一个事件位。FreeRTOS+TCP API调用这个阻塞来等待一个事件或一组事件，只是在事件组上阻塞。

事件组还包含一个中止位，允许TCP连接被终止，不管套接字当时在等待哪个事件。



## 使用事件组的事件管理

### xEventGroupCreate() API函数

FreeRTOS V9.0.0还包括`xEventGroupCreateStatic()`函数，该函数在编译时分配静态创建事件组所需的内存:在使用事件组之前，必须显式地创建它。

使用EventGroupHandle_t类型的变量引用事件组。API函数`xEventGroupCreate()`用于创建事件组，并返回一个EventGroupHandle_t来引用它创建的事件组。

```groovy
EventGroupHandle_t xEventGroupCreate( void );
```

清单 132.  `xEventGroupCreate()` API函数原型

表 42. `xEventGroupCreate() `的返回值

| 参数名 | 描述                                                         |
| :----- | ------------------------------------------------------------ |
| 返回值 | 如果返回`NULL`，则不能创建事件组，因为没有足够的堆内存供FreeRTOS分配事件组数据结构。第2章提供了堆内存管理的更多信息。如果返回非null值，则表示事件组创建成功。返回值应该存储为创建的事件组的句柄。 |



### xEventGroupSetBits() API函数

`xEventGroupSetBits()` API函数在事件组中设置一个或多个位，通常用于通知任务，由被设置的位表示的事件已经发生。

> 注意:永远不要在中断服务程序中调用`xEventGroupSetBits()`。应该使用中断安全版本`xEventGroupSetBitsFromISR()`来代替它。

```groovy
EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup,
								const EventBits_t uxBitsToSet );
```

清单133.  `xEventGroupSetBits()` API函数原型

表 43.  `xEventGroupSetBits()`参数和返回值

| 参数名      | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `xEventGroup` | 要在其中设置位的事件组的句柄。事件组句柄已经从用于创建事件组的`xEventGroupCreate()`调用中返回。 |
| `uxBitsToSet` | 一个位掩码，用于指定事件组中的事件位或事件位设置为1。事件组的值通过用`uxBitsToSet`传递的值按位或事件组的现有值来更新。例如，将`uxBitsToSe`t设置为0x04(二进制0100)将导致事件组中的事件位3被设置(如果它还没有被设置)，而事件组中的所有其他事件位保持不变。 |
| 返回值      | 调用`xEventGroupSetBits()`返回时事件组的值。注意，返回的值不一定是`uxBitsToSet`指定的位，因为这些位可能已经被另一个任务再次清除。 |



### xEventGroupSetBitsFromlSR() API函数

`xEventGroupSetBitsFromlSR()`是`xEventGroupSetBits()`的中断安全版本。

给出信号量是一个确定性操作，因为预先知道给出信号量最多会导致一个任务离开阻塞状态。当在事件组中设置位时，不知道有多少任务将离开阻塞状态，因此在事件组中设置位不是确定性操作。

FreeRTOS设计和实现标准不允许不确定的操作在一个中断服务程序执行,或者当中断禁用这个原因，`xEventGroupSetBitsFromlSR()`不直接设置事件比特在中断服务程序,而是延缓RTOS守护进程的行动任务。

```groovy
BaseType_t xEventGroupSetBitsFromISR( EventGroupHandle_t xEventGroup,
 									  const EventBits_t uxBitsToSet,
									  BaseType_t *pxHigherPriorityTaskWoken );
```

清单 134. `xEventGroupSetBitsFromISR()` API函数原型

表 44. `xEventGroupSetBitsFromISR()`参数和返回值

| 参数名                    | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `xEventGroup`               | 要在其中设置位的事件组的句柄。事件组句柄已经从用于创建事件组的`XEventGroupCreate()`调用中返回。 |
| `uxBitsToSet`               | 一个位掩码，用于指定事件组中的事件位或事件位设置为1。事件组的值通过用`uxBitsToSet`传递的值按位或事件组的现有值来更新。例如，将`uxBitsToSet`设置为0x05(二进制0101)将导致事件组中的事件位3和事件位0被设置(如果它们还没有被设置)，而事件组中的所有其他事件位保持不变。 |
| `pxHigherPriorityTaskWoken` | `xEventGroupSetBitsFromlSR()`不直接在中断服务例程中设置事件位，而是通过在定时器命令队列上发送命令，将操作推迟给RTOS守护进程任务。如果守护任务处于阻塞状态以等待定时器命令队列上的数据可用，那么写入定时器命令队列将导致守护任务离开阻塞状态。如果守护进程任务的优先级高于当前正在执行的任务(被中断的任务)的优先级，那么，`xEventGroupSetBitsFromlSR()`将在内部将`*pxHigherPriorityTaskWoken`设置为pdTRUE。如果`xEventGroupSetBitsFromlSR()`将这个值设置为pdTRUE，那么应该在中断退出之前执行上下文切换。这将确保中断直接返回到守护任务，因为守护任务将是最高优先级的就绪状态任务。 |
| 返回值                    | 有两个可能的返回值：<br/>pdPASS<br/>pdPASS只在数据成功发送到定时器命令队列时返回。<br/>pdFALSE<br/>如果设置位命令不能写入定时器命令队列，因为队列已经满了，将返回pdFALSE。|

### xEventGroupWaitBits() API函数

`xEventGroupWaitBits()` APl函数允许任务读取事件组的值，并且可以选择在阻塞状态等待事件组中的一个或多个事件位被设置，如果这些事件位还没有设置。

```groovy
EventBits_t xEventGroupWaitBits( const EventGroupHandle_t xEventGroup,
 								 const EventBits_t uxBitsToWaitFor,
 								 const BaseType_t xClearOnExit,
 								 const BaseType_t xWaitForAllBits,
 								 TickType_t xTicksToWait );
```

清单135.  `xEventGroupWaitBits()` API函数原型

调度程序用来确定任务是否进入阻塞状态，以及任务何时离开阻塞状态的条件称为“解封条件”。解封条件由`uxBitsToWaitFor`和`xWaitForAllBits`参数值的组合指定：

- `uxBitsToWaitFor`指定要测试事件组中的哪个事件位
- `xWaitForAllBits`指定是使用位数OR测试，还是位数AND测试。

如果在调用`xEventGroupWaitBits()`时，任务的解锁条件得到满足，那么该任务将不会进入阻塞状态。

表45提供了导致任务进入阻塞状态或退出阻塞状态的条件示例。表45只显示了事件组和`uxBitsToWaitFor`值中最不重要的四个二进制位——这两个值的其他位被假定为零。

表45 `uxBitsToWaitFor`和`xWaitForAllBits`参数的影响

| Existing Event Group Value | uxBitsToWaitFor value | xWaitForAllBitsvalue | Resultant Behavior                                           |
| -------------------------- | --------------------- | -------------------- | ------------------------------------------------------------ |
| 0000                       | 0101                  | pdFALSE              | 由于在事件组中没有设置0位或2位，调用任务将进入阻塞状态，并且在事件组中设置0位或2位时将离开阻塞状态。 |
| 0100                       | 0101                  | pdTRUE               | 由于0位和2位没有同时设置在事件组中，调用任务将进入阻塞状态，当0位和2位同时设置在事件组中，调用任务将离开阻塞状态。 |
| 0100                       | 0110                  | pdFALSE              | 调用任务不会进入阻塞状态，因为`xWaitForAllBits`是pdFALSE，并且`uxBitsToWaitFor`指定的两个位中的一个已经在事件组中设置。 |
| 0100                       | 0110                  | pdTRUE               | 由于`xWaitForAllBits`为pdTRUE，并且`uxBitsToWaitFor`指定的两个位中只有一个已经在事件组中设置，因此调用任务将进入阻塞状态。当事件组中的第2位和第3位都被设置时，任务将离开阻塞状态。 |

调用任务使用`uxBitsToWaitFor`参数指定要测试的位，调用任务很可能需要在满足解封条件后将这些位清除为零。事件位可以使用`xEventGroupClearBits()` API函数来清除，但如果使用该函数手动清除事件位将导致应用程序代码中的竞争条件：

- 使用同一事件组的任务不止一个。
- 位由不同的任务或中断服务程序在事件组中设置。

提供了`xClearOnExit`参数以避免这些潜在的竞争条件。如果`xClearOnExit`设置为pdTRUE，那么对调用任务来说，事件位的测试和清除似乎是一个原子操作(不能被其他任务或中断中断)。

表46 `xEventGroupWaitBits()`参数和返回值

| 参数名            | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| `xEventGroup`     | 事件组的句柄，其中包含正在读取的事件位。事件组句柄已经从用于创建事件组的`xEventGroupCreate()`调用中返回。 |
| `uxBitsToWaitFor` | 指定事件组中要测试的事件位或事件位的位掩码。例如，如果调用任务想要等待事件位0和/或事件位2在事件组中被设置，那么将`uxBitsToWaitFor`设置为0x05(二进制0101)。更多的例子请参见表45。 |
| `xClearOnExit`    | 如果调用任务的解封条件已经被满足，并且`xClearOnExit`被设置为pdTRUE，那么在调用任务退出`xEventGroupWaitBits()` API函数之前，`uxBitsToWaitFor`指定的事件位将被清除回事件组中的0。如果`xClearOnExit`设置为pdFALSE，则事件组中的事件位的状态不会被`xEventGroupWaitBits()` API函数修改。 |
| `xWaitForAllBits` | `uxBitsToWaitFor`参数指定要在事件组中测试的事件位。`xWaitForAllBits`指定当`uxBitsToWaitFor`参数指定的一个或多个事件位被设置时，或者只有当`uxBitsToWaitFor`参数指定的所有事件位被设置时，调用任务才应该从阻塞状态中删除。如果`xWaitForAllBits`为pdFALSE,那么一个任务进入阻塞状态等待其开启条件满足时将阻塞状态的任何部分规定`uxBitsToWaitFor`成为集(或指定的超时`xTicksToWait`参数到期)。示例请参见表45。 |
| `xTicksToWait`    | 任务保持阻塞状态以等待其解除阻塞条件满足的最大时间。如果`xTicksTolWait`为零，或者在调用`xEventGroupWaitBits()`时满足解封条件，则`xEventGroupWaitBits()`将立即返回。块时间以滴答周期指定，因此它所代表的绝对时间依赖于滴答频率。宏`pdMS_TO_TICKS()`可用于将以毫秒为单位指定的时间转换为以ticks为单位指定的时间。将`xTicksToWait`设置为`portMAX_DELAY`将导致任务无限期等待(不会超时)，前提是在`FreeRTOSConfig.h`中将`INCLUDE_vTaskSuspend`设置为1。 |
| 返回值            | 如果`xEventGroupWaitBits()`返回是因为调用任务的解封条件被满足，那么返回值是调用任务的解封条件被满足时的事件组的值(在`xClearOnExit`为pdTRUE时自动清除任何位之前)。在这种情况下，返回值也将满足解封条件。如果`xEventGroupWaitBits()`返回是因为`xTicksToWait`参数指定的块时间过期，那么返回的值是块时间过期时事件组的值。在这种情况下，返回值将不满足解封条件。 |



### 示例22. 实验事件组

这个例子演示了如何进行：

- 创建事件组。
- 从中断服务程序中设置事件组中的位。

- 从任务中设置事件组中的位。
- 阻塞事件组。

`xEventGroupWaitBits()` `xWaitForAllBits`参数的效果是通过首先执行`xWaitForAllBits`设置为pdFALSE的示例，然后执行`xWaitForAllBits`设置为pdTRUE的示例来演示的。

事件位0和事件位1由一个任务设置。事件位2是由中断服务程序设置的。例程设置。这三个位通过#define语句被赋予描述性的名字，如清单136所示。

```groovy
/*"事件组中事件位的定义。 */
#define mainFIRST_TASK_BIT  ( 1UL << 0UL ) /* 事件位0，由任务设置。 */
#define mainSECOND_TASK_BIT ( 1UL << 1UL ) /* 事件位1，由任务设置。 */
#define mainISR_BIT         ( 1UL << 2UL ) /* 事件位2，由ISR设置。 */
```

清单136. 示例22中使用的事件位定义

清单137显示了设置事件位0和事件位1的任务的实现。它位于一个循环中，反复设置一个位，然后再设置另一个位，每次调用`xEventGroupSetBits()`之间有200毫秒的延迟。在设置每个位之前打印一个字符串，以允许在控制台中看到执行序列。

```groovy
static void vEventBitSettingTask( void *pvParameters )
{
const TickType_t xDelay200ms = pdMS_TO_TICKS( 200UL ), xDontBlock = 0;
	for( ;; )
	{
 		/*在开始下一个循环之前稍作延迟。 */
 		vTaskDelay( xDelay200ms );
        
 		/*输出一条消息，表示任务即将设置事件位0，然后设置事件位0。*/
 		vPrintString( "Bit setting task -\t about to set bit 0.\r\n" );
 		xEventGroupSetBits( xEventGroup, mainFIRST_TASK_BIT );
        
 		/*在设置其他位之前稍作延迟。 */
 		vTaskDelay( xDelay200ms );
        
 		/*输出一个消息，说事件位1即将被任务设置，然后设置事件位1。*/
 		vPrintString( "Bit setting task -\t about to set bit 1.\r\n" );
 		xEventGroupSetBits( xEventGroup, mainSECOND_TASK_BIT );
 	} 
}
```

清单137.  例22中设置事件组中两个比特的任务

清单138显示了在事件组中设置第2位的中断服务例程的实现。同样，在设置位之前打印一个字符串，以允许在控制台中看到执行序列。但是，在这种情况下，因为控制台输出不应该直接在中断服务例程中执行，所以使用`xTimerPendFunctionCallFromISR()`在RTOS守护进程任务的上下文中执行输出。

与前面的示例一样，中断服务程序是由一个简单的周期性任务触发的，该任务强制软件中断。在本例中，中断每500毫秒产生一次。

```groovy
static uint32_t ulEventBitSettingISR( void )
{
/* 该字符串没有在中断服务程序中打印，而是被发送到RTOS守护进程任务中打印。因此，它被声明为static，以确保编译器不会在ISR的堆栈上分配字符串，因为当从守护进程任务打印字符串时，ISR的堆栈帧将不存在。*/
static const char *pcString = "Bit setting ISR -\t about to set bit 2.\r\n";
BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    /*输出一个消息说第2位即将被设置。消息不能从ISR打印，因此通过挂起函数调用在RTOS守护进程任务上下文中运行，将实际输出推迟到RTOS守护进程任务。*/
 	xTimerPendFunctionCallFromISR( vPrintStringFromDaemonTask, 
 				( void * ) pcString, 
 				0, 
				&xHigherPriorityTaskWoken );
    
    /* 在事件组中设置位2。 */
 	xEventGroupSetBitsFromISR( xEventGroup, mainISR_BIT, &xHigherPriorityTaskWoken );
    
    /* xTimerPendFunctionCallFromISR()和xEventGroupsetBitsFromISR()都写定时器命令队列，并使用相同的xHigherPriorityraskwoken变量。如果写入定时器命令队列导致RTOS守护进程任务，并且RTOS守护进程任务的优先级较高，则返回Blocked状态比当前正在执行的任务(此中断的任务)的优先级高然后xhigherprioritytaskkoken将被设置为pdTRUE。
    
    xHigherPriorityTaskwoken用作portYIELD FROM ISR()的参数。如果xHigherPriorityraskwoken等于pdTRUE，那么从ISR()调用portYIEID将请求一个上下文切换。如果xHigherPriorityraskwoken仍然是pdFALSE，那么从ISR()调用portYIELD将没有效果。
    
    由windows端口使用的portYIELD_FROM_ISR()的实现包括一个返回语句，这就是为什么这个函数没有显式返回值。*/
	portYIELD_FROM_ISR( xHigherPriorityTaskWoken );
}
```

清单138.  例22中设置事件组中第2位的ISR

清单139显示了调用`xEventGroupWaitBits()`来阻塞事件组的任务的实现。任务为事件组中设置的每个位打印一个字符串。

`xEventGroupWaitBits()` `xClearOnExit`参数被设置为pdTRUE，因此导致调用`xEventGroupWaitBits()`返回的事件位或位将在`xEventGroupWaitBits()`返回之前被自动清除。

```groovy
static void vEventBitReadingTask( void *pvParameters )
{
EventBits_t xEventGroupValue;
const EventBits_t xBitsToWaitFor = ( mainFIRST_TASK_BIT  | 
 				     mainSECOND_TASK_BIT | 
 				     mainISR_BIT );
 	for( ;; )
 	{
        /* 阻塞以等待事件位在事件组中被设置。*/
 		xEventGroupValue = xEventGroupWaitBits( /* 要读取的事件组。 */
 							xEventGroup,
 							/* 位测试。 */
 							xBitsToWaitFor, 

 							/*如果满足解封条件，则在退出时清除位。*/
 							pdTRUE,    

 							/*不要等待所有的位。对于第二次执行，该参数被设置为pdTRUE。 */
 							pdFALSE,   

 							/* 不要超时。 */
 							portMAX_DELAY );

 		/*为设置的每个位打印一条消息。 */
 		if( ( xEventGroupValue & mainFIRST_TASK_BIT ) != 0 )
 		{
 			vPrintString( "Bit reading task -\t Event bit 0 was set\r\n" );
 		}
 		if( ( xEventGroupValue & mainSECOND_TASK_BIT ) != 0 )
 		{
 			vPrintString( "Bit reading task -\t Event bit 1 was set\r\n" );
 		}
 		if( ( xEventGroupValue & mainISR_BIT ) != 0 )
 		{
 			vPrintString( "Bit reading task -\t Event bit 2 was set\r\n" );
 		}
 	} 
}
```

清单139. 例22中等待事件位被设置而阻塞的任务

`main()`函数在启动调度程序之前创建事件组和任务。有关它的实现，请参见清单140。从事件组中进行读操作的任务优先级高于向事件组中进行写操作的任务优先级，确保每次满足读任务的解阻塞条件时，读任务都会抢占写任务。

```groovy
int main(void)
{
    /* 在使用事件组之前，必须先创建事件组。 */
    xEventGroup = xEventGroupCreate();
    
    /* 创建在事件组中设置事件位的任务。 */
    xTaskCreate(vEventBitSettingTask, "Bit Setter", 1000, NULL, 1, NULL);
    
    /* 创建等待事件组中事件位设置的任务。*/
    xTaskCreate(vEventBitReadingTask, "Bit Reader", 1000, NULL, 2, NULL);
    
    /* 创建用于周期性产生软件中断的任务。 */
    xTaskCreate(vInterruptGenerator, "Int Gen", 1000, NULL, 3, NULL);
    
    /*安装软件中断的处理程序。完成此操作所需的语法取决于所使用的FreeRTOS端口。这里显示的语法只能在FreeRros Windows端口中使用，在该端口中这种中断只能被模拟。 */
    vPortSetInterruptHandler(mainINTERRUPT_NUMBER, ulEventBitSettingISR);
    
    /* 启动调度程序，使创建的任务开始执行。 */
    vTaskStartScheduler();
    
    /* 启动调度程序，使已创建的任务开始执行。 */
    for (;;);
    return 0;
}
```

清单140. 创建例22中的事件组和任务

在执行示例22时，将`xEventGroupWaitBits()` xWaitForAllBits参数设置为pdFALSE，所产生的输出如图73所示。在图73中，可以看到，由于调用`xEventGroupWaitBits()`中的xWaitForAllBits参数被设置为pdFALSE，从事件组中读取的任务将离开阻塞状态，并在每次设置任何事件位时立即执行。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636101093971-7eb3bbab-0706-410b-a4db-9aac8e2f3dc6.png)

图73 示例22执行`xWaitForAllBits`设置为pdFALSE时产生的输出

在执行示例22时，将`xEventGroupWaitBits()` xWaitForAllBits参数设置为pdTRUE，所产生的输出如图74所示。在图74中可以看到，因为`xWaitForAllBits`参数被设置为pdTRUE，从事件组中读取的任务只有在设置了所有三个事件位之后才会离开状态阻塞。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636101269447-89193a38-41d7-4f19-be10-3e4ac2e23fb6.png)

图74 将`xWaitForAlBits`设置为pdTRUE执行示例22时产生的输出



## 使用事件组进行任务同步

有时，应用程序的设计需要两个或多个任务来彼此同步。例如，考虑这样一种设计:任务A接收一个事件，然后将该事件所需的一些处理委托给另外三个任务:任务B、任务C和任务D。如果任务A无法接收到另一个事件，直到任务B、C和D都完成了对前一个事件的处理，那么这四个任务就需要彼此同步。每个任务的同步点将在该任务完成其处理之后，并且不能继续进行，直到其他每个任务都完成了同样的处理。只有当所有四个任务都到达它们的同步点时，任务A才能接收到另一个事件。

需要这种类型的任务同步的一个不那么抽象的例子可以在一个FreeRTOS+TCP演示项目中找到。演示在两个任务之间共享一个TCP套接字;一个任务向套接字发送数据，另一个任务从同一个套接字1接收数据。在确定其他任务不会再次尝试访问该套接字之前，关闭TCP套接字对任何一个任务来说都是不安全的。如果两个任务中有一个希望关闭套接字，那么它必须通知另一个任务它的意图，然后等待另一个任务停止使用该套接字，然后再继续。清单140所示的伪代码演示了将数据发送到希望关闭套接字的任务的场景。

清单140所展示的情景是微不足道的，因为只有两个任务需要互相同步，但很容易看出，如果有其他任务在执行同步，该方案会变得更复杂，需要更多的任务加入同步，如果其他的任务在执行依赖于套接字的处理的话。处理依赖于套接字被打开。

> 套接字：在编写本文的时候，这是在任务之间共享单个FreeRTOS+TCP套接字的唯一方法。

```groovy
void SocketTxTask(void *pvParameters)
{
xSocket_t xSocket;
uint32_t ulTxCount = 0UL;
    
    for (;;)
    {
       /*创建一个新的socket。这个任务将发送到这个套接字，而另一个任务将从这个套接字接收。*/
        xSocket = FreeRTOS_socket(...);
        
        /* 连接套接字。 */
        FreeRTOS_connect(xSocket, ...);

        /* 使用队列将套接字发送到接收数据的任务。 */
        xQueueSend(xSocketPassingQueue, &xSocket, portMAX_DELAY);
        
        /* 在关闭套接字之前向套接字发送1000条消息。*/
        for (ulTxCount = 0; ulTxCount < 1000; ulTxCount++)
        {
            if (FreeRTOS_send(xSocket, ...) < 0)
            {
                /* 意外错误-退出循环，之后套接字将被关闭。 */
                break;
            }
        }
        
        /* 让Rx任务知道Tx任务想要关闭套接字。 */
        TxTaskWantsToCloseSocket();
        
        /* 这是Tx任务的同步点。Tx任务在这里等待Rx任务到达它的同步点。Rx任务只有在不再使用套接字时才会到达它的同步点，并且可以安全地关闭套接字。*/
        xEventGroupSync(...);
        
        /* 这两个任务都没有使用套接字。关闭连接，然后关闭套接字。*/
        FreeRTOS_shutdown(xSocket, ...);
        WaitForSocketToDisconnect();
        FreeRTOS_closesocket(xSocket);
    }
}
/*-----------------------------------------------------------*/

void SocketRxTask(void *pvParameters)
{
xSocket_t xSocket;
    
    for (;;)
    {
        /* 等待接收由Tx任务创建并连接的套接字。 */
        xQueueReceive(xSocketPassingQueue, &xSocket, portMAX_DELAY);
        
        /* 继续接收套接字，直到Tx任务想要关闭套接字。 */
        while (TxTaskWantsToCloseSocket() == pdFALSE)
        {
            /* 接收然后处理数据。 */
            FreeRTOS_recv(xSocket, ...);
            ProcessReceivedData();
        }
        
        /*这是Rx任务的同步点——它只在不再使用套接字时到达这里，因此Tx任务关闭套接字是安全的。*/
        xEventGroupSync(...);
    }
}
```

清单141.两个任务的伪代码，它们彼此同步以确保在套接字关闭之前，任何一个任务都不再使用共享的TCP套接字

事件组可用于创建同步点:

- 必须参与同步的每个任务在事件组中被分配一个唯一的事件位。
- 每个任务在到达同步点时设置自己的事件位。

- 设置了自己的事件位后，每个任务阻塞在事件组上，等待代表所有其他同步任务的事件位也被设置。

但是，`xEventGroupSetBits()`和`xEventGroupWaitBits()` API函数不能在此场景中使用。如果使用它们，那么位的设置(指示一个任务已经到达它的同步点)和位的测试(确定其他同步任务是否已经到达它们的同步点)将作为两个独立的操作执行。为了了解为什么这是一个问题，考虑一个场景，任务A任务B和任务C尝试使用事件组进行同步:

1. 任务A和任务B已经到达同步点，事件位已经在事件组中设置，处于阻塞状态，等待任务C的事件位也被设置。
2. 任务C到达同步点后，使用`xEventGroupSetBits()`设置其在事件组中的位。一旦任务C的位被设置，任务A和任务B就会离开阻塞状态，并清除所有三个事件位。

3. 任务C然后调用`xEventGroupWaitBits()`等待这三个事件比特成为集,但到那时,所有三个事件已经被清除,任务和任务B离开各自的同步点,所以同步失败了。

要成功地使用事件组创建同步点，事件位的设置和后续事件位的测试必须作为一个单独的不可中断操作执行。为此提供了`xEventGroupSync()` API函数。



### xEventGroupSync() API函数

提供`xEventGroupSync()`允许两个或多个任务使用事件组彼此同步。该功能允许任务在一个事件组中设置一个或多个事件位，然后等待在同一事件组中设置多个事件位的组合，作为一个单独的不可中断操作。

`xEventGroupSync()` uxBitsTolaitFor参数指定调用任务的解除阻塞条件。如果`xEventGroupSync()`返回是因为满足了不阻塞条件，`uxBitsTolaitFor`指定的事件位将在`xEventGroupSync()`返回之前被清除回零。

```groovy
EventBits_t xEventGroupSync(EventGroupHandle_t xEventGroup,
                            const EventBits_t uxBitsToSet,
                            const EventBits_t uxBitsToWaitFor,
                            TickType_t xTicksToWait);
```

清单142. `xEventGroupSync()` API函数原型

表47. `xEventGroupSync()`参数和返回值

| 参数名          | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `xEventGroup`     | 要在其中设置事件位然后测试的事件组的句柄。事件组句柄已经从用于创建事件组的`xEventGroupCreate()`调用中返回。 |
| `uxBitsToSet`     | 一个位掩码，用于指定事件组中的事件位或事件位设置为1。事件组的值通过用`uxBitsToSet`传递的值按位或事件组的现有值来更新。例如，将`uxBitsToSet`设置为0x04(二进制0100)将导致事件3位被设置(如果它还没有被设置)，而事件组中的所有其他事件位保持不变。 |
| `uxBitsToWaitFor` | 指定事件组中要测试的事件位或事件位的位掩码。例如，如果调用任务想要等待事件位0，1和2在事件组中被设置，那么将`uxBitsToWaitFor`设置为0x07(二进制111). |
| `xTicksToWait`    | 任务保持阻塞状态以等待其解除阻塞条件满足的最大时间。如果`xTicksToWait`为零，或者在调用`xEventGroupSync()`时满足解除条件，则`xEventGroupSync()`将立即返回。块时间以滴答周期指定，因此它所代表的绝对时间依赖于滴答频率。宏`pdMS_TO_TICKS()`可用于将以毫秒为单位的时间转换为以节拍为单位的时间。将`xTicksToWait`设置为`portMAX_DELAY`将导致任务无限期等待(不会超时)，前提是在`FreeRTOSConfig.h`中将`INCLUDE_vTaskSuspend`设置为1。 |
| 返回值          | 如果`xEventGroupSync()`返回是因为调用任务的阻塞条件被满足，那么返回的值是调用任务的阻塞条件被满足时的事件组的值(在任何位被自动清除回零之前)。在这种情况下，返回值也将满足调用任务的阻塞条件。如果`xEventGroupSync()`返回是因为`xTicksToWait`参数指定的块时间过期，那么返回的值是块时间过期时事件组的值。在这种情况下，返回值将不满足调用任务的阻塞条件。 |



### 示例23. 同步任务

示例23使用`xEventGroupSync()`同步一个任务实现的三个实例。任务参数用于将任务调用`xEventGroupSync()`时设置的事件位传递给每个实例。

任务在调用`xEventGroupsync()`之前打印一条消息，在调用`xEventGroupsync()`返回之后再次打印一条消息。每条消息都包含一个时间戳。这允许在生成的输出中观察执行顺序。伪随机延迟用于防止所有任务同时到达同步点。

有关任务的实现，请参见清单143。

```groovy
static void vSyncingTask(void *pvParameters)
{
    const TickType_t xMaxDelay = pdMS_TO_TICKS(4000UL);
    const TickType_t xMinDelay = pdMS_TO_TICKS(200UL);
    TickType_t xDelayTime;
    EventBits_t uxThisTasksSyncBit;
    const EventBits_t uxAllSyncBits = (mainFIRST_TASK_BIT |
                                       mainSECOND_TASK_BIT |
                                       mainTHIRD_TASK_BIT);

    /*创建该任务的三个实例-每个任务在同步中使用不同的事件位。使用的事件位通过任务参数传递到每个任务实例。将其存储在uxthisaskssyncbit变量中。*/
    uxThisTasksSyncBit = (EventBits_t)pvParameters;
    
    for (;;)
    {
        /*通过延迟一个伪随机时间来模拟这个任务花费一些时间来执行一个动作。这可以防止该任务的所有三个实例同时到达同步点，因此可以更容易地观察示例的行为。*/
        xDelayTime = (rand() % xMaxDelay) + xMinDelay;
        vTaskDelay(xDelayTime);

        /*打印一条消息，显示这个任务已经到达它的同步点。pcTaskGetTaskName()是一个API函数，它返回在创建任务时分配给任务的名称。*/
        vPrintTwoStrings(pcTaskGetTaskName(NULL), "reached sync point");

        /*等待所有任务到达各自的同步点。*/
        xEventGroupSync(/* 用于同步的事件组。 */
                        xEventGroup,
            
                        /* 该任务设置的位，表示它已到达同步点。 */
                        uxThisTasksSyncBit,
            
                        /*需要等待的比特，每个参与同步的任务都有一个位*/
                        uxAllSyncBits,
            
                        /*无限等待所有三个任务到达vnchronization点。*/
                        portMAX_DELAY);

        /*打印一条消息，显示这个任务已经通过了它的同步点。由于使用了无限期延迟，所以只有在所有任务到达各自的同步点后才会执行下面的行。*/
        vPrintTwoStrings(pcTaskGetTaskName(NULL), "exited sync point");
    }
}
```

清单143.  示例23中使用的任务的实现

`main()`函数创建事件组，创建所有三个任务，然后启动调度程序。有关它的实现，请参见清单144。

```groovy
/* 事件组中的事件位的定义。 */
#define mainFIRST_TASK_BIT (1UL << 0UL) /* 事件位0，由第一个任务设置。 */
#define mainSECOND_TASK_BIT(1UL << 1UL) /* 事件位1，由第二个任务设置。 */
#define mainTHIRD_TASK_BIT (1UL << 2UL) /* 事件位2，由第三个任务设置。 */

/* 声明用于同步三个任务的事件组。 */
EventGroupHandle_t xEventGroup;

int main(void)
{
    /* 在使用事件组之前，必须先创建它。 */
    xEventGroup = xEventGroupCreate();
    
    /*创建任务的三个实例。每个任务都有一个不同的名称，稍后将打印出来，以直观地指示正在执行的任务。当任务到达其同步点时要使用的事件位使用任务参数传递给任务。*/
    xTaskCreate(vSyncingTask, "Task 1", 1000, mainFIRST_TASK_BIT, 1, NULL);
    xTaskCreate(vSyncingTask, "Task 2", 1000, mainSECOND_TASK_BIT, 1, NULL);
    xTaskCreate(vSyncingTask, "Task 3", 1000, mainTHIRD_TASK_BIT, 1, NULL);
    
    /* 启动调度器，使创建的任务开始执行。 */
    vTaskStartScheduler();
    
    /* 像往常一样，永远不要达到下面这行。 */
    for (;;);
    return 0;
}
```

清单144. 示例23中使用的`main()`函数

执行示例23时产生的输出如图75所示。可以看到，即使每个任务在不同的(伪随机)时间到达同步点，每个任务在同一时间退出同步点1(这是最后一个任务到达同步点的时间)。

> 同步点：图75显示了在FreeRTOS Windows端口中运行的示例，它不提供真正的实时行为(特别是当使用Windows系统调用打印到控制台时)，因此会显示一些时间变化。



![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636106241870-6e67e703-5286-4d20-ad9b-6e847b751628.png)

图75执行示例23时产生的输出



