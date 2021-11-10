# **中断管理**



## 章节介绍和范围

### 事件

嵌入式实时系统必须对源自环境的事件做出响应。例如，到达以太网外设的数据包（事件）可能需要传递到 TCP/IP 堆栈进行处理（操作）。非琐碎的系统必须服务于来自多个源的事件，所有这些源都有不同的处理开销和响应时间要求。在每种情况下，都必须对最佳事件处理实施策略作出判断：

1. 如何检测事件？通常使用中断，但也可以轮询输入。
2. 当使用中断时，中断服务程序（ISR）内部应执行多少处理，外部应执行多少处理？通常希望每个 ISR尽可能短。

3. 如何将事件与主（非 ISR）代码通信，以及如何构造此代码以最佳地适应潜在异步事件的处理？

FreeRTOS 不会对应用程序设计者强加任何特定的事件处理策略，但确实提供了允许以简单且可维护的方式实施所选策略的功能。



重要的是区分任务的优先级和中断的优先级：

- 任务是与运行 FreeRTOS 的硬件无关的软件功能。 应用程序编写者在软件中分配任务的优先级，软件算法（调度程序）决定哪个任务将处于运行状态。
- 尽管是用软件编写的，但中断服务程序是硬件特性，因为硬件控制将运行哪个中断服务程序以及何时运行。 任务只有在没有 ISR 运行时才会运行，所以最低优先级的中断会中断最高优先级的任务，任务没有办法抢占 ISR。

所有运行FreeRTOS的体系结构都能够处理中断，但有关中断条目和中断优先级分配的细节因体系结构而异。



### 范围

本章旨在让读者更好地理解：

- 哪些 FreeRTOS API 函数可以在中断服务程序中使用。
- 将中断处理推迟到任务的方法。

- 如何创建和使用二进制信号量和计数信号量。
- 二进制和计数信号量之间的区别。

- 如何使用队列将数据传入和传出中断服务程序。
- 一些 FreeRTOS 移植时可用的中断嵌套模型。





## 从 ISR 使用 FreeRTOS API

### 中断安全 API

通常需要使用来自中断服务函数 (ISR) 的 FreeRTOS API 函数提供的功能，但许多 FreeRTOS API 函数执行在 ISR 内无效的操作——其中最值得注意的是放置调用 API函数进入阻塞状态； 如果从 ISR 调用 API 函数，则它不是从任务调用的，因此没有调用任务可以置于阻塞状态。 FreeRTOS 通过提供一些 API 函数的两个版本来解决这个问题； 一种用于任务的版本，一种用于 ISR 的版本。 旨在从 ISR 使用的函数在其名称后加上“`FromISR`”。

> 注意：切勿从 ISR 调用名称中不含“`FromISR`”的 FreeRTOS API 函数。





### 使用单独的中断安全 API 的好处

具有用于中断的单独 API 可以使任务代码更高效，ISR 代码更有效，并且中断输入更简单。 要了解为什么，考虑另一种解决方案，它将提供一个API函数的单一版本，可以从一个任务和一个ISR调用。如果可以从任务和ISR调用相同版本的API函数，则：

- API 函数需要额外的逻辑来确定它们是从任务还是从 ISR 调用的。 额外的逻辑会在函数中引入新的路径，使函数变得更长、更复杂、更难测试。

- 当从任务调用函数时，某些 API 函数参数将过时，而当从 ISR 调用函数时，其他函数参数将过时。

- 每个 FreeRTOS 移植都需要提供一种机制来确定执行上下文（任务或 ISR）。

- 难以确定执行上下文（任务或 ISR）的架构将需要额外的、浪费的、使用更复杂的非标准中断入口代码，允许软件提供执行上下文。

  

### 使用单独的中断安全 API 的缺点

拥有两个版本的某些 API 函数可以提高任务和 ISR 的效率，但会带来新的问题；有时需要从任务和 ISR 调用不属于 FreeRTOS API 的函数，但需要使用 FreeRTOS API。

这通常只有在集成第三方代码时才会出现问题，因为这是软件设计不受应用程序编写者控制的唯一时候。 如果这确实成为一个问题，那么可以使用以下技术之一来克服该问题：

1. 将中断处理推迟到任务，因此 API 函数只能从任务的上下文中调用。

2. 如果您使用的是支持中断嵌套的 FreeRTOS 移植，则使用以 “`FromISR`” 结尾的 API 函数版本，因为该版本可以从任务和 ISR 中调用（反之则不然，不以 “`FromISR`” 结尾的 API 函数 不能从 ISR 调用“`FromISR`”）。

3. 第三方代码通常包括一个 RTOS 抽象层，可以实现该抽象层来测试调用函数的上下文（任务或中断），然后调用适合该上下文的 API 函数。

   

### xHigherPriorityTaskWoken 参数

本节介绍了 `xHigherPriorityTaskWoken` 参数的概念。 如果您尚未完全理解本节，请不要担心，因为以下部分提供实际示例。

如果上下文切换是由中断执行的，那么中断退出时运行的任务可能与进入中断时运行的任务不同——中断将中断一个任务，但返回到另一个任务。

一些 FreeRTOS API 函数可以将任务从阻塞状态移动到就绪状态。 这已经在诸如 `xQueueSendToBack()` 之类的函数中看到了，如果有一个任务在阻塞状态等待数据在主题队列上可用，它将解除对任务的阻塞。

如果被 FreeRTOS API 函数解除阻塞的任务的优先级高于处于运行状态的任务的优先级，那么根据 FreeRTOS 调度策略，应该切换到更高优先级的任务。 实际切换到更高优先级任务的时间取决于调用 API 函数的上下文：

- 如果从任务中调用 API 函数

  如果在 `FreeRTOSConfig.h` 中将 `configUSE_PREEMPTION` 设置为 1，那么在 API 函数中会自动切换到更高优先级的任务——所以在 API 函数退出之前。 这已经在图 43 中看到，其中写入定时器命令队列导致在写入命令队列的函数退出之前切换到 RTOS 守护程序任务。

- 如果从中断调用 API 函数

  中断内不会自动切换到更高优先级的任务。 相反，设置了一个变量来通知应用程序编写者应该执行上下文切换。 中断安全 API 函数（以 “`FromISR`” 结尾的函数）有一个名为 `pxHigherPriorityTaskWoken` 的指针参数，用于此目的。

  如果应该执行上下文切换，则中断安全 API 函数会将 `*pxHigherPriorityTaskWoken` 设置为 pdTRUE。 为了能够检测到这种情况，`pxHigherPriorityTaskWoken` 指向的变量在第一次使用之前必须初始化为 pdFALSE。

  如果应用程序编写者选择不从 ISR 请求上下文切换，则更高优先级的任务将保持就绪状态，直到调度程序下次运行——在最坏的情况下将在下一次滴答中断期间。

  FreeRTOS API 函数只能将 `*pxHighPriorityTaskWoken` 设置为 pdTRUE。 如果一个 ISR 调用了多个 FreeRTOS API 函数，那么可以在每个 API 函数调用中传递相同的变量作为 `pxHigherPriorityTaskWoken` 参数，并且该变量只需要在第一次使用之前初始化为 pdFALSE。

在 API 函数的中断安全版本中，上下文切换不会自动发生有几个原因：

1. 避免不必要的上下文切换

   在任务需要执行任何处理之前，中断可能会执行多次。 例如，考虑一个任务处理一个由中断驱动的 `UART` 接收到的字符串的场景； 每次接收到一个字符时，`UART ISR` 都切换到任务是一种浪费，因为任务只有在接收到完整字符串后才能执行。

2. 控制执行序列

   中断可能偶尔发生，而且时间不可预测。 专业的 FreeRTOS 用户可能希望暂时避免在其应用程序的特定点不可预测地切换到不同的任务——尽管这也可以使用 FreeRTOS 调度程序锁定机制来实现。

3. 可移植性

   它是可以跨所有 FreeRTOS 移植使用的最简单的机制。

4. 效率

   面向较小处理器架构的端口仅允许在 ISR 的最后请求上下文切换，而消除该限制将需要额外且更复杂的代码。 它还允许在同一 ISR 内多次调用 FreeRTOS API 函数，而不会在同一 ISR 内生成多个上下文切换请求。

5. 在 RTOS 滴答定时中断执行

   正如本书后面将看到的，可以将应用程序代码添加到 RTOS 滴答中断中。 尝试在滴答中断内进行上下文切换的结果取决于正在使用的 FreeRTOS 移植。 充其量，它会导致对调度程序的不必要调用。

