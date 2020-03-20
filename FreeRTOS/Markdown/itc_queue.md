# 队列

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

![xQueueCreate Structure][2]

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

**FreeRTOS** 使用 `prvLockQueue` 和 `prvUnlockQueue` 对队列进行锁操作。
这是因为 **FreeRTOS** 为了提高中断的响应时间在队列阻塞和唤醒操作时不会关闭中断，
而是使用 `vTaskSuspendAll` 和 `xTaskResumeAll` 以关闭调度器的方式进行临界区保护。
在调度器关闭时，任务链表会被锁定（比如 **pxDelayedTaskList**），但是

[1]: ./images/xQueueCreate.jpg
[2]: ./images/xQueueCreate_Structure.jpg
[3]: ./images/xQueueGenericSend.jpg
[4]: ./images/prvCopyDataToQueue.jpg
[5]: ./images/xQueueGenericSendFromISR.jpg
[6]: ./images/xQueueReceive.jpg
[7]: ./images/xQueueReceiveFromISR.jpg
