---
layout:     post
title:      "Java并发(五) -- ExecutorService源码简析"
subtitle:   ""
date:       2020-05-18
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java并发
---

# 背景
ExecutorService是java.concurrent包下面的一个接口，并继承了Executor接口。是线程池常使用到的类，所以准备学习下此类
# Executor
```
/**
 * An object that executes submitted {@link Runnable} tasks. This
 * interface provides a way of decoupling task submission from the
 * mechanics of how each task will be run
 **/
public interface Executor {
    void execute(Runnable command);
}
```
Executor接口只有一个execute方法，但是看类上面的注释说最大的作用是对任务提交和任务执行进行了解耦，可以看下官方给的例子
```
class SerialExecutor implements Executor {
        final Queue<Runnable> tasks = new ArrayDeque<Runnable>();
        final Executor executor;
        Runnable active;

        SerialExecutor(Executor executor) {
            this.executor = executor;
        }

        public synchronized void execute(final Runnable r) {
            tasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (active == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((active = tasks.poll()) != null) {
                executor.execute(active);
            }
        }
    }}
```
原始任务提交方直接调用new Thread(runnable).start执行，所以任务提交和调用方相同。而例子中任务提交方只需要将任务提交给SerialExecutor执行就好了。
# ExecutorService
```
public interface ExecutorService extends Executor {
    //有序关闭之前提交的任务，但是不再接受新任务
    void shutdown();
    //停止所有正在执行的任务，取消等待中的任务并返回等待中的任务
    List<Runnable> shutdownNow();
    //是否已经shut down,当调用shutdown或者shutdownnow就为true
    boolean isShutdown();
    //shutdown或shutdownnow中所有完成
    boolean isTerminated
    //提交一个任务并返回一个Future
    <T> Future<T> submit(Callable<T> task);
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);
    //提交多个任务并返回多个Future    
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit) throws InterruptedException;
    //提交多个任务并返回执行成功的一个任务的结果
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
我们可以看到ExecutorService继承了Executor接口，所以ExecutorService也可以使用execute（Runnable runable）方法