---
layout: post
title:  JVM中对象的创建过程
categories: [JVM, Java]
tags:  [JVM, Java]
keywords: JVM,对象,创建过程
---


在使用java开发程序的时候，用new就可以创建出一个对象。在这个创建对象的过程中，JVM做了不少的工作，流程大体如下：




<!-- ```flow
st=>start: Start
e=>end: End
op1=>operation: new 指令
sub1=>subroutine: 执行类加载
cond=>condition: 定位类引用,是否被加载?
io=>operation: 分配内存并初始化零值
set=>operation: 设置对象
set=>operation: 按java代码进行初始化
st->op1->cond
cond(yes)->io->set->e
cond(no)->sub1(right)->io
``` -->

![flow](https://raw.githubusercontent.com/ethendev/data/master/silo/img/jvm/jvm_object_create_flow.png)


### 定位符号引用
首先，JVM接到new指令时，将会检查这个指令的参数能否在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的的类是否已被加载、解析和初始化过。如果没有就执行加载过程。


### 分配内存
在HotSpot虚拟机中，对象在内存中的存储的布局可分为3部分：对象头、示例数据和对齐填充。其中，对象头包括两部分，一部分是存储对象运行时数据，如HashCode、GC分带年龄、锁状态标志等。对象头另一部分是类型指针，即对象指向它的元数据类型的指针，JVM通过这个指针来确定这个对象是哪个类的实例。但并不是所有的JVM都需要保留类型指针。

在完成上述的操作后，JVM需要对对象进行必要的设置，如设置对象头等。这个时候，从JVM的视角来说，一个新的对象已经产生了，但从java程序的视角来说，对象的创建才刚刚开始。接着会把对象按照程序员的意愿进行初始化，这样一个可用的对象才算完全产生。


### 设置
在HotSpot虚拟机中，对象在内存中的存储的布局可分为3部分：对象头、示例数据和对齐填充。其中，对象头包括两部分，一部分是存储对象运行时数据，如HashCode、GC分带年龄、锁状态标志等。对象头另一部分是类型指针，即对象指向它的元数据类型的指针，JVM通过这个指针来确定这个对象是哪个类的实例。但并不是所有的JVM都需要保留类型指针。

在完成上述的操作后，JVM需要对对象进行必要的设置，如设置对象头等。这个时候，从JVM的视角来说，一个新的对象已经产生了，但从java程序的视角来说，对象的创建才刚刚开始。接着会把对象按照程序员的意愿进行初始化，这样一个可用的对象才算完全产生。

