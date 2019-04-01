# 任务切换

## 前言

调度器的核心功能是**使最高优先级的任务获得CPU的使用权**，那么任务切换就是调度器的一项基本功能。本章重点描述调度器的任务切换。

## 总览

任务切换如下图：

![task_yield][1]

由图上可知，FreeRTOS 有两个方式可以触发任务切换：

 1. 通过系统调用执行任务切换
 2. 通过时钟中断执行任务切换

对于系统调用方式，FreeRTOS 针对不同场景提供了两个API：

 1. taskYIELD：任务上下文中可以使用的强制任务切换。
 2. portYIELD_FROM_ISR：中断上下文中可以使用的强制任务切换。

## 触发任务切换

``` C
portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
```

对于系统调用和时钟中断的方式，触发任务切换的方式是一致的，都是通过设置 **ICSR** 寄存器来请求任务切换。此时，系统产生 **PendSV exception**。

## 任务切换流程

## API

 [1]: ./images/task_yield.jpg
