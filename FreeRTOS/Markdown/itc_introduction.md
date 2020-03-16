# 任务间通信

## 前言

任务间通信(ITC，Inter-task Communication)是一组编程接口，能让用户在不同的任务或中断与任务之间交换消息。

## 主要的方式

 - RTOS Task Notifications
 - Stream and Message Buffers
 - Queues
 - Binary Semaphores
 - Counting Semaphores
 - Mutexes
 - Recursive Mutexes

## 全局数据

### Queue_t

`Queue_t` 的成员如下：

> int8_t *pcHead;

用作标记，指向队列存储区域的开始处。

> int8_t *pcWriteTo;

指向队列存储区域中，下一个待写入的空闲位置。(即使用 \*(pcWriteTo++) = Item 的方式写入)。

> int8_t *pcTail;

用作标记，指向队列存储区域的结束处。
**注意：该队列类型为 `Queue` 时使用该结构。**

> int8_t *pcReadFrom;

指向队列存储区域中，最后一次读取队列项所在的位置。(即使用 Item = \*(++pcReadFrom) 的方式读取)。
**注意：该队列类型为 `Queue` 时使用该结构。**

> TaskHandle_t xMutexHolder;

记录保持互斥的任务的句柄。
**注意：该队列类型为 `Semaphore` 时使用该结构。**

> UBaseType_t uxRecursiveCallCount;

记录被递归的次数。
**注意：该队列类型为 `Semaphore` 时使用该结构。**

> List_t xTasksWaitingToSend;

事件链表的链表头。按照任务的优先级，挂载因等待向该队列发送消息而处于阻塞状态的任务。

> List_t xTasksWaitingToReceive;

事件链表的链表头。按照任务的优先级，挂载因等待从该队列获取消息而处于阻塞状态的任务。

> volatile UBaseType_t uxMessagesWaiting;

当前队列上的消息个数。

> UBaseType_t uxLength;

定义Item的个数。

> UBaseType_t uxItemSize;

定义每个Item的大小。

> volatile int8_t cRxLock;

队列的读锁。

> volatile int8_t cTxLock;

队列的写锁。

> uint8_t ucStaticallyAllocated;

记录队列的内存是否为动态申请。

> struct QueueDefinition *pxQueueSetContainer;

指向队列所属的消息组。

> UBaseType_t uxQueueNumber;

调试追踪时使用，对该队列提供一个可以追溯的数值。

> uint8_t ucQueueType;

调试追踪时使用，记录当前队列的类型。
