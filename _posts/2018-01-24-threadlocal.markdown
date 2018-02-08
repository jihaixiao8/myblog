---
layout:     post
title:      JDK1.8中ThreadLocal源码分析
subtitle:   " \"jdk1.8的实现，了解ThreadLocal的运作方式和原理\""
date:       2018-01-24 00:00:00
author:     "Jihaixiao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
    - 并发编程
---

> “Yeah It's on. ”

## 前言

java中最常用的并发框架就是线程池，如果需要异步或者并发任务都可以使用它，使用它有几个好处：

1. 降低资源消耗。通过重复使用创建的线程降低线程创建和销毁的开销。
2. 提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。
3. 提高线程的可管理性。线程是稀缺资源，如果无限制创建，会消耗系统资源，降低系统稳定性，使用线程池统一分配调度，加强了可管理性。

## 正文

#### 1. 线程池内部流程

当一个任务提高到线程池，线程池是如何处置任务的，流程如下：

1. 线程池先判断核心线程池的线程是否都在执行任务，如果不是，则创建一个新的线程执行任务，如果核心线程池的线程都在执行任务，则进入下个流程。
2. 线程池判断工作队列是否已经满了，如果没满，则将新提交的任务放入工作队列，如果满了，则进入下一个流程。
3. 线程池判断线程池的线程是否都在工作，如果没有，则创建一个线程执行任务，如果已经饱和，则交给饱和策略去处理。

#### 2.线程池内部流程代码描述

1. 如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（此处需要获取全局锁）
2. 如果运行的线程多余corePoolSize,则将任务加入BlockingQueue。
3. 如果无法将任务加入BlockingQueue,则创建新的线程来执行任务。（此处需要获取全局锁）
4. 如果创建新线程使当前运行的线程超过maximumPoolSize,任务将被拒绝，并调用RejectedExecutionHandler.rejectedExecution()方法来执行饱和策略。

ThreadPoolExecutor整体流程如上，我们要尽可能避免获取全局锁，如果ThreadPoolEdxcutor完成预热之后（当前运行的线程大于等于corePoolSize），以后执行execute()方法都是执行步骤2，不需要获取全局锁。

#### 3.ThreadPoolExecutor的全局变量

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

ThreadPoolExecutor一个很重要的全局变量就是ctl，使用的原子的int类型,为了节约资源，并且位运算有与生俱来的高效率，所以jdk使用32位的ctl，高3位用来表示线程池的状态，低29位用来表示工作线程的数量。ctl默认是1110 0000 0000 0000 0000 0000 0000 0000 ,即默认是运行状态，工作线程数为0。

```java
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```

* COUNT_BITS：代表工作线程数量的位数。
* CAPACITY：线程池的最大容量，1左移29位减1 就是 0001 1111 1111 1111 1111 1111 1111 1111

```java
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

该方法的作用是获取线程的数量，入参是ctl的值，CAPACITY在此方法中充当掩码的作用，屏蔽高3位，&操作后得到的就是工作线程的数量。

```java
private static int runStateOf(int c)     { return c & ~CAPACITY; }
```

runStateOf方法是获取线程池的状态，入参也是ctl的值，先对CAPACITY取反码，然后做掩码，屏蔽低29位，只看高3位，即获得了线程池的状态。

###### 线程池的状态

```java
1110 0000 0000 0000 0000 0000 0000 0000
private static final int RUNNING    = -1 << COUNT_BITS; 
0000 0000 0000 0000 0000 0000 0000 0000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
0010 0000 0000 0000 0000 0000 0000 0000
private static final int STOP       =  1 << COUNT_BITS;
0100 0000 0000 0000 0000 0000 0000 0000
private static final int TIDYING    =  2 << COUNT_BITS;
0110 0000 0000 0000 0000 0000 0000 0000
private static final int TERMINATED =  3 << COUNT_BITS;
```

* Running:

  111状态，接收新的任务，也处理任务队列里的任务。

* ShutDown:

  000状态，不接收新的任务，但是处理任务队列里的任务。

* Stop:

  001状态，不接收新的任务，不处理任务队列里的任务，并且中断正在执行的任务。

* Tidying：

  010状态，所有的任务都停止了，工作线程数量为0（ctl低29位为0），往tidying状态转换的线程都会去调用terminated这个钩子函数。

* Terminated:

  011状态，terminated()方法执行完成。

  ###### 

###### 其他常量和变量



```java
private final BlockingQueue<Runnable> workQueue;
```

待执行的线程放入的队列。

```java
private final ReentrantLock mainLock = new ReentrantLock();
```

线程池内的全局锁，用来加锁一些必要的操作。

```java
private final HashSet<Worker> workers = new HashSet<Worker>();
```

存放线程池内所有工作线程的HashSet，操作的时候必须加锁(HashSet非线程安全)

```java
private final Condition termination = mainLock.newCondition();
```

awaitTermination 用来线程通信的condition。

```java
private int largestPoolSize;
```

用来追踪线程池达到的最大size，操作它必须要加锁，这个变量用来做统计，监控。

```java
private long completedTaskCount;
```