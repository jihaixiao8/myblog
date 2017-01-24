---
layout:     post
title:      "Java Condition使用与源码分析"
subtitle:   " \"如何使用Condition实现线程之间的等待与唤醒 \""
date:       2017-01-23 00:00:00
author:     "Jihaixiao"
header-img: "img/post-bg-2017.jpg"
catalog: true
tags:
    - Java
    - 并发编程
---

## Condition使用简介

### 1:Condition与Java对象监视器方法的比较

任何一个Java对象都有一组监视器方法（在java.lang.Ojbect上），主要包括wait()，wait(long timeout)，notify()，notifyAll()等方法，这几个方法跟synchronized关键字结合，可以实现等待/通知模式。

Condition接口也提供了类似的方法，通过与Lock配合，也可以实现等待通知模式，下面通过一个表格笔记下两者的共同点和区分方式。

|             对比项             |   Object Monitor Method    |                Condition                 |
| :-------------------------: | :------------------------: | :--------------------------------------: |
|            前置条件             |           获取对象的锁           | 调用Lock.lock()获取锁<br/>调用Lock.newCondition()获取Condition对象 |
|            调用方式             | 直接调用，例如：<br/>object.wait() |      直接调用，如：<br/>condition.await()       |
|           等待队列个数            |             1个             |                    多个                    |
|       当前线程释放锁并进入等待状态        |             支持             |                    支持                    |
| 当前线程释放锁，并进入等待状态，在等待的时候不响应中断 |            不支持             |                    支持                    |
|      当前线程释放锁并进入超时等待状态       |             支持             |                    支持                    |
|   当前线程释放锁并进入等待状态到将来的某个时间    |            不支持             |                    支持                    |
|        唤醒等待队列中的一个线程         |             支持             |                    支持                    |
|        唤醒等待队列中的全部线程         |             支持             |                    支持                    |

从上表可以看出，Object监视器方法能做到的，Condition都能做到，Object监视器方法不能做的，Condition也能做到，它提供了更加丰富的功能。（基于AQS做的）。



### 2:Condition使用方法

Condition定义了等待/通知两种类型的方法，当前线程调用这些方法时，必须要提前获取到Condition对象关联的锁。Condition对象是由Lock对象创建出来的（调用newCondition()方法），也就是说，Condition对象是依赖Lock对象的。看下面的例子：

```java
public class ConditionUseCase {

    static Lock lock = new ReentrantLock();
    static Condition condition = lock.newCondition();


    public static void main(String[] args) throws InterruptedException {
        //1:启动A线程，然后让A线程获取到锁，调用await()方法，释放锁，然后当前线程进入等待状态。
        AThread aThread = new AThread();
        aThread.start();
        //2:睡1秒，确保A线程肯定现在B线程之前启动。
        Thread.sleep(1000);
        //3:启动B线程，让B线程获取到锁，然后调用sigal()方法，
        //去唤醒在等待队列中的线程（此处只有A调用了await()，
      	//所以只有A进入等待队列，等待被唤醒）
        BThread bThread = new BThread();
        bThread.start();
    }

    /**
     * A线程，在获取到锁以后会进入等待状态
     */
    static class AThread extends Thread{

        private volatile boolean flag = false;

        @Override
        public void run() {
            //注意:一定要先获取锁，不然会报错
            lock.lock();
            System.out.println("开始执行啦A"+Thread.currentThread().getName());
            try {
                condition.await();
                System.out.println("结束执行了A"+Thread.currentThread().getName());
            } catch (InterruptedException e) {
                System.out.println("任务A被中断");
            } finally {
                lock.unlock();
            }

        }


    }

    /**
     * B线程，用来在获取到锁以后，将A线程唤醒。
     */
    static class BThread extends Thread {
        @Override
        public void run() {

            try {
                //注意:一定要先获取锁，不然会报错
                lock.lock();
                System.out.println("开始执行B了");
                condition.signal();
            } finally {
                lock.unlock();
            }
            System.out.println("结束执行B了");

        }
    }

}
```

执行结果如下：

**开始执行啦AThread-0**<br/>
**开始执行B了**<br/>
**结束执行B了**<br/>
**结束执行了AThread-0**<br/>

