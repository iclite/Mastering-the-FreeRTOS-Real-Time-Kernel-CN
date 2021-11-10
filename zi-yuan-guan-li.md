# **资源管理**



## 引言及范围

在多任务系统中，如果一个任务开始访问资源，但在转换出运行状态之前没有完成其访问，则可能发生错误。如果任务使资源处于不一致的状态，那么任何其他任务或中断对相同资源的访问都可能导致数据损坏或其他类似问题。

以下是一些例子:

1. 访问外设<br/>
   考虑以下场景，其中两个任务试图写入一个液晶显示器(LCD)。

   1. 任务A执行并开始写入字符串 “Hello world”到LCD。
   2. 在输出字符串“Hello w”的开头部分后，任务A被任务B抢占。
   3. 任务B在进入阻塞状态前将“中止, 重试, 失败?”写入LCD。
   4. 任务A从它被抢占的位置继续，并完成其字符串-“orld”的剩余字符的输出。 

   液晶显示器现在显示损坏的字符串“Hello w中止, 重试, 失败? orld”。

2. 读、修改、写操作<br/>
   清单111显示了一行C代码，以及一个如何将C代码转换为汇编代码的示例。可以看到，`PORTA`的值首 先从内存中读入寄存器，在寄存器中修改，然后写回内存。这被称为读、修改、写操作。

   ```
    /* 正在编译的C代码。*/
    PORTA |= 0x01; 
    
    /* 编译C代码时生成的汇编代码。*/
    LOAD   R1, [#PORTA]  ; Read a value from PORTA into R1
    MOVE   R2, #0x01     ; Move the absolute constant 1 into R2
    OR     R1, R2        ; Bitwise OR R1 (PORTA) with R2 (constant 1)
    STORE  R1, [#PORTA]  ; Store the new value back to PORTA
   ```

   清单 111. 读、修改、写顺序的示例

   

   这是一个“非原子”操作，因为它需要多个指令才能完成，并且可以被中断。考虑以下场景，其中两个任务试图更新名为`PORTA`的内存映射寄存器。

   1. 任务A将`PORTA`的值加载到寄存器中——操作的读部分。
   2. 任务A在完成同一操作的修改和写部分之前被任务B抢占。
   3. 任务B更新`PORTA`值。则进入阻塞状态。
   4. 任务A从它被抢占的位置继续执行。在将更新后的值写回`PORTA`之前，它修改已经保存在寄存器中的`PORTA`值的副本。

   在这个场景中，任务A更新和回写`PORTA`的过期值。任务B在任务A获取`PORTA`值的副本之后修改`PORTA`，并且	在任务A将其修改后的值写回`PORTA`寄存器之前修改`PORTA`。当任务A写入`PORTA`时，它会覆盖任务B已经执行的修改，从而有效地破坏`PORTA`寄存器值。

   本例使用外设寄存器，但在对变量执行读、修改、写操作时也适用相同的原则。

3. 变量的非原子访问

   更新结构的多个成员，或更新比体系结构的自然字长更大的变量（例如，在16位机器上更新32位变量），这些都是非原子操作的例子。如果它们被打断，可能会导致数据丢失或损坏。

4. 函数可重入性

   如果从多个任务调用函数是安全的，那么该函数就是可重入的。或者来自任务和中断。重入函数被称为线程安全，因为它们可以从多个执行线程访问，而不会有数据或逻辑操作被破坏的风险。

   每个任务维护自己的堆栈和自己的一组处理器(硬件)寄存器值。如果函数只访问存储在堆栈或寄存器中的数据，那么该函数是可重入的，线程安全的。清单112是一个 重入函数的例子。清单113是一个非可重入函数的例子。

```groovy
/* 传递一个参数给函数。这将被传递到堆栈上。或在处理器寄存器中。两种方法都是安全的，因为调用该函数的每个任务或中断都维护自己的堆栈和自己的寄存器值集，所以调用该函数的每个任务或中断都有自己的
1Var1副本。*/
long lAddOneHundred( long lVar1 )
{
/*该函数的作用域变量也将被分配到堆栈或寄存器，这取决于编译器和优化级别。每个调用该函数的任务或中断都有自己的1Var2副本。*/
long lVar2;
    
    lVar2 = lVar1 + 100;
    return lVar2;
}
```

清单112. 一个可重入函数的例子

```groovy
/*在本例中，1Varl是一个全局变量，因此每个调用1NonsenseFunction的任务都会访问该变量的同一个
副本。*/
long lVar1;

long lNonsenseFunction( void )
{
/* lstate是静态的，所以没有在堆栈上分配。每个调用该函数的任务都将访问变量的同一副本。*/
static long lState = 0;
long lReturn;
    
     switch( lState )
 	{
 		case 0 : lReturn = lVar1 + 10;
 				 lState = 1;
 				 break;
		case 1 : lReturn = lVar1 + 20;
 				 lState = 0;
 				 break;
 	}   
 }
```

清单113. 一个非可重入函数的例子



### 互斥

为了确保始终保持数据一致性，可以访问任务之间或任务与中断之间共享的资源。必须使用“互斥”进行管理。目标是确保，一旦一个任务开始访问不可重入且非线程安全的共享资源，在资源返回到一致状态之前，相同的任务对资源具有独占访问权。

