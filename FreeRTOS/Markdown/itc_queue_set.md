# 队列集

## 简介

队列集提供了一种机制，允许任务同时阻塞在多个队列或信号量的读取操作中。队列和信号量被划分为集合，其中不管哪一个消息到来，都可以让任务退出阻塞状态。

[Blocking on Multiple RTOS Objects][1]

## 使用方法

 1. 使用 `xQueueCreateSet` 显示创建队列集。
 2. 使用 `xQueueAddToSet` 将队列或信号量添加至集合中。
 3. 使用 `xQueueSelectFromSet` 或 `xQueueSelectFromSetFromISR` 监视队列集的状态改变。
 4. 如果集合接收到队列消息，则 **步骤3** 返回准备读取的队列集成员的句柄。需通过 **xQueueReceive** 或 **xQueueSemaphoreTake** 读取具体的队列消息。

详细的例子可以参见：[Queue Set Example][2]

## 替代方案

定义一个结构，其中包含一个成员来保存事件类型，另一个成员保存与事件相关联的数据或指针。然后使用单个队列来发送和接收定义的消息。

 [1]: https://www.freertos.org/Pend-on-multiple-rtos-objects.html
 [2]: https://www.freertos.org/xQueueCreateSet.html
