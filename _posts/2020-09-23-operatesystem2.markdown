---
layout:     post
title:      "操作系统(二) -- 程序执行"
subtitle:   ""
date:       2020-09-23
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 操作系统
---

**本文参考[图灵机简介](https://zhuanlan.zhihu.com/p/143834012)、[内存是如何存储数据的](https://www.zhihu.com/question/289555024/answer/1134803042?utm_source=wechat_session&utm_medium=social&utm_oi=754455964719022080&utm_content=group3_Answer&utm_campaign=shareopn)**
# 图灵机
图灵机是cpu的一般示例，可以通过读取上面的数据和指令来完成计算等操作。大家可以去看下上面的图灵机简介。

# 冯诺依曼模型
![冯诺依曼模型](http://upload-images.jianshu.io/upload_images/9082703-63d7edcad8d0de1e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
冯诺依曼模型分为输入设备，输出设备，内存，中央处理器(CPU)，总线。

- CPU
CPU负责控制和运算，按照CPU每次计算字节数的不同分为32位CPU和64位CPU。这里的32和64称为CPU位宽。
CPU里面有寄存器，控制单元和逻辑运算单元。控制单元控制CPU工作类型，算数逻辑单用于数学运算。寄存器是CPU内部用于存储的模块，为了弥补CPU与内存通信较慢的问题，分为通用寄存器，特殊寄存器，指令寄存器。

- 内存
![](http://upload-images.jianshu.io/upload_images/9082703-a149c6cda50861be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
内存是一个存储程序和数据的二进制位的线性存储区域。最小存储单位是字节，每个字节对应于一个内存地址。由数个IC组成，IC内部有VCC，GND电源引脚，D0-D7八个数据引脚，所以一次输入1个字节数据，A0-A9十个地址信号引脚，可以指定1024个地址，所以一个IC表示1024 * 1byte = 1KB。

- 总线
总线主要分为控制总线，数据总线，内存总线。地址总线专门用来指定CPU将要操作的内存地址，数据总线用来读写内存数据，控制总线用来发送和接收关键信号，比如中断，设备复位等信号。

- I/O输入输出设备
输入输出就是我们常见的键盘，鼠标，显示器等外设，通过IO流与计算机进行交互。

# 程序执行
我们以JAVA程序int a = 1+2为例来看下程序执行。
1：首先JVM将常量1和2放到JVM中的方法区的常量池中（JDK1.7），1.8改到了堆里面，1存放到地址0x100，2存放到地址0x102;
2：JVM生成汇编指令，load 0x100 -> R1，load 0x102 -> R2，add R1 R2 R3，set R3 -> 0x104
3：构建的指令依然使用32位或者64位二进制数据表示，然后由CPU完成指令的解析，并执行。
