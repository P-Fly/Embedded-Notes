# 队列

## 简介

`Queue` 是任务间通信的主要形式。可以用于在任务之间，以及中断和任务之间发送消息。在大多数情况下，它们被用作线程安全FIFO（先进先出）缓冲区，新数据可以被发送到队列的后面也可以发送到队列的前面。

主要特点如下：

 1. 数据交互使用复制而不是引用的方式。
    - 对于短消息，不需要内存动态申请，可以减少内存碎片。
    - 对于大数据量的消息，可以通过传递指针的方式实现数据交互。
 2. 内核负责分配队列存储区域的内存，并提供数据的发送和读取接口，因此容易实现权限管理（MPU）。
 3. 中断中不能使用不以 **FromISR** 结尾的函数。

 [FreeRTOS Queues][8]

## 创建队列

| API | 实际调用函数 |
| :--- | :--- |
| xQueueCreate | xQueueGenericCreate |
| xQueueCreateStatic | xQueueGenericCreateStatic |

`xQueueGenericCreate` 与 `xQueueGenericCreateStatic` 的区别在于所需的内存是否为动态分配，因此我们以 `xQueueGenericCreate` 为例进行分析:

### xQueueGenericCreate

``` C
QueueHandle_t xQueueGenericCreate( const UBaseType_t uxQueueLength,
        const UBaseType_t uxItemSize,
        const uint8_t ucQueueType )
```

 - **uxQueueLength**： 队列可以包含的最大项数。
 - **uxItemSize**： 队列中每一项所需的字节数。
 - **ucQueueType**： 队列类型，对于 `Queue` 该参数为 **queueQUEUE_TYPE_BASE**。
 - **Return Value**： 返回新创建的队列的句柄。

使用 `xQueueGenericCreate` 创建队列的流程如下图：

![xQueueCreate Sequence][1]

 1. 需要注意 `xQueueGenericReset` 中，对阻塞任务的处理策略。

    - 如果有任务处于等待读取的状态，则该阻塞任务将继续阻塞。
    - 如果有任务处于等待写入的状态，则取消 **优先级最高的任务** 的阻塞状态。

    > If there are tasks blocked waiting to read from the queue, then the tasks will remain blocked as after this function exits the queue will still be empty.  If there are tasks blocked waiting to write to the queue, then one should be unblocked as after this function exits it will be possible to write to it.

使用 `xQueueGenericCreate` 创建队列的数据结构如下图：

![QueueCreate Structure][2]

## 入队

队列的入队方式如下表：

 - 按照运行场景可分为两个版本：
    - 可以在中断服务例程中使用
    - 不可在中断服务例程中使用

 - 按照插入位置可分为三种方式：
    - 从队列首部入队
    - 从队列尾部入队
    - 从队列尾部覆盖式入队

| API | 实际调用函数 | 入队方式 | 备注 |
| :--- | :--- | :--- | :--- |
| xQueueSendToFront        | xQueueGenericSend | 从队列首部入队 |  |
| xQueueSendToBack         | xQueueGenericSend | 从队列尾部入队 |  |
| xQueueOverwrite          | xQueueGenericSend | 从队列尾部覆盖式入队 | 只用于长度为1的队列 |
| xQueueSend               | xQueueGenericSend | 从队列尾部入队 | 不推荐使用，该接口存在是为了保持版本的向后兼容性 |
| xQueueSendToFrontFromISR | xQueueGenericSendFromISR | 从队列首部入队 |  |
| xQueueSendToBackFromISR  | xQueueGenericSendFromISR | 从队列尾部入队 |  |
| xQueueOverwriteFromISR   | xQueueGenericSendFromISR | 从队列尾部覆盖式入队 | 只用于长度为1的队列 |
| xQueueSendFromISR        | xQueueGenericSendFromISR | 从队列尾部入队 | 不推荐使用，该接口存在是为了保持版本的向后兼容性 |

### xQueueGenericSend

``` C
BaseType_t xQueueGenericSend( QueueHandle_t xQueue,
        const void * const pvItemToQueue,
        TickType_t xTicksToWait,
        const BaseType_t xCopyPosition )
```

 - **xQueue**： 队列的句柄。
 - **pvItemToQueue**： 需要入队的Item。
 - **xTicksToWait**： 入队任务被阻塞的最大时间，如果设置为0则不被阻塞。
 - **xCopyPosition**：设置入队方式，可以为 **queueSEND_TO_BACK**，**queueSEND_TO_FRONT**，**queueOVERWRITE**。
 - **Return Value**： 入队成功或队列满。

使用 `xQueueGenericSend` 入队的流程如下图：

![xQueueGenericSend Sequence][3]

 1. 图中绿色部分为队列未满，能正常写入的流程。
 2. `prvCopyDataToQueue` 为数据入队的执行函数。有三种入队方式：**queueSEND_TO_BACK**，**queueSEND_TO_FRONT**，**queueOVERWRITE**。
 3. 红色部分为队列的阻塞和唤醒流程：
    - 当有高优先级任务被唤醒时，`xTaskResumeAll` 产生上下文切换。
    - 当没有高优先级任务被唤醒时，`portYIELD_WITHIN_API` 产生上下文切换，此时切换至同优先级或低优先级任务。

