# 任务切换

## 前言

调度器的核心功能是**使最高优先级的任务获得CPU的使用权**，而任务切换就是调度器的一项基本功能。本章重点描述调度器的任务切换。

## 总览

任务切换的接口如下图：

![task_yield][1]

由上图可知，FreeRTOS 有两个方式可以触发任务切换：

 1. 通过系统调用执行任务切换。
 2. 通过时钟中断执行任务切换。

对于系统调用方式，FreeRTOS 针对不同场景提供了两个API：

 1. taskYIELD：任务上下文中可以使用的强制任务切换。
 2. portYIELD_FROM_ISR：中断上下文中可以使用的强制任务切换。

## 触发任务切换

``` C
portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
```

无论是在系统调用场景还是时钟中断场景，任务切换的方式都是一致的：通过设置 **ICSR** 寄存器来请求任务切换。此时，系统产生 **PendSV exception**。

## PendSV exception

 - 产生中断前使用的是 **psp**，因此硬件自动将 **8** 个寄存器压入到任务栈。
 - 进入 **PendSV exception**，此时 **sp** 默认使用 **msp**。

```armasm
mrs r0, psp
isb
ldr r3, =pxCurrentTCB
ldr r2, [r3]
````

对寄存器进行赋值：
 - **r0** = **psp**
 - **r3** = **pxCurrentTCB**
 - **r2** = **TCB_t**

```armasm
stmdb r0!, {r4-r11, r14}
````

将 **r4 - r11, r14** 寄存器的值存入用户堆栈，注意：
 - 当前默认的栈寄存器是 **msp**，而压入的栈是 **psp** 指向的位置。
 - 此处 **R0** 指向了栈顶，而 **psp** 没有更新。

```armasm
str r0, [r2]
````

更新 **pxCurrentTCB->pxTopOfStack**。

```armasm
stmdb sp!, {r3}
mov r0, #configMAX_SYSCALL_INTERRUPT_PRIORITY
msr basepri, r0
dsb
isb
bl vTaskSwitchContext
mov r0, #0
msr basepri, r0
ldmia sp!, {r3}
````

 - 备份 **R3** 寄存器到系统堆栈中 （因为在 `bl vTaskSwitchContext` 后还会用到该寄存器）。
 - 进入临界区，相当于调用了 `vPortRaiseBASEPRI`。
 - 调用 vTaskSwitchContext，该函数负责使 **pxCurrentTCB** 指向下一个需要执行的任务。
 - 退出临界区，相当于调用了 `vPortClearBASEPRIFromISR`。（按照规定 **Cortex-M** 的 **PendSV exception** 必须是 **最低优先级**，因此可以用将 **basepri** 寄存器请 **0** 的方式打开中断）。
 - 恢复 **R3** 寄存器。

```armasm
ldr r1, [r3]
ldr r0, [r1]
ldmia r0!, {r4-r11, r14}
```

 - 获取新的栈顶地址，r0 = pxCurrentTCB->pxTopOfStack。
 - 恢复 **R4 - R11, R14** 寄存器。

```armasm
msr psp, r0
isb
```

重新设置堆栈寄存器。

```armasm
bx r14
nop
nop
```

从 **PendSV exception** 返回，进入 **Thread mode**。硬件使用 **psp** 恢复其它寄存器。后续的代码也继续使用 **psp** 寄存器。

 [1]: ./images/task_yield.jpg
