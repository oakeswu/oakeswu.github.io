---
layout:     post
title:      "ThreadPoolExecutor类"
subtitle:   ""
date:       2020-05-25
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java并发
---

# 背景
当我们在项目中创建线程池时，可以用Executors里面封装好的newSingleThreadExecutor，newFixedThreadPool等线程池，但是阿里的java开发规范建议我们自己创建一个线程池，其实我们看Executors里面的线程池底层也依旧是使用的ThreadPoolExecutor类，为什么这么做呢，因为只有在自己new一个ThreadPoolExecutor对象过程中才会知道各个参数的意义，并选择合适的参数构建线程池。ThreadPoolExecutor的关系继承图如下
![](https://upload-images.jianshu.io/upload_images/9082703-648f5597d0de66d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 源码
ThreadPoolExecutor是一个内容很多的类，我们先看下他的完整构造函数，了解一个每个参数的含义
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
- corePoolSize 线程池中核心线程数，核心线程创建了之后就会一直在线程池中，不会因为超时设置而消失
- maximumPoolSize 线程池中允许的最大线程数量
- keepAliveTime 超过corePoolSize小于等于maximumPoolSize的线程无任务执行后存活的时间
- unit keepAliveTime的时间单位，毫秒，秒等
- workQueue 阻塞队列，用于存放被添加到线程池中尚未执行的任务
- threadFactory 用于创建具体执行线程
- handler 拒绝策略，当任务太多来不及处理如何拒绝任务

然后我们看下拒绝策略
```
    public static class AbortPolicy implements RejectedExecutionHandler {
      
        public AbortPolicy() { }

        /**
         * 直接抛出异常
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }

    public static class CallerRunsPolicy implements RejectedExecutionHandler {
      
        public CallerRunsPolicy() { }

        /**
         * 线程池的线程数量达到上限，该策略会把任务队列中的任务放在调用者线程当中运行；
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

    public static class DiscardPolicy implements RejectedExecutionHandler {
      
        public DiscardPolicy() { }

        /**
         * 默默丢弃无法处理的任务，不予任何处理。
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
    
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
      
        public DiscardOldestPolicy() { }

        /**
         * 丢弃任务队列中最老的一个任务，也就是当前任务队列中最先被添加进去的
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```
拒绝策略现在就这四种，当然我们也可以自己去实现RejectedExecutionHandler 接口，不过jdk提供的这四种目前足够使用了

# 执行
我们想看下线程具体到哪里执行的，先看下源码
```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. 如果小于corePoolSize数的线程正在执行，那么当前就新建一个线程执行
         *
         * 2. 如果一个任务成功排队，我们仍然需要检查我们是否需要添加一个线程，
         *     因为上次检查后有的线程已经死亡或者线程池可能已经shutdown
         *
         * 3. 如果任务无法入队，那我们应该添加一个新线程，如果仍然失败，
         *     那我们知道线程池已经shutodown或者已满拒绝添加任务
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

    private final HashSet<Worker> workers = new HashSet<Worker>();
     
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
上面execute方法就是正式执行Runnable任务的方法，可以看到Runnable任务被封装成了worker并放入了一个HashSet中进行已创建线程的管理。

# 案例
```
public class ThreadPoolTest {
    private static ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("Test-Thread").build();
    private static ExecutorService threadPool = new ThreadPoolExecutor(1,5,2, TimeUnit.SECONDS, new LinkedBlockingQueue<>(), threadFactory,  new ThreadPoolExecutor.AbortPolicy());

    public static void main(String[] args) {
        try {
            Future future1 = threadPool.submit(() -> System.out.println(Thread.currentThread().getName() + " 执行第一个"));
            Future future2 = threadPool.submit(() -> System.out.println(Thread.currentThread().getName() + " 执行第二个"));
            finishThreads(future1, future2);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void finishThreads(Future... futures) throws Exception {
        for (Future future : futures) {
            future.get();
        }
    }
}
```
我们创建了一个线程池并执行了两个打印任务。