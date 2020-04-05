# 任务通知

## 功能描述

每个任务都有一个32位的通知值，该值可以被任务通知更新，并且解除该任务的阻塞状态。这种灵活性使得以前使用 **Binary Semaphores**，**Counting Semaphores**，**Event Groups**，**Mailbox** 的情形，可以由任务通知替换。

 [Task Notifications][1]

### 通知值更新方式

> - Set the receiving task’s notification value without overwriting a previous value
> - Overwrite the receiving task’s notification value
> - Set one or more bits in the receiving task’s notification value
> - Increment the receiving task’s notification value

### 使用限制

 - 一组任务通知只能有一个接收任务（多发单收模式）。
 - 接收通知的任务可以因为等待通知而进入阻塞状态，但发送通知的任务如果不能发出通知也不能进入阻塞状态。

## 样例

### 替代 Binary Semaphores

| API | 功能 | 备注 |
| :--- | :--- | :--- |
| ulTaskNotifyTake | 等待任务通知 | **参数 xClearCountOnExit = True** |
| xTaskNotifyGive | 发出任务通知 |  |
| vTaskNotifyGiveFromISR | 发出任务通知（中断上下文） |  |

 [Task Notification As Binary Semaphore][2]

### 替代 Counting Semaphores

| API | 功能 | 备注 |
| :--- | :--- | :--- |
| ulTaskNotifyTake | 等待任务通知 | **参数 xClearCountOnExit = False** |
| xTaskNotifyGive | 发出任务通知 |  |
| vTaskNotifyGiveFromISR | 发出任务通知（中断上下文） |  |

 [Task Notification As Counting Semaphore][3]

### 替代 Event Groups

| API | 功能 | 备注 |
| :--- | :--- | :--- |
| xTaskNotifyWait | 等待任务通知 |  |
| xTaskNotify | 发出任务通知 |  |
| xTaskNotifyFromISR | 发出任务通知（中断上下文） |  |

 [Task Notification As Event Group][4]

### 替代 Mailbox

| API | 功能 | 备注 |
| :--- | :--- | :--- |
| xTaskNotifyWait | 等待任务通知 |  |
| xTaskNotify | 发出任务通知 | **eAction = eSetValueWithOverwrite or eSetValueWithoutOverwrite** |
| xTaskNotifyFromISR | 发出任务通知（中断上下文） | **eAction = eSetValueWithOverwrite or eSetValueWithoutOverwrite** |

 [Task Notification As Mailbox][5]

 [1]: https://www.freertos.org/RTOS-task-notifications.html
 [2]: https://www.freertos.org/RTOS_Task_Notification_As_Binary_Semaphore.html
 [3]: https://www.freertos.org/RTOS_Task_Notification_As_Counting_Semaphore.html
 [4]: https://www.freertos.org/RTOS_Task_Notification_As_Event_Group.html
 [5]: https://www.freertos.org/RTOS_Task_Notification_As_Mailbox.html
