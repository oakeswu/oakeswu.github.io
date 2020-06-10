---
layout:     post
title:      "List和Map的并发测试"
subtitle:   ""
date:       2020-06-09
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java并发
---

# 背景
Java中List和Map是常用的集合，但是List和Map都是接口类，在实现类中有线程安全的，也有线程不安全的，今天就来整理一下。

# ArrayList和HashMap
ArrayList是有一个Object[]数组组成的列表，我们就看下add方法的源码
```
public class ArrayList<E> extends AbstractList<E> 
           implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    /**
     * 在list的尾部增加一个元素
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

     private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

     private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
}
```
我们看到add方法里面会对全局变量size，elementData等做修改，但是没有加锁，也可以看下ArrayList中其他方法发现也没有加锁，所以ArrayList多线程环境下有并发风险，那我们再来看看1.8当中的HashMap的put方法
```
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //当判断没有hash碰撞时候就直接加newNode
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
}
```
这里可以看到HashMap的put方法也没有加锁，可以看到我特地注释的地方当两个线程同时操作的时候很可能会发生数据覆盖的问题，这里特别推荐一篇文章，将JDK1.7和1.8中HashMap线程不安全的地方简单的表述出来（[JDK1.7和JDK1.8中HashMap为什么是线程不安全的](https://blog.csdn.net/swpu_ocean/article/details/88917958)）

# Vector和Hashtable
我们也来看下Vector的add方法和Hashtable中的put方法
```
public class Vector<E> extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
    public synchronized boolean add(E e) {
        modCount++;
        ensureCapacityHelper(elementCount + 1);
        elementData[elementCount++] = e;
        return true;
    }
}
```
```
public class Hashtable<K,V> extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable {
    
    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
}
```
我们可以看到Vector里面和Hashtable中都增加了synchronized来实现线程安全。但是由于synchronized在以前算一个很重的锁，效率不高，所以我们平时一般用的是CopyOnWriteArrayList和ConcurrentHashMap来实现线程安全

# CopyOnWriteArrayList和ConcurrentHashMap
那我们来简单看一下CopyOnWriteArrayList的add方法和ConcurrentHashMap的put方法
```
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    
    public boolean add(E e) {
        //乐观锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
}
```
```
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
}
```
我们可以看到CopyOnWriteArrayList里面使用了ReentrantLock锁，ConcurrentHashMap使用了分段锁来实现线程安全

# 例子
```
public class ListMapTest {
    private static final int size = 10000;

    public static void main(String[] args) throws Exception{
        SharedListAndMap sharedListAndMap = new SharedListAndMap();
        CountDownLatch countDownLatch = new CountDownLatch(size);
        sharedListAndMap.setCountDownLatch(countDownLatch);
        
        for (int i = 0; i < size; i++) {
            Thread thread = new Thread(sharedListAndMap, "test"+i);
            thread.start();
        }
        countDownLatch.await();
        System.out.println("notSafeList:" + sharedListAndMap.getNotSafeList().size());
        System.out.println("safeList:" + sharedListAndMap.getSafeList().size());
        System.out.println("notSafeMap:" +sharedListAndMap.getNotSafeMap().size());
        System.out.println("safeMap:" +sharedListAndMap.getSafeMap().size());
    }

    private static class SharedListAndMap implements Runnable{
        private List<String> notSafeList = new ArrayList<>(size);
        private List<String> safeList = new CopyOnWriteArrayList<>();
        private Map<String, String> notSafeMap = new HashMap(size, 1);
        private Map<String, String> safeMap = new ConcurrentHashMap<>(size, 1);

        private CountDownLatch countDownLatch;
        public void setCountDownLatch(CountDownLatch countDownLatch) {
            this.countDownLatch = countDownLatch;
        }

        public List<String> getNotSafeList() {
            return notSafeList;
        }

        public List<String> getSafeList() {
            return safeList;
        }

        public Map<String, String> getNotSafeMap() {
            return notSafeMap;
        }

        public Map<String, String> getSafeMap() {
            return safeMap;
        }

        @Override
        public void run() {
            String threadName = Thread.currentThread().getName();
            notSafeList.add(threadName);
            safeList.add(threadName);
            notSafeMap.put(threadName, threadName);
            safeMap.put(threadName, threadName);
            countDownLatch.countDown();
        }
    }
}
```
上面贴了不少源码来说明，此时我们再看一个代码实例来验证一下，这里没有验证Vector，Hashtable，读者可以自己去验证。我们可以看三次的执行结果
```
Connected to the target VM, address: '127.0.0.1:64887', transport: 'socket'
notSafeList:9999
safeList:10000
notSafeMap:9999
safeMap:10000
Disconnected from the target VM, address: '127.0.0.1:64887', transport: 'socket'
```
```
Connected to the target VM, address: '127.0.0.1:49155', transport: 'socket'
notSafeList:9998
safeList:10000
notSafeMap:10000
safeMap:10000
Disconnected from the target VM, address: '127.0.0.1:49155', transport: 'socket'
```
```
Connected to the target VM, address: '127.0.0.1:49174', transport: 'socket'
notSafeList:9985
safeList:10000
notSafeMap:10000
safeMap:10000
Disconnected from the target VM, address: '127.0.0.1:49174', transport: 'socket'
```
从上面就可以看到ArrayList和HashMap是会丢失数据的，而CopyOnWriteArrayList和ConcurrentHashMap线程安全。其实里面还有很多值得探究的点，譬如synchronized的原理，ReentrantLock是啥，什么是分段锁，CountDownLatch是什么等等，所以技术有很多探究的地方。