---
layout:     post
title:      "fork/join框架"
subtitle:   " \"jdk1.7提供的任务拆分框架\""
date:       2017-04-08 00:00:00
author:     "Jihaixiao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
    - 并发编程
---

> “Yeah It's on. ”

## 前言

JDK1.7后，提供了一个完善的任务并行框架—Fork/Join。

现实中有很多任务，如果单线程执行，效率很低，但是可以对大的任务拆分，拆分开来的小任务互不影响，多个线程并行执行，执行完合并结果，这么明显提高了效率。例如：有一张数据库的表，我们想把数据库这张表的数据抽出来，全量写到redis里，如果每次分页查一页数据，然后单线程写redis，效率一般。我们可以考虑用并行任务框架，每次查出一页数据，对这页数据拆分任务，并行执行的去写redis，这么执行起来会比以前最多快N倍。（N是拆分的粒度）

## 正文

#### 1.Fork/Join应用





