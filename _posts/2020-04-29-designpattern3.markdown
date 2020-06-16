---
layout:     post
title:      "设计模式(三) -- 单例模式"
subtitle:   ""
date:       2020-04-29
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 设计模式
---

# 什么是单例模式
单例模式可以按照字面上的意思理解，保证类只有一个实例对象生成的模式。一般分为懒汉式和饿汉式，懒汉式中需要考虑下线程不安全的问题，可以在实例中展示

# 代码实例
饿汉式
```
public class SingletonHunger {
    private static SingletonHunger singletonHunger = new SingletonHunger();

    private static SingletonHunger getSingletonHunger() {
        return singletonHunger;
    }
}
```
我们可以看到因为singletonHunger变量是private和static修饰的，所以饿汉式其实就是项目启动的时候就初始化了对象并且外面并不能直接获取，必须通过getSingletonHunger方法获取，所以返回的都是已初始化完成的singletonHunger变量

懒汉式（非线程安全）
```
public class SingletonLazy {
    private static SingletonLazy singletonLazy;

    public static SingletonLazy getSingletonLazy() {
        if (singletonLazy == null) {
            singletonLazy = new SingletonLazy();
        }
        return singletonLazy;
    }
}
```
我们可以看到懒汉式并没有一开始就初始化对象，而是在获取对象方法中的时候判断一下对象是否已经初始化，但上面的代码其实存在并发风险，因为线程初始化并不是一个原子性的操作，当多个线程同时调用getSingletonLazy方法时很可能会多次初始化，所以我们可以改进一下代码
```
public class SingletonLazy {
    private static volatile SingletonLazy singletonLazy;

    public static synchronized SingletonLazy getSingletonLazy() {
        if (singletonLazy == null) {
            singletonLazy = new SingletonLazy();
        }
        return singletonLazy;
    }
}
```
第一种方法就是直接方法上加锁，这样可以保证线程的安全，但是我们使用的是方法锁，所有调用方都要竞争锁，所以我们可以再优化一版，改成对象锁
```
public class SingletonLazy {
    private static volatile SingletonLazy singletonLazy;

    public static SingletonLazy getSingletonLazy() {
        if (singletonLazy == null) {
            synchronized (SingletonLazy.class) {
                if (singletonLazy == null) {
                    singletonLazy = new SingletonLazy();
                }
            }
        }
        return singletonLazy;
    }
}
```
最终版的代码就写好了，我们仔细观察上面的代码会发现两个有意思的地方，一是singletonLazy变量增加了一个volatile修饰，二是方法中进行了两次是否为null的判断，为什么要这样做？

- 第一个null判断是为了提升效率，如果singletonLazy已经初始化之后就可以不用去竞争锁操作
- 第二个null判断是为了避免多次实例化对象，因为当两个线程A和B进入方法都通过了第一次null判断进入到竞争锁的阶段，当B先获取到锁之后进入方法并初始化一个对象之后释放锁然后由A获取到，如果不加第二次判断A会再次实例化对象
- volitale的作用是保证变量的可见性和避免指令重排序，但是synchronized己经保证了可见性，所以这里volitale的作用就是为了避免指令重排序，其实实例化对象并不是一个原子性的操作，其中有三个步骤，1是分配内存空间，2是初始化对象，3是将内存地址赋给对象引用。如果没有volitale，那这个顺序在编译或者运行过程中可能发生指令重排，从123的运行顺序变成了132的顺序，此时另一个线程进来判断对象引用不为null，但是实际对象并没有初始化成功，从而导致报错，所以这里volitale为了避免重排序