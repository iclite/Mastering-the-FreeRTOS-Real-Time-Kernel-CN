# 队列管理

## 章节介绍与范围

“队列” 提供了一个**任务到任务**，**任务到中断**和**中断到任务**的通讯机制。

### 范围

本章节的目标是告诉读者很好的理解：

* 如何创建队列。
* 一个队列是如何管理它所包含的数据。
* 如何发送数据到队列。
* 如何从队列接收数据。
* 阻塞队列意味着什么。
* 如何阻塞多个队列。
* 如何覆盖队列中的数据。
* 如何清除一个队列。
* 读取和写入一个队列对任务优先级的影响。

本章节只涵盖了任务到任务通讯。任务到中断与中断到任务通讯在第 6 章中说明。

## 队列的特征

### 数据存储

一个队列能保存有限数量的固定大小的数据单元。一个队列能保存单元的最大数量叫做 “长度”。每个队列数据单元的长度与大小是在创建队列时设置的。

队列通常是一个先入先出（FIFO）的缓冲区，即数据在队列末尾（tail）被写入，在队列前部（head）移出。图 31 展示了数据被写入和移出作为 FIFO 使用的队列。也可以写入队列的前端，并覆盖已位于队列前端的数据。

![&#x56FE; 31. &#x5199;&#x5165;&#x961F;&#x5217;&#x548C;&#x4ECE;&#x961F;&#x5217;&#x8BFB;&#x53D6;&#x7684;&#x793A;&#x4F8B;&#x5E8F;&#x5217;](.gitbook/assets/wei-xin-jie-tu-20190911112254.png)

有两种方法可以实现队列的行为：

1. 通过复制实现队列：复制队列是指将发送到队列的数据一个字节一个字节地复制到队列中。
2. 通过引用实现队列：引用队列意味着队列只持有指向发送到队列的数据的指针，而不是数据本身。

FreeRTOS 是通过使用复制方法实现队列。这是考虑到复制队列比引用队列更强大，更容易使用，因为：

* 堆栈变量可以直接发送到队列，即使该变量将在声明它的函数退出后，不再存在。
* 可以将数据发送到队列，而无需先分配缓冲区来保存数据，然后将数据复制到分配的缓冲区中。
* 发送任务可以立即重用发送到队列的变量或缓冲区。
* 发送任务和接收任务是完全解耦的，应用程序设计人员不需要关心哪个任务拥有数据，或者哪个任务负责发布数据。
* 复制队列并不会阻止队列也被用于引用队列。例如，当正在排队的数据的大小使得将数据复制到队列不切实际时，可以将指向数据的指针复制到队列中。
* RTOS 完全负责分配用于存储数据的内存。
* 在受内存保护的系统中，任务可以访问的 RAM 将受到限制。在这种情况下，只有当发送和接收任务都可以访问存储数据的 RAM 时，才可以使用引用排队。按复制排队不受此限制；内核总是以完全特权运行，允许使用队列跨内存保护边界传递数据。

### 多任务访问

队列本身就是对象，任何知道它们存在的任务或 ISR 都可以访问它们。任意数量的任务可以写入同一个队列，任意数量的任务也可以从同一个队列读取。在实践中，队列有多个写入者是非常常见的，但是队列有多个读取者就不那么常见了。

### 阻塞队列读取

当任务尝试从队列中读取时，它可以选择指定 “阻塞” 时间。 如果队列已经为空，则这是任务将保持在阻塞状态以等待队列中的数据可用的时间。 当另一个任务或中断将数据放入队列时，处于阻塞状态且等待数据从队列中变为可用的任务将自动移至就绪状态。 如果指定的阻塞时间在数据可用之前到期，则任务也将自动从 “阻塞” 状态移动到 “就绪” 状态。

队列可以有多个读取者，因此单个队列可能会由多个在其上阻塞等待数据的任务。 在这种情况下，只有一个任务在数据可用时将被解除阻塞。 取消阻塞的任务始终是等待数据的最高优先级任务。 如果被阻塞的任务具有相同的优先级，那么等待数据最长的任务将被阻塞。

### 阻塞队列写入

与从队列读取数据时一样，任务也可以在向队列写入数据时指定阻塞时间。在这种情况下，如果队列已经满了，则阻塞时间是任务应该保持在阻塞状态以等待队列上可用空间的最长时间。

队列可以有多个写入者，因此对于一个完整的队列，可能有多个任务阻塞在队列上，等待完成发送操作。在这种情况下，当队列上的空间可用时，只有一个任务将被解除阻塞。未阻塞的任务总是等待空间的最高优先级任务。如果阻塞的任务具有相同的优先级，那么等待空间最长的任务将被解除阻塞。

### 阻塞多个队列

队列可被分组到集合中，允许任务进入阻塞状态来等待数据在集合的任何队列中变为可用。队列集合在第 4.6 章节 “从多个队列接收” 中展示。

## 使用队列

### xQueueCreate\(\) API 函数

一个队列在使用前必须被显式的创建。

队列被句柄引用，句柄是类型为 `QueueHandle_t` 类型的变量。`xQueueCreate()` API 函数会创建一个队列，并给一个 `QueueHandle_t` 的变量来引用这个被创建的队列。

FreeRTOS V9.0.0 也包含了 `xQueueCreateStatic()` 函数，它创建队列是在编译时静态地分配内存。当一个队列创建时，FreeRTOS 是从 FreeRTOS 堆中分配所需 RAM。这一段 RAM 被用来保存队列数据结构和队列所含的各个单元。`xQueueCreate()` 在创建队列所需 RAM 不足时会返回 `NULL` 。第 2 章提供了 FreeRTOS 堆的更多信息。

```c
QueueHandle_t xQueueCreate( UBaseType_t uxQueueLength, UBaseType_t uxItemSize);
```

清单 40. `xQueueCreate()` API 函数原型

表 18. `xQueueCreate()` 参数和返回值

| 参数名 | 描述 |
| :--- | :--- |
| uxQueueLength | 正在创建的队列一次可以容纳的最大项数。 |
| uxItemSize | 可以存储在队列中的每个数据项的字节大小。 |
| 返回值 | 如果返回 `NULL`，则无法创建队列，因为 FreeRTOS 没有足够的堆内存来分配队列数据结构和存储区域。返回的非空值表示队列已成功创建。返回的值应该存储为已创建队列的句柄。 |

创建队列后，可以使用 `xQueueReset()` API 函数将队列返回到其原始的空状态。

### xQueueSendToBack\(\) 与 xQueueSendToFront\(\) API 函数

正如所料，`xQueueSendToBack()` 用于将数据发送到队列的后端（尾部），`xQueueSendToFront()` 用于将数据发送到队列的前端（头部）。

`xQueueSend()` 与 `xQueueSendToBack()` 等价，并且完全相同。

{% hint style="info" %}
永远不要从中断服务例程调用 `xQueueSendToFront()` 或 `xQueueSendToBack()`。应该使用中断安全转换 `xQueueSendToFrontFromISR()` 和 `xQueueSendToBackFromISR()` 。这些将在第 6 章中描述。
{% endhint %}

```c
BaseType_t xQueueSendToFront( QueueHandle_t xQueue,
                              const void * pvItemToQueue,
                              TickType_t xTicksToWait );
```

清单 41. `xQueueSendToFront()` API 函数原型

```c
BaseType_t xQueueSendToBack( QueueHandle_t xQueue,
                             const void * pvItemToQueue,
                             TickType_t xTicksToWait );
```

清单 42. `xQueueSendToBack()` API 函数原型

表 19. `xQueueSendToFront()` 和 `xQueueSendToSendToBack()` 函数参数和返回值

<table>
  <thead>
    <tr>
      <th style="text-align:left">&#x53C2;&#x6570;&#x540D;&#x79F0;/&#x8FD4;&#x56DE;&#x503C;</th>
      <th style="text-align:left">&#x63CF;&#x8FF0;</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">xQueue</td>
      <td style="text-align:left">&#x53D1;&#x9001;(&#x5199;&#x5165;)&#x6570;&#x636E;&#x7684;&#x961F;&#x5217;&#x7684;&#x53E5;&#x67C4;&#x3002;&#x961F;&#x5217;&#x53E5;&#x67C4;&#x5C06;&#x4ECE;&#x7528;&#x4E8E;&#x521B;&#x5EFA;&#x961F;&#x5217;&#x7684; <code>xQueueCreate()</code> &#x8C03;&#x7528;&#x4E2D;&#x8FD4;&#x56DE;&#x3002;</td>
    </tr>
    <tr>
      <td style="text-align:left">pvItemToQueue</td>
      <td style="text-align:left">
        <p>&#x6307;&#x5411;&#x8981;&#x590D;&#x5236;&#x5230;&#x961F;&#x5217;&#x4E2D;&#x7684;&#x6570;&#x636E;&#x7684;&#x6307;&#x9488;&#x3002;</p>
        <p>&#x5728;&#x521B;&#x5EFA;&#x961F;&#x5217;&#x65F6;&#xFF0C;&#x5C06;&#x8BBE;&#x7F6E;&#x961F;&#x5217;&#x53EF;&#x4EE5;&#x5BB9;&#x7EB3;&#x7684;&#x6BCF;&#x4E2A;&#x9879;&#x76EE;&#x7684;&#x5927;&#x5C0F;&#xFF0C;&#x56E0;&#x6B64;&#x8FD9;&#x591A;&#x4E2A;&#x5B57;&#x8282;&#x5C06;&#x4ECE; <code>pvItemToQueue</code> &#x590D;&#x5236;&#x5230;&#x961F;&#x5217;&#x5B58;&#x50A8;&#x533A;&#x57DF;&#x4E2D;&#x3002;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">xTicksToWait</td>
      <td style="text-align:left">
        <p>&#x5982;&#x679C;&#x961F;&#x5217;&#x5DF2;&#x7ECF;&#x6EE1;&#x4E86;&#xFF0C;&#x4EFB;&#x52A1;&#x5E94;&#x8BE5;&#x4FDD;&#x6301;&#x963B;&#x585E;&#x72B6;&#x6001;&#x4EE5;&#x7B49;&#x5F85;&#x961F;&#x5217;&#x4E0A;&#x53EF;&#x7528;&#x7A7A;&#x95F4;&#x7684;&#x6700;&#x5927;&#x65F6;&#x95F4;&#x91CF;&#x3002;</p>
        <p>&#x5982;&#x679C; <code>xTicksToWait</code> &#x4E3A;&#x96F6;&#x4E14;&#x961F;&#x5217;&#x5DF2;&#x6EE1;&#xFF0C;&#x5219; <code>xQueueSendToFront()</code> &#x548C; <code>xQueueSendToBack()</code> &#x90FD;&#x5C06;&#x7ACB;&#x5373;&#x8FD4;&#x56DE;&#x3002;</p>
        <p>&#x963B;&#x585E;&#x65F6;&#x95F4;&#x4EE5;&#x6EF4;&#x7B54;&#x5468;&#x671F;&#x6307;&#x5B9A;&#xFF0C;&#x56E0;&#x6B64;&#x5B83;&#x6240;&#x8868;&#x793A;&#x7684;&#x7EDD;&#x5BF9;&#x65F6;&#x95F4;&#x4F9D;&#x8D56;&#x4E8E;&#x6EF4;&#x7B54;&#x9891;&#x7387;&#x3002;&#x5B8F; <code>pdMS TO TICKS()</code> &#x53EF;&#x7528;&#x4E8E;&#x5C06;&#x4EE5;&#x6BEB;&#x79D2;&#x4E3A;&#x5355;&#x4F4D;&#x7684;&#x65F6;&#x95F4;&#x8F6C;&#x6362;&#x4E3A;&#x4EE5;&#x8282;&#x62CD;&#x4E3A;&#x5355;&#x4F4D;&#x7684;&#x65F6;&#x95F4;&#x3002;</p>
        <p>&#x5982;&#x679C;&#x5728; <code>FreeRTOSConfig.h</code> &#x4E2D;&#x5C06; <code>INCLUDE_vTaskSuspend</code> &#x8BBE;&#x7F6E;&#x4E3A;
          1&#xFF0C;&#x5219;&#x5C06; <code>xTicksToWait</code> &#x8BBE;&#x7F6E;&#x4E3A; <code>portMAX_DELAY</code> &#x5C06;&#x5BFC;&#x81F4;&#x4EFB;&#x52A1;&#x65E0;&#x9650;&#x671F;&#x5730;&#x7B49;&#x5F85;&#xFF08;&#x6CA1;&#x6709;&#x8D85;&#x65F6;&#xFF09;&#x3002;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">&#x8FD4;&#x56DE;&#x503C;</td>
      <td style="text-align:left">
        <p>&#x6709;&#x4E24;&#x79CD;&#x53EF;&#x80FD;&#x7684;&#x8FD4;&#x56DE;&#x503C;&#xFF1A;</p>
        <ol>
          <li>pdPASS&#xFF1A;&#x4EC5;&#x5F53;&#x6570;&#x636E;&#x6210;&#x529F;&#x53D1;&#x9001;&#x5230;&#x961F;&#x5217;&#x65F6;&#xFF0C;&#x624D;&#x4F1A;&#x8FD4;&#x56DE; <code>pdPASS</code>&#x3002;&#x5982;&#x679C;&#x6307;&#x5B9A;&#x4E86;&#x963B;&#x585E;&#x65F6;&#x95F4;&#xFF08;<code>xTicksToWait</code> &#x4E0D;&#x4E3A;&#x96F6;&#xFF09;&#xFF0C;&#x90A3;&#x4E48;&#x8C03;&#x7528;&#x4EFB;&#x52A1;&#x53EF;&#x80FD;&#x88AB;&#x7F6E;&#x4E8E;Blocked&#x72B6;&#x6001;&#xFF0C;&#x7B49;&#x5F85;&#x7A7A;&#x95F4;&#x5728;&#x961F;&#x5217;&#x4E2D;&#x53D8;&#x4E3A;&#x53EF;&#x7528;&#xFF0C;&#x5728;&#x51FD;&#x6570;&#x8FD4;&#x56DE;&#x4E4B;&#x524D;&#xFF0C;&#x4F46;&#x6570;&#x636E;&#x5DF2;&#x6210;&#x529F;&#x5199;&#x5165;&#x961F;&#x5217;
            &#x5728;&#x963B;&#x6B62;&#x65F6;&#x95F4;&#x5230;&#x671F;&#x4E4B;&#x524D;&#x3002;</li>
          <li>errQUEUE_FULL&#xFF1A;&#x5982;&#x679C;&#x7531;&#x4E8E;&#x961F;&#x5217;&#x5DF2;&#x6EE1;&#xFF0C;&#x65E0;&#x6CD5;&#x5C06;&#x6570;&#x636E;&#x5199;&#x5165;&#x961F;&#x5217;&#xFF0C;&#x5C06;&#x8FD4;&#x56DE;<code>errQUEUE_FULL</code> &#x3002;&#x5982;&#x679C;&#x6307;&#x5B9A;&#x4E86;&#x963B;&#x585E;&#x65F6;&#x95F4;&#xFF08;<code>xTicksToWait</code> &#x4E0D;&#x4E3A;&#x96F6;&#xFF09;&#xFF0C;&#x5219;&#x8C03;&#x7528;&#x4EFB;&#x52A1;&#x5C06;&#x88AB;&#x7F6E;&#x4E8E;&#x963B;&#x585E;&#x72B6;&#x6001;&#x4EE5;&#x7B49;&#x5F85;&#x53E6;&#x4E00;&#x4E2A;&#x4EFB;&#x52A1;&#x6216;&#x4E2D;&#x65AD;&#x5728;&#x961F;&#x5217;&#x4E2D;&#x817E;&#x51FA;&#x7A7A;&#x95F4;&#xFF0C;&#x4F46;&#x6307;&#x5B9A;&#x7684;&#x963B;&#x585E;&#x65F6;&#x95F4;&#x5728;&#x8BE5;&#x72B6;&#x6001;&#x4E4B;&#x524D;&#x5230;&#x671F;&#x3002;</li>
        </ol>
      </td>
    </tr>
  </tbody>
</table>`xQueueReceive()` API 函数

xQueueReceive\(\) 是用来从队列中接收（读取）一个元素。收到的元素将从队列中删除。

{% hint style="info" %}
切勿从中断服务程序调用 `xQueueReceive()`。 中断安全 `xQueueReceiveFromISR()` API 函数在第 6 章中描述。
{% endhint %}

```c
BaseType_t xQueueReceive( QueueHandle_t xQueue,
                          void *const pvBuffer,
                          TickType_t xTicksToWait );
```

清单 43. `xQueueReceive()` API 函数原型

表 20. `xQueueReceive()` 函数参数和返回值 

<table>
  <thead>
    <tr>
      <th style="text-align:left">&#x53C2;&#x6570;&#x540D;&#x79F0;/&#x8FD4;&#x56DE;&#x503C;</th>
      <th style="text-align:left">&#x63CF;&#x8FF0;</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">xQueue</td>
      <td style="text-align:left">
        <p>&#x6B63;&#x5728;&#x63A5;&#x6536;&#xFF08;&#x8BFB;&#x53D6;&#xFF09;&#x6570;&#x636E;&#x7684;&#x961F;&#x5217;&#x53E5;&#x67C4;&#x3002;</p>
        <p>&#x5C06;&#x4ECE;&#x7528;&#x4E8E;&#x521B;&#x5EFA;&#x961F;&#x5217;&#x7684; <code>xQueueCreate()</code> &#x8C03;&#x7528;&#x8FD4;&#x56DE;&#x961F;&#x5217;&#x53E5;&#x67C4;&#x3002;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">pvBuffer</td>
      <td style="text-align:left">
        <p>&#x6307;&#x5411;&#x8981;&#x5C06;&#x63A5;&#x6536;&#x5230;&#x7684;&#x6570;&#x636E;&#x590D;&#x5236;&#x5230;&#x7684;&#x5185;&#x5B58;&#x7684;&#x6307;&#x9488;&#x3002;</p>
        <p>&#x5728;&#x521B;&#x5EFA;&#x961F;&#x5217;&#x65F6;&#x8BBE;&#x7F6E;&#x961F;&#x5217;&#x4FDD;&#x5B58;&#x7684;&#x6BCF;&#x4E2A;&#x6570;&#x636E;&#x9879;&#x7684;&#x5927;&#x5C0F;&#x3002; <code>pvBuffer</code> &#x6307;&#x5411;&#x7684;&#x5185;&#x5B58;&#x5FC5;&#x987B;&#x81F3;&#x5C11;&#x8DB3;&#x4EE5;&#x5BB9;&#x7EB3;&#x90A3;&#x4E48;&#x591A;&#x5B57;&#x8282;&#x3002;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">xTicksToWait</td>
      <td style="text-align:left">
        <p>&#x5982;&#x679C;&#x961F;&#x5217;&#x5DF2;&#x7ECF;&#x4E3A;&#x7A7A;&#xFF0C;&#x5219;&#x4EFB;&#x52A1;&#x5E94;&#x4FDD;&#x6301;&#x5728;&#x963B;&#x585E;&#x72B6;&#x6001;&#x4EE5;&#x7B49;&#x5F85;&#x6570;&#x636E;&#x7684;&#x6700;&#x957F;&#x65F6;&#x95F4;&#x5728;&#x961F;&#x5217;&#x4E2D;&#x53EF;&#x7528;&#x3002;</p>
        <p>&#x5982;&#x679C; <code>xTicksToWait</code> &#x4E3A;&#x96F6;&#xFF0C;&#x90A3;&#x4E48;&#x5982;&#x679C;&#x961F;&#x5217;&#x5DF2;&#x7ECF;&#x4E3A;&#x7A7A;&#xFF0C;&#x5219; <code>xQueueReceive()</code> &#x5C06;&#x7ACB;&#x5373;&#x8FD4;&#x56DE;&#x3002;</p>
        <p>&#x963B;&#x585E;&#x65F6;&#x95F4;&#x5728;&#x6EF4;&#x7B54;&#x5468;&#x671F;&#x4E2D;&#x6307;&#x5B9A;&#xFF0C;&#x56E0;&#x6B64;&#x5B83;&#x8868;&#x793A;&#x7684;&#x7EDD;&#x5BF9;&#x65F6;&#x95F4;&#x53D6;&#x51B3;&#x4E8E;&#x6EF4;&#x7B54;&#x9891;&#x7387;&#x3002;
          &#x5B8F; <code>pdMS_TO_TICKS()</code>&#x53EF;&#x7528;&#x4E8E;&#x5C06;&#x4EE5;&#x6BEB;&#x79D2;&#x4E3A;&#x5355;&#x4F4D;&#x6307;&#x5B9A;&#x7684;&#x65F6;&#x95F4;&#x8F6C;&#x6362;&#x4E3A;&#x523B;&#x5EA6;&#x4E2D;&#x6307;&#x5B9A;&#x7684;&#x65F6;&#x95F4;&#x3002;</p>
        <p>&#x5C06; <code>xTicksToWait</code> &#x8BBE;&#x7F6E;&#x4E3A; <code>portMAX_DELAY</code> &#x4F1A;&#x5BFC;&#x81F4;&#x4EFB;&#x52A1;&#x65E0;&#x9650;&#x671F;&#x5730;&#x7B49;&#x5F85;&#xFF08;&#x6CA1;&#x6709;&#x8D85;&#x65F6;&#xFF09;&#xFF0C;&#x524D;&#x63D0;&#x662F; <code>FreeRTOSConfig.h</code> &#x4E2D;&#x7684; <code>INCLUDE_vTaskSuspend</code> &#x8BBE;&#x7F6E;&#x4E3A;
          1&#x3002;</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">&#x8FD4;&#x56DE;&#x503C;</td>
      <td style="text-align:left">
        <p>&#x6709;&#x4E24;&#x79CD;&#x53EF;&#x80FD;&#x7684;&#x8FD4;&#x56DE;&#x503C;&#xFF1A;</p>
        <ol>
          <li>pdPASS&#xFF1A;&#x4EC5;&#x5F53;&#x4ECE;&#x961F;&#x5217;&#x4E2D;&#x6210;&#x529F;&#x8BFB;&#x53D6;&#x6570;&#x636E;&#x65F6;&#x624D;&#x4F1A;&#x8FD4;&#x56DE; <code>pdPASS</code>&#x3002;
            &#x5982;&#x679C;&#x6307;&#x5B9A;&#x4E86;&#x963B;&#x585E;&#x65F6;&#x95F4;&#xFF08;<code>xTicksToWait</code> &#x4E0D;&#x4E3A;&#x96F6;&#xFF09;&#xFF0C;&#x90A3;&#x4E48;&#x8C03;&#x7528;&#x4EFB;&#x52A1;&#x53EF;&#x80FD;&#x88AB;&#x7F6E;&#x4E8E;&#x963B;&#x585E;&#x72B6;&#x6001;&#xFF0C;&#x7B49;&#x5F85;&#x6570;&#x636E;&#x5728;&#x961F;&#x5217;&#x4E2D;&#x53EF;&#x7528;&#xFF0C;&#x4F46;&#x662F;&#x5728;&#x963B;&#x585E;&#x65F6;&#x95F4;&#x5230;&#x671F;&#x4E4B;&#x524D;&#x5DF2;&#x6210;&#x529F;&#x4ECE;&#x961F;&#x5217;&#x4E2D;&#x8BFB;&#x53D6;&#x6570;&#x636E;&#x3002;</li>
          <li>errQUEUE_EMPTY&#xFF1A;&#x5982;&#x679C;&#x7531;&#x4E8E;&#x961F;&#x5217;&#x5DF2;&#x7ECF;&#x4E3A;&#x7A7A;&#x800C;&#x65E0;&#x6CD5;&#x4ECE;&#x961F;&#x5217;&#x4E2D;&#x8BFB;&#x53D6;&#x6570;&#x636E;&#xFF0C;&#x5219;&#x5C06;&#x8FD4;&#x56DE;<code>errQUEUE_EMPTY</code>&#x3002;&#x5982;&#x679C;&#x6307;&#x5B9A;&#x4E86;&#x963B;&#x585E;&#x65F6;&#x95F4;&#xFF08;<code>xTicksToWait</code> &#x4E0D;&#x4E3A;&#x96F6;&#xFF09;&#xFF0C;&#x90A3;&#x4E48;&#x8C03;&#x7528;&#x4EFB;&#x52A1;&#x5C06;&#x88AB;&#x7F6E;&#x4E8E;&#x963B;&#x585E;&#x72B6;&#x6001;&#x4EE5;&#x7B49;&#x5F85;&#x53E6;&#x4E00;&#x4E2A;&#x4EFB;&#x52A1;&#x6216;&#x4E2D;&#x65AD;&#x5C06;&#x6570;&#x636E;&#x53D1;&#x9001;&#x5230;&#x961F;&#x5217;&#xFF0C;&#x4F46;&#x963B;&#x585E;&#x65F6;&#x95F4;&#x5728;&#x8BE5;&#x65F6;&#x95F4;&#x4E4B;&#x524D;&#x5230;&#x671F;&#x3002;</li>
        </ol>
      </td>
    </tr>
  </tbody>
</table>### uxQueueMessagesWaiting\(\) API 函数

`uxQueueMessagesWaiting()` 用于查询当前队列中的项目数。

{% hint style="info" %}
切勿从中断服务程序调用 `uxQueueMessagesWaiting()`。 应该在其位置使用中断安全 `uxQueueMessagesWaitingFromISR()`。
{% endhint %}

```c
UBaseType_t uxQueueMessagesWaiting( QueueHandle_t xQueue );
```

清单 44. `uxQueueMessagesWaiting()` API 函数原型

表 21. `uxQueueMessagesWaiting()` 函数参数或返回值

| 参数名称/返回值 | 描述 |
| :--- | :--- |
| xQueue | 正在查询队列的句柄。 将从用于创建队列的 `xQueueCreate()` 调用返回队列句柄。 |
| 返回值 | 正在查询的队列当前持有的项目数。 如果返回零，则队列为空。 |

### 示例 10. 从队列接收时阻塞

此示例演示了正在创建的队列，从多个任务发送到队列的数据以及从队列中接收的数据。 创建队列以保存 `int32_t` 类型的数据项。 发送到队列的任务不指定阻塞时间，从队列接收的任务执行。

发送到队列的任务的优先级低于从队列接收的任务的优先级。 这意味着队列永远不应包含多个项目，因为只要数据被发送到队列，接收任务就会解锁，抢占发送任务，并删除数据 - 再次将队列留空。

清单 45 显示了写入队列的任务的实现。 创建此任务的两个实例，一个将值 100 连续写入队列，另一个将值 200 连续写入同一队列。 任务参数用于将这些值传递到每个任务实例中。

```c
static void vSenderTask( void *pvParameters )
{
int32_t lValueToSend;
BaseType_t xStatus;
    
    /* 创建此任务的两个实例，以便通过任务参数传递发送到队列的值 —— 这样每个实例可以使用不同
    的值。创建队列是为了保存 int32_t 类型的值，因此将参数转换为所需的类型。 */
    lValueToSend = ( int32_t ) pvParameters;
    
    /* 对于大多数任务，这个任务是在一个无限循环中实现的。 */
    for( ;; )
    {
        /* 将值发送到队列。
        
        第一个参数是数据发送到的队列。队列是在调度程序启动之前创建的，因此在此任务开始执行
        之前。
        
        第二个参数是要发送的数据的地址，在本例中是 lValueToSend 的地址。
        
        第三个参数是阻塞时间 —— 如果队列已经满了，任务应该保持在阻塞状态，等待队列上的空间
        可用。在这种情况下，未指定块时间，因为队列永远不应包含多个元素，因此永远不会满。*/
        xStatus = xQueueSendToBack( xQueue, &lValueToSend, 0 );
        
        if( xStatus != pdPASS )
        {
            /* 发送操作无法完成，因为队列已满 —— 这一定是一个错误，因为队列不能包含更多的
            元素 */
            vPrintString( "Could not send to the queue.\r\n" );
        }
    }
}
```

清单 45. 示例 10 中使用的发送任务的实现。

清单 46 显示了从队列接收数据的任务的实现。 接收任务指定块时间为 100 毫秒，因此将进入阻塞状态以等待数据变为可用。 当队列中的数据可用时，它将离开阻塞状态，或者在没有数据可用的情况下，它将离开 100 毫秒。 在此示例中，100 毫秒超时应该永不过期，因为有两个任务连续写入队列。

```c
static void vReceiverTask( void *pvParameters )
{
/* 声明将保存从队列接收的值的变量。 */
int32_t lReceivedValue;
BaseType_t xStatus;
const TickType_t xTicksToWait = pdMS_TO_TICKS( 100 );
    
    /* 此任务也在无限循环中定义。 */
    for( ;; )
    {
        /* 此调用应该始终发现队列为空，因为此任务将立即删除写入队列的任何数据。 */
        if( uxQueueMessagesWaiting( xQueue ) != 0 )
        {
            vPrintString( "Queue should have been empty!\r\n" );
        }
        
        /* 从队列中接收数据。 
        
        第一个参数是接收数据的队列。队列在调度程序启动之前创建，因此在此任务第一次运
        行之前创建。 
        
        第二个参数是将接收到的数据放置到其中的缓冲区。在这种情况下，缓冲区只是具有保存
        接收数据所需大小的变量的地址。
        
        最后一个参数是阻塞时间如果队列已经为空，任务将保持在阻塞状态等待数据可用的最大
        时间量。 */
        xStatus = xQueueReceive( xQueue, &lReceivedValue, xTicksToWait );
        
        if( xStatus == pdPASS )
        {
            /* 从队列中成功接收到数据，打印出接收到的值。 */
            vPrintStringAndNumber( "Received = ", lReceivedValue );
        }
        else
        {
            /* 即使在等待了100ms 之后，也没有从队列接收到数据。这一定是一个错误，因为
            发送任务是免费运行的，并且将不断地写入队列。*/
            vPrintString( "Could not receive from the queue.\r\n" );
        }
    }
}
```

清单 46. 示例 10  接受任务的实现

清单 47 包含 `main()` 函数的定义。 这只是在启动调度程序之前创建队列和三个任务。 创建队列以最多保存五个 `int32_t` 值，即使设置了任务的优先级，使得队列一次也不会包含多个项目。

```c
/* 声明一个类型为 QueueHandle_t 的变量。该变量用于将句柄存储到所有三个任务都访问的队列中。 */
QueueHandle_txQueue;int main( void )
{
    /* 创建队列最多可以容纳5个值，每个值都足够大，可以容纳 int32_t 类型的变量。 */
    xQueue= xQueueCreate( 5, sizeof( int32_t) );if( xQueue != NULL )
    {
        /* 创建将发送到队列的任务的两个实例。任务参数用于传递任务将写入队列的值，因此一个任务
        将持续向队列写入 100，而另一个任务将持续向队列写入 200。这两个任务都在优先级 1 处创
        建。 */
        xTaskCreate( vSenderTask, "Sender1", 1000, ( void * ) 100, 1, NULL );
        xTaskCreate( vSenderTask, "Sender2", 1000, ( void * ) 200, 1, NULL );
        
        /* 创建将从队列中读取的任务。创建任务的优先级为 2，因此高于发送方任务的优先级。 */
        xTaskCreate( vReceiverTask, "Receiver", 1000, NULL, 2, NULL );
        
        /* 启动调度程序，以便创建的任务开始执行。 */
        vTaskStartScheduler();
    }
    else
    {
        /* 无法创建队列。 */
    }
    
    /* 如果一切正常，那么 main() 将永远不会到达这里，因为调度程序现在将运行这些任务。如果
    main() 确实到达这里，那么很可能没有足够的 FreeRTOS 堆内存可用来创建空闲任务。第 2 章
    提供了关于堆内存管理的更多信息。 */
    for( ;; );
}
```

清单 47. 例 10 中 main\(\) 的实现

发送到队列的两个任务都具有相同的优先级。 这导致两个发送任务依次将数据发送到队列。 例 10 中产生的输出如图 32 所示。