FreeRTOS提供了几个可以用来实现互斥的特性。但是，最好的互斥方法是(只是可能，因为通常不实用)将应用程序设计成不共享资源的方式，并且只从单个任务访问每个资源。



### 范围

本章旨在让读者更好地理解：

- 何时以及为什么需要资源管理和控制。
- 多么关键的部分啊。

- 互斥是什么意思。
- 暂停调度器意味着什么。

- 如何使用互斥锁。
- 如何创建和使用看门人任务。

- 什么是优先级反转，以及优先级继承如何减少(而不是消除)它的影响。



## 关键部分和暂停调度器

### 基本的关键部分

基本临界区是由调用宏`taskENTER_critical()`和`taskEXIT_CRITICAL()`包围的代码区域有个。关键部分也是被称为临界区域。

`taskENTER_CRITICAL()`和`taskEXIT_CRITICAL()`不接受任何参数，或返回一个值。注释清单114演示了它们的使用。

> 值：像macro这样的函数并不像实际函数那样“返回一个值”。本书将术语“返回值”应用到宏上，最简单的做法是把宏看成一个函数。

```groovy
/*确保对PORTA寄存器的访问不会被中断，将其置于临界区，进入临界区。*/
taskENTER_CRITICAL();

/*在调用taskENTER_CRITICAL()和调用taskEXIT_CRITICAL()之间不能切换到另一个任务。中断仍然可以在允许中断嵌套的EreeRTos端口上执行，但只允许逻辑优先级高于configMAX_SYSCALL_INTERRUPT_PRIORITY常量的值的中断执行，并且这些中断不允许调用FreeRTOS API函数。*/
PORTA |= 0x01;

/*对PORTA的访问已经完成，因此可以安全退出临界区。*/
taskEXIT_CRITICAL();
```

清单111.  使用临界区来保护对寄存器的访问

本书附带的示例项目使用了一个名为`vPrintString()`的函数来将字符串写入标准输出(即使用FreeRTOS Windows端口时的终端窗口)。`vPrintString()`从许多不同的任务调用;因此，理论上，它的实现可以使用临界区保护对标准输出的访问，如清单115所示。

```groovy
void vPrintString( const char *pcString )
{
 	/*将字符串写入标准输出，使用临界区作为一个粗略的互斥方法。 */
 	taskENTER_CRITICAL();
 	{
 		printf( "%s", pcString );
 		fflush( stdout );
 	}
 	taskEXIT_CRITICAL();
}
```

清单115.  `vPrintString()`的一个可能实现

以这种方式实现的临界区是一种提供互斥的非常粗糙的方法。它们通过禁用中断来工作，要么完全禁用，要么直到`configMAX_SYSCALL_INTERRUPT_PRIORITY`设置的中断优先级—取决于正在使用的FreeRTOS por。抢占式的上下文切换只能发生在一个中断中，因此，只要中断保持禁用，调用`taskENTER_CRITICAL()`的任务就会被保证保持在运行状态，直到临界区段退出。

基本临界段必须保持很短，否则它们将对中断响应时间产生不利影响。对`taskENTER_CRITICAL()`的每个调用都必须与对`taskEXIT_CRITICAL()`的调用紧密匹配。因此，不应该使用临界区来保护标准输出(stdout，即计算机写入其输出数据的流)(如清单115所示)，因为向终端写入数据可能是一个相对较长的操作。本章的例子探讨了可选的解决方案。

关键部分嵌套是安全的，因为内核保留了嵌套深度的计数。只有当嵌套深度返回0时，即每次调用`taskEXIT_CRITICAL()`都执行一次对`taskENTER_CRITICAL()`的调用，临界区才会退出。

调用`taskENTER_CRITICAL()`和`taskEXIT_CRITICAL()`是任务改变FreeRTOS运行的处理器的中断启用状态的唯一合法方法。通过任何其他方法改变中断启用状态都会使宏的嵌套计数无效。

`taskENTER_CRITICAL()`和`taskEXIT_CRITICAL()`不会以“FromlSR”结束，所以一定不能从中断服务程序中调用。`taskENTER_CRITICAL_FROM_ISR()` 是一个中断安全版本的`taskENTER_CRITICAL()`，和`taskEXIT_CRITICAL_FROM_ISR()`是一个允许中断嵌套的FreeRTOS端口，它们将在不允许中断嵌套的端口中被废弃。

`taskENTER_CRITICAL_FROM_ISR()`返回一个值，这个值必须被传递到匹配的调用`taskEXIT_CRITICAL_FROM_ISR()`。清单116演示了这一点。

```groovy
void vAnInterruptServiceRoutine( void )
{
/*声明一个变量，该变量的返回值来自taskENTER_CRITICAL_FROM_ISR()将被保存。*/
UBaseType_t uxSavedInterruptStatus;
    
	/* ISR的这一部分可以被任何更高优先级的中断中断。*/ 
	/*使用taskENTER_CRITICAL_FROM_ISR()来保护该ISR的一个区域。保存从taskENTER_CRITICAL_FROM_ISR()返回的值，以便它可以被传递到匹配的调用taskEXIT_CRITICAL_FROM_ISR()。*/
	uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();
 
	/* ISR的这一部分在调用taskENTER_CRITICAL_FROM_ISR()和taskEXIT_CRITICAL_FROM_ISR()之间，所以只能被优先级高于configMAX_SYSCALL_INTERRUPT_PRIORITY常量设置的优先级。*/
    
	/*通过调用taskEXIT_CRITICAL_FROM_ISR()再次退出临界区，并传入由匹配的调用返回的值taskENTER_CRITICAL_FROM_ISR()。*/
 	taskEXIT_CRITICAL_FROM_ISR( uxSavedInterruptStatus );
	/* ISR的这一部分可以被任何更高优先级的中断中断。*/
}
```