使用 `pxHigherPriorityTaskWoken` 参数是可选的。 如果不需要，则将 `pxHigherPriorityTaskWoken` 设置为 `NULL`。



### portYIELD_FROM_ISR() 和 portEND_SWITCHING_ISR() 宏

本节介绍用于从 ISR 请求上下文切换的宏。 如果您尚未完全理解本节，请不要担心，因为以下部分提供实际示例。

`taskYIELD()` 是一个宏，可以在任务中调用以请求上下文切换。 `portYIELD_FROM_ISR()` 和 `portEND_SWITCHING_ISR()` 都是 `taskYIELD()` 的中断安全版本。 `portYIELD_FROM_ISR()` 和 `portEND_SWITCHING_ISR()` 都以同样的方式使用，并且做同样的事情。 某些 FreeRTOS 移植仅提供两个宏之一。 较新的 FreeRTOS 移植提供这两种宏。本书中的示例使用 `portYIELD_FROM_ISR()`。

```verilog
portEND_SWITCHING_ISR( xHigherPriorityTaskWoken );
```

清单 87. `portEND_SWITCHING_ISR()` 宏 



```verilog
portYIELD_FROM_ISR( xHigherPriorityTaskWoken );
```

清单 88. `portYIELD_FROM_ISR()` 宏



从中断安全 API 函数传出的 `xHigherPriorityTaskWoken` 参数可以直接用作调用 `portYIELD_FROM_ISR()`中的参数。

如果 `portYIELD_FROM_ISR()` `xHigherPriorityTaskWoken` 参数是 pdFALSE（0），则不请求上下文切换，并且宏不起作用。 如果 `portYIELD_FROM_ISR()` `xHigherPriorityTaskWoken` 参数不是 pdFALSE，则请求上下文切换，并且处于 运行状态的任务可能会更改。 中断将始终返回到处于运行状态的任务，即使在中断执行期间处于运行状态的任务发生了变化。

大多数 FreeRTOS 移植允许在 ISR 内的任何位置调用 `portYIELD_FROM_ISR()`。 一些 FreeRTOS 移植（主要用于较小架构的端口）仅允许在 ISR 的最后调用 `portYIELD_FROM_ISR()`。



## 延迟中断处理

保持 ISR 尽可能短通常被认为是最佳实践。 原因包括：

- 即使任务已被分配非常高的优先级，如果硬件不提供中断，则它们只会运行。
- ISR 可以中断（添加“抖动”）任务的开始时间和执行时间。

- 根据运行 FreeRTOS 的架构，在执行 ISR 时可能无法接受任何新中断，或至少是新中断的子集。
- 应用程序编写者需要考虑并防范任务和 ISR 同时访问变量、外设和内存缓冲区等资源的后果。

- 一些 FreeRTOS 移植允许中断嵌套，但中断嵌套会增加复杂性并降低可预测性。中断越短，嵌套的可能性就越小。



中断服务程序必须记录中断的原因，并清除中断。 中断所需的任何其他处理通常可以在任务中执行，从而允许中断服务程序尽可能快地退出。这称为“延迟中断处理”，因为中断所需的处理被从ISR“延迟”到任务。

将中断处理推迟到任务还允许应用程序编写者优先处理相对于应用程序中其他任务的处理，并使用所有 FreeRTOS API 函数。

如果中断处理的任务的优先级高于任何其他任务的优先级，则处理将立即执行，就像处理已在 ISR 本身中执行一样。 这种情况如图 48 所示，其中任务 1 是一个正常的应用程序任务，任务 2 是中断处理被推迟到的任务。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636075883656-39b14edf-eb77-4a3d-b477-c442fc65c0e8.png)

图 48. 高优先级任务中的中断处理



在图 48 中，中断处理从时间 t2 开始，实际上在时间 t4 结束，但只有时间 t2 和 t3 之间的时间段用于 ISR。 如果未使用延迟中断处理，则时间 t2 和 t4 之间的整个时间段都将花费在 ISR 中。

关于何时最好执行 ISR 中的中断所需的所有处理以及何时最好将部分处理推迟到任务，没有绝对的规则。 在以下情况下将处理推迟到任务最有用：

- 中断所需的处理并非微不足道。 例如，如果中断只是存储模数转换的结果，那么几乎可以肯定这最好在 ISR 内部执行，但如果转换结果还必须通过软件过滤器，那么它可能是最好在任务中执行过滤器。
- 中断处理可以方便地执行 ISR 内部无法执行的操作，例如写入控制台或分配内存。

- 中断处理不是确定性的——这意味着事先不知道处理需要多长时间。



以下各节描述并演示了本章到目前为止介绍的概念，包括可用于实现延迟中断处理的 FreeRTOS 功能。



## 用于同步的二进制信号量 

二进制信号量API的中断安全版本可以用于在每次特定中断发生时解除任务阻塞，从而有效地将任务与中断同步。这允许大部分中断事件处理在同步任务中实现，只有非常快和短的部分直接留在ISR中。如前一节所述，二进制信号量用于将中断处理“推迟”到任务。

> 任务：使用直接到任务的通知将一个任务从中断中解禁，比使用二进制信号量更有效率。直接到任务的通知在第9章 "任务通知 "中才会涉及。

正如之前在图48中所展示的，如果中断处理是特别关键的时间，那么可以设置延迟处理任务的优先级，以确保该任务总是优先于系统中的其他任务。然后，ISR可以被实现，包括对 `portYIELD_FROM_ISR()` ，确保ISR直接返回到中断处理的任务中。这样做的效果是确保整个事件处理的在时间上连续执行（没有中断），这就像所有的事件都是在ISR本身。图49重复了图48中的情景，但文字被更新为描述如何使用信号量来控制延迟处理任务的执行。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636079382522-076e62b6-ff99-42b3-9216-0ffa86e99374.png)

图49. 使用二进制信号来实现延迟的中断处理



延迟处理任务使用一个阻塞的 "拿走" 调用到 信号量 ，作为一种手段进入阻塞状态以等待事件的发生。当事件发生时，ISR在同一信号上使用 "给出" 操作来解除对任务的封锁，这样所需的事件处理可以继续进行。

“接受信号”和“给予信号”是具有不同含义的概念根据它们的使用情况，有不同的含义。在这种中断同步的情况下，二进制信号量在概念上可以被认为是一个长度为1的队列。该队列可以在任何时候都最多包含一个项，所以总是要么是空的，要么是满的（hence, binary）。通过调用 `xSemaphoreTake()` ，中断处理被推迟到的任务有效地试图用一个阻塞时间从队列中读取，如果队列是空的，则导致任务进入阻塞状态。当事件发生时，ISR使用 `xSemaphoreGiveFromISR()` 函数将一个令牌（信号量）放入队列，使队列变满。这将导致任务退出阻塞状态并移除令牌，使队列再次变空。当任务完成其处理后，它再次尝试从队列中读出并发现队列是空的，于是重新进入阻塞状态以等待下一个事件。这个序列在图50中展示。

图50显示了中断“给出”信号，尽管它并没有首先“拿走”它，以及任务“拿”走了信号，但没有把它送回去。这就是为什么这种情况被描述为在概念上类似于向队列中写和从队列中读。它经常引起混淆因为它并不遵循与其他信号使用场景相同的规则，在其他场景中，一个任务使用信号量的任务必须一直把它送回来--比如第7章中资源管理描述的场景。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636081992050-7c154d1a-1644-4f15-978d-aefaf283a81c.png)

图50. 使用二进制信号量使一个任务与中断同步



### xSemaphoreCreateBinary() API函数

FreeRTOS V9.0.0还包括 `xSemaphoreCreateBinaryStatic()` 函数，该函数在编译时分配内存。所需的内存，以便在编译时静态地创建一个二进制信号：对所有各种类型的FreeRTOS 信号量的句柄都存储在一个`SemaphoreHandle_t`类型的变量中。

在使用信号量之前，必须先创建它。要创建一个二进制信号量，请使用 `xSemaphoreCreateBinary() API`函数。

一些信号量API函数实际上是宏，而不是函数。为了简单起见，在本书中被称为函数。

```verilog
SemaphoreHandle_t xSemaphoreCreateBinary( void );
```

清单89. `xSemaphoreCreateBinary()` API函数原型



表33. `xSemaphoreCreateBinary()` 的返回值

| 参数名称 | 描述                                                         |
| ---------- |  ------------------------------------------------------------ |
| 返回值  | 如果返回的是`NULL`，那么这个信号就不能被创建，因为没有足够的堆内存可供FreeRTOS分配给信号量数据结构。如果返回的值不是`NULL`，则表明该信号已被成功创建。返回的值应该作为创建的信号量的句柄。 |

