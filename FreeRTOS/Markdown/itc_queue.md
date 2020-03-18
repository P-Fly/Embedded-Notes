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

 1. 需要注意

使用 `xQueueGenericSend` 入队后队列的数据结构如下图：

![xQueueGenericSend Structure][4]

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

 1. 需要注意

使用 `xQueueGenericSendFromISR` 入队后队列的数据结构如下图：

![xQueueGenericSendFromISR Structure][6]

## 出队

| API | 实际调用函数 | 入队方式 | 
| :--- | :--- | :--- |
|  |  |  |
|  |  |  |
|  |  |  |

## 删除队列

##

锁是用于防止访问queue event lists，即：
	List_t xTasksWaitingToSend;
	List_t xTasksWaitingToReceive;



xQueuePeek
xQueuePeekFromISR
xQueueReceive
vQueueDelete
xQueueGiveFromISR
xQueueReceiveFromISR
xQueueReset
xQueueGenericSend
xQueueGenericSendFromISR
xQueueGenericCreateStatic
xQueueGenericReset -- No Public

[1]: ./images/xQueueCreate.jpg
[2]: ./images/xQueueCreate_structure.jpg

