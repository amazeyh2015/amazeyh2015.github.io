---
layout: post
title: "Backtrace From Register"
date: 2025-10-03 10:00:00 +0800
categories: learning
---

本文描述了在iOS开发中如何从寄存器还原出调用堆栈

# Register Description

* SP：栈顶地址
* FP：栈底地址，X29
* PC：下一条指令的地址
* LR：函数返回后的地址，X30

# Frame Backtracing

先看下测试的代码，定义了一个类和三个方法

<img src="https://blog-amazeyh2015.oss-cn-beijing.aliyuncs.com/backtrace-from-register/test_methods.png" alt="test_methods.png" width="300" />

然后在main函数中调用-\[TestClassA methodA:]

<img src="https://blog-amazeyh2015.oss-cn-beijing.aliyuncs.com/backtrace-from-register/invoke_method.png" alt="invoke_method.png" width="600" />

代码运行并命中断点后，查看调用堆栈

<img src="https://blog-amazeyh2015.oss-cn-beijing.aliyuncs.com/backtrace-from-register/address_in_backtrace.png" alt="address_in_backtrace.png" width="800" />

下面我们看看这个堆栈是如何生成的

先看下SP、FP、PC、LR的值

<img src="https://blog-amazeyh2015.oss-cn-beijing.aliyuncs.com/backtrace-from-register/address_of_registers.png" alt="address_of_registers.png" width="600" />

PC寄存器保存下一条指令的地址，所以Frame0的地址为PC的值0x0000000100ed436c
备注：调用堆栈中的地址都是下一条指令的地址，在调用函数时，会把PC的值设置给LR
调用函数时，会把当前Frame的信息保存在栈中，并通过FP的值串联起来，结构如下

<img src="https://blog-amazeyh2015.oss-cn-beijing.aliyuncs.com/backtrace-from-register/address_chain.png" alt="address_chain.png" width="500" />

Frame1的地址为LR的值0x0000000100ed4348
Frame2的地址为前面LR的值，通过当前FP的值（0x000000016ef2f460）找到保存前面FP值的地址（0x000000016ef2f490）

<img src="https://blog-amazeyh2015.oss-cn-beijing.aliyuncs.com/backtrace-from-register/value_at_address.png" alt="value_at_address.png" width="300" />

保存前面LR值的地址为保存前面FP值的地址+8（0x000000016ef2f498），值为0x0000000100ed4310
依此类推，找到其他Frame的地址，直到遇到保存的FP的值为0x0000000000000000，表示结束