### xSemaphoreTake() API函数

“取用”信号量是指 "获得 "或 "接收 "该信号量。只有在信号量是可用的情况下才可以采取。

所有不同类型的FreeRTOS信号，除了递归互斥，都可以使用 `xSemaphoreTake()` 函数。

`xSemaphoreTake()` 不能从一个中断服务例程中使用。

```verilog
BaseType_t xSemaphoreTake( SemaphoreHandle_t xSemaphore, TickType_t xTicksToWait );
```

清单90. `xSemaphoreTake()` 的API函数原型



表34. `xSemaphoreTake()` 参数和返回值

| 参数名称/返回值 | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| `xSemaphore`    | 被“带走”的信号量。一个信号量是由一个 `SemaphoreHandle_t` 类型的变量来引用的。它必须在使用前明确地创建。 |
| `xTicksToWait`  | 任务在阻塞状态下保持的最大时间状态下等待信号量，如果信号量还没有可用的话。如果`xTicksToWait`是0，那么`xSemaphoreTake()`将立即返回，如果 则将立即返回，因为信号量是不可用的。区块时间是以tick周期为单位的，所以它的绝对时间代表的绝对时间取决于tick频率。宏指令`pdMS_TO_TICKS()`可以用来将一个以毫秒为单位的时间转换成以刻度为单位的时间。将`xTicksToWait`设置为`portMAX_DELAY`，如果`INCLUDE_vTaskSuspend` 在 `FreeRTOSConfig.h` 中被设置为 1，将导致任务无限期地等待（没有超时）。 |
| 返回值          | 有两个可能的返回值：<br/>1.pdPASS<br/>pdPASS仅在调用 `xSemaphoreTake()` 成功获得信号时返回。如果指定了一个块状时间（`xTicksToWait`不是0），那么就有可能调用的任务被放置到阻塞状态，以等待信号量，如果它不是立即可用的，但是在阻塞时间结束前，信号量变得可用。<br/>2. pdFALSE<br/>该信号灯不可用。如果指定了一个阻塞时间（`xTicksToWait`不是0），那么调用任务将被置入阻塞状态，以等待信号量变成可用的状态，但是在这之前阻断时间已过期。 |

### xSemaphoreGiveFromISR() API函数

可以使用`xSemaphoreGiveFromISR()`函数“给出”二进制和计数信号量。

> 计数信号量将在本书后面的章节中介绍。



`xSemaphoreGiveFromISR()` 是 `xSemaphoreGive()` 的中断安全版本，所以具有`pxHigherPriorityTaskWoken`参数，这在本章开始时已经描述过。

```verilog
BaseType_t xSemaphoreGiveFromISR( SemaphoreHandle_t xSemaphore, 
 BaseType_t *pxHigherPriorityTaskWoken );
```

清单91. `xSemaphoreGiveFromISR()` API函数原型



表35. `xSemaphoreGiveFromISR()` 参数和返回值

| 参数名称/返回值             | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `xSemaphore`                | 被 "给予 "的信号量。一个信号量是由一个类型为`SemaphoreHandle_t`的变量来引用，并且在使用前必须明确地被创建 使用。 |
| `pxHigherPriorityTaskWoken` | 有可能在一个单一的信号量上会有一个或多个 任务阻塞在其上，等待信号量成为可用的。调用 `xSemaphoreGiveFromISR()` 可以使 可以使信号量可用，从而导致等待信号量的任务离开阻塞状态。如果调用 `xSemaphoreGiveFromISR()` 导致一个任务离开 阻塞状态，而解除阻塞的任务的优先级高于 比当前执行的任务（被中断的任务）的优先级高。那么，在内部，`xSemaphoreGiveFromISR()` 将`*pxHigherPriorityTaskWoken `设置为pdTRUE。如果 `xSemaphoreGiveFromISR() `把这个值设置为pdTRUE。那么通常情况下，在退出中断之前应该进行上下文切换。这将确保该中断直接返回到最高优先级的就绪状态任务。 |
| 返回值                      | 有两个可能的返回值：<br/> 1. pdPASS<br/>pdPASS只有在调用 `xSemaphoreGiveFromISR()` 是成功的。<br/> 2、pdFAIL<br/> 如果一个semaphore已经可用，则不能被给予。并且 `xSemaphoreGiveFromISR()` 将返回pdFAIL。 |



### 示例16. 使用二进制信号量使一个任务与中断同步

这个例子使用一个二进制信号量来解除一个任务与中断服务程序之间的阻断——有效地使该任务与中断同步。

一个简单的周期性任务被用来每500毫秒产生一个软件中断。一个软件中断是为了方便而使用的，因为在一些目标环境中，连接到一个真正的中断的复杂性，所以使用软件中断是为了方便。清单92显示了周期性任务的实现。请注意，该任务在中断产生前后都会打印出一个字符串。这使得执行的顺序可以在例子执行时产生的输出中被观察到。

```verilog
/* 本例中使用的软件中断的编号。显示的代码来自Windows项目，其中数字0到2是由FreeRTOS Windows端口本身使用的，所以3是应用程序可用的第一个数字。*/
#define mainINTERRUPT_NUMBER 3
static void vPeriodicTask( void *pvParameters )
{
    const TickType_t xDelay500ms = pdMS_TO_TICKS(500UL);
 		/* 和大多数任务一样，这个任务是在一个无限循环中实现的。*/
 		for( ;; )
 		{
 				/* 阻止直到再次产生软件中断的时间。*/
 			 vTaskDelay( xDelay500ms );
 			/* 产生中断，在中断产生前后打印一条信息。这样执行的顺序就可以从输出中看出来。
 用于生成软件中断的语法取决于所使用的FreeRTOS的端口。下面使用的语法只能用于FreeRTOS Windows端口，在该端口中，这种中断只是模拟的。*/
 			 vPrintString( "Periodic task - About to generate an interrupt.\r\n" );
 			 vPortGenerateSimulatedInterrupt( mainINTERRUPT_NUMBER );
 			 vPrintString( "Periodic task - Interrupt generated.\r\n\r\n\r\n" );
 		} 
}
```

清单92. 实现例16中定期生成软件中断的任务 



清单93显示了中断处理被推迟的任务的实现——该任务通过使用一个二进制信号量同步。同样，在任务的每次迭代中都会打印出一个字符串，所以任务和中断的执行顺序可以从例子执行时产生的输出中看出来。

```verilog
{
    /* 和大多数任务一样，这个任务是在一个无限循环中实现的。 */
    for (;;)
    {
        /* 使用semaphore来等待事件的发生。这个semaphore是在调度器启动之前创建的，所以在这个任务第一次运行之前的时候。这个任务会无限期地阻塞，这意味着这个函数调用只会在成功获得semaphore返回——所以不需要检查xSemaphoreTake()返回的值。*/
        xSemaphoreTake(xBinarySemaphore, portMAX_DELAY);
      /* 要到达这里，这个事件必须已经发生。处理该事件（在本例中，只是打印出一条信息）。 */
        vPrintString("Handler task - Processing event.\r\n");
    }
}
```

清单93. 中断处理所涉及的任务的实现 （与中断同步的任务）。



清单94显示了这个ISR，除了 "给 "信号以解除对中断处理的任务的封锁外，没有什么作用。

注意`xHigherPriorityTaskWoken`变量是如何被使用的。在调用`xSemaphoreGiveFromISR() `之前，它被设置为pdFALSE，然后在调用`portYIELD_FROM_ISR() `时作为参数使用。如果`xHigherPriorityTaskWoken`等于pdTRUE，在 `portYIELD_FROM_ISR() `宏中会要求进行上下文切换。

与大多数FreeRTOS运行的架构不同，FreeRTOS的Windows端口需要一个ISR来返回一个值。与Windows端口一起提供的 `portYIELD_FROM_ISR()` 宏的实现包括返回语句，所以清单94没有明确显示返回值。