可以看出，A先执行，然后释放了锁（此处在Condition源码中有体现），一直在等待，然后B线程启动，然后B线程调用了sigal()方法，唤醒了A，然后A才执行结束了。这样使用Condition接口就可以实现线程之间互相协调，等待和唤醒功能。（ArrayBlockingQueue 有界阻塞队列就是利用Condition去实现的）。

### 3:Condition部分方法简介

如下表格所示：

| 方法名称                                     | 描述                                       |
| :--------------------------------------- | :--------------------------------------- |
| void await() throws InterruptedException | 当前线程进入等待状态，直到被通知（sigal）或中断，当前线程将进入运行状态且从await()方法返回的情况，如下：<br/> 1：其他线程调用该Condition的singal()或者singalAll()方法，而当前线程被选中唤醒。<br/>2：其他线程调用interrupt()方法中断当前线程。<br>3：如果当前等待线程从await()方法返回，那么表明该线程已经获取了Condition对象所对应的锁。 |
| void awaitUninterruptibly()              | 当前线程进入等待状态直到被通知，该方法不响应中断。                |
| long awaitNanos(long nanosTimeout) throws InterruptedException | 当前线程进入等待状态直到被通知，超时或者中断，返回值是表示剩余的时间，如果在nanosTimeout纳秒前被唤醒，那么返回值就是（nanosTimeout - 实际耗时），如果返回值是0或者负数，那么可以认为超时了。 |
| boolean awaitUntil(Date deadline) throws InterruptedException | 当前线程进入等待状态直到被通知，中断或者到某个时间。如果没有到指定时间就被唤醒，那么方法返回true，否则，表示到了指定时间，自动被唤醒，并返回false。 |
| boolean await(long time,TimeUnit unit) throws InterruptedException | 当前线程进入等待状态直到被通知，中断，或者超时，可以使用TimeUnit指定时间单位，如果未超时就被唤醒，返回true,否则，超时了，返回false。 |
| void singal()                            | 唤醒一个等待在Condition上的线程，该线程从等待方法返回前必须获取与Condition相关联 |
| void singalAll()                         | 唤醒所有等待在Condition上的线程，能够从等待方法返回的线程必须获取与Condition相关联的锁 |

下面提供一个简单有界阻塞队列的例子，有界阻塞队列的特性就是当队列为空的时候，队列的获取操作将会阻塞获取线程，直到队列中有新增元素，才会被唤醒继续获取；当队列已满时，队列的插入操作将会阻塞插入线程，直到队列中出现空位：

