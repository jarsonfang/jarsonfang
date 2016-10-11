---
title: 字符设备驱动原理图解
tags:
  - kernel
categories:
  - linux设备驱动内核机制
  - 设备驱动
id: 469
date: 2014-03-13 10:05:38
---

现实世界中存在着大量的设备，这些设备在电气特性和I/O方式上都各不相同。为了简化设备驱动程序员的工作，linux系统从这些各异的设备中抽象出共性的特征，将其划分为三大类：字符设备、块设备和网络设备。内核针对每一类设备都提供了对应的驱动模型框架，包括基本的内核设施和文件系统接口。这样，设备驱动程序员在编写某类设备驱动程序时，就有一套完整的驱动模型框架可以使用，从而可以将大量的精力放在设备本身的操作上。下图2-1展示了一个粗略的linux设备驱动程序结构图：
[![device_driver_framework](http://jarson.in/wp-content/uploads/2014/03/device_driver_framework-277x300.png)](http://jarson.in/wp-content/uploads/2014/03/device_driver_framework.png)
<!--more-->
以下内容转自 [《深入Linux设备驱动程序机制》学习心得---字符设备驱动原理图解](http://blog.chinaunix.net/uid-20543672-id-3203690.html) By Tekkaman Ninja

在第二章讲解字符设备的时候，个人觉得比较有收获的主要是两个方面的知识：
1、字符设备号的管理（char_device_struct）
2、字符设备驱动的file_operation中的函数如何与file结构体中的相应接口关联，并被应用程序调用。

对于以上两个主要的知识点，我觉得书上的条理已经很清楚的，很容易看懂，我在这里复述就多余了。我把学到的两个知识点用图的方式总结出来，供大家参考。

**字符设备号的管理**

重点在于内核在管理时所依赖的数据结构char_device_struct以及全局的散列表chrdevs。
还有就是要知道内核对于设备号的注册与注销和驱动功能的实现是没有必然的联系的。设备号的管理是一个独立的机制，以避免驱动在使用设备号的时候发生冲突，导致驱动中的file_operation对应错误，进而出现应用层操作错误的设备。因为应用层对设备的操作就是通过设备号对应到具体的file_operation的。

[![chr_dev](http://jarson.in/wp-content/uploads/2014/03/chr_dev-249x300.jpg)](http://jarson.in/wp-content/uploads/2014/03/chr_dev.jpg)

**字符设备驱动的file_operation中的函数如何与file结构体中的相应接口关联**

这部分的内容主要是要熟悉open函数的调用流程，驱动中的file_operation结构体就是在open函数中通过设备号与进程相关的file结构体中相应函数进行对应的。在完成了open操作之后，其他的文件操作就可以直接调用驱动file_operation中的函数了。
内核对于char_device_struct结构体的管理方式和设备号的cdev_map是类似的，都是通过主设备号作为哈希表的key来索引。

[![chr_dev2](http://jarson.in/wp-content/uploads/2014/03/chr_dev2-256x300.jpg)](http://jarson.in/wp-content/uploads/2014/03/chr_dev2.jpg)