```verilog
static uint32_t ulExampleInterruptHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken;
    /* xHigherPriorityTaskWoken参数必须被初始化为pdFALSE，因为如果需要进行上下文切换，它将在中断安全API函数中被设置为pdTRUE。 */
    xHigherPriorityTaskWoken = pdFALSE;
    /* “给予”semaphore以解除对任务的封锁任务，将xHigherPriorityTaskWoken的地址作为中断安全API函数的参数pxHigherPriorityTaskWoken传入。 */
    xSemaphoreGiveFromISR(xBinarySemaphore, &xHigherPriorityTaskWoken);
  /* 将xHigherPriorityTaskWoken的值传给portYIELD_FROM_ISR()。如果xHigherPriorityTaskWoken在xSemaphoreGiveFromISR()中被设置为pdTRUE，那么调用portYIELD_FROM_ISR()将请求进行上下文切换。如果xHigherPriorityTaskWoken仍然是pdFALSE，那么调用   portYIELD_FROM_ISR()将没有任何影响。与大多数 FreeRTOS 端口不同， Windows 端口要求 ISR 返回一个值——返回语句在 Windows 版本的 portYIELD_FROM_ISR() 中。*/
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单94. 示例16中使用的软件中断的ISR



`main()`函数创建了二进制信号量，创建了任务，安装了中断处理程序。处理并启动调度程序。该函数的实现见清单95。

为安装中断处理程序而调用的函数的语法是针对FreeRTOS Windows端口的，对于其他FreeRTOS端口可能有所不同。请参考FreeRTOS.org网站上的特定端口文档页面，以及FreeRTOS下载中提供的例子，以找到你正在使用的端口所需的语法。

```verilog
int main(void)
{
    /* 在使用semaphore之前，必须明确地创建它。在这个例子中，我们创建了一个binary semaphore。 */
    xBinarySemaphore = xSemaphoreCreateBinary();
    /* 检查semaphore是否成功创建。 */
    if (xBinarySemaphore != NULL)
    {
        /* 创建 "处理程序 "任务，这是一个被推迟处理的中断任务,一个将与中断同步的任务。处理程序任务是以高优先级创建的，以确保它在中断退出后立即运行。在这种情况下，我们选择了3的优先级。 */
        xTaskCreate(vHandlerTask, "Handler", 1000, NULL, 3, NULL);
        /* 创建将定期产生一个软件中断的任务。这个任务的优先级低于处理任务，以确保每次处理任务退出阻塞状态时，它将被抢占。 */
        xTaskCreate(vPeriodicTask, "Periodic", 1000, NULL, 1, NULL);
        /* 安装软件中断的处理程序。这样做所需的语法取决于正在使用的FreeRTOS端口。这里显示的语法只能用于FreeRTOS的windows端口，在那里这种中断只是模拟的。 */
        vPortSetInterruptHandler(mainINTERRUPT_NUMBER, ulExampleInterruptHandler);
        /* 启动调度器，使创建的任务开始执行。 */
        vTaskStartScheduler();
    }
    /* 正常情况下，绝对不能达到下面这一行。 */
    for (;;);
}
```

清单95. 示例16中 `main() `的实现



例16产生的输出如图51所示。正如预期的那样，`vHandlerTask() `在中断产生后立即进入了运行状态，所以任务的输出与周期性任务产生的输出相分离。图52中提供了进一步的解释。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636093177939-15bcfd4b-fdd7-45ea-8215-02b2af35bafc.png)

图51. 执行示例16时产生的输出

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636093274205-f5851fb4-dffb-41f7-a29c-a2ac0c5f7907.png)

图52. 执行例16时的执行顺序



### 完善示例16中使用的任务的实现

示例16使用一个二进制信号量来同步一个任务和一个中断。其执行顺序如下：

1. 中断发生了。

2. ISR执行并“给出”信号量以解除对任务的封锁。

3. 任务在ISR之后立即执行，并“拿走”了信号量。

4. 任务处理了该事件，然后试图再次“拿走”信号量——进入阻塞状态，因为信号量还不可用（另一个中断还没有发生）。



只有当中断发生的频率相对较低时，示例16中的任务结构才是充分的。为了理解这个原因，我们可以考虑一下，如果在任务完成对第一个中断的处理之前，又发生了第二个、第三个中断，会发生什么：

- 当第二个ISR执行时，信号量将是空的，所以ISR将给出信号量，且任务在完成对第一个事件的处理后将立即处理第二个事件。这种情况在图53中显示。

- 当第三个ISR执行时，信号量已经可用，防止ISR再次给出信号量，所以任务不会知道第三个事件已经发生。这种情况在图54中显示。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636094583388-c25d8f39-1a95-4fb2-a2ef-5a9fba60794a.png)

图53. 在任务完成处理第一个事件之前发生一个中断时的情景



![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636094676453-cb3a2122-221e-4fbc-9fb5-bf133a9d19f8.png)

图54 在任务完成处理第一个事件之前发生两个中断时的情景



示例16中使用的延迟中断处理任务，如清单93所示，其结构是在每次调用 `xSemaphoreTake() `时只处理一个事件。这对示例16来说是足够的，因为产生事件的中断是由软件触发的，而且发生的时间是可预测的。在实际应用中，中断是由硬件产生的，并且发生在不可预测的时间。因此，为了尽量减少错过中断的机会，延迟中断处理任务的结构必须使它在每次调用 `xSemaphoreTake() `之间处理所有已经存在的`事件`。清单96证明了这一点，它显示了一个UART的延迟中断处理任务的结构。在清单96中，假设UART在每次收到字符时都会产生一个接收中断，并且UART将收到的字符放入一个硬件的 FIFO（一个硬件缓冲区）。

> 事件：另外，也可以使用计数信号或直接到任务的通知来计数事件。计数信号将在下一节中描述。直接到在第9章,任务的通知中描述。直接到任务的通知是首选方法，因为它们在运行时间和RAM的使用上都是最有效的。

示例16中使用的延迟中断处理任务还有一个弱点；它在调用 `xSemaphoreTake() `时没有使用超时。相反，该任务将`portMAX_DELAY`作为 `xSemaphoreTake() `的`xTicksToWait`参数，这导致该任务无限期地等待（没有超时），等待 信号量的出现。无限期超时经常被用在示例代码中，因为它们的使用简化了示例的结构，从而使示例更容易理解。然而，在实际应用中，无限期超时通常是不好的做法，因为它们使人们很难从错误中恢复。作一个例子，考虑这样的情景。一个任务正在等待一个中断来给出 信号量，但是硬件中的错误状态阻止了中断的产生：

- 如果任务在没有超时的情况下等待，它将不知道错误状态，并将永远等待。
- 如果任务在等待时有超时，那么`xSemaphoreTake() `将在超时结束时返回pdFAIL，然后任务可以在下次执行时检测并清除错误。这种情况在清单96中也有演示。

```verilog
static void vUARTReceiveHandlerTask(void *pvParameters)
{
    /* xMaxExpectedBlockTime持有两个中断之间的最大预期时间。 */
    const TickType_t xMaxExpectedBlockTime = pdMS_TO_TICKS(500);
    /* 和大多数任务一样，这个任务是在一个无限循环中实现的。 */
    for (;;)
    {
        /* 该 semaphore 是由UART的接收(Rx)中断"给予"的。等待下一个中断的时间最多为xMaxExpectedBlockTime ticks。 */
        if (xSemaphoreTake(xBinarySemaphore, xMaxExpectedBlockTime) == pdPASS)
        {
            /* 获得了semaphore。在再次调用xSemaphoreTake()之前，处理所有待处理的Rx事件。每个Rx事件都会在UART的接收FIFO中放置一个字符，UART_RxCount()被认为是返回FIFO中的字符数。*/
            while (UART_RxCount() > 0)
            {
                /* UART_ProcessNextRxEvent()被假定为处理一个Rx字符，使FIFO中的字符数减少1。 */
                UART_ProcessNextRxEvent();
            }
            /* 没有更多的Rx事件等待处理（FIFO中没有更多的字符），所以回环并调用xSemaphoreTake()以等待下一个中断。从代码的这一点到调用xSemaphoreTake()之间发生的任何中断都会被锁在semaphore中，所以不会丢失。 */
        }
        else
        {
            /* 没有在预期的时间内收到一个事件。检查并在必要时清除UART中可能阻止UART产生更多中断的任何错误条件。 */
            UART_ClearErrors();
        }
    }
}
```

清单96. 推荐的延迟中断处理任务的结构，以UART接收处理程序为例



------

## 计数信号量

正如二进制信号量可以被认为是长度为1的队列一样，计数信号量可以被认为是长度为1以上的队列。任务对存储在队列中的数据不感兴趣，只是对队列中的项目数量感兴趣。 `configUSE_COUNTING_SEMAPHORES`必须在`FreeRTOSConfig.h`中设置为1，才能使用计数信号量 。

每当一个 计数信号量 被 "给予 "时，其队列中的另一个空间就被使用。队列中的项目数量是 信号量的“计数”值。

1. `计数事件`

   在这种情况下，事件处理程序会在每次事件发生时 “给”一个信号量——导致信号的计数值在每次“给出 ”时被递增。 一个任务在每次处理一个事件时都会“拿走”一个信号量——导致信号量的数值被递减。在每次 "取 "的时候都会递减。计数值是已经发生的事件数与已经发生的事件数之间的差值。这个机制是图55中所示。

   用于统计事件的计数信号器在创建时，其初始计数值为0。

> 计数事件：使用直接到任务的通知来计数事件比使用计数信号量更有效。直接到任务的通知在第9章才会涉及。

2. 资源管理

   在这种情况下，计数值表示可用资源的数量。为了获得资源的控制权，一个任务必须首先获得一个信号量——减少 信号量的计数值。当计数值达到0时，就没有可用的资源了。当一个任务完成了对资源的控制，它就会“给”回 信号量——增加 信号量的计数值。

   用于管理资源的计数信号量在创建时，其初始计数值等于可用资源的数量。第7章介绍了使用信号量来管理资源。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636097763206-7d7472d6-fae3-4560-a851-4917b7a68aca.png)

图55. 使用一个计数信号量来 "计数 "事件



### xSemaphoreCreateCounting() API函数

FreeRTOS V9.0.0还包括` xSemaphoreCreateCountingStatic() `函数，该函数在编译时分配静态地创建一个计数信号量的内存：所有不同类型的FreeRTOS信号的句柄都存储在一个`SemaphoreHandle_t`类型的变量中。

在使用信号量之前，必须先创建它。要创建一个计数信号量，请使用 `xSemaphoreCreateCounting() `API函数。

```verilog
SemaphoreHandle_t xSemaphoreCreateCounting( UBaseType_t uxMaxCount,
 UBaseType_t uxInitialCount );
