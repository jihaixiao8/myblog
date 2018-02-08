---
layout:     post
title:      "ThreadPoolExecutor线程池源码分析"
subtitle:   " \"jdk1.8的实现，了解线程池的运作方式和原理\""
date:       2017-10-26 00:00:00
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

PS：负数在计算机中的二进制表示为原码的补码：例如-1  原码是 1000 0000 0000 0000 0000 0000 0000 0001，它的反码是1111 1111 1111 1111 1111 1111 1111 1110，然后取补码：1111 1111 1111 1111 1111 1111 1111 1111,所以-1的二进制表达就是1111 1111 1111 1111 1111 1111 1111 1111。

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

待执行的任务放入的队列。

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
统计完成任务的计数器，仅在工作线程终止后更新，操作时需要加锁。

```java
private volatile ThreadFactory threadFactory;
```

线程工厂，用来创建线程池的新线程。

```java
private volatile RejectedExecutionHandler handler;
```

饱和或者执行shutdown时调用的处理策略。

```java
private static final RejectedExecutionHandler defaultHandler =
    new AbortPolicy();
```

默认的饱和处理策略，丢弃任务。

```java
private volatile long keepAliveTime;
```

空闲的工作线程等待新任务的超时时间，如果allowCoreThreadTimeOut 为true 或者线程数大于corePoolSize，这个时间才会生效，否则空闲线程会一直等待新的任务。

```java
private volatile boolean allowCoreThreadTimeOut;
```

如果为false(默认情况)，核心线程即使空闲，也会一直存活下去，如果为true，核心线程会使用keepAliveTime作为等待新任务的超时时间，超时则销毁。

```java
private volatile int corePoolSize;
```

保持存活的最小的核心线程数（前提是allowCoreThreadTimeOut为false）

```java
private volatile int maximumPoolSize;
```

线程池的最大线程数，最大不能超过CAPACITY。

#### 4: 重要方法解析

###### 4.1:execute(Runnable command)方法

```Java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
  //获取线程池的ctl(状态+线程数)
    int c = ctl.get();
  /**
  如果当前工作线程数小于线程池的核心线程数，那么调用addWorker增加一个线程执行任务，
  如果成功，直接return,如果没成功，继续往下走
  */
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
      //增加线程不成功的情况，重新获取线程池ctl(状态+线程数)
        c = ctl.get();
    }
  	//情况1：线程池是RUNNING态，任务放入队列成功，则进入if块。
  	//情况2：线程池是RUNNING态，但是任务放入队列失败，表示任务队列满了，不进入if块。
  	//情况3：线程池是非RUNNING态，不进入if块。
    if (isRunning(c) && workQueue.offer(command)) {
      	//再次获取线程池状态
        int recheck = ctl.get();
      	//如果线程池状态变成非运行状态并且从等待队列中成功移除，那么就会调用饱和策略，默认丢弃任务，抛出异常。
        if (! isRunning(recheck) && remove(command))
            reject(command);
      	//如果当前工作线程数为0，那么再尝试增加一个新线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
	//走到这儿，表示出现了情况2和情况3
  	//情况2下，在工作队列已满的情况下，如果工作线程数小于maximumPoolSize,尝试添加一个Worker，大于的话当然就添加失败了
  	//情况3下，同样尝试去添加一个新的Worker
    else if (!addWorker(command, false))
      	//添加新Worker失败，执行配置的饱和策略，默认是丢弃
        reject(command);
}
```