清单116.  在中断服务程序中使用临界区

使用更多的处理时间执行进入并随后退出临界区段的代码，而不是执行实际受临界区段保护的代码，这是一种浪费。基本临界区段进入和退出都非常快，而且总是确定性的，因此当受保护的代码区域非常短时，它们的使用非常理想。



### 挂起(或锁定)调度器

还可以通过挂起调度器来创建临界区。暂停调度器有时也被称为“锁定”调度器。

基本临界区保护代码区域不受其他任务和中断的访问。通过暂停调度器实现的临界区仅保护代码区域不被其他任务访问。因为中断仍然处于启用状态。

如果临界区段太长，不能简单地通过禁用中断来实现，则可以通过暂停调度程序来实现。然而，在调度器挂起时中断活动会使恢复(或“取消挂起”)调度器成为一个相对较长的操作，因此必须考虑在每种情况下使用哪种方法最好。



### vTaskSuspendAll() API函数

```groovy
void vTaskSuspendAll( void );
```

清单 117.  `vTaskSuspendAll()` API函数原型

调度程序通过调用`vTaskSuspendAll()`被挂起。挂起调度程序可以防止发生上下文切换，但保留中断。如果在调度器挂起时中断请求上下文切换，则请求将被挂起。并且只在恢复(未挂起)调度程序时执行。

当调度器挂起时，不能调用FreeRTOS API函数。



### xTaskResumeAll()的API函数

```groovy
BaseType_t xTaskResumeAll( void );
```

清单118.  `xTaskResumeAll()` API函数原型

通过调用`xTaskResumeAl()`恢复(未挂起)调度器。

表 40. `xTaskResumeAll()`的返回值

| 返回值 | 描述                                                         |
| ------ | ------------------------------------------------------------ |
| 返回值 | 在调度器挂起时请求的上下文切换将被挂起并仅在恢复调度器时执行。如果在`xTaskResumeAll()`返回之前执行了挂起的上下文切换，则返回pdTRUE。否则返回pdFALSE。 |

对`vTaskSuspendAll()`和`xTaskResumeAl()`的调用嵌套是安全的，因为内核保留了嵌套深度的计数。调度器只有在嵌套深度返回0时才会恢复，也就是在每次调用`vTaskSuspendAll()`都执行一次`xTaskResumeAll()`时。

清单119显示了`vPrintString()`的实际实现，它挂起调度程序以保护对终端输出的访问。

```groovy
void vPrintString( const char *pcString )
{
 	/*将字符串写入标准输出，以互斥的方式挂起调度程序*/
 	vTaskSuspendScheduler();
 	{
 		printf( "%s", pcString );
 		fflush( stdout );
 	}
 	xTaskResumeScheduler();
}
```

清单119. `vPrintString()`的实现



### 互斥锁(和二进制信号量)

互斥锁是一种特殊类型的二进制信号量，用于控制对两个或多个任务共享的资源的访问。MUTEX这个词来源于“互斥”。`configUSE_MUTEXES`必须在`FreeRTOSConfig.h`中设置为1，互斥才能可用。

当在互斥场景中使用互斥时，可以将互斥看作与共享资源相关联的令牌。以便任务合法地访问资源。它必须首先成功地“接受”令牌(作为令牌持有者)。当令牌持有者使用完资源后，它必须“归还”令牌。只有当该令牌被返回时，另一个任务才能成功地获取该令牌，然后安全地访问相同的共享资源。任务不允许访问共享资源，除非它持有令牌。这种机制如图63所示。

尽管互斥锁和二进制信号量共享许多特性。图63所示的场景(其中使用互斥锁进行互斥)与图53所示的场景(其中使用二进制信号量进行同步)完全不同。主要的区别在于信号量被获取后发生了什么：

- 必须始终返回用于互斥的信号量。
- 用于同步的信号量通常被丢弃且不返回。