```

清单97. `xSemaphoreCreateCounting()` API函数原型 



表36. `xSemaphoreCreateCounting()`参数和返回值

| 参数名称/返回值  | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| `uxMaxCount`     | 信号量的最大值。继续用队列来比喻，`uxMaxCount`值实际上就是队列的长度。当信号量被用来计算或锁住事件时，`uxMaxCount`是可以锁住的最大事件数。当信号量被用来管理对资源集合的访问时，`uxMaxCount`应该被设置为可用资源的总数。 |
| `uxInitialCount` | 信号量被创建后的初始计数值。当信号量被用来计算或锁定事件时，`uxInitialCoun`t应该被设置为0——因为当信号量被创建时，可能还没有事件发生。当信号量被用来管理对一组资源的访问时，`uxInitialCount`应该被设置为等于`uxMaxCount`，因为当信号量被创建时，所有的资源都是可用的。 |
| 返回值           | 如果返回`NULL`，说明不能创建信号量，因为没有足够的堆内存可供FreeRTOS分配信号量的数据结构。第2章提供了更多关于堆内存管理的信息。返回的非`NULL`值表示已经成功创建了信号量。返回的值应该作为创建的信号量的句柄来存储。 |

### 示例17. 使用计数信号来使一个任务与中断同步

示例17改进了示例16的实现，用binary 信号量代替了计数信号量。`main()`被修改为包含一个对 `xSemaphoreCreateCounting()`，以取代对 `xSemaphoreCreateBinary() `的调用。新的API 调用显示在清单98中。

```verilog
/* 在使用semaphore之前，必须明确地创建它。在这个例子中，一个counting semaphore被创建。创建的semaphore的最大计数值为10，初始计数值为0。 */
xCountingSemaphore = xSemaphoreCreateCounting(10, 0);
```

清单98. 示例17中用于创建计数信号量的 `xSemaphoreCreateCounting() `的调用



为了模拟高频率发生的多个事件，中断服务例程被改变为在每个中断中“给予”信号量不止一次。每个事件都被锁在 信号量的计数值中。修改后的中断服务例程显示在清单99中。

```verilog
static uint32_t ulExampleInterruptHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken;
    /* xHigherPriorityTaskWoken参数必须被初始化为pdFALSE，因为如果需要进行上下文切换，它将在中断安全API函数中被设置为pdTRUE。 */
    xHigherPriorityTaskWoken = pdFALSE;
    /* 多次 “给”semaphore 。第一次将解除对推迟的 中断处理任务，接下来的 "给予 "是为了证明  semaphore锁住事件，以允许被推迟的中断处理任务依次处理它们，而不会丢失事件。这模拟了处理器收到的多个中断，尽管在这种情况下，事件是在一个中断发生中模拟的。 */
    xSemaphoreGiveFromISR(xCountingSemaphore, &xHigherPriorityTaskWoken);
    xSemaphoreGiveFromISR(xCountingSemaphore, &xHigherPriorityTaskWoken);
    xSemaphoreGiveFromISR(xCountingSemaphore, &xHigherPriorityTaskWoken);
    /* 将xHigherPriorityTaskWoken的值传给portYIELD_FROM_ISR()。如果
 xHigherPriorityTaskWoken在xSemaphoreGiveFromISR()中被设置为pdTRUE，那么调用portYIELD_FROM_ISR()将请求进行上下文切换。如果xHigherPriorityTaskWoken仍然是pdFALSE，那么调用portYIELD_FROM_ISR()将没有任何影响。与大多数 FreeRTOS 端口不同， Windows 端口要求 ISR 返回一个值——返回语句在 Windows 版本的 portYIELD_FROM_ISR() 中。 */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单99. 示例17所使用的中断服务例程的实现



所有其他的函数与示例16中使用的函数相比都没有修改。

示例17执行时产生的输出如图56所示。可以看出，在每次产生中断时，被推迟处理的任务会处理所有三个[模拟]事件。这些事件被锁在信号的计数值中，允许任务依次处理它们。


![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636101378827-75ccc71d-0a24-4dd0-b078-f9e41f7bdabf.png)

图56. 执行示例17时产生的输出




## 推迟工作到RTOS守护进程任务

到目前为止，所介绍的延迟中断处理的例子都要求应用程序编写者为每个使用延迟处理技术的中断创建一个任务。也可以使用` xTimerPendFunctionCallFromISR() `API函数将中断处理推迟到RTOS守护任务中——无需为每个中断创建一个单独的任务。将中断处理推迟到守护任务被称为“集中的推迟中断处理”。

> xTimerPendFunctionCallFromISR()：在第五章中指出，守护任务最初被称为定时器服务任务，因为它最初只用来执行软件定时器回调函数。因此，`xTimerPendFunctionCall() `是在 `timers.c `中实现的，根据将函数的名称与实现该函数的文件名放在一起的惯例，该函数的名称前缀为`Timer`。

第5章描述了与软件定时器有关的FreeRTOS API函数如何在定时器命令队列中向守护任务发送命令。`xTimerPendFunctionCall() `和` xTimerPendFunctionCallFromISR()` API函数使用相同的定时器命令队列，向守护任务发送 "执行函数 "命令。发送给守护任务的函数随后在守护任务的上下文中被执行。

集中式延时中断处理的优点包括：

- 降低资源使用量

  它消除了为每个延迟中断创建一个单独任务的需要。

- 简化的用户模

  递延中断处理函数是一个标准的C函数。



集中式延时中断处理的缺点包括：

- 灵活性较低

  不可能单独设置每个递延中断处理任务的优先级。每个递延中断处理功能以守护任务的优先级执行。如第五章所述，守护任务的优先级是由 `configTIMER_TASK_PRIORITY `在` FreeRTOSConfig.h` 中的编译时间配置常数来设置。

- 减少决定论

  `xTimerPendFunctionCallFromISR()` 将一个命令发送到定时器命令队列的后面。在`xTimerPendFunctionCallFromISR() `发送给队列的"`execute function`"命令之前，已经在定时器命令队列中的命令将被守护任务处理。



不同的中断有不同的时间限制，所以在同一个应用程序中使用两种推迟中断处理的方法是很常见的。



### xTimerPendFunctionCallFromISR() API函数

`xTimerPendFunctionCallFromISR() `是 `xTimerPendFunctionCall() `的中断安全版本。

这两个API函数都允许由应用程序编写者提供的函数由RTOS守护任务执行，因此也是在RTOS守护任务的上下文中执行。要执行的函数和函数的输入参数值都被发送到定时器命令队列中的守护任务。因此，该函数实际执行的时间取决于守护任务相对于应用程序中其他任务的优先级。

```verilog
BaseType_t xTimerPendFunctionCallFromISR( PendedFunction_t xFunctionToPend,
 void *pvParameter1,
 uint32_t ulParameter2,
 BaseType_t *pxHigherPriorityTaskWoken );
```

清单100. `xTimerPendFunctionCallFromISR() `API函数原型



```verilog
void vPendableFunction( void *pvParameter1, uint32_t ulParameter2 );
```

清单101. `xTimerPendFunctionCallFromISR()` 的`xFunctionToPend`参数中传递的函数必须符合的原型。



