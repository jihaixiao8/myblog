---
layout:     post
title:      "ReentrantLock源码解析"
subtitle:   " \"根据ReentrantLock源码理解并发工具类的核心AQS \""
date:       2016-12-12 00:00:00
author:     "Jihaixiao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
    - Spring
---




## 前言

这篇笔记都是自己根据JDK源码自己理解的，希望有人能看到，然后跟我探讨下，不一定理解的就好。

## 正文

#### ReentrantLock简介

这是jdk1.5 并发大师Doug Lea 增加的一个锁工具，它可以替代synchronized，并且它还具有synchronized不具备的功能，例如可中断锁，定时锁等。

ReetrantLock同时也是可重入的，而且它有两种形态：公平锁和非公平锁。简单说下这俩概念：

- 公平锁：在线程竞争锁过程中，只有一个会竞争到锁，那么其他线程都会阻塞，当锁被释放时，其他线程才可被唤醒，进入临界区，那么如果下一个进入临界区的线程跟当时阻塞顺序有关的锁，就叫公平锁。阻塞队列依照FIFO队列的形式，先阻塞的，第一个被唤醒，进入临界区。
- 非公平锁：锁释放的时候不会按照阻塞的顺序去唤醒下一个线程，进入临界区，而是全部唤醒，让线程去竞争，成功获取到锁的进入临界区，不公平的地方就是在于不会依仗上一次的状态，还是需要自己去争取。

ReentrantLock 默认是非公平模式，可以看它提供的构造器：

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

默认构造器会 构造一个NofairSync对象（非公平锁同步器），你也可以自己传入fair属性来决定使用哪种锁。这个Sync就是并发工具类的核心，一切都是围绕它来做的。

#### AQS(抽象队列同步器简介)

可以看一下ReentrantLock的源码，会发现，它的核心就是一个Sync对象，那么这个对象是什么呢？再看，Sync这个内部类 实现了AbstractQueuedSynchronizer 类，这才是核心。

AbstractQueuedSynchronized ，抽象队列同步器，顾名思义，就是一个抽象类，使用了模板方法模式，为继承它的子类提供了一系列可以自己定制的模板方法，同时它内部又提供了一个int类型的同步状态state和一个FIFO双向队列（链表实现）：当前线程获取同步状态失败时，同步器会将当前线程以及等待状态等信息构造成一个节点，并将其加入队列，同时阻塞当前线程，当同步状态释放时，会把首节点的线程唤醒，使其再次尝试获取同步状态。

简而言之，就是它提供给你一个同步状态，用来协调线程，还提供给你一个FIFO阻塞队列，用来存放阻塞的线程信息，这俩AQS替你实现了，你就可以用它来自己做一些线程同步类啊，自己定制锁了。

###### 1：同步状态state

看AQS的源码发现如下，它的3个重要属性：

```java
/**
* 同步阻塞队列的头结点,volatile修饰，确保对头结点的修改对
* 所有线程可见。
*/
private transient volatile Node head;

/**
 *	同步阻塞队列的尾节点，volatile修饰，确保对尾节点的修改对所有
 * 	线程可见，只会在调用enq方法增加新的等待节点的时候才会修改尾节点。
 */
private transient volatile Node tail;

/**
 * 	同步状态，非常重要，volatile修饰，确保对它的修改对所有线程可见，线程之间通过
 *	修改它的状态来实现阻塞，唤醒等状态。
 */
private volatile int state;
```

state同步状态AQS提供了3个方法来操作它：

* getState():该方法不会被重写，获取同步状态，在多线程下一定能获取到最新的（volatile的内存语义）

  ```java
  protected final int getState() {
      return state;
  }
  ```

* setState(int newState):该方法不会被重写，设置同步状态，在多线程下，调用该方法设置state以后其他线程能够立即可见（volatile写的内存语义）

  ```java
  protected final void setState(int newState) {
      state = newState;
  }
  ```

