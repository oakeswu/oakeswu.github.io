---
layout:     post
title:      "Java并发(十) -- CAS思想"
subtitle:   ""
date:       2020-06-06
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java并发
---

# 概念
CAS的全称是compare and swap，也叫自旋锁，翻译过来就是比较并交换。类似于数据库里面有悲观锁和乐观锁一样，并发编程中也有乐观锁和悲观锁区分，synchronize就是并发中的悲观锁，认为线程每次都会修改，会引起锁的争抢。而CAS思想则是乐观锁，认为线程很少会修改数据，适合读多写少的场景。

# 原理
说到CAS就不得不提Unsafe类，Unsafe是java并发包底层的核心，里面的compareAndSwapInt等方法就是CAS思想的实现。我们可以看一下方法
```
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

// var5先当前内存中的值，如果别的线程修改内存中值就会再次循环过程，没有修改就直接把内存中的值更新成var5+var4
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}

/**
* offset就是偏移地址
* expected是期望值，x是替换值
**/
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x)
```
compareAndSwapInt就是通过对象o和offset获取对象内存中的值，当获取到内存中的值与expected期望值一样的话，就把内存中的值更换成x。通过一个while循环判断来更新值。当我们理解了上面的逻辑之后我们就可以发现cas为啥适合读多写少的场景以及可能出现ABA问题。

- ABA问题
我们看上面的代码就知道在while判断的时候间隔期间可能会发生其他线程修改内存中值的情况，我们假设当前线程获取到的内存值var5等于A,然后另一个线程修改了内存中的值为B,再修改成A，此时对于当前线程判断var5的值跟内存中的值都为A，此时内存中值曾经修改为B的情况就对当前线程透明了，所以会存在ABA问题，解决方案是可以增加版本号来附加判断。

- 读多写少
我们可以看到CAS是循环判断的， 当写多的情况发生时很可能造成compareAndSwapInt判断失败导致一直循环下去。