![img](https://cdn.nlark.com/yuque/0/2021/jpeg/23129846/1636013046498-a572266e-5d62-4262-8ab5-36482102960d.jpeg)

图63 使用互斥对象实现的互斥

该机制纯粹通过应用程序编写人员的规程来工作。没有理由一个任务不能在任何时候访问资源，但是每个任务“同意”不这样做。除非它能成为互斥锁持有者。



### xSemaphoreCreateMutex() API函数

FreeRTOS V9.0.0还包括`xSemaphoreCreateMutexStatic()`函数，该函数在编译时分配静态创建互斥锁所需的内存:互斥锁是一种信号量类型。所有不同类型的FreeRTOS信号量的句柄都存储在`SemaphoreHandle_t`类型的变量中。

在使用互斥锁之前，必须先创建它。要创建互斥量类型的信号量，请使用`xSemaphoreCreateMutex()` APl函数。

```groovy
SemaphoreHandle_t xSemaphoreCreateMutex( void );
```

清单120. `xSemaphoreCreateMutex()` API函数原型

表 41. `xSemaphoreCreateMutex()`的返回值

| 参数名称/返回值 | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| 返回值          | 如果返回`NULL`，则无法创建互斥锁，因为没有足够的堆内存供FreeRTOS分配互斥锁数据结构。第2章提供了堆内存管理的更多信息。非`NULL`返回值表明互斥锁已经成功创建。返回值应该存储为创建的互斥锁的句柄。 |



### 示例20. 重写vPrintString()以使用信号灯

本例创建了`vPrintString()`的新版本，称为`prvNewPrintString()`，然后从多个任务中调用这个新函数。`prvNewPrintString()`在功能上与`vPrintString()`相同，但使用互斥锁来控制对标准输出的访问，而不是通过锁定调度器。清单121显示了`prvNewPrintString()`的实现。

```groovy
static void prvNewPrintString( const char *pcString )
{
	/*互斥锁是在调度器启动之前创建的，所以在任务执行时已经存在尝试获取互斥锁，如果互斥锁不能立即可用，则无限期阻塞以等待它。对xSemaphoreTake()的调用只有在成功获得互斥锁时才会返回。所以不需要检查函数的返回值。如果使用了任何其他延迟周期，那么代码必须在访问共享资源(在本例中是标准输出)之前检查xSemaphorerake()是否返回pdTRUE。正如本书前面提到的，不建议在生产代码中使用无限期超时。*/
 	xSemaphoreTake( xMutex, portMAX_DELAY );
 	{
	/*只有成功获得互斥锁后，才会执行下面的行。现在可以自由访问标准输出，因为在任何时候只有一个任务可以拥有互斥锁。*/
	 printf( "%s", pcString );
     fflush( stdout );
        
    /* 互斥锁必须返回! */
 	}
 	xSemaphoreGive( xMutex );
}
```

清单121. `prvNewPrintString()`的实现

`prvNewPrintString()`由`prvPrintTask()`实现的任务的两个实例反复调用。每次调用之间使用随机延迟时间。task参数用于向任务的每个实例传递唯一的字符串。清单122显示了`prvPrintTask()`的实现。

```groovy
static void prvPrintTask( void *pvParameters )
{
char *pcStringToPrint;
const TickType_t xMaxBlockTimeTicks = 0x20;
 	/*创建该任务的两个实例。任务打印的字符串“使用任务的参数传递给任务”。参数被转换为所需的类型。*/
 	pcStringToPrint = ( char * ) pvParameters;
    
 	for( ;; )
 	{
 		/*使用新定义的函数输出字符串。*/
 		prvNewPrintString( pcStringToPrint );
		/*等待一个伪随机时间。请注意rand()不一定是可重入的，但在这种情况下，它实际上并不重要，因为代码并不关心返回的值是什么。在更安全的应用程序中，应该使用已知可重入的rand()版本——或者应该在临界区段保护rand()的调用。*/
 		 vTaskDelay( ( rand() % xMaxBlockTimeTicks ) );
 	} 
}
```

清单122.  示例20中`prvPrintTask()`的实现

正常情况下，`main()`只是创建互斥锁，创建任务，然后启动调度器。实现如清单123所示。

`prvPrintTask()`的两个实例以不同的优先级创建，因此优先级较低的任务有时会被优先级较高的任务抢占。由于使用互斥锁来确保每个任务对终端的访问是互斥的，所以即使发生了抢占，被删除的字符串也将是正确的，并且不会损坏。通过减少任务处于阻塞状态的最大时间，可以增加抢占的频率，该时间是由`xMaxBlockTimeTicks`常量设置的。

使用FreeRTOS Windows端口的示例20的注意事项:

- 调用`printf()`将生成一个Windows系统调用。Windows系统调用不在FreeRTOS的控制范围之内，并且会带来不稳定性。
- Windows系统调用的执行方式意味着，即使没有使用互斥锁，也很少看到损坏的字符串。

```groovy
int main( void )
{
	/*在使用信号量之前，必须显式地创建信号量。在这个例子中，创建了一个互斥量类型的信号量。*/
 	xMutex = xSemaphoreCreateMutex();
 	/*在创建任务之前检查信号量是否创建成功。*/
 	if( xMutex != NULL )
 	{
		/*创建两个写入stdout的任务实例。他们写入的字符串作为任务的参数传递给任务。任务按不同的优先级创建，因此会发生一些抢占。*/
 		xTaskCreate( prvPrintTask, "Print1", 1000, 
 					"Task 1 ***************************************\r\n", 1, NULL );
 		xTaskCreate( prvPrintTask, "Print2", 1000, 
 					"Task 2 ---------------------------------------\r\n", 2, NULL );
        
		/*启动调度程序，使创建的任务开始执行。*/
 		vTaskStartScheduler();
 }
 
/*如果一切正常，那么main()将永远不会到达这里，因为调度程序现在将运行任务。如果main()确实到达了这里，那么很可能是没有足够的堆内存可用来创建空闲任务。第2章提供了堆内存管理的更多信息。*/
 for( ;; );
}
```

清单123. 示例20的·main()·的实现

执行示例20时产生的输出如图64所示。图65描述了可能的执行顺序

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636015921770-e7998136-2952-4ab0-85cf-b2098474a4c4.png)

