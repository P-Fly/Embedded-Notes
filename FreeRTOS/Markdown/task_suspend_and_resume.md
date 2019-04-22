# 任务的挂起与恢复

## 前言

本章详细介绍了任务挂起 / 恢复的接口。

## vTaskSuspend

### 功能

挂起一个任务。

注意：该函数没有累积性，即使调用多次 **vTaskSuspend**，调用一次 **vTaskResume** 也能恢复任务。

### 原型

``` C
void vTaskSuspend( TaskHandle_t xTaskToSuspend );
```

### 参数

 - xTaskToSuspend：需要挂起的任务句柄。

### 流程

![vTaskSuspend][1]

## vTaskResume

### 功能

恢复一个被挂起的任务。

### 原型

``` C
void vTaskResume( TaskHandle_t xTaskToResume );
```

### 参数

 - xTaskToResume：需要恢复的任务句柄。

### 流程

![vTaskResume][2]

## xTaskResumeFromISR

### 功能

在中断中恢复一个被挂起的任务。

### 原型

``` C
BaseType_t xTaskResumeFromISR( TaskHandle_t xTaskToResume );
```

### 参数

 - xTaskToResume：需要恢复的任务句柄。

### 返回值

如果该函数返回非0，表明应该发起一次上下文切换（有高优先级任务或同优先级任务需要切换）。

### 流程

 [1]: ./images/vTaskSuspend.jpg
 [2]: ./images/vTaskResume.jpg
