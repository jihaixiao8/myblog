---
layout:     post
title:      "JMM中的Happen before"
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

JMM(Java Memory Model）对于理解并发编程有很重要的帮助。

[跳过废话，直接看技术实现 ](#build) 



JMM在程序猿的需求跟编译器，处理器之间的需求做了一个比较折中的选择。

* 程序猿想让JMM能够提供一个强一致内存模型，这样编程越方便。

* 编译器，处理器想让JMM对自己的限制越少越好，这样它们就能尽可能的用各种手段去优化代码，希望JMM提供一个弱一致模型。

  ​


<p id = "build"></p>
---

## 正文

JMM java内存模型分为工作内存和主内存。

每个线程有自己工作内存，

—— Hux 后记于 2015.10


