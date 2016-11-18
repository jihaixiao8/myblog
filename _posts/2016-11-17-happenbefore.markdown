---
layout:     post
title:      "JMM中的happen before"
subtitle:   " \"Hello World, Hello Blog\""
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



JMM对于这两种重排序，采用了不通的处理策略：

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