表37. `xTimerPendFunctionCallFromISR() `参数和返回值

| 参数名称/返回值             | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `pvParameter1`              | 将被传递到由守护任务执行的函数中的值，作为该函数的`pvParameter1`参数。该参数有一个void *类型，允许它用来传递任何数据类型。例如，整数类型可以直接转换为void *，或者，void *可以用来指向一个结构。 |
| `ulParameter2 `             | 将被传递到由守护任务执行的函数中的值，作为该函数的`ulParameter2`参数。 |
| `pxHigherPriorityTaskWoken` | `xTimerPendFunctionCallFromISR()` 写到定时器命令队列。如果RTOS守护任务处于阻塞状态以等待定时器命令队列上的数据，那么写到定时器命令队列将导致守护任务离开阻塞状态。如果守护任务的优先级高于当前执行的任务（被中断的任务）的优先级，那么在内部，`xTimerPendFunctionCallFromISR()` 将把`*pxHigherPriorityTaskWoken`设为pdTRUE。如果 `xTimerPendFunctionCallFromISR()` 将此值设置为pdTRUE，那么在退出中断之前必须进行上下文切换。这将确保中断直接返回到守护任务，因为守护任务将是最高优先级的就绪状态任务。 |
| 返回值                   | 有两个可能的返回值：<br/>1. pdPASS<br/>如果"`execute function`"命令被写入定时器命令队列，将返回pdPASS。<br/>2. pdFAIL<br/>如果"`execute function`"命令不能被写入定时器命令队列，因为定时器命令队列已经满了，则将返回pdFAIL。第5章描述了如何设置定时器命令队列的长度。 |



### 示例18. 集中的延迟中断处理


示例18提供了与示例16类似的功能，但没有使用信号量，也没有专门创建一个任务来执行中断所需要的处理。相反，处理是由RTOS守护任务执行的。

示例18所使用的中断服务例程如清单102所示。它调用` xTimerPendFunctionCallFromISR()`，将一个指向`vDeferredHandlingFunction()` 函数的指针传递给守护任务。延迟的中断处理由`vDeferredHandlingFunction()` 函数执行。

中断服务程序每次执行时都会增加一个名为 `ulParameterValue` 的变量。`ulParameterValue` 在调用`xTimerPendFunctionCallFromISR() `时被用作 `ulParameter2 `的值，因此在守护任务执行`vDeferredHandlingFunction() `时也将被用作 `ulParameter2 `的值。该函数的另一个参数`pvParameter1`，在这个例子中没有使用。

```verilog
static uint32_t ulExampleInterruptHandler(void)
{
    static uint32_t ulParameterValue = 0;
    BaseType_t xHigherPriorityTaskWoken;
    /* xHigherPriorityTaskWoken参数必须被初始化为pdFALSE，因为如果需要进行上下文切换，它将在中断安全API函数中被设置为pdTRUE。 */
    xHigherPriorityTaskWoken = pdFALSE;
    /* 向守护任务发送一个指向中断的延迟处理函数的指针。递延处理函数的pvParameter1参数不使用，所以直接设置为NULL。递延处理函数的ulParameter2参数用于传递一个数字，这个数字在每次执行这个中断处理程序时都会递增1。 */
    xTimerPendFunctionCallFromISR(vDeferredHandlingFunction, /* 要执行的功能。 */
                                  NULL,                      /* 未使用。 */
                                  ulParameterValue,          /* 递增值。 */
                                  &xHigherPriorityTaskWoken);
    ulParameterValue++;
    /* 将xHigherPriorityTaskWoken的值传给portYIELD_FROM_ISR()。如果 xHigherPriorityTaskWoken 在 xTimerPendFunctionCallFromISR() 中被设置为 pdTRUE， 那么调用 portYELD_FROM_ISR() 将请求进行上下文切换。如果 xHigherPriorityTaskWoken 仍然是 pdFALSE， 那么调用 portYIELD_FROM_ISR() 将没有任何影响。与大多数 FreeRTOS 端口不同，Windows 端口要求 ISR 返回一个值——返回语句在 Windows 版本的 portYIELD_FROM_ISR() 中。 */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单102. 示例18中使用的软件中断处理程序



`vDeferredHandlingFunction()` 的实现在清单103中显示。它打印出一个固定的字符串，以及其`ulParameter2`的参数值。

`vDeferredHandlingFunction() `必须具有清单101中的原型，尽管在这个例子中，只有一个参数被实际使用。

```verilog
static void vDeferredHandlingFunction(void *pvParameter1, uint32_t ulParameter2)
{
    /* 处理事件————在本例中只是打印出一条信息和ulParameter2的值。 pvParameter1在本例中没有使用。 */
    vPrintStringAndNumber("Handler function - Processing event ", ulParameter2);
}
```

清单103. 执行示例18中中断所需的处理的函数



示例18使用的`main() `函数在清单104中显示。它比示例16使用的`main() `函数更简单，因为它没有创建一个信号量或一个任务来执行延迟中断处理。



`vPeriodicTask() `是周期性产生软件中断的任务。它被创建时的优先级低于守护任务的优先级，以确保一旦守护任务离开阻塞状态，它就会被守护任务抢占。

```verilog
int main(void)
{
    /* 产生软件中断的任务是以低于守护任务的优先级创建的。守护任务的优先级由FreeRTOSConfig.h中的configTIMER_TASK_PRIORITY编译时配置常数设置。 */
    const UBaseType_t ulPeriodicTaskPriority = configTIMER_TASK_PRIORITY - 1;
    /* 创建将周期性产生软件中断的任务。 */
    xTaskCreate(vPeriodicTask, "Periodic", 1000, NULL, ulPeriodicTaskPriority, NULL);
    /* 安装软件中断的处理程序。这样做所需的语法取决于正在使用的FreeRTOS端口。这里显示的语法只能用于FreeRTOS的windows端口，在那里这种中断只是模拟的。 */
    vPortSetInterruptHandler(mainINTERRUPT_NUMBER, ulExampleInterruptHandler);
    /* 启动调度器，使创建的任务开始执行。 */
    vTaskStartScheduler();

    /* 正常情况下，绝对不能达到下面这行。 */
    for (;;)
        ;
}
```

清单104. 示例18中 `main() `的实现



示例18产生的输出如图57所示。守护任务的优先级高于产生软件中断的任务的优先级，所以`vDeferredHandlingFunction()` 在中断产生后立即被守护任务执行。这导致 `vDeferredHandlingFunction() `输出的消息出现在周期性任务输出的两个消息之间，就像用 信号量来解锁专门的延迟中断处理任务时那样。图58中提供了进一步的解释。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636105289736-5c5c060b-774c-4943-a6af-84c5675d2b25.png)

图57. 执行示例18时产生的输出



![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636105350694-0dc0ebe9-b795-4087-a35e-20942bf90516.png)

图58 执行示例18时的执行顺序



## 在中断服务程序中使用队列

二进制和计数信号被用来交流事件。队列被用来交流事件，并传输数据。

`xQueueSendToFrontFromISR()` 是 `xQueueSendToFront() `的版本，可以在中断服务例程中安全使用；`xQueueSendToBackFromISR() `是 `xQueueSendToBack()` 的版本，可以在中断服务例程中安全使用；而`xQueueReceiveFromISR()` 是 `xQueueReceive() `的版本，可以在中断服务例程中安全使用。



### xQueueSendToFrontFromISR() 和xQueueSendToBackFromISR() API 函数

```verilog
BaseType_t xQueueSendToFrontFromISR(QueueHandle_t xQueue,
                                    void *pvItemToQueue
                                        BaseType_t *pxHigherPriorityTaskWoken
                                   );
```

清单105. `xQueueSendToFrontFromISR()` API函数原型



```verilog
BaseType_t xQueueSendToBackFromISR(QueueHandle_t xQueue,
                                   void *pvItemToQueue
                                       BaseType_t *pxHigherPriorityTaskWoken
                                  );
