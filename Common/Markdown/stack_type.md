# 栈类型

## 前言

本章简要描述了栈的四种类型。

## 

![stack type][1]

如图所示，可以通过两个维度区别栈。

Full：栈指针指向最后压入数据的位置。
Empty：栈指针指向下一个将要压入数据的位置。

Ascending：栈指针由低地址向高地址生长。
Descending：栈指针由高地址向低地址生长。

每种栈的代码可以参考上图。我们常用的 Cortex-M 使用的是 **Full Descending Stack**。

 [1]: ./images/stack_type.jpg