```java
package cn.com.jd.util;

import java.util.Arrays;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Created by jihaixiao on 2017/1/24.
 */
public class MyArrayBlockingQueue<E> {

    //队列数组
    private final Object[] items;

    //添加元素索引，最大值是items.length
    private int addIndex;

    //删除元素索引，最大值是items.length
    private int removeIndex;

    //当前队列数组元素数量
    private int count;

    //锁
    private final Lock lock;

    //删除元素Condition
    private final Condition notEmpty;

    //添加元素Condition
    private final Condition notFull;

    /**
     *
     * 构造器，此处items，lock，notEmpty，notFull 都被final修饰
     * 都具有final的内存语义，能够保证在引用构造过程中没有被溢出的
     * 情况下，MyArrayBlockingQueue实例可以被正确构造，所有线程都
     * 能读到正确的items，lock，notEmpty，notFull的值。
     *
     * @param size
     */
    public MyArrayBlockingQueue(int size) {
        items = new Object[size];
        lock = new ReentrantLock();
        notEmpty = lock.newCondition();
        notFull = lock.newCondition();
    }

    /**
     * 往队列中添加元素，如果队列满了，当前添加线程阻塞，直到队列有空位位置
     * @param e
     * @throws InterruptedException
     */
    public void add(E e) throws InterruptedException {
        /**
         * 1：想使用Condition，先获取它对应的锁再说
         */
        lock.lock();
        try {
            /**
             * 2:循环检测队列的长度，如果满了，调用notFull这个Condition的
             * await()方法让当前add方法的线程进入等待状态，如果队列随后有空位了，
             * 那么该线程被唤醒，发现不符合count == item.length条件，就能跳出此
             * 循环
             */
            while (count == items.length) {
                notFull.await();
            }
            /**
             * 3：把添加的元素放到指定的索引上
             */
            items[addIndex] = e;
            /**
             * 4：此处如果元素索引已经达到了队列最大值，那么要
             * 置0，从头开始。因为这是个先进先出FIFO队列，所以，元素被取走
             * 都是从头部开始取，头部肯定是先出空位的
             * (此处对普通int类型addIndex 进行++操作能保证原子性和可见性，是因为
             * 这个操作被锁住了)
             */
            if (++addIndex == items.length)
                addIndex = 0;
            /**
             * 5：添加完元素，数组当前元素数量+1,此处操作被锁住，所以可以保证
             * 线程安全
             */
            ++count;
            /**
             * 6：添加完元素，队列里面有元素了，此时调用下sigal()方法，去唤醒一个
             * 在notEmpty等待队列里面等着取元素的线程。
             */
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    /**
     * 从队列头移除元素，如果队列空了，那么当前移除元素线程会被阻塞，直到队列中有元素了
     * @return
     * @throws InterruptedException
     */
    public E remove() throws InterruptedException {
        /**
         * 1：想使用Condition，先获取它对应的锁再说
         */
        lock.lock();
        try {
            /**
             * 2：循环检测队列当前元素的数量，如果为0了，表示
             * 当前队列为空，此时不能取元素了，要阻塞当前remove的线程
             * 并将该线程加入到notEmpty的等待队列中，如果当前线程被唤醒
             * 表示队列有元素了，那么就不符合count == 0这个条件，就能跳出
             * 这个循环。
             */
            while (count == 0){
                notEmpty.await();
            }
            /**
             * 3：根据删除元素索引拿到被remove的元素o
             */
            Object o = items[removeIndex];
            /**
             * 4：将该数组中被删除元素索引那儿的引用置为null,便于GC
             */
            items[removeIndex] = null;
            /**
             * 5：如果被移出元素索引达到队列长度，那么
             * 就会被removeIndex置为0，从头开始
             */
            if (++removeIndex == items.length)
                removeIndex = 0;
            /**
             * 6：当前队列中元素数量-1，因为锁，保证线程安全
             */
            --count;
            /**
             * 7：删除完元素，尝试去唤醒一个在notNull队列里
             * 等待的添加元素的线程。
             */
            notFull.signal();
            return (E) o;
        } finally {
            lock.unlock();
        }
    }

    @Override
    public String toString() {
        return Arrays.toString(items);
    }
}
```

可以用下面的Main函数，测一下这个类能不能正常工作：队列长度设置为5，启动了6个 添加元素的线程，添加了5个满了以后其中一个线程被阻塞，直到一个删除元素的线程启动，去删除元素后，那个被阻塞的添加线程被唤醒，并往队列里添加了元素。

```java
package cn.com.jd.test.concurrent;

import cn.com.jd.util.MyArrayBlockingQueue;

/**
 * Created by jihaixiao on 2017/1/24.
 */
public class MyQueueTest {

    static MyArrayBlockingQueue queue = new MyArrayBlockingQueue(5);

    public static void main(String[] args) throws InterruptedException {
        for (int i=0;i<6;i++){
            AddThread addThread = new AddThread(i);
            addThread.start();
        }
        Thread.sleep(1000);
        RemoveThread removeThread = new RemoveThread();
        removeThread.start();
        Thread.sleep(1000);
        System.out.println(queue);
    }

    static class AddThread extends Thread {

        private Object o;

        public AddThread(Object o) {
            this.o = o;
        }

        @Override
        public void run() {
            try {
                queue.add(o);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }


    static class RemoveThread extends Thread {

        @Override
        public void run() {
            try {
                queue.remove();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

}
```

## Condition原理，源码分析

ConditionObject是AQS的内部类，每个Condition对象都有一个关联的等待队列，这个队列是实现await/singal的关键。

### 1:ConditionOjbect 主要属性

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

ConditionObject只有两个属性，链表的头结点引用firstWaiter和链表的尾节点引用lastWaiter，这两个属性没有加volatile时因为对Condition的操作都必须获取到对应的锁，所以一定能够保证线程可见性，所以此处不多此一举。

> 未完待续...