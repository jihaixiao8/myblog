---
layout:     post
title:      "讲讲线程间的通信"
subtitle:   " \"线程与线程之间互相独立，那么他们是靠什么互相通信的呢\""
date:       2017-04-03 00:00:00
author:     "Jihaixiao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
    - 并发
---

> “Yeah It's on. ”


## 前言

线程之间都各自有各自的工作空间，自己的工作副本，如果互相之间不通信，自己搞自己的事，那么价值一般般，如果多个线程能够相互通信，互相配合去完成工作，这就明显提升了效率。

## 正文

#### 1.volatile和synchronized关键字

这两个java提供的关键字可以用作线程间的通信，先说volatile：

java支持多个线程同时访问一个成员变量，但是在现代计算机多核处理器中，每个线程都拥有这个成员变量的一份自己的拷贝（为了提高程序运行效率），也就说，在程序运行期间，线程读到的成员变量并不一定是新的，例如A线程读变量X，存放到自己的工作内存中一份副本，但后来X被线程B改变了值，A并不知道，还用自己的工作内存的副本继续计算，那就会算错了。

而volatile关键字则可以起到让A,B线程通信的作用：被volatile修饰的成员变量会告知所有线程对X变量的使用要从主内存中读取，而任何线程对变量X的修改必须同步刷新到主内存中去，它能保证所有线程对变量X的可见性。

例如有个布尔类型的开关成员变量：boolean swtich = true。某个线程对它执行关闭操作，swtich = false;这儿涉及到多个线程的访问，所以要设置成volatile boolean swtich = true  才能保证结果的正确性，因为这么一弄，所有线程对swtich变量的访问和修改都需要以主内存为准。但切勿滥用volatile，它只是保证可见性的一种方式，不能保证操作原子性，而且会对程序性能有所损耗。

再来说说synchronized关键字：

它修饰在方法上或者代码块上，用来保证多个线程在同一时刻只有一个线程可以访问被修饰的方法或者代码块，可以保证线程对变量访问的可见性和排他性。

看如下代码：

```java
package com.jd.jhx.concurrent;

/**
 * Created by jihaixiao on 2017/4/4.
 */
public class Synchronized {

    public static void main(String[] args) {
        //对Synchronized.class对象加锁
        synchronized (Synchronized.class) {
            //静态同步方法，对Synchronized.class对象加锁
            m();
        }
    }

    public static synchronized void m() {
        System.out.println("123");
    }
}
```

我们可以到编译的class文件目录下 使用，javap -v Synchronized.class命令，部分输出如下：

    public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #2                  // class com/jd/jhx/concurrent/Synchronized
         2: dup
         3: astore_1
         4: monitorenter
         5: invokestatic  #3                  // Method m:()V
         8: aload_1
         9: monitorexit
        10: goto          18
        13: astore_2
        14: aload_1
        15: monitorexit
        16: aload_2
        17: athrow
        18: return
      Exception table:
         from    to  target type
             5    10    13   any
            13    16    13   any
      LineNumberTable:
        line 9: 0
        line 10: 5
        line 11: 8
        line 12: 18
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      19     0  args   [Ljava/lang/String;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 13
          locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

    public static synchronized void m();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=0, args_size=0


可以看到，JVM对于同步块使用了monitorenter 和monitorexit命令去实现的，对于同步方法则依靠方法修饰符上的ACC_SYNCHRONIZED完成的。不管那种，本质都是对一个对象的监视器进行获取，而这个获取过程是排他的，也就是说同一个时刻，只有一个线程能获取到由sysnchronized所保护的对象的监视器。而没有获取到监视器的线程将进入BLOCKED状态，被阻塞到同步方法或同步块的入口处。

synchronized通过这种监视器的方式，实现线程的通信，当A，B线程尝试获取变量X的监视器的时候，A线程获取到了，执行代码块，B线程阻塞，进入同步队列等待，当A执行完，monitorexit命令执行后，唤醒等待队列里的B线程，B线程被唤醒后，再次去尝试获取X的监视器。

#### 2.线程等待/通知机制