* compareAndSetState(int expect,int update):该方法不会被重写，CAS设置当前状态，该方法能够保证设置的原子性。

  ```java
  protected final boolean compareAndSetState(int expect, int update) {
      // See below for intrinsics setup to support this
      return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
  }
  ```

###### 2：AQS提供的可重写方法

|                   方法名称                   |                    描述                    |
| :--------------------------------------: | :--------------------------------------: |
|  protected boolean tryAcquire(int arg)   | 独占式获取同步状态，实现该方法需要查询当前状态并判断同步状态是否符合预期，然后再CAS设置同步状态。 |
|  protected boolean tryRelease(int arg)   |    独占式的释放同步状态，等待获取同步状态的线程将有机会获取同步状态。     |
| protected int tryAcquireShared(int arg)  |   共享式获取同步状态，返回大于或等于0的值，表示获取成功，反之，获取失败。   |
| protected boolean tryReleaseShared(int arg) |                共享式释放同步状态。                |
|  protected boolean isHeldExclusively()   |   当前同步器是否在独占模式下被线程占用，一般该方法表示是否被当前线程占用。   |

###### 3：AQS提供的模板方法

AQS提供的模板方法分为3类：独占式获取与释放同步状态，共享式获取与释放同步状态和查询同步队列中等待线程的状况。

|                   方法名称                   |                    描述                    |
| :--------------------------------------: | :--------------------------------------: |
|          void acquire(int arg)           | 独占式的获取同步状态，如果获取成功，则由该方法返回；如果获取失败，则进入同步队列等待，该方法将会调用重写的tryAcquire(int arg) 方法。该方法不响应中断。 |
|    void acquireInterruptibly(int arg)    | 跟acquire()差不多，只是该方法响应中断，如果当前线程被中断，会抛出InterruptedException异常并返回。 |
| boolean tryAcquireNanos(int arg,long nanos) | 在acquireInterruptibly()上增加了超时限制，如果当前线程在指定时间内（纳秒级别）没有获取到同步状态，那么就返回false,获取到了返回true。 |
|       void acquireShared(int arg)        | 共享式的获取同步状态，与独占式的唯一区别，同一时刻可以有多个线程获取到同步状态。该方法不响应中断。 |
|    void acquireSharedInterruptibly()     |                 该方法响应中断。                 |
| boolean tryAcquireSharedNanos(int arg,long nanos) | 在acquireSharedInterruptibly()上面增加了超时限制。  |
|         boolean release(int arg)         | 独占式的释放同步状态，在同步状态释放之后，会将同步队列中第一个节点包含的线程唤醒。 |
|      boolean releaseShared(int arg)      |               共享式的释放同步状态。                |
|  Collection<Thread> getQueuedThreads()   |             获取等待在同步队列上的线程集合。             |

###### 4：AQS的同步队列

FIFO队列，如果获取同步状态失败，AQS会将失败线程和等待状态构造成一个Node,然后加入同步队列，同时阻塞当前线程，当同步状态释放时，会唤醒头结点的线程，使其再次尝试获取同步状态。

Node包括当前获取同步状态失败线程的引用，等待状态，前置和后继节点，如下所示：

```java
static final class Node{

    /**
     * 
     * 等待状态，节点上线程的等待状态，详情见表格
     * 
     */
    volatile int waitStatus;

    /**
     *
     * 同步队列Node的前置节点，volatile修饰，确保对它的修改能被其他
     * 线程可见
     */
    volatile Node prev;

    /**
     *
     * 同步队列的Node的后继节点，volatile修饰，确保对它的修改能被其他
     * 线程可见
     */
    volatile Node next;

    /**
     *
     * 获取同步状态失败的线程引用，volatile修饰，确保对它的修改能够对其他线程
     * 读到
     */
    volatile Thread thread;

    /**
     *
     * 等待队列中的后继节点，如果当前节点是共享的，那么这个字段将是一个SHARED常量
     * 也就是说节点类型和等待队列中的后继节点共用同一个字段。
     *
     */
    Node nextWaiter;

}
```



#### ReentrantLock 的实现

###### 1：Lock()方法的实现