图64. 执行示例20时产生的输出

如图64所示，终端上显示的字符串没有损坏。随机排序是任务所使用的随机延迟周期的结果。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636016039460-db7c1686-d562-441a-b61c-13b7c2b2e5d0.png)

图65. 例20中可能的执行序列



### 优先级反转

图65演示了使用互斥锁提供互斥的一个潜在缺陷。所描述的执行顺序显示了高优先级任务2必须等待低优先级任务1放弃对互斥锁的控制。高优先级的任务被低优先级的任务以这种方式延迟，我们称之为“优先级反转”。如果一个中等优先级的任务在高优先级任务等待信号量的同时开始执行，这种不受欢迎的行为将进一步扩大——结果将是一个高优先级任务在等待一个低优先级任务——而低优先级任务甚至无法执行。图66显示了这种最糟糕的情况。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636016540177-16c01691-37c6-413e-a887-810009f27c71.png)

图66 最坏情况下的优先权反转情况

优先级反转可能是一个严重的问题，但在小型嵌入式系统中，通过考虑如何访问资源，通常可以在系统设计时避免这个问题。



### 优先级继承

FreeRTOS互斥体和二进制信号量非常相似，不同之处是互斥体包含一个基本的优先级继承机制。而二进制信号量则不然。优先级继承是一种最小化优先级倒置负面影响的方案。它并不修复优先级倒置，只是通过确保倒置总是有时间限制来减少其影响。但是，优先级继承使系统定时分析变得复杂，依赖它来进行正确的系统操作并不是一个好的实践。

优先级继承的工作原理是，暂时将互斥锁持有者的优先级提高到试图获取同一个互斥锁的优先级最高的任务的优先级。持有互斥锁的低优先级任务继承了等待互斥锁的任务的优先级。图67演示了这一点。互斥锁持有者的优先级在返回互斥锁时自动重置为初始值。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636016800950-a3500eeb-745b-4217-b4a4-ae26f73ce0ee.png)

图67. 优先权继承使优先权反转的影响最小化

正如刚才看到的，优先级继承功能会影响使用互斥锁的任务的优先级。因此，互斥锁不能在中断服务程序中使用。



### 死锁（或致命拥抱）

“死锁”是使用互斥对象进行互斥的另一个潜在陷阱。僵局有时也被称为“致命拥抱”。

当两个任务都在等待对方持有的资源而无法继续时，就会发生死锁。考虑以下场景，任务A和任务B都需要获取互斥锁X和互斥锁Y来执行一个操作：

1. 任务A执行并成功接受互斥锁X。

2. 任务A被任务B抢占。

3. 任务B在尝试获取互斥锁X之前成功获取了互斥锁Y，但是互斥锁X被任务A持有，所以对任务B不可用。任务B选择进入阻塞状态，等待互斥锁X被释放。

4. 任务A继续执行。它尝试取互斥量Y，但是互斥量Y被任务B持有，所以不能被任务A	使用。任务A选择进入阻塞状态来等待互斥量Y被释放。

在这个场景的最后，任务A正在等待任务B持有的互斥锁，任务B也在等待任务A持有的互斥锁。

与优先级倒置一样，避免死锁的最佳方法是在设计时考虑它的潜力，并设计系统以确保不会发生死锁。特别是，正如本书前面所述，任务无限期地等待(没有超时)来获得互斥锁通常是不好的做法。相反，使用比期望等待互斥锁的最大时间稍长一点的超时时间——那么在此时间内无法获得互斥锁将是设计错误的症状，可能是死锁。

在实践中，死锁在小型嵌入式系统中不是一个大问题，因为系统设计人员可以很好地理解整个应用程序，因此可以识别和消除可能发生死锁的区域。



### 递归互斥锁

任务本身也有可能死锁。如果任务试图执行，就会发生这种情况。多次使用同一个互斥锁，而不首先返回互斥锁。考虑以下场景：

1. 任务成功获取互斥锁。

2. 在持有互斥锁时，任务调用库函数

3. 库函数的实现尝试获取相同的互斥锁，并进入阻塞状态以等待互斥锁可用。

在本场景结束时，任务处于阻塞状态，等待返回互斥锁，但任务已经是互斥锁持有者了。出现死锁，因为任务处于阻止状态以等待其自身。

这种类型的死锁可以通过使用递归锁来代替标准互斥锁来避免。一个递归锁可以被同一个任务多次使用。并且只有在每次调用“取”递归互斥时执行一次“给”递归后才返回。

标准互斥体和递归互斥体的创建和使用方式类似:

- 标准互斥对象是使用`xSemaphoreCreateMutex()`创建的。递归互斥是使用`xSemaphoreCreateRecursiveMutex()`创建的。这两个API函数具有相同的原型。
- 标准互斥对象使用`xSemaphoreTake()`来“获取”。使用`xSemaphoreTakeRecursive()`“获取”递归互斥。这两个API函数具有相同的原型。

- 标准互斥对象使用`xsemaphoregve()`来“给定”。递归互斥是使用`xSemaphoreTakeRecursive()`给出的。这两个API函数具有相同的原型。

清单124. 演示了如何创建和使用递归锁

