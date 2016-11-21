---
layout:     post
title:      "JMM中的happen before"
subtitle:   " \"help to understand concurrency programming \""
date:       2016-11-17 12:00:00
author:     "Jihaixiao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
    - 并发编程
---

> “Yeah It's on. ”


## 前言

JMM(Java Memory Model）对于理解并发编程有很重要的帮助，而happen-before是JMM的核心。

[跳过废话，直接看技术实现 ](#build) 



JMM在程序猿的需求跟编译器，处理器之间的需求做了一个比较折中的选择。

* 程序猿想让JMM能够提供一个强一致内存模型，这样编程越方便。

* 编译器，处理器想让JMM对自己的限制越少越好，这样它们就能尽可能的用各种手段去优化代码，希望JMM提供一个弱一致模型。

  ​



<p id = "build"></p>
---

## 正文

JMM把happen-before要求禁止的重排序分为两类：

* 会改变程序结果的重排序。
* 不会改变程序执行结果的重排序。



JMM对于这两种重排序，采用了不同的处理策略：

* 对于会改变程序执行结果的重排序，JMM要求编译器和处理器必须禁止这种重排序。
* 对于不会改变程序执行结果的重排序，JMM对编译器和处理器不作要求，换句话说，就是它默许这种重排序。

例如下面这段代码：

```java
double a = 1;               //A
double b = 2;               //B
double c = a * b;           //C
```

上面的代码有3个happen-before关系：

* A happen-before B
* B happen-before C
* A happen-before C

入下图所示：

![](http://ogu2tysfa.bkt.clouddn.com/reorder4.jpg)



黄色区域是JMM的实现，程序猿只关注happen-before的规则，并不关心实现。这种实现为程序猿提供了足够的内存可见性，并且对编译器和处理器的约束尽可能的小。（编译器和处理器只要不改变程序执行结果，JMM不会限制它们的优化），但是有些JMM保证的内存可见性，并不一定真实存在，例如上面代码的A happen-before B。

### happen-before的定义

《JSR-133：Java Memory Model and Thread Specification》 对happen-before关系的定义如下：

1. 如果一个操作happen-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，并且第一个操作的执行顺序在第二个操作之前。
2. **如果两个操作存在happen-before关系，并不意味着java平台的具体实现必须要按照happen-before关系执行的顺序执行**。如果重排序之后的执行结果，与按happen-before关系执行的结果一致，那么这种重排序并不非法（也就是说，JMM允许这种重排序）。

上面的第一条是jmm对程序员的承诺，例如a happen-before b，内存模型将向程序员保证，a的操作结果对b可见，并且a的执行顺序排在b之前，当然这只是JMM对程序员的保证。

第二条是JMM对编译器和处理器重排序的约束规则。JMM一直在遵从一个原则：只要别改变程序的执行结果（单线程程序和正确同步的程序），编译器和处理器随便优化，JMM这么做是因为程序员只关心程序执行时语义不要被改变，对两个操作实际上是否重排序了不关心。

### happen-before原则

1. 程序顺序规则：一个线程中的每个操作，happen-before于被线程中的任意后续操作。
2. 监视器锁规则：对一个锁的解锁，happen-before于随后对这个锁的加锁。
3. volatile变量规则：对一个volatile变量的写，happen-before于任意后续对这个volatile的读。
4. 传递性：如果A happen-before B，且B happen-before C，那么A happen-before C。
5. start()规则：如果线程A 执行操作 threadB.start()，那么线程A的ThreadB.start()操作happen-before于线程B中的任意操作。
6. join()规则：如果线程A执行threadB.join()操作并成功返回，那么线程B中的任意操作happen-before于线程A从threadB.join()操作成功返回。