```

清单106. `xQueueSendToBackFromISR() `API函数原型



`xQueueSendFromISR()` 和 `xQueueSendToBackFromISR() `在功能上等同。

表38. `xQueueSendToFrontFromISR()` 和 `xQueueSendToBackFromISR()` 参数和返回值

| 参数名称/返回值             | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `xQueue`                    | 发送（写入）数据的队列的句柄。该队列句柄将从用于创建队列的`xQueueCreate()`调用中返回。 |
| `pvItemToQueue`             | 一个指向将被复制到队列中的数据的指针。队列所能容纳的每个项目的大小是在创建队列时设置的，所以这个字节将从 `pvItemToQueue `复制到队列存储区。 |
| `pxHigherPriorityTaskWoken` | 一个队列有可能会有一个或多个任务被阻塞在上面，等待数据的出现。调用 `xQueueSendToFrontFromISR(`) 或 `xQueueSendToBackFromISR() `可以使数据可用，从而导致这样一个任务离开阻塞状态。如果调用API函数导致一个任务离开阻塞状态，并且解除阻塞的任务的优先级高于当前执行的任务（被中断的任务），那么在内部，API函数将把`*pxHigherPriorityTaskWoken` 设置为pdTRUE。如果 `xQueueSendToFrontFromISR()` 或`xQueueSendToBackFromISR() `将此值设置为pdTRUE，那么在中断退出之前应该进行上下文切换。这将确保中断直接返回到最高优先级的就绪状态任务。 |
| 返回值                      | 有两个可能的返回值:<br/>1. pdPASS<br/> pdPASS仅在数据被成功发送到队列时返回。 <br/>2. `errQUEUE_FULL`<br/> `errQUEUE_FULL`如果数据不能被发送到队列，则返回，因为队列已经满了。 |



### 从ISR中使用队列时的考虑因素

队列提供了一种简单方便的方式将数据从中断传递给任务，但如果数据到达的频率很高，使用队列的效率就不高。

FreeRTOS下载中的许多演示程序包括一个简单的UART驱动程序，它使用一个队列将字符从UART的接收ISR中传递出来。在这些演示程序中，使用队列有两个原因：演示队列在ISR中的使用，以及故意加载系统以测试FreeRTOS端口。以这种方式使用队列的ISR绝对不是为了代表一种有效的设计，除非数据到达的速度很慢，否则建议生产代码不要复制这种技术。更有效的技术，那些适合于生产代码，包括：

- 使用直接内存访问（DMA）硬件来接收和缓冲字符。这种方法实际上没有软件开销。然后可以使用直接到任务的`通知`来解除对任务的封锁，只有在检测到传输中断后才会处理缓冲区。
- 将收到的每个字符复制到线程安全的RAM`缓冲区`。同样，在收到一个完整的消息后，或在检测到传输中断后，可以使用一个直接到任务的通知来解除对处理缓冲区的任务的封锁。

- 在ISR中直接处理收到的字符，然后使用队列将数据处理的结果（而不是原始数据）发送给一个任务。这在之前由图34演示过。



> 通知：直接到任务的通知提供了从ISR中解除任务阻塞的最有效方法。直接到任务的通知将在第9章 "任务通知 "中介绍。
>
> 缓冲区：作为FreeRTOS+TCP(http://www.FreeRTOS.org/tcp)的一部分提供的 "流缓冲 "可用于此目的。



### 示例19. 在一个队列中从一个中断中发送和接收信息

这个例子演示了 `xQueueSendToBackFromISR()` 和 `xQueueReceiveFromISR()` 在同一个中断中使用。和以前一样，为了方便，中断是由软件产生的。

创建一个周期性任务，每200毫秒向一个队列发送五个数字。只有在所有五个数字都被发送后，它才会产生一个软件中断。该任务的实现在清单107中显示。

```verilog
static void vIntegerGenerator(void *pvParameters)
{
    TickType_t xLastExecutionTime;
    uint32_t ulValueToSend = 0;
    int i;
    /* 初始化调用vTaskDelayUntil()所使用的变量。 */
    xLastExecutionTime = xTaskGetTickCount();
    for (;;)
    {
        /* 这是一个定期任务。阻止直到它再次运行的时间。该任务将每200ms执行一次。 */
        vTaskDelayUntil(&xLastExecutionTime, pdMS_TO_TICKS(200));
        /* 向队列发送五个数字，每个数字比前一个数字高一个。这些数字由中断服务例程从队列中读取。
中断服务例程总是清空队列，所以这个任务保证能够写入所有五个值，而不需要指定一个块的时间。 */
        for (i = 0; i < 5; i++)
        {
            xQueueSendToBack(xIntegerQueue, &ulValueToSend, 0);
            ulValueToSend++;
        }
        /* 产生中断，以便中断服务程序可以从队列中读取数值。用于产生软件中断的语法取决于正在使用的FreeRTOS端口。下面使用的语法只能用于FreeRTOS的Windows端口，在该端口中，这种中断只是模拟的。*/
        vPrintString("Generator task - About to generate an interrupt.\r\n");
        vPortGenerateSimulatedInterrupt(mainINTERRUPT_NUMBER);
        vPrintString("Generator task - Interrupt generated.\r\n\r\n\r\n");
    }
}
```

清单 107. 示例19中写入队列的任务的实现



中断服务例程反复调用 `xQueueReceiveFromISR() `，直到所有由周期性任务写入队列的值都被读出，队列留空。每个收到的值的最后两位被用作一个字符串数组的索引。然后通过调用 `xQueueSendFromISR()` ，将相应索引位置的字符串指针发送到不同的队列。中断服务例程的实现见清单108。

```verilog
static uint32_t ulExampleInterruptHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken;
    uint32_t ulReceivedNumber;
    /* 字符串被声明为静态常量，以确保它们不被分配到中断服务程序的堆栈中，因此即使在中断服务程序不执行时也存在。 */
    static const char *pcStrings[] =
        {
            "String 0\r\n",
            "String 1\r\n",
            "String 2\r\n",
            "String 3\r\n"};
    /* 一如既往，xHigherPriorityTaskWoken被初始化为pdFALSE，以便能够检测到它在中断安全API函数中被设置为pdTRUE。请注意，由于中断安全API函数只能将xHigherPriorityTaskWoken设置为pdTRUE，所以在调用xQueueReceiveFromISR()和调用xQueueSendToBackFromISR()时使用同一个xHigherPriorityTaskWoken变量是安全的。 */
    xHigherPriorityTaskWoken = pdFALSE;
    /* 从队列中读取，直到队列为空。 */
    while (xQueueReceiveFromISR(xIntegerQueue,
                                &ulReceivedNumber,
                                &xHigherPriorityTaskWoken) != errQUEUE_EMPTY)
    {
        /* 将接收到的值截断到最后两位（值0到3，包括在内），然后用截断的值作为pcStrings[]数组的索引，选择一个字符串（char *）发送到另一个队列上。 */
        ulReceivedNumber &= 0x03;
        xQueueSendToBackFromISR(xStringQueue,
                                &pcStrings[ulReceivedNumber],
                                &xHigherPriorityTaskWoken);
    }
    /* 如果从xIntegerQueue接收导致任务离开阻塞状态，并且如果离开阻塞状态的任务的优先级高于运行状态的任务的优先级，那么xHigherPriorityTaskWoken将在xQueueReceiveFromISR()中被设置为pdTRUE。
 如果向xStringQueue发送导致任务离开阻塞状态，并且如果离开阻塞状态的任务的优先级高于运行状态的任务的优先级，那么xHigherPriorityTaskWoken将在xQueueSendToBackFromISR()中被设置为pdTRUE。xHigherPriorityTaskWoken被用作portYIELD_FROM_ISR()的参数。如果xHigherPriorityTaskWoken等于pdTRUE，那么调用portYIELD_FROM_ISR()将请求进行上下文切换。如果xHigherPriorityTaskWoken仍然是pdFALSE，那么调用portYIELD_FROM_ISR()将没有任何作用。
Windows端口使用的portYIELD_FROM_ISR()的实现包括一个返回语句，这就是为什么这个函数没有明确地返回一个值。 */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

清单108. 例19所使用的中断服务例程的实现



从中断服务例程中接收字符指针的任务在队列中阻塞，直到有消息到达，在接收到每个字符串时打印出来。它的实现在清单109中显示。

```verilog
static void vStringPrinter( void *pvParameters )
{
char *pcString;
 for( ;; )
 {
 /* Block on the queue to wait for data to arrive. */
 xQueueReceive( xStringQueue, &pcString, portMAX_DELAY );
 /* Print out the string received. */
 vPrintString( pcString );
 } }
```

清单109. 打印出从示例19中的中断服务例程收到的字符串的任务



像往常一样，`main()`在启动调度程序之前创建所需的队列和任务。它的实现在清单110中显示。

```verilog
int main( void )
{
 /* 在使用一个队列之前，必须首先创建它。创建本例所使用的两个队列。一个队列可以容纳uint32_t类型的变量，另一个队列可以容纳char*类型的变量。两个队列最多可以容纳10个项目。真正的应用程序应该检查返回值以确保队列被成功创建。 */
 xIntegerQueue = xQueueCreate( 10, sizeof( uint32_t ) );
 xStringQueue = xQueueCreate( 10, sizeof( char * ) );
 /* 创建一个任务，使用队列将整数传递给中断服务例程。该任务的优先级为1。 */
 xTaskCreate( vIntegerGenerator, "IntGen", 1000, NULL, 1, NULL );
 /* 创建一个任务，打印出从中断服务例程发送给它的字符串。这个任务是以较高的优先级2创建的。 */
 xTaskCreate( vStringPrinter, "String", 1000, NULL, 2, NULL );
 /* 安装软件中断的处理程序。这样做所需的语法取决于正在使用的FreeRTOS端口。这里显示的语法只能用于FreeRTOS的Windows端口，在那里这种中断只是模拟的。 */
 vPortSetInterruptHandler( mainINTERRUPT_NUMBER, ulExampleInterruptHandler );
 /* 启动调度程序，使创建的任务开始执行。 */
 vTaskStartScheduler();
 
 /* 如果一切顺利，那么main()将永远不会到达这里，因为调度器现在正在运行任务。如果main()确实到达这里，那么很可能是没有足够的堆内存可供创建空闲任务。第二章提供了更多关于堆内存管理的信息。 */
 for( ;; );
}
```

清单110. 示例19的`main()`函数



执行示例19时产生的输出如图59所示。可以看出，中断接收了所有五个整数，并产生了五个字符串作为响应。图60中给出了更多的解释。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636337624983-fa5577b7-cf32-4898-927b-584773549def.png)



