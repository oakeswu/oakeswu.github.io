---
layout:     post
title:      "Java并发(十二) -- synchronized和volatile底层原理"
subtitle:   ""
date:       2020-06-11
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Java并发
---

**本文主要参考Java并发编程的艺术，IA-32架构软件开发人员手册并基于JDK1.8**

# 背景
我们在之前的文章中多次用到了volatile和synchronized，volatile可以保证可见性，有序性，但是不能保证原子性，所以线程中如果包含volatile共享变量的非原子性操作就可能存在并发风险，此时可以考虑使用synchronized锁，所以我们探究下volatile是如何保证可见性和有序性， synchronized是如何实现锁机制的。

# volatile
- 可见性
首先我们在[JMM](https://www.jianshu.com/writer#/notebooks/45155628/notes/69448465)中可以得知线程工作内存其实就是CPU寄存器和高速缓存的抽象描述，在单CPU中由于线程其实是共用同一个高速缓存中的，所以不存在缓存不一致问题，而在多CPU中存在多个高速缓存区，从而存在缓存不一致的情况。
1：线程A在CPU1对应的Cache1中同步了主内存的一份数据Data1
2：线程B在CPU2对应的Cache2中同步了主内存的一份数据Data2
3：线程开始前Data1和Data2都是同步的主内存中的数据，所以Data1的数据跟Data2的数据相同
4：线程A开始执行并修改Cache1中的Data1数据，然后更新到主内存中
5：线程B开始执行并修改Cache2中的Data2数据，Data2数据同步到主内存中，此时覆盖了Data1的数据
那么这时候加上volatile可以避免这种情况发生，当第四步Data1同步到主内存中时就会让Cache2中的Data2失效并重新拉取主内存数据，保证Data2数据与主内存最新数据一致。我们看下volatile如何实现的
```
public class VolatileTest {
    private static volatile String volatileStr;
    public static void main(String[] args) {
        volatileStr = "test";
    }
}
```
将代码对应的汇编语言打印出来看看，我这里只截取了一部分
```
CompilerOracle: compileonly *VolatileTest.main
Connected to the target VM, address: '127.0.0.1:65126', transport: 'socket'
Loaded disassembler from hsdis-amd64.dll
Decoding compiled method 0x0000000002d99d50:
Code:
Argument 0 is unknown.RIP: 0x2d99ea0 Code size: 0x00000170
[Disassembling for mach='amd64']
[Entry Point]
[Verified Entry Point]
[Constants]
# {method} {0x000000001bcd0298} 'main' '([Ljava/lang/String;)V' in 'newthread/VolatileTest'
# parm0:    rdx:rdx   = '[Ljava/lang/String;'
#           [sp+0x40]  (sp of caller)
0x0000000002d99ea0: mov     dword ptr [rsp+0ffffffffffffa000h],eax
0x0000000002d99ea7: push    rbp
0x0000000002d99ea8: sub     rsp,30h
0x0000000002d99eac: mov     rsi,1bcd0388h     ;   {metadata(method data for {method} {0x000000001bcd0298} 'main' '([Ljava/lang/String;)V' in 'newthread/VolatileTest')}
0x0000000002d99eb6: mov     edi,dword ptr [rsi+0dch]
0x0000000002d99ebc: add     edi,8h
0x0000000002d99ebf: mov     dword ptr [rsi+0dch],edi
0x0000000002d99ec5: mov     rsi,1bcd0290h     ;   {metadata({method} {0x000000001bcd0298} 'main' '([Ljava/lang/String;)V' in 'newthread/VolatileTest')}
0x0000000002d99ecf: and     edi,0h
0x0000000002d99ed2: cmp     edi,0h
0x0000000002d99ed5: je      2d99f1dh          ;*ldc  ; - newthread.VolatileTest::main@0 (line 12)
0x0000000002d99edb: mov     rsi,76b4bb498h    ;   {oop(a 'java/lang/Class' = 'newthread/VolatileTest')}
0x0000000002d99ee5: mov     rdi,76b60f5e8h    ;   {oop("test")}
0x0000000002d99eef: mov     r10,rdi
0x0000000002d99ef2: shr     r10,3h
0x0000000002d99ef6: mov     dword ptr [rsi+68h],r10d
0x0000000002d99efa: shr     rsi,9h
0x0000000002d99efe: mov     rdi,0ea43000h
0x0000000002d99f08: mov     byte ptr [rsi+rdi],0h
0x0000000002d99f0c: lock add dword ptr [rsp],0h  ;*putstatic volatileStr
                                              ; - newthread.VolatileTest::main@2 (line 12)
0x0000000002d99f11: add     rsp,30h
0x0000000002d99f15: pop     rbp
0x0000000002d99f16: test    dword ptr [0c30100h],eax
                                              ;   {poll_return}
0x0000000002d99f1c: ret
0x0000000002d99f1d: mov     qword ptr [rsp+8h],rsi
0x0000000002d99f22: mov     qword ptr [rsp],0ffffffffffffffffh
0x0000000002d99f2a: call    2d8f360h          ; OopMap{rdx=Oop off=143}
                                              ;*synchronization entry
                                              ; - newthread.VolatileTest::main@-1 (line 12)
                                              ;   {runtime_call}
0x0000000002d99f2f: jmp     2d99edbh
0x0000000002d99f31: nop
0x0000000002d99f32: nop
0x0000000002d99f33: mov     rax,qword ptr [r15+2a8h]
0x0000000002d99f3a: mov     r10,0h
0x0000000002d99f44: mov     qword ptr [r15+2a8h],r10
0x0000000002d99f4b: mov     r10,0h
0x0000000002d99f55: mov     qword ptr [r15+2b0h],r10
0x0000000002d99f5c: add     rsp,30h
0x0000000002d99f60: pop     rbp
0x0000000002d99f61: jmp     2cff6e0h          ;   {runtime_call}
0x0000000002d99f66: hlt
0x0000000002d99f67: hlt
```
我们主要找到lock add dword ptr [rsp],0h  ;*putstatic volatileStr ; - newthread.VolatileTest::main@2 (line 12)这句话，我们可以发现加了一个lock命令，此时你把代码中的volatile去掉你会发现没有lock命令了，那么这个lock命令的作用是啥，我们查查IA-32架构手册可以看到
```
在Pentium 和早期的IA-32 处理器中，LOCK 前缀会使处理器执行当前指令时产生
一个LOCK#信号，这总是引起显式总线锁定出现。
在Pentium 4、Intel Xeon 和P6 系列处理器中，加锁操作是由高速缓存锁或总线
锁来处理。如果内存访问有高速缓存且只影响一个单独的高速缓存线，那么操作中就
会调用高速缓存锁，而系统总线和系统内存中的实际内存区域不会被锁定。同时，这
条总线上的其它Pentium 4、Intel Xeon 或者P6 系列处理器就回写所有的已修改数据
并使它们的高速缓存失效，以保证系统内存的一致性。如果内存访问没有高速缓存且/
或它跨越了高速缓存线的边界，那么这个处理器就会产生LOCK#信号，并在锁定操作期
间不会响应总线控制请求。
```
所以我们可以看到lock命令会保证系统内存的一致性

- 有序性
jvm会对代码进行编译器重排序和处理器重排序进行代码执行效率的优化，但是在并发中可能造成并发风险，volatile通过内存屏障禁止指令重排序来保证有序性。
1：在每个volatile写操作的前面插入一个StoreStore屏障，防止写volatile与后面的写操作重排序。
2：在每个volatile写操作的后面插入一个StoreLoad屏障，防止写volatile与后面的读操作重排序。
3：在每个volatile读操作的后面插入一个LoadLoad屏障，防止读volatile与后面的读操作重排序。
4：在每个volatile读操作的后面插入一个LoadStore屏障，防止读volatile与后面的写操作重排序。

# synchronized
synchronized是java很早就有的一个重量级锁，同步方法和同步代码块是通过不同方式实现线程安全的。同步方法是通过lock cmpxchg命令，使用了CAS思想判断。同步代码块则是通过monitorenter和monitorexit实现，JVM要保证每个monitorenter必须有对应的monitorexit配对，线程执行到monitorenter指令时将会尝试获取对象对应的moitor的所有权，即对象锁。我们可以看下代码实例
```
public class SynchronizedPrincipleTest {
    private static synchronized void test() {
        System.out.println("test");
    }
    public static void main(String[] args) {
        test();
    }
}

0x00000000029ed2c8: and     rax,37fh
0x00000000029ed2cf: mov     r8,rax
0x00000000029ed2d2: or      r8,r15
0x00000000029ed2d5: lock cmpxchg qword ptr [r10],r8
0x00000000029ed2da: jne     29ed33dh          ;*synchronization entry
                                              ; - newthread.SynchronizedPrincipleTest::test@-1 (line 9)

0x00000000029ed2dc: mov     edx,17h
0x00000000029ed2e1: nop
0x00000000029ed2e3: call    29257a0h          ; OopMap{off=136}
                                              ;*getstatic out
                                              ; - newthread.SynchronizedPrincipleTest::test@0 (line 9)
                                              ;   {runtime_call}
0x00000000029ed2e8: int3                      ;*getstatic out
                                              ; - newthread.SynchronizedPrincipleTest::test@0 (line 9)

0x00000000029ed2e9: lock cmpxchg qword ptr [r10],r11
0x00000000029ed2ee: lea     rbx,[rsp+0h]
0x00000000029ed2f3: mov     rax,qword ptr [r10]
```
```
public class SynchronizedPrincipleTest {

   public static void main(String[] args) {
        synchronized (SynchronizedTest.class) {
            System.out.println("test");
        }
    }
}

0x000000000378a020: and     rax,0fffffffffffff007h
0x000000000378a027: mov     qword ptr [r8],rax
0x000000000378a02a: jne     378a248h          ;*monitorenter
                                              ; - newthread.SynchronizedPrincipleTest::main@4 (line 15)

0x000000000378a030: mov     qword ptr [rsp+28h],rsi
0x000000000378a035: mov     qword ptr [rsp+20h],rdx
0x000000000378a03a: nop     word ptr [rax+rax+0h]
0x000000000378a040: jmp     378a2b8h          ;   {no_reloc}
```

# 总结
我们上面讲到了synchronized和volatile的原理发现并发中的可见性，有序性和原子性保证之后就可以并发安全，但是synchronized的有序性和volatile的有序性含义不一样，synchronized通过锁的争抢保证线程执行的有序性，而volatile保证的是指令重排序的有序性。这也是为什么**懒汉式单例模式加了synchronized也仍然需要给变量加volatile的原因**