###### 4.2:addWorker方法

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        //如果状态 > SHUTDOWN 直接返回false，此时不执行新任务，也不执行阻塞队列的任务
      	//但是 在 = SHUTDOWN的状态，不会处理新任务，还是可以继续执行在阻塞队列的任务
      	//如果状态 = SHUTDOWN 而且 firstTask 不为 null 直接返回false
      	//如果状态 = SHUTDOWN 而且 firstTask 为 null 而且阻塞队列为空 直接返回false
      	//综上所述就是如果状态>SHUTDOWN,则不会处理当前任务，返回false，当=SHUTDOWN时，如果任务不为null也不会处理，返回false，或者是阻塞队列已经被执行完成了，也是不会处理当前任务的，同样返回false
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
         //第二个循环开始获取当前工作线程数ws，如果ws大于限制的数量，直接return false，创建worker不成功。
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
          //此处采用了经典的for循环CAS方式去添加新的工作线程数，如果成功，直接跳出第一个循环
          //但是存在这么一种情况：线程池状态在这段时间可能会发生改变，如果cas成功，因为操作的是整个ctl的值
          //那么肯定能够确保线程池状态没有发生变化，所以不用担心CAS成功，线程池状态改变这种情况，是不存在的
          //如果CAS失败，再次获取ctl的值，如果当前线程池状态不等于rs，即线程池状态变了，需要重新开始第一个循环体，重新获取线程池状态，继续如上操作。
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

  //整个上面的循环体代码就是为了校验线程池状态是否可以添加新线程，如果可以，CAS操作ctl+1，否则，直接return false。
  
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
      	//新建一个Worker对象，并拿到Worker对应的线程t，这个线程是ThreadFactory创建的
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
   				//重新获取线程池的状态
                int rs = runStateOf(ctl.get());
				//如果线程池处于运行状态或者shutdown状态并且没有任务，则创建一个新线程，
              	//按道理来说shutdown状态是不创建新线程的，但如果提交的任务为null，那么就会创建一个线程，只是不执行任务。
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                      //如果该线程已经存活，则抛出异常，肯定不正常，线程还是刚创建，压根没启动
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                      //largestPoolSize 表示线程池中存在的最大线程数的情况
                        largestPoolSize = s;
                  	//往工作线程中加入了新的worker成功
                    workerAdded = true;
                }
            } finally {
              // 锁被释放，其他的线程可以进入了
                mainLock.unlock();
            }
            if (workerAdded) {
              // 这里才是真正的线程启动的地方，由于Worker本身是继承Runnable
               // 所以这个start最终运行的也是Worker的run方法，也就是`runWorker`方法
                t.start();
                workerStarted = true;
            }
        }
    } finally {
      //没有正常添加线程到worker中或者线程启动失败
        if (! workerStarted)
          //从线程Set删除操作，ctl的线程数-1,并且尝试将线程池转换到terminated状态，后续会分析
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

###### 4.3的runWorker方法

在addWorker新增一个线程后，其实调用了Thread.start()方法，因为Worker实现了Runnable接口，所以最终会调用runWorker方法。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
  	//解绑worker和当前任务的关系
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
          	//死循环，一直去主动获取任务(包括去工作队列里面取，getTask就是 )，直到取不到为止。
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
              	//此处可以重写这个方法，增加一些对线程和任务的监控
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                  	//真实的执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                  	//此处可以重写次方法，在执行任务完成以后，增加监控
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

###### 4.4 getTask方法

这个方法用来在执行任务之前，死循环工作从队列中获取可用的任务。

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
      	//死循环去获取当前线程池的状态
        int c = ctl.get();
        int rs = runStateOf(c);

        // 如果状态是非运行态，并且是shutdown状态，而且队列是空的，那么表示没有任务了，工作线程数 -1
      	// 如果状态是非运行态，并且是shutdown状态，队列不为空，那么表示队列还有任务，需要执行完。
      	//如果状态是非运行态，并且不是shutdown状态，那么即使队列不为空，也不会继续执行了，工作线程数-1
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }
		//校验完前置条件以后，获取当前工作线程数量
        int wc = workerCountOf(c);

        // 上文的allowCoreThreadTimeOut属性在此用，设置完后，如果为true，或者当前工作线程数大于corePoolSize
      	
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		//此处又要校验：
      	//如果工作线程数大于maximumPoolSize，非正常状态，尝试CAS工作线程-1，如果失败，表示ctl变了，重新开始循环，重新获取ctl。
      	//如果timed && timedOut为true，即从队列中获取任务超时，只要存在工作线程或者队列空了，尝试CAS工作线程-1，如果失败，表示ctl变了，重新开始循环，重新获取ctl。
      	//如果都不属于上面的情况，继续。
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
		
        try {
          	//如果设置了allowCoreThreadTimeOut或者工作线程数大于核心线程数
          	//就会从阻塞队列中超时读取，没达到超时限制，会一直阻塞等待新任务到keepAliveTime
          	//如果超时还未拿到新任务，则返回null。
          	//如果timed为false，会一直阻塞，直到获取到新任务。
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
          //如果r不为空，表示拿到任务，返回，如果为空，要么队列没任务，要么获取超时了，timeOut设置为true。
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
          	//如果线程中断了，则恢复timeout为false，重新开始循环，重试。
            timedOut = false;
        }
    }
}
```

###### 4.5 processWorkerExit方法

该方法在runWorker方法中finally块中被调用，用来对执行完任务的Worker进行善后操作。

入参

* Worker w ：当前工作的Worker
* boolean completedAruptly : 如果为false，表示当前Worker已经取不到新任务了，如果为true，则不是拿不到任务，而是任务执行期间出现异常情况。

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
  	//如果为true，表示任务执行期间出现异常，那么工作线程数量使用CAS方式操作，减1
  	//如果为false，表示执行正常，只是队列中取不到任务了，那么工作线程数量不处理。
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();
	//此处加锁
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
      	//把Worker w完成的任务数量添加到总数量上，此处加锁，并把w从线程池里的工作线程集合Set中移除。
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
	//尝试把线程池状态改为terminate状态。
    tryTerminate();

    int c = ctl.get();
  	// 线程池状态位RUNNING 或者 SHUTDOWN 
    if (runStateLessThan(c, STOP)) {
      	// 如果当前worker是由于没有有效task而退出的
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
      	//如果执行任务异常中断，新加一个线程
      	//如果执行任务线程没有非意外退出，工作线程数小于corePoolSize，也尝试新增一个线程。
        addWorker(null, false);
    }
}
```

