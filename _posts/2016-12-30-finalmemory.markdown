---
layout:     post
title:      "Java中final关键字的内存语义"
subtitle:   " \"理解final关键字在并发中的用法 \""
date:       2016-12-30 00:00:00
author:     "Jihaixiao"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Java
    - 并发编程
---

> “Yeah It's on. ”

## 前言

final关键字修饰的域是不可变的，即在初始化后不能被程序改变，当然，如果final修饰的是对象的引用，不可变是指该对象的引用不可变，这个对象的内部还是可以被修改的。

## 正文

###  1：final域的重排序规则

对于final域，编译器和处理器要遵循如下重排序规则：

* 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量是不可重排序的。
* 初次读一个包含final域的对象的引用，与随后初次读这个final域是不可重排序的。

用一个例子来说明final域的读写重排序问题：

```java
public class FinalExample {
    int i;              //普通变量
    final int j;        //final变量

    static FinalExample obj;

    public FinalExample() {
        i=1;                    //构造函数内写普通域
        j=2;                    //构造函数内写final域
    }

    public static void writer(){   //写线程A执行，初始化FinalExample，并把它的引用赋值给obj
        obj = new FinalExample();
    }

    public static void reader(){   //读线程B执行
        FinalExample object = obj;  //读对象引用
        int a = object.i;           //读普通域
        int b = object.j;           //读final域
    }

}
```

### 2:写final域的重排序规则

写final域的重排序规则禁止把final的写重排序到构造函数之外，这个规则的实现包括两个方面：

* JMM禁止编译器把final域的写重排序到构造函数之外。
* 编译器会在final域写之后，构造函数return之前，插入一个StoreStore屏障，这个屏障禁止了处理器把final域的写重排序到构造函数之外了。（StoreStore屏障确保屏障前的store操作先于屏障后的，对后的可见，放在return之前可以确保对final域的写肯定是在构造函数结束之前发生的。）

上面例子的write()方法会有两个步骤：

* 构造一个FinalExample对象
* 把这个对象的引用赋值给obj

假设线程B对对象引用与读对象的成员域没有重排序，下图是一种可能的执行顺序，分析下：

![图1](http://ogu2tysfa.bkt.clouddn.com/111.jpg)

上图中因为StoreStore屏障只限制写final域肯定不会被重排序到构造函数外，如果普通域i=1被重排序到构造函数之外了，那么读线程B读i的时候就有可能读到未初始化的值，而读线程B能读到正确的final域的值。

写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被初始化了，而普通域没这个保证，上图中读线程B看到对象引用的时候，该对象很可能还没构造完成。

### 3:读final域的重排序规则

读final域的重排序规则：

在一个线程中，初次读对象的引用与初次读该对象包含的final域，JMM是禁止处理器对它们重排序的（仅针对处理器），编译器会在读final域前面插入一个LoadLoad屏障，确保对对象引用的读先于对final域的读。

初次读对象的引用与读该对象的域的值，这俩存在间接依赖关系，因为编译器会遵守这种关系，所以编译器不会重排序这俩操作，大多数处理器也会遵守，但是有小部分处理器对存在间接依赖关系的操作允许重排序，这个规则就是用来针对这些处理器的。

上面例子的reader()方法包括3个操作：

* 初次读引用变量obj
* 初次读引用变量obj所指对象的普通域i
* 初次读引用变量obj所指对象的final域j

下图是一种可能的执行顺序，来分析下：

![图片2](http://ogu2tysfa.bkt.clouddn.com/22.png)



上图中，读普通域操作被重排序到读对象引用之前，读的时候，还没被初始化，发生错误的读。

而读final域的重排序规则把读final域限制在读对象引用之后，此时final域已被初始化，可以读到正确的值。

读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用。



> 未完待续....