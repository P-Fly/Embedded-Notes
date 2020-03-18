# 队列

## 创建队列

| API | 实际调用函数 |
| :--- | :--- |
| xQueueCreate | xQueueGenericCreate |
| xQueueCreateStatic | xQueueGenericCreateStatic |

`xQueueGenericCreate` 与 `xQueueGenericCreateStatic` 的区别在于内存是否为动态分配，因此以 `xQueueGenericCreate` 为例:

### xQueueGenericCreate

``` C
QueueHandle_t xQueueGenericCreate( const UBaseType_t uxQueueLength,
        const UBaseType_t uxItemSize,
        const uint8_t ucQueueType )
```

 - **uxQueueLength**： 队列可以包含的最大项数。
 - **uxItemSize**： 队列中每一项所需的字节数。
 - **ucQueueType**： 队列类型，对于 `Queue` 该参数为 **queueQUEUE_TYPE_BASE**。

![xQueueCreate Sequence][1]

![xQueueCreate Structure][2]

 1. 需要注意 xQueueGenericReset 中，对阻塞任务的处理策略。

    - 如果有任务处于等待读取的状态，则该阻塞任务将继续阻塞。
    - 如果有任务处于等待写入的状态，则取消**一个任务**的阻塞状态。

    > If there are tasks blocked waiting to read from the queue, then the tasks will remain blocked as after this function exits the queue will still be empty.  If there are tasks blocked waiting to write to the queue, then one should be unblocked as after this function exits it will be possible to write to it.

## 发送消息

| API | 入队方式 | 实际调用函数 |
| :--- | :--- | :--- |
|  |  |  |
|  |  |  |
|  |  |  |



xQueueSendToFront
xQueueSendToBack
xQueueSend
xQueueOverwrite
xQueuePeek
xQueuePeekFromISR
xQueueReceive
vQueueDelete
xQueueSendToFrontFromISR
xQueueSendToBackFromISR
xQueueOverwriteFromISR
xQueueSendFromISR
xQueueGiveFromISR
xQueueReceiveFromISR
xQueueReset

xQueueGenericCreate
xQueueGenericSend
xQueueGenericSendFromISR
xQueueGenericCreateStatic
xQueueGenericReset -- No Public









## 接收消息

## 删除队列

##

锁是用于防止访问queue event lists，即：
	List_t xTasksWaitingToSend;
	List_t xTasksWaitingToReceive;

[1]: ./images/xQueueCreate.jpg
[2]: ./images/xQueueCreate_structure.jpg
