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
    - 并发编程
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

等待/通知机制是任何java对象都具有的，所以的相关方法都被定义到java.lang.Object中，下面是几个重要方法的解释：

* notify()：

  通知一个对象上等待的线程，使其从wait()方法返回，其返回的前提是获取到了对象的锁。

* notifyAll()：

  通知所有等待在该对象上的线程，作用类似于notify()，是个全量的功能。

* wait()：

  调用该方法的线程进入WAITING状态，只有等待另外线程的通知或者被中断才会返回，调用wait()方法后，该线程会释放对象的锁。

* wait(long)

  超时等待一段时间，单位是毫秒，如果这段时间没有通知，那么就超时返回。

* wait(long,int)

  超时时间更细粒度控制的方法，可精确到纳秒。

等待/通知机制，就例如A,B线程，A调用了对象O的wait()方法，进入等待状态，并释放锁，而另一个线程B获取到A释放的锁，调用notify()或者notifyAll()方法，待线程B释放锁后，线程A会收到通知，从对象O的wait()方法返回，进而执行后续操作。也就是说，线程A,B通过对象O来进行通信。

可以看如下一个例子，分别new两个线程，一个WaitThread，调用wait()方法后，等待后续操作，一个NotifyThread，调用notifyAll()方法，然后继续加锁，最后释放。

```java
package com.jd.jhx.concurrent;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.TimeUnit;

/**
 * Created by jihaixiao on 2017/4/4.
 */
public class WaitNotify {


    static boolean flag = true;
    static Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {
        Thread waitThread = new Thread(new Wait(),"WaitThread");
        waitThread.start();
        TimeUnit.SECONDS.sleep(1);
        Thread notifyThread = new Thread(new Notify(),"NotifyThread");
        notifyThread.start();
    }

    static class Wait implements Runnable {

        @Override
        public void run() {
            //加锁
            synchronized (lock) {
                //当 条件不满足时，继续wait,同时释放了lock的锁。
                while (flag) {
                    System.out.println(Thread.currentThread()+"flag is true.wait@"+
                    new SimpleDateFormat("HH:mm:ss").format(new Date()));
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                //条件满足，完成工作。
                System.out.println(Thread.currentThread()+"flag is false.running@"+
                new SimpleDateFormat("HH:mm:ss").format(new Date()));
            }
        }
    }

    static class Notify implements Runnable {

        @Override
        public void run() {
            //加锁
            synchronized (lock) {
                //获取到lock的锁，然后进行通知，通知时不会释放lock的锁
                //直到当前线程释放锁，WaitThread才会从wait方法中返回。
                System.out.println(Thread.currentThread()+"hold lock.notify"+
                new SimpleDateFormat("HH:mm:ss").format(new Date()));
                lock.notifyAll();
                flag = false;
                SleepUtil.second(5);
            }

            //再次加锁
            synchronized (lock) {
                System.out.println(Thread.currentThread()+"hold lock again.notify"+
                        new SimpleDateFormat("HH:mm:ss").format(new Date()));
                SleepUtil.second(5);

            }
        }
    }

}
```

打印的结果如下：

Thread[WaitThread,5,main]flag is true.wait@20:49:19<br>
Thread[NotifyThread,5,main]hold lock.notify20:49:20<br>
Thread[NotifyThread,5,main]hold lock again.notify20:49:25<br>
Thread[WaitThread,5,main]flag is false.running@20:49:30<br>

这个例子可以得到如下结论：

1. 调用wait(),notify(),notifyAll()方法时需要先获取到对应的对象的锁。
2. 调用wait()方法后，线程的状态由RUNNING变为WAITING，并将当前线程放入等待队列。
3. notify()或者notifyAll()方法调用后，等待线程不会立刻从wait()方法返回，需要调用notify()或notifyAll()的方法释放锁以后，等待线程才有机会从wait()方法返回。
4. notify()方法是将等待队列中的一个等待线程从等待队列移到同步队列中，而notifyAll()方法是将等待队列中的所有线程全部移到同步队列，被移动的线程从WAITING状态变为BLOCKED。这也解释了第三条，为何需要等通知线程释放锁wait()方法才会返回，只有释放了锁，等待线程才有机会去竞争锁，然后，从wait()方法中返回。
5. 从wait()方法返回的前提是获得了调用对象的锁。

#### 3.Thead.join()的使用

thread.join()的含义就是：当前线程A等待thead线程终止后才从thread.join()方法返回。

这个也是线程间的通知/等待机制实现的，可以看看源码，如下：

```java
public final synchronized void join(long millis)
throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
      	//如果线程一直未终止，那么就持续等待，直到线程终止，跳出循环。
        while (isAlive()) {
            wait(0);
        }
    } else {
    	//暂且不看
    }
```

调用该方法会对调用的对象加锁，如果调用的线程一直没有终止，那么该方法会一直让线程处于WAITING状态，直到线程终止，才从join方法返回。

可以看如下例子：

```java
package com.jd.jhx.concurrent;

import java.util.concurrent.TimeUnit;

/**
 * Created by jihaixiao on 2017/4/4.
 */
public class Join {

    public static void main(String[] args) throws InterruptedException {
        Thread previous = Thread.currentThread();
        for (int i=0;i<6;i++) {
            Thread t = new Thread(new Demino(previous),String.valueOf(i));
            t.start();
            previous = t;
        }
        TimeUnit.SECONDS.sleep(5);
        System.out.println(Thread.currentThread().getName()+" terminate");
    }


    static class Demino implements Runnable {

        private Thread thread;

        public Demino(Thread thread) {
            this.thread = thread;
        }

        @Override
        public void run() {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+" terminate");
        }
    }
}
```

结果如下：

main terminate<br>
0 terminate<br>
1 terminate<br>
2 terminate<br>
3 terminate<br>
4 terminate<br>
5 terminate<br>

线程0持有主线程的引用，然后在线程0执行的时候，调用主线程的join()方法，然后线程0处于等待状态，只有当主线程终止的时候，线程0从join()方法返回，然后执行完成...以此类推。

同理，如下这个例子：

```java
public class Join1 {

    public static void main(String[] args) throws InterruptedException {
        Thread.currentThread().join();
    }
}
```

这个例子，看过join的源码就知道，main线程会一直存活，然后一直死循环在等待，但是main线程终止不了，join方法也就退出不了了，造成了死锁。