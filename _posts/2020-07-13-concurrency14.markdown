---
layout:     post
title:      "Java并发(十四) --ReentrantLock源码简析"
subtitle:   ""
date:       2020-07-13
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java并发
---

# 背景
之前学习了AQS源码，现在就可以好好学习下AQS的一个应用ReentrantLock。ReentrantLock跟synchronized一样是一个可重入锁，但是ReentrantLock可以实现公平和非公平锁，而synchronized只是非公平锁，所以我们现在就来看下ReentrantLock的源码

# 源码
- 实现Lock接口
```
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
    //取锁，获取不到进入等待队列
    public void lock() {
        sync.lock();
    }
    //响应中断的获取锁
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
    //获取锁，获取不到就直接返回false，因为nonfairTryAcquire是也是lock中一个中间步骤
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
    //释放锁
    public void unlock() {
        sync.release(1);
    }
  
    public Condition newCondition() {
        return sync.newCondition();
    }
    //默认是非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
我们可以看到上面实现Lock接口的方法最终都是操作一个Sync对象，我们就来看下sync是啥？
- Sync
```
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    abstract void lock();

    //非公平的获取锁
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    //释放锁
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }

    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }

    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }

    final boolean isLocked() {
        return getState() != 0;
    }

    private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```
我们可以看Sync是ReentrantLock一个内部类，已经实现了nonfairTryAcquire方法和tryRelease方法，我们一般使用过程中都是分公平锁和非公平锁，其实公平锁和非公平锁就是继承了Sync类，我们可以看下代码
- 非公平锁
```
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    //非公平锁的获取锁
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```
我们可以看到非公平锁的获取锁过程会先尝试一下获取锁，如果获取成功则直接调用AQS里面的setExclusiveOwnerThread方法将exclusiveOwnerThread改成当前线程。如果失败则调用AQS里面的acquire方法获取锁。
- 公平锁
```
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```
我们可以看到公平锁的lock是直接调用AQS的acquire方法，但是前面AQS源码中提到，会先走到子类复写的tryAcquire，那么是如何区分公平和非公平的呢？其实就是hasQueuedPredecessors方法会先判断等待队列是否有等待的线程，如果有则先保证等待的线程先获取锁，从而实现公平。

# 实例
```
public class ThreadPoolTest {
    private static ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("Test-Thread").build();
    private static ExecutorService threadPool = new ThreadPoolExecutor(2,2,2, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(100), threadFactory,  new ThreadPoolExecutor.AbortPolicy());
    static ReentrantLock lock = new ReentrantLock(true);

    public static void main(String[] args) {

        try {
            Future future1 = threadPool.submit(() -> printInfo("第一个"));
            Future future2 = threadPool.submit(() -> printInfo("第二个"));
            finishThreads(future1, future2);
            System.out.println(threadPool);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void printInfo(String index) {
        lock.lock();
        try {
            if ("第一个".equalsIgnoreCase(index)) {
                Thread.sleep(5000);
            }
            System.out.println(Thread.currentThread().getName() + " " + index);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    private static void finishThreads(Future... futures) throws Exception {
        for (Future future : futures) {
            future.get();
        }
    }
}
```
上面通过ReentrantLock公平锁和ThreadPoolExecutor实现了一个顺序打印的程序，大家可以运行试试。