###### 4.6 tryTerminate方法

在线程池为SHUTDOWN状态，并且任务队列为空的时候，或者STOP状态，线程池空的情况下，将线程池状态改为TERMINATED状态。

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
      	//1：如果线程池还处于RUNNING状态，不会修改为TERMINATED，直接返回
      	//2：如果已经是TIDYING或者TERMINATED状态，不会重新执行，直接返回
      	//3：如果是SHUTDOWN状态，但是队列中还有任务没执行完，也不会修改为TERMINATED，直接返回
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
      	//以上情况都不符合，继续
      	//1：SHUTDOWN状态，队列为空
      	//2：STOP状态
      	//此时如果工作线程数不为0，那么中断一个空闲的线程。
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
		
      	//能走到这儿的有2种情况
      	//1:SHUTDOWN状态，工作线程为0
      	//2:STOP状态，工作线程为0
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
          	//尝试更改线程池为TIDYING状态，如果失败，继续循环
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                  	//如果成功，此方法为空方法，子类可以重写
                    terminated();
                } finally {
                  	//最后更新线程池状态为TERMINATED
                    ctl.set(ctlOf(TERMINATED, 0));
                  	//给termination condition上的其他线程发消息，告知线程池已为TERMINATED，唤醒阻塞再上面的线程。
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

###### 4.7 interruptIdleWorkers方法

参数：

* boolean one :为true，只中断线程池中最多1个空闲的工作线程。如果为false，则中断线程池中所有空闲的线程。

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
      	//循环线程池的工作线程集合
        for (Worker w : workers) {
            Thread t = w.thread;
          	//如果工作线程没有被中断，并且tryLock成功(能成功表示竞争到了AQS的state，表示当前线程是空闲的)
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                  	//执行中断
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
          	//如果为true，只中断最多一个空闲线程，然后返回
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

###### 4.8 shutDown方法

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
      	//1:获取线程池的锁，然后调用checkShutdownAccess方法检查每一个线程池的线程是否有可以ShutDown的权限。
        checkShutdownAccess();
      	//2:通过自旋将线程池状态更新为SHUTDOWN状态
        advanceRunState(SHUTDOWN);
      	//3:中断所有的空闲线程
        interruptIdleWorkers();
      	//4:子类实现
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
  	//完成以上操作，尝试更新线程池为TERMINATED
  	//这儿可以说明，即使调用了shutdown方法，如果队列中还有任务，即还有非空闲线程，那么非空闲线程不会被中断
  	//所以队列的任务可以继续执行，直到完成。然后通过 processWorkerExit方法再次调用tryTerminate
  	//最终更新线程池状态为TERMINATED状态。
    tryTerminate();
}
```

###### 4.9 shutDowNow方法

该方法会中断所有存活的线程，不管是不是空闲的还是正在执行任务的，队列中的任务也不再执行，并且返回队列中未执行的任务。

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
      	//1:获取线程池的锁，然后调用checkShutdownAccess方法检查每一个线程池的线程是否有可以ShutDown的权限。
        checkShutdownAccess();
      	//2:自旋更新状态为STOP状态
        advanceRunState(STOP);
      	//3:中断线程池中所有存活的线程
        interruptWorkers();
      	//4:返回任务队列中未执行的任务
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
  	//尝试将线程池状态更新为TERMINATED
    tryTerminate();
    return tasks;
}
```