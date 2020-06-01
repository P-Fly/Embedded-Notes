# 内存管理

## 前言

标准 **malloc** 和 **free** 作为内存分配方案有以下缺点：
 - they are not always available on embedded systems.
 - they take up valuable code space.
 - they are not thread safe.
 - they are not deterministic (the amount of time taken to execute the function will differ from call to call).

因此 **FreeRTOS** 将内存分配 **API** 保存在可移植层中，允许适合当前特定应用的内存管理方案。

[Memory Management][1]

## 内存分配方案

### heap_1

**heap_1** 是最简单的实现。但是一旦分配了内存，它就不允许释放内存。

### heap_2

**heap_2** 允许你释放先前分配的块，但它不将相邻的自由块组合成单个大块。建议被 **heap_4** 替代。

### heap_3

**heap_3** 简单的封装了 **malloc** 和 **free**，以实现 **thread safe**。

### heap_4

**heap_4** 允许你释放先前分配的块，与 **heap_2** 不同，它会将相邻的空闲内存块组合成一个大块。

### heap_5

**heap_5** 使用 **heap_4** 中的合并算法，并且允许堆栈跨越多个非连续的内存区。

## 代码分析

内存分配方案的详细代码分析可以参看文章：[FreeRTOS内存管理分析][2]

 [1]: https://www.freertos.org/a00111.html
 [2]: https://blog.csdn.net/zhzht19861011/article/details/51606068
