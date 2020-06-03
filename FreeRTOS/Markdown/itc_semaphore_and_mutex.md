# 信号量和互斥量

## 工作原理

信号量的使用主要是用来保护共享资源，实现任务的同步或异步。

一般将信号量抽象成两个操作 **P(s)** 和 **V(s)**：
 - P(s)：如果s的值大于零，就减1；如果它的值为零，就挂起该进程。
 - V(s)：如果有其他进程因等待s而被挂起，就让它恢复；如果没有就加1。

## 信号量简介 

**FreeRTOS** 提供了 **Binary Semaphores**，**Mutexes**，**Counting Semaphores** and **Recursive Semaphores** 四种形式的信号量和互斥量。

 - [Binary Semaphores][1]：只能容纳一个项的队列，只关心队列是空或满（二值性），不关心具体的值，这个机制可以用来同步任务和中断。
 - [Counting Semaphores][2]：被认为是长度大于1的队列，同样对存储在队列中的数据不感兴趣，仅仅关注队列是否为空或满。按照初始值的不同，分为两个典型应用场景  **Counting Events** 和 **Resource Management**。
 - [Mutexes][3]：与 **Binary Semaphores**非常相似，但是包含了优先级继承机制。一般来说，**Mutexes**是实现互斥的更好选择；而 **Binary Semaphores** 是实现同步的更好选择。

    优先级继承机制：如果一个高优先级任务块试图获得当前由较低优先级任务持有的互斥(Token)，那么持有令牌的任务的优先级将暂时提高到阻塞任务的优先级。该机制旨在确保较高优先级的任务在尽可能短的时间内保持在阻塞状态，并在此过程中尽量减少已经发生的“优先级反转”。

 - [Recursive Semaphores][4]：与 **Mutexes** 相似，但是该互斥体可以被所有者反复获取。当该所有者不在使用该资源时，需要反复释放相同的次数才会将资源释放。

## **Binary Semaphores**

| API | comments |
| :--- | :--- |
| xSemaphoreCreateBinary | Creates a new binary semaphore with dynamic memory |
| xSemaphoreCreateBinaryStatic | Creates a new binary semaphore with static memory |
| xSemaphoreTake | To obtain a semaphore |
| xSemaphoreTakeFromISR | To obtain a semaphore from ISR |
| xSemaphoreGive | To release a semaphore |
| xSemaphoreGiveFromISR | To release a semaphore from ISR |

## **Counting Semaphores**

| API | comments |
| :--- | :--- |
| xSemaphoreCreateCounting | Creates a new counting semaphore with dynamic memory |
| xSemaphoreCreateCountingStatic | Creates a new counting semaphore with static memory |
| xSemaphoreTake | To obtain a semaphore |
| xSemaphoreTakeFromISR | To obtain a semaphore from ISR |
| xSemaphoreGive | To release a semaphore |
| xSemaphoreGiveFromISR | To release a semaphore from ISR |

## **Mutexes**

| API | comments |
| :--- | :--- |
| xSemaphoreCreateMutex | Creates a new mutex type semaphore with dynamic memory |
| xSemaphoreCreateMutexStatic | Creates a new mutex type semaphore with static memory |
| xSemaphoreTake | To obtain a semaphore |
| xSemaphoreTakeFromISR | To obtain a semaphore from ISR |
| xSemaphoreGive | To release a semaphore |
| xSemaphoreGiveFromISR | To release a semaphore from ISR |

## **Recursive Semaphores**

| API | comments |
| :--- | :--- |
| xSemaphoreCreateRecursiveMutex | Creates a new recursive mutex type semaphore with dynamic memory |
| xSemaphoreCreateRecursiveMutexStatic | Creates a new recursive mutex type semaphore with static memory |
| xSemaphoreTakeRecursive | To obtain a semaphore |
| xSemaphoreGiveRecursive | To release a semaphore |

 [1]: https://www.freertos.org/Embedded-RTOS-Binary-Semaphores.html
 [2]: https://www.freertos.org/Real-time-embedded-RTOS-Counting-Semaphores.html
 [3]: https://www.freertos.org/Real-time-embedded-RTOS-mutexes.html
 [4]: https://www.freertos.org/RTOS-Recursive-Mutexes.html
