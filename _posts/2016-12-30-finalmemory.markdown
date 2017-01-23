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

### 4:final域是引用类型

```java
public class FinalReferenceExample {

    final int[] intArray;               //final是引用类型

    static FinalReferenceExample obj;

    public FinalReferenceExample() {    //构造函数
        intArray = new int[1];          //操作1
        intArray[0]=1;                  //操作2
    }

    public static void writeOne(){              //写线程A执行
        obj = new FinalReferenceExample();      //3
    }

    public static void writeTwo(){              //写线程B执行
        obj.intArray[0] = 2;                    //4
    }

    public static void reader(){                //读线程C执行
        if (obj != null){                       //5
            int temp1 = obj.intArray[0];        //6
        }
    }

}
```

上面的例子：

​	intArray是引用类型的，它引用一个int类型的数组引用，对于引用类型，final域的写重排序对编译器和处理器有如下约束：

* 在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造的对象的引用赋给一个引用变量，这两个操作不能重排序。

如下图是一个可能的执行顺序，来分析下：

![图片3](http://ogu2tysfa.bkt.clouddn.com/33.png)

上面的操作1是对final域的写，2是对final的引用的对象的成员域的写，3是把被构造对象的引用赋给引用变量。

前面说过1跟3不能重排序，2和3也不能重排序。（个人认为1和2应该是可以重排序的，对程序执行结果没有影响）

JMM可以确保线程C可以正确的读到线程A在构造函数内对final引用的对象的域的写入，即能看到数组下标为0的值是1，而线程B对数组元素的写入，线程C可能能读到，也可能读不到。因为JMM不保证线程B的写入对线程C可见，可以用volatile或者Lock解决，来确保内存可见性。

### 5:final引用不能从构造函数中溢出

前面提到：写final域的重排序规则可以保证，在引用变量为所有线程可见之前，该引用变量所指向的对象的final域一定都已经在构造函数中正确初始化了。其实要得到这个效果，还有一点必须保证：在构造函数内部，不能让被构造对象的引用为其他线程可见，也就是说对象引用不能再构造函数中溢出。看如下代码：

```java
public class FinalReferenceEscapeExample {

    final int i;

    static FinalReferenceEscapeExample obj;

    public FinalReferenceEscapeExample(){
        i=1;                            //1 写final域
        obj = this;                     //2 this引用在此“逃逸”
    }

    public static void writer() {
        new FinalReferenceEscapeExample();
    }

    public static void reader(){
        if (obj != null) {              //3
            int temp = obj.i;           //4
        }
    }

}
```

如果A线程调用writer方法，此时调用构造函数，2操作其实就让被构造对象的引用逃逸了，而且，1和2其实是可以重排序的，所以如果1,2重排序，此时B线程调用reader方法，发现obj不是空，然后去读final域 i的值，但是很可能读到的i的值并未初始化。如下图：

![](http://ogu2tysfa.bkt.clouddn.com/44.png)

### 6:final语义在处理器的实现

以X86处理器为例，说明final域在处理器中的实现。

上面说过，写final域的重排序规则要求在final域写之后，构造函数return之前，插入一个StoreStore屏障，读final域的重排序规则要求编译器在读final域的操作之前插入一个LoadLoad屏障。

由于X86不会对写-写进行重排序，所以在X86处理器中，写final域需要的StoreStore屏障会被省略掉，由于X86不会对存在间接依赖关系的操作做重排序，所以读final域的LoadLoad屏障也会被省略，所以X86不会对final域的读写插入任何屏障。

### 7:JSR-133为什么要增强final的语义

在旧的Java内存模型中，最严重的一个BUG就是线程可能会看到final域的值会改变，例如一个线程当前看到的一个final int的值是0（还未被初始化），过一段时间该线程再读的话final域的值可能变成1了（被某个线程初始化了），最常见的的例子就是在旧的JAVA内存模型中，String的值可能会改变（[例子][1]）

为了修补这个漏洞，JSR-133专家组增强了final语义，通过final域增加写读重排序规则，可以为java程序猿提供初始化安全保证：只要对象是正确构造的（被构造对象的引用在构造过程中没有逃逸），那么不需要同步（lock和volatile），就可以保证任意线程都能看到这个final域在构造函数中被初始化的值。









[1]:https://www.cs.umd.edu/users/pugh/java/memoryModel/jsr-133-faq.html