```groovy
/*递归互斥是SemaphoreHandle_t类型的变量。 */
SemaphoreHandle_t xRecursiveMutex;

/* 创建和使用递归互斥锁的任务的实现。 */
void vTaskFunction( void *pvParameters )
{
const TickType_t xMaxBlock20ms = pdMS_TO_TICKS( 20 );
 	/* 在使用递归互斥锁之前，必须显式地创建它。 */
 	xRecursiveMutex = xSemaphoreCreateRecursiveMutex();
    
	/*检查信号量是否创建成功。configASSERT()在11.2节中有描述。*/
 	configASSERT( xRecursiveMutex );
    
 	/*对于大多数任务，这个任务是作为一个无限循环来实现的。*/
 	for( ;; )
 	{
		/* ... */
        /*取递归互斥锁。 */
        if( xSemaphoreTakeRecursive( xRecursiveMutex, xMaxBlock20ms ) == pdPASS )
        {
            		/*成功获取递归互斥锁。任务现在可以访问互斥锁保护的资源。此时，递归调用计数(即对xSemaphoreTakeRecursive()的嵌套调用的数量)为1，因为递归互斥对象只被取过一次。* /
            
 			/*当它已经持有递归的互斥对象时，任务再次接受互斥对象。在实际的应用程序中这只可能发生在该任务调用的子函数中，因为没有实际的理由要多次使用同一个互斥锁。调用任务已经是互斥锁的持有者，所以对xSemaphorerakeRecursive()的第二次调用只是将递归调用的声音增加到2。*/
 		   	xSemaphoreTakeRecursive( xRecursiveMutex, xMaxBlock20ms );
            
 			/* ... */
            		/*任务在访问了互斥锁保护的资源后返回互斥锁。此时，递归调用计数为2，因此对xSemaphoreGiveRecursive()的第一次调用不返回互斥锁。相反它只是将递归调用计数减回1。*/
 			xSemaphoreGiveRecursive( xRecursiveMutex );
            
			/*下一次调用xSemaphoreGiveRecursive()时，返回的调用计数减少为0。所以这次返回的是递归互斥锁。*/
 			xSemaphoreGiveRecursive( xRecursiveMutex );
            
			/*现在每次调用xSemaphoreGiveRecursive()都会执行一个xSemaphoreGiveRecursive()，所以任务不再是互斥锁的持有者。*/
		}
	} 
}
```

清单124.  创建和使用递归互斥锁



### 互斥体和任务调度

如果两个具有不同优先级的任务使用同一个互斥锁，那么FreeRTOS调度策略会明确任务执行的顺序;能够运行的最高优先级任务将被选择为进入运行状态的任务。例如，如果高优先级任务处于阻塞状态，等待低优先级任务持有的互斥锁，那么一旦低优先级任务返回互斥锁，高优先级任务就会抢占低优先级任务。高优先级的任务将成为互斥锁的持有者。在图67中已经看到了这个场景。

然而，当任务具有相同的优先级时，通常会对任务的执行顺序做出错误的假设。如果任务1和任务2有相同的优先级。任务1处于阻塞状态，等待由任务2持有的互斥，那么当任务2 "给出 "突变时，任务1不会抢占任务2。相反，任务2将保持在 运行状态，而任务1将简单地从阻塞状态移动到就绪状态。这种情况由图68显示了，其中垂直线标记了勾号中断发生的时间。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636018020725-90971f22-7bdb-457e-a772-b0aa1f623fe6.png)

图68 具有相同优先级的任务使用同一个互斥锁时可能的执行顺序

在图68所示的场景中，当互斥锁可用时，FreeRTOS调度器不会让任务一成为运行状态任务，因为:

1. 任务1和任务2具有相同的优先级，所以除非任务2进入阻塞状态，否则在下一次tick中断之前不会切换到任务1(假设`configUSE TIME SLICING`在`FreeRTOSConfig.h`中设置为1)。
2. 如果一个任务在紧循环中使用了一个互斥锁，并且每次任务给出互斥锁时都会发生上下文切换，那么这个任务只会在很短的时间内保持运行状态。如果两个或多个任务在一个紧循环中使用同一个互斥锁，那么在任务之间快速切换会浪费处理时间。

如果一个互斥锁在一个紧循环中被多个任务使用，并且使用互斥锁的任务具有相同的优先级，那么必须小心确保任务收到大约相等的处理时间。图69演示了任务可能无法获得相同数量的处理时间的原因。它显示了以相同优先级创建清单125所示任务的两个实例时可能发生的执行序列。

```groovy
/*在紧循环中使用互斥锁的任务的实现。该任务在本地缓冲区中创建一个文本字符串，然后将该字符串写入显示。对显示器的访问受互斥锁的保护。*/
void vATask( void *pvParameter )
{
extern SemaphoreHandle_t xMutex;
char cTextBuffer[ 128 ];
    for( ;; )
	{
		/* 生成文本字符串-这是一个快速的操作。 */
        vGenerateTextInALocalBuffer( cTextBuffer );
        
		/*获取保护对显示的访问的互斥锁。*/
 		xSemaphoreTake( xMutex, portMAX_DELAY );
        
		/* 将生成的文本写入显示器-这是一个缓慢的操作。*/
		vCopyTextToFrameBuffer( cTextBuffer );
        
 		/* 文本已经写入显示器，因此返回互斥锁。*/
 		xSemaphoreGive( xMutex );
 	} 
}
```

