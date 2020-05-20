---
layout:     post
title:      "ThreadLocal类"
subtitle:   ""
date:       2020-05-07
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java并发
---

# 为什么使用
ThreadLocal类常用于线程间独立数据的使用，譬如每个线程自己日志信息的存储传递等。我们可以看看官方的注释
```
/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 **/
```
# 源码解析（精简版）
```
public class ThreadLocal<T> { 
    // 用于ThreadLocalMap中Entry的index计算
    private final int threadLocalHashCode = nextHashCode();
    private static AtomicInteger nextHashCode = new AtomicInteger();
    private static final int HASH_INCREMENT = 0x61c88647;
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
    
    // 初始化值
    protected T initialValue() {
        return null;
    }

    //初始化值，SuppliedThreadLocal是ThreadLocal的子类并重写了上面initialValue方法
    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }
    static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

        private final Supplier<? extends T> supplier;

        SuppliedThreadLocal(Supplier<? extends T> supplier) {
            this.supplier = Objects.requireNonNull(supplier);
        }

        @Override
        protected T initialValue() {
            return supplier.get();
        }
    }
    
    // 获取当前线程中当前ThreadLocal调用对象的数据
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    
    // 设置初始值
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
  
    // 设置当前对象threadlocal的值
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    // ThrealLocalMap是ThreadLocal的内部类
    static class ThreadLocalMap {
         //table初始化大小
        private static final int INITIAL_CAPACITY = 16;
        // 存储数组table
        private Entry[] table;
        // resize临界值
        private int threshold; // Default to 0
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
        
        // 注意这里的key是弱引用
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
        
        // Map添加值
        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        // 根据key值获取Map数据
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
    }
}
```
- ThreadLocal类其实可以看成是Thread类中单独存储数据的类，底层的存储实现其实是通过一个内部类ThreadLocalMap实现的
- ThreadLocalMap里面主要由Entry数组构成，但是需要注意到Entry数组里面的key是WeakReference(threadLocal)，也就是一个弱引用
- 实现数据再线程之中隔离的方法就是靠SuppliedThreadLocal类重写了initialValue方法，可以保证方法在threadlocal.get()方法时会创建新的数据

# 案例
```
public class ThreadLocalTest {
   class Bank {
       private ThreadLocal<Integer> account = ThreadLocal.withInitial(() -> 100);
       private ThreadLocal<Counter> counter = ThreadLocal.withInitial(Counter::new);

       public int getAccount() {
           return account.get();
       }

       public long getCounterValue() {
           return counter.get().getValue();
       }

       public void save(int money) {
           account.set(account.get() + money);
       }

       public void add() {
           counter.get().increment();
       }
   }

   class NewThread {
       private Bank bank;
       public NewThread(Bank bank) {
           this.bank = bank;
       }

       public void multiTest(){
           for (int i = 0; i < 3; i++) {
                new Thread(() -> {
                    bank.save(10);
                    bank.add();
                    System.out.println(Thread.currentThread().getName() + ":" + "money:" + bank.getAccount() + ",counter value:" + bank.getCounterValue());
                }, "thread-"+i).start();
           }
       }
   }

    public static void main(String[] args) {
       ThreadLocalTest test = new ThreadLocalTest();
       ThreadLocalTest.Bank bank = test.new Bank();
       ThreadLocalTest.NewThread newThread = test.new NewThread(bank);
       newThread.multiTest();
    }
}
```
我们可以看看执行结果
```
Connected to the target VM, address: '127.0.0.1:55030', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:55030', transport: 'socket'
thread-0:money:110,counter value:1
thread-1:money:110,counter value:1
thread-2:money:110,counter value:1

Process finished with exit code 0
```
我们可以看到虽然Bank虽然是线程共享的，但是ThreadLocal数据是每个线程独自的