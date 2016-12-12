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

###### 2：AQS提供的模板方法