清单125. 一个在紧密循环中使用互斥的任务

清单125中的注释注意到，创建字符串是一个快速的操作，而更新显示则是一个缓慢的操作。因此，当显示被更新时，互斥锁被持有，任务将在大部分运行时间内持有互斥锁。

在图69中，垂直线标记了记号中断发生的时间。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636018507067-3d56d6a9-cf05-44e0-a084-80d0db8b5523.png)

图69 任务的两个实例可能发生的执行序列如清单125所示，以相同的优先级创建

图69中的第7步显示任务1重新进入阻塞状态，这发生在 `xSemaphoreTake()` API函数中发生。

图69表明，任务1将被阻止获得互斥锁，直到时间片的开始与任务2不是互斥锁持有者的短时间的重合。

图69中所示的情况可以通过在调用`taskYIELD()`后添加一个调用来避免。调用`xSemaphoreGive()`后，可以避免这种情况。这在清单126中得到了证明，如果在任务持有互斥锁时`taskYIELD()`被调用，那么它就会被调用。如果在任务持有互斥锁的时候，标记计数发生了变化，就会调用`taskYIELD()`。

```groovy
void vFunction( void *pvParameter )
{
extern SemaphoreHandle_t xMutex;
char cTextBuffer[ 128 ];
TickType_t xTimeAtWhichMutexWasTaken;
 
    for( ;; )
    {
 		/* 生成文本字符串-这是一个快速的操作。 */
 		vGenerateTextInALocalBuffer( cTextBuffer );
        
 		/*获取对显示器进行protectinq访问的互斥锁。 */
 		xSemaphoreTake( xMutex, portMAX_DELAY );
        
 		/* 记录使用互斥锁的时间。 */
 		xTimeAtWhichMutexWasTaken = xTaskGetTickCount();
        
 		/* 将生成的文本写入显示器——这是一个缓慢的操作。 */
 		vCopyTextToFrameBuffer( cTextBuffer );
        
 		/* 文本已经写入显示器，因此返回互斥锁。*/
 		xSemaphoreGive( xMutex );
        
		/*如果每次迭代都调用taskYIELD()，那么该任务只会在很短的一段时间内保持运行状态，而在任务之间快速切换会浪费处理时间。因此，只有当在互斥锁被持有时时钟计数改变时才调用taskYIELD()。* /
 		if( xTaskGetTickCount() != xTimeAtWhichMutexWasTaken )
 		{
 			taskYIELD();
		 }
 	} 
}
```

清单126. 确保在循环中使用互斥锁的任务可以获得更多的处理时间，同时也确保不会因为在任务之间切换太快而浪费处理时间。



## 看门人任务

看门人任务提供了一种干净的实现互斥的方法，并且没有优先级倒置或死锁的风险。

看门人任务是对资源拥有唯一所有权的任务。只有看门人任务被允许直接访问资源，其他需要访问资源的任务只能通过使用看门人的服务间接访问资源。



### 示例21. 重写vPrintString()以使用看门人任务

示例21提供了`vPrintString()`的另一种替代实现。这一次，使用一个看门人任务来管理对标准输出的访问。当任务想要将消息写入标准输出时，它不会直接调用print函数，而是将消息发送给看门人。

看门人任务使用FreeRTOS队列将访问序列化到标准输出。任务的内部实现不必考虑互斥，因为它是唯一允许直接访问标准输出的任务。

看门人任务大部分时间处于阻塞状态，等待消息到达队列。当消息到达时，看门人只需将消息写入标准输出，然后返回阻塞状态以等待下一条消息。看门人任务的实现如清单128所示。

中断可以发送到队列，因此中断服务程序也可以安全地使用看门人的服务向终端写入消息。在本例中，使用了一个钩子函数，每200次写一条消息。

滴答钩子(或滴答回调)是内核在每次滴答中断期间调用的函数。使用滴答钩子函数:

1. 在`FreeRTOSConfig.h`中设置`configUSE_TICK_HOOK`为1。
2. 提供钩子函数的实现，使用清单127所示的准确的函数名称和原型如清单127所示。

```groovy
void vApplicationTickHook( void );
```

清单127.  钩子函数的名称和原型

钩子函数在勾选中断的上下文中执行，因此必须保持非常短的时间。所以必须保持非常短，必须只使用适量的堆栈空间，并且不能调用任何不以“`FromISR()`”结尾的FreeRTOS API 函数，而不是以 “`FromISR() `”结尾。

调度器将总是在滴答钩子函数之后立即执行，所以中断安全。从滴答钩子调用的FreeRTOS API函数不需要使用其 `pxHigherPriorityTaskWoken`参数，并且该参数可以被设置为`NULL`。

```groovy
static void prvStdioGatekeeperTask( void *pvParameters )
{
char *pcMessageToPrint;
    
    /*这是唯一允许写入标准输出的任务。任何其他想要将字符串写入输出的任务都不会直接访问标准输出，而是将字符串发送给该任务。因为只有该任务访问标准输出，所以在任务本身的实现中不需要考虑互斥或序列化问题。*/
 	for( ;; )
	{
		/*等待消息到达。指定了一个不确定的块时间，因此不需要检查返回值-该函数只在成功接收到消息时返回。*/
 		xQueueReceive( xPrintQueue, &pcMessageToPrint, portMAX_DELAY );
        
 		/* Output the received string. */
 		printf( "%s", pcMessageToPrint );
 		fflush( stdout );
        
 		/*返回循环以等待下一条消息。*/
 	} 
}
```

