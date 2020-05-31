---
layout:     post
title:      "两个线程交替打印数字"
subtitle:   ""
date:       2020-05-25
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java并发
---

# 背景
今天偶然看到一个题目，刚好可以复习一下之前学的线程，题目就会两线程交替打印数字，我们先看个**错误**例子，也是一般的想法
```
public class ExchangePrint {
    private volatile boolean onePrint = false;
    private volatile int index = 1;

    class OneThread implements Runnable {

        @Override
        public void run() {
            while (index < 10) {
                if (onePrint) {
                    System.out.println("oneThread:" + index);
                    index++;
                    onePrint = false;
                }
            }
        }
    }

    class TwoThread implements Runnable {

        @Override
        public void run() {
            while (index < 10) {
                if (!onePrint) {
                    System.out.println("twoThread:" + index);
                    index++;
                    onePrint = true;
                }
            }
        }
    }

    public static void main(String[] args) {
        ExchangePrint exchangePrint = new ExchangePrint();
        new Thread(exchangePrint.new OneThread()).start();
        new Thread(exchangePrint.new TwoThread()).start();
    }
}
```
上面这个例子试了运行了几次都是正确的结果，但是我们发现index++是一个非原子性操作，极端情况下可能会打印两次相同值的情况。所以我们想到一种方案用AtomicInteger代替index++避免，改进后的正确代码是
```
public class ExchangePrint {
    private volatile boolean ePrint = false;
    private volatile AtomicInteger efValue = new AtomicInteger(0);

    class EThread implements Runnable {

        @Override
        public void run() {
            while (efValue.get() < 100) {
                if (ePrint) {
                    System.out.println("EThread:" + efValue);
                    efValue.getAndIncrement();
                    ePrint = false;
                }
            }
        }
    }

    class FThread implements Runnable {

        @Override
        public void run() {
            while (efValue.get() < 100) {
                if (!ePrint) {
                    System.out.println("FThread:" + efValue);
                    efValue.getAndIncrement();
                    ePrint = true;
                }
            }
        }
    }

     public static void main(String[] args) {
        ExchangePrint exchangePrint = new ExchangePrint();
        new Thread(exchangePrint.new EThread()).start();
        new Thread(exchangePrint.new FThread()).start();
    }
}
```
但是我们发现两边线程会一直循环判断直到自己执行，那我们有没有办法是不该自己执行的时候就线程挂起，不占用资源， 轮到自己执行时再通知一下自己执行，这个方案听起来不就是线程的wait，notify和lock Condition里面的await，singal场景吗？那我们来实现一下。
```
public class ExchangePrint {
    private static final Object simpleLock= new Object();
    private volatile int abIndex = 1;
    private volatile boolean aHasPrint = false;

    class AThread implements Runnable {

        @Override
        public void run() {
            for (int i = 0; i < 50; i++) {
                synchronized (simpleLock) {
                    while (aHasPrint) {
                        try {
                            simpleLock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    System.out.println("AThread:" + abIndex);
                    abIndex++;
                    aHasPrint = true;
                    simpleLock.notifyAll();
                }
            }
        }
    }

    class BThread implements Runnable {

        @Override
        public void run() {
            for (int i = 0; i < 50; i++) {
                synchronized (simpleLock) {
                    while (!aHasPrint) {
                        try {
                            simpleLock.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    System.out.println("BThrea:" + abIndex);
                    abIndex++;
                    aHasPrint = false;
                    simpleLock.notifyAll();
                }
            }
        }
    }

    public static void main(String[] args) {
        ExchangePrint exchangePrint = new ExchangePrint();
        new Thread(exchangePrint.new AThread()).start();
        new Thread(exchangePrint.new BThread()).start(); 
    }
}
```
我们看到通过一个对象锁和判断条件就可以交替执行打印了，我们再看看ReentrantLock类实现的代码
```
public class ExchangePrint {
    private static final ReentrantLock reentrantLock = new ReentrantLock();
    private Condition cPrint = reentrantLock.newCondition();
    private Condition dPrint = reentrantLock.newCondition();
    private volatile int cdIndex = 1;
    private volatile boolean cHasPrint = false;

    class CThread implements Runnable {

        @Override
        public void run() {
            while (cdIndex < 100) {
                reentrantLock.lock();
                try {
                    if (cHasPrint) {
                        cPrint.await();
                    }
                    System.out.println("CThread:" + cdIndex);
                    cdIndex++;
                    cHasPrint = true;
                    dPrint.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    reentrantLock.unlock();
                }
            }
        }
    }

    class DThread implements Runnable {

        @Override
        public void run() {
            while (cdIndex < 100) {
                reentrantLock.lock();
                try {
                    if (!cHasPrint) {
                        dPrint.await();
                    }
                    System.out.println("DThread:" + cdIndex);
                    cdIndex++;
                    cHasPrint = false;
                    cPrint.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    reentrantLock.unlock();
                }
            }
        }
    }

    public static void main(String[] args) {
        ExchangePrint exchangePrint = new ExchangePrint();
        new Thread(exchangePrint.new CThread()).start();
        new Thread(exchangePrint.new DThread()).start();
    }
}
```
我们可以看到其实通过等待和通知实现方式都差不多，不过要特别注意wait，notifyall必须要在sychronized代码块里面，不然会报IllegalMonitorStateException错误，这个可以看下Object类wait和notifyall方法的说明。上面就是三种正确的交替打印方法，后面会再复习下Lock接口以及实现类等