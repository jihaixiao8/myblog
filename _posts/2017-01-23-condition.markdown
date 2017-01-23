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

### 1:Condition与Java对象监视器方法的比较。

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



### 2:Condition使用方法。

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

> 未完待续...