清单128.  看门人任务

写入队列的任务如清单129所示。与前面一样，将创建任务的两个独立实例，并使用任务参数将任务写入队列的字符串传递给任务。

```groovy
static void prvPrintTask( void *pvParameters )
{
int iIndexToString;
const TickType_t xMaxBlockTimeTicks = 0x20;
    
    /*创建该任务的两个实例。任务参数用于将字符串数组的索引传递给任务。将其转换为所需的类型。*/
 	iIndexToString = ( int ) pvParameters;
    
    for( ;; )
 	{
        /*输出字符串，不是直接输出，而是通过队列将字符串的指针传递给看门人任务。队列是在调度程序启动之前创建的，因此在该任务第一次执行时已经存在。没有指定块时间，因为队列中总是有空间。*/
 		xQueueSendToBack( xPrintQueue, &( pcStringsToPrint[ iIndexToString ] ), 0 );
        
        /*等待一个伪随机时间。请注意rand()不一定是可重入的，但在这种情况下，它实际上并不重要，因为代码并不关心返回的值是什么。在更安全的应用程序中，应该使用已知可重入的rand()版本——或者应该使用临界区来保护对rand()的调用。*/
 		vTaskDelay( ( rand() % xMaxBlockTimeTicks ) );
 	}
}
```

清单129.  示例21的打印任务实现

滴答钩子函数对自己被调用的次数进行计数，每次计数达到200时，就向看门人任务发送消息。仅出于演示目的，滴答钩子写到队列的前面，而任务写到队列的后面。滴答钩子实现如清单130所示。

```groovy
void vApplicationTickHook( void )
{
static int iCount = 0;

	/*每200个节拍打印一条消息。消息不是直接写出来的，而是发送给看门人任务。*/
 	iCount++;
 	if( iCount >= 200 )
 	{
		/*由于xQueueSendToFrontFromISR()是从标记钩子调用的，所以不需要使用xHigherPriorityTaskwoken参数(第三个参数)，该参数设置为NULL。*/
 		xQueueSendToFrontFromISR( xPrintQueue, 
 					  &( pcStringsToPrint[ 2 ] ), 
 					  NULL );
 
 		/* 重置计数，准备在200滴答时间内再次打印字符串。 */
 		iCount = 0;
 	} 
}
```

清单130.  滴答钩子函数的实现

正常情况下，`main()`创建运行示例所需的队列和任务，然后启动调度程序。`main()`的实现如清单131所示。

```groovy
/*定义任务和中断将通过看门人打印出来的字符串。 */
static char *pcStringsToPrint[] =
{
 	"Task 1 ****************************************************\r\n",
 	"Task 2 ----------------------------------------------------\r\n",
 	"Message printed from the tick hook interrupt ##############\r\n"
};
    
/*-----------------------------------------------------------*/
/*声明一个类型为QueueHandle_t的变量。该队列用于发送消息，从打印任务和标记中断到看门人任务。*/
QueueHandle_t xPrintQueue;
/*-----------------------------------------------------------*/

int main( void )
{
	/*在使用队列之前，必须显式创建队列。队列被创建为最多容纳5个字符的指针。*/
 	xPrintQueue = xQueueCreate( 5, sizeof( char * ) );
 	/* 检查队列是否创建成功。 */
 	if( xPrintQueue != NULL )
 	{
        /*创建任务的两个实例，发送消息给守门人。任务使用的字符串的索引通过任务参数传递给任务(xTaskcreate()的第4个参数)。这些任务按不同的优先级创建，因此高优先级任务偶尔会抢占低优先级任务。*/
 		xTaskCreate( prvPrintTask, "Print1", 1000, ( void * ) 0, 1, NULL );
 		xTaskCreate( prvPrintTask, "Print2", 1000, ( void * ) 1, 2, NULL );
       
        /*创建看门人任务。这是唯一允许直接访问标准输出的任务。*/
 		xTaskCreate( prvStdioGatekeeperTask, "Gatekeeper", 1000, NULL, 0, NULL );
 
 		/* 启动调度程序，使创建的任务开始执行。 */
 		vTaskStartScheduler();
 	}
    
    /*如果一切正常，那么main()将永远不会到达这里，因为调度程序现在将运行任务。如果main()确实到达了这里，那么很可能是没有足够的堆内存可用来创建空闲任务。第2章提供了堆内存管理的更多信息。*/
 	for( ;; );
}
```

清单131.  示例21的`main()`的实现

执行示例21时产生的输出如图70所示。可以看到，来自任务的字符串和来自中断的字符串都正确输出，没有损坏。

![img](https://cdn.nlark.com/yuque/0/2021/png/23129846/1636078578045-f89e7516-8666-4c7d-9fa8-aabc2e5f588e.png)

图70  执行例21时产生的输出

看门人任务的优先级比打印任务低，因此发送给看门人的消息会一直留在队列中，直到两个打印任务都处于阻塞状态。在某些情况下，为看门人分配更高的优先级是合适的，这样消息就可以立即得到处理——但是这样做的代价是，看门人会延迟较低优先级的任务，直到完成对受保护资源的访问。