图59. 执行示例19时产生的输出

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636337712214-d0dc3f3d-9031-4fba-bf94-1675c821d4f7.png)

图60. 例19产生的执行顺序



## 中断嵌套

在任务优先级和中断优先级之间出现混淆是很常见的。本节讨论的是中断优先级，即中断服务例程（ISR）执行时相对于对方的优先级。分配给一个任务的优先级与分配给一个中断的优先级没有任何关系。硬件决定ISR的执行时间，而软件决定任务的执行时间。为响应硬件中断而执行的ISR将中断一个任务，但一个任务不能抢先执行ISR。

支持中断嵌套的端口需要在 FreeRTOSConfig.h 中定义表39中详述的一个或两个常量。

`configMAX_API_CALL_INTERRUPT_PRIORITY `都定义了同一个属性。旧的 FreeRTOS 端口使用 `configMAX_SYSCALL_INTERRUPT_PRIORITY` ，而新的 FreeRTOS 端口使用 `configMAX_API_CALL_INTERRUPT_PRIORITY `。



表39. 控制中断嵌套的常量

| 恒定                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `configMAX_SYSCALL_INTERRUPT_PRIORITY` or `configMAX_API_CALL_INTERRUPT_PRIORITY` | 设置最高的中断优先级，从中可以调用中断安全的FreeRTOS API函数。 |
| `configKERNEL_INTERRUPT_PRIORITY `                           | 设置tick中断所使用的中断优先级，并且必须始终设置为可能的最低中断优先级。如果使用的FreeRTOS端口没有同时使用`configMAX_SYSCALL_INTERRUPT_PRIORITY `常量，那么任何使用中断安全的FreeRTOS API函数的中断也必须以 `configKERNEL_INTERRUPT_PRIORITY `定义的优先级执行。 |



每个中断源都有一个数值优先级，和一个逻辑优先级：

- 数值优先级

  数字优先级只是分配给中断优先级的数字。例如，如果一个中断被分配的优先级为7，那么它的数字优先级就是7。 同样，如果一个中断被分配的优先级为200，那么它的数字优先级就是200。

- 逻辑优先级

  一个中断的逻辑优先级描述了该中断对其他中断的优先级。



如果两个不同优先级的中断同时发生，那么处理器将执行两个中断中逻辑优先级较高的那个中断的ISR，然后再执行两个中断中逻辑优先级较低那个中断的ISR。

一个中断可以中断（嵌套）任何具有较低逻辑优先级的中断，但一个中断不能中断（嵌套）任何具有相同或更高逻辑优先级的中断。

中断的数字优先级和逻辑优先级之间的关系取决于处理器结构。在某些处理器上，分配给一个中断的数字优先级越高，该中断的逻辑优先级就越高，而在其他处理器架构上，分配给一个中断的数字优先级越高，该中断的逻辑优先级就越低。

通过将 `configMAX_SYSCALL_INTERRUPT_PRIORITY `设置为比 `configKERNEL_INTERRUPT_PRIORITY `更高的逻辑中断优先级，可以创建一个完整的中断嵌套模型。这在图61中得到了证明，图中显示了这样一种情况：

- 处理器有七种独特的中断优先级。
- 分配给数字优先级为7的中断比分配给数字优先级为1的中断有更高的逻辑优先级。

- `configKERNEL_INTERRUPT_PRIORITY` 被设置为1。
- `configMAX_SYSCALL_INTERRUPT_PRIORITY` 被设置为3。



![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636338947247-e45b09e7-f928-400c-ac68-47b21a63983a.png)

图61. 影响中断嵌套行为的常量



参考图61：

- 当内核或应用程序处于关键部分时，使用优先级1到3（包括）的中断被阻止执行。以这些优先级运行的ISR可以使用中断安全的FreeRTOS API函数。关键部分的描述见第7章。
- 使用优先级为4或以上的中断不受关键部分的影响，因此在硬件本身的限制下，调度员所做的任何事情都不会阻止这些中断立即执行。以这些优先级执行的ISR不能使用任何 FreeRTOS的API函数。

- 通常，需要非常严格的时间精度的功能（例如电机控制）会使用高于`configMAX_SYSCALL_INTERRUPT_PRIORITY`的优先级，以确保调度器不会在中断响应时间中引入抖动。



### 对ARM `Cortex-M`和ARM GIC用户的说明

`Cortex-M`处理器上的中断配置是混乱的，而且容易出错。为了帮助你的开发，`FreeRTOS Cortex-M`端口自动检查中断配置，但只有在 `configASSERT() `被定义的情况下可以。`configASSERT() `在第11.2节描述。

> Cortex-M：本节仅部分适用于`Cortex-M0`和`Cortex-M0+`内核。



`ARM Cortex`内核和ARM通用中断控制器（GIC）使用数字上的低优先级数字来表示逻辑上的高优先级中断。这似乎有悖于直觉，而且很容易忘记。如果你想给一个中断分配一个逻辑上的低优先级，那么它必须被分配一个数字上的高值。如果你想给一个中断分配一个逻辑上的高优先级。 那么它就必须被分配一个数字上的低值。

`Cortex-M`中断控制器允许最多有8位用于指定每个中断的优先级，使255成为最低的优先级,零是最高优先级。然而，`Cortex-M`微控制器通常只实现8个可能的位数。实际实现的位数取决于微控制器系列。

当八个可能的位中只有一个子集被实现时，只有字节中最有意义的位可以被使用--留下最没有意义的位。未实现的位可以取任何值，但把它们设置为1是正常的。 图62展示了二进制101的优先级是如何存储在一个实现了四个优先位的`Cortex-M`微控制器中。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129867/1636339801928-abdc7bff-8d30-4479-86b1-3cfc39b1608b.png)

图62 二进制101的优先级是如何被一个实现了四个优先位的Cortex-M微控制器存储的



在图62中，二进制值101被移到了最有意义的四个位，因为最没有意义的四个位没有实现。未实现的位被设置为1。

一些库函数希望在优先权值被移到已执行的（最重要的）位子上之后再指定。当使用这样的函数时，图62中所示的优先级可以被指定为十进制95。十进制95是二进制101向上移位4，成为二进制101nnnn（其中"n"是一个未实现的位），未实现的位设置为1，成为二进制1011111。

一些库函数希望在优先权值被移到已执行的（最有意义的）位之前就被指定。当使用这样的函数时，图62中的优先权必须被指定为十进制5。十进制5是二进制101，没有任何移位。

`configMAX_SYSCALL_INTERRUPT_PRIORITY `和 `configKERNEL_INTERRUPT_PRIORITY `必须以允许它们直接写入`Cortex-M`寄存器的方式来指定，所以在优先级值被上移到实现的位后。

`configKERNEL_INTERRUPT_PRIORITY`必须始终设置为可能的最低中断优先级。 未实现的优先级位可以被设置为1，所以不管实际实现了多少个优先级位，这个常数总是可以被设置为255。

`Cortex-M`中断的默认优先级为0--可能的最高优先级。Cortex-M硬件的实现不允许`configMAX_SYSCALL_INTERRUPT_PRIORITY `被设置为0，所以使用FreeRTOS API的中断的优先级决不能停留在其默认值。