使用 `prvCopyDataToQueue` 入队后队列的数据结构如下图：

![prvCopyDataToQueue][4]

### xQueueGenericSendFromISR

``` C
BaseType_t xQueueGenericSendFromISR( QueueHandle_t xQueue,
        const void * const pvItemToQueue,
        BaseType_t * const pxHigherPriorityTaskWoken,
        const BaseType_t xCopyPosition )
```

 - **xQueue**： 队列的句柄。
 - **pvItemToQueue**： 需要入队的Item。
 - **pxHigherPriorityTaskWoken**： 当该值返回 **True** 时，表明有更高优先级的任务被解除阻塞，需要在退出中断前发起一次上下文切换。
 - **xCopyPosition**：设置入队方式，可以为 **queueSEND_TO_BACK**，**queueSEND_TO_FRONT**，**queueOVERWRITE**。
 - **Return Value**： 入队成功或队列满。

使用 `xQueueGenericSendFromISR` 入队的流程如下图：

![xQueueGenericSendFromISR Sequence][5]

## 出队

队列的出队方式如下表：

 - 按照运行场景可分为两个版本：
    - 可以在中断服务例程中使用
    - 不可在中断服务例程中使用

 - 按照出队方式可分为两种方式：
    - 出队并删除数据项
    - 出队不删除数据项

| API | 出队方式 | 
| :--- | :--- |
| xQueueReceive | 出队并删除数据项 |
| xQueuePeek | 出队不删除数据项 |
| xQueueReceiveFromISR | 出队并删除数据项 |
| xQueuePeekFromISR | 出队不删除数据项 |

`xQueueReceive` 与 `xQueuePeek` 的区别在于是否删除队列中出队的数据，因此我们以 `xQueueReceive` 和 `xQueueReceiveFromISR` 为例进行分析:

### xQueueReceive

``` C
BaseType_t xQueueReceive( QueueHandle_t xQueue,
        void * const pvBuffer,
        TickType_t xTicksToWait )
```

 - **xQueue**： 队列的句柄。
 - **pvBuffer**： 接受出队Item的Buffer。
 - **xTicksToWait**： 出队任务被阻塞的最大时间，如果设置为0则不被阻塞。
 - **Return Value**： 出队成功或队列空。

使用 `xQueueReceive` 出队的流程如下图：

![xQueueReceive Sequence][6]

### xQueueReceiveFromISR

``` C
BaseType_t xQueueReceiveFromISR( QueueHandle_t xQueue,
        void * const pvBuffer,
        BaseType_t * const pxHigherPriorityTaskWoken )
```

 - **xQueue**： 队列的句柄。
 - **pvBuffer**： 接受出队Item的Buffer。
 - **pxHigherPriorityTaskWoken**： 当该值返回 **True** 时，表明有更高优先级的任务被解除阻塞，需要在退出中断前发起一次上下文切换。
 - **Return Value**： 出队成功或队列空。

使用 `xQueueReceiveFromISR` 出队的流程如下图：

![xQueueReceiveFromISR Sequence][7]

## 删除队列

### vQueueDelete

``` C
void vQueueDelete( QueueHandle_t xQueue )
```

 - **xQueue**： 队列的句柄。

## 其它

### 关于队列锁

**FreeRTOS** 为了提高中断的响应时间，在队列阻塞和唤醒操作时并不会关闭中断，而是使用 `vTaskSuspendAll` 和 `xTaskResumeAll` 以关闭调度器的方式进行临界区保护。但是当调度器关闭时，任务虽然不会被切换，但是中断服务依然会响应。如果中断服务函数此时操作队列的事件列表 **xTasksWaitingToSend** 和 **xTasksWaitingToReceive**，则很有可能会与任务产生竞争冲突。因此 **FreeRTOS** 使用 `prvLockQueue` 和 `prvUnlockQueue` 对队列进行锁操作。

对于队列锁的简要逻辑是：
 - 对于ISR，如果队列未上锁，则不但操作队列数据，同时还唤醒阻塞队列（如果需要）。
 - 对于ISR，如果队列上锁，则只操作队列数据，并通过通过锁记录数据的操作次数。
 - 对于Task，在队列解锁时会检查锁的状态，如果有中断对队列进行了操作，则此时才唤醒阻塞队列。

 [1]: ./images/xQueueCreate.jpg
 [2]: ./images/QueueCreate_Structure.jpg
 [3]: ./images/xQueueGenericSend.jpg
 [4]: ./images/prvCopyDataToQueue.jpg
 [5]: ./images/xQueueGenericSendFromISR.jpg
 [6]: ./images/xQueueReceive.jpg
 [7]: ./images/xQueueReceiveFromISR.jpg
 [8]: https://www.freertos.org/Embedded-RTOS-Queues.html
