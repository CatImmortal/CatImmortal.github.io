---
title: 《垃圾回收的算法与实现》3.复制法
date: 2022-04-17 15:32:18
tags: GC
categories: 
  - 程序
  - 学习笔记
  - 垃圾回收的算法与实现
description: 《垃圾回收的算法与实现》的读书笔记
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_Head.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_Head.png
---

# 简介

GC复制法将堆空间分为两块空间（From和To），当From空间被全部占满时，会将其中的活动对象全部复制到To空间，然后回收From空间，复制完成后会将From和To空间互换

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_1.png)



复制法下的对象有2个额外的域成员

1. tag：用于标记此对象是否已被复制过
2. forwarding：用于指向此对象被复制后的新对象。方便将原来指向此的指针指到新对象上



# 执行过程

假设以下图的堆空间开始GC

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_2.png)



首先从根出发，将B复制到To空间，将复制后的对象称为B'，此时B的tag被勾选，forwarding也指向了B'

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_3.png)



接下来把B的子对象A也复制到To空间，这样才算真正意义上复制了B，因为A没有子对象，所以对A的复制也完成了

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_4.png)



然后复制对象G及其子对象E，虽然B也是G的子对象，但B已经复制过了，只需要将指向B的指针指到B'上就行

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_5.png)

最后互换From和To，GC就结束了



# 优点

## 优秀的吞吐量

标记-清除法消耗的吞吐量是搜索活动对象所花费的时间和搜索整体堆所花费的时间

而复制法只搜索并复制活动对象，所以相比标记-清除法能更快的完成GC，也就是说吞吐量优秀

## 可实现高速分配

复制法不使用空闲链表，因为To分块是一个连续的内存空间，可以直接进行分配，不需要遍历空闲链表然后找分块

## 不会发生碎片化

GC后活动对象都被集中分配在了From空间开头，像这样把对象重新集中，放在堆一段的行为叫做压缩，在复制法每次运行GC后都会执行压缩，因此不会发生碎片化

## 与缓存兼容

在复制法中有引用关系的对象都会被安排在堆里离彼此较接近的位置，使得在CPU读取对象时容易发生缓存命中

# 缺点

## 堆使用效率低下

复制法将堆二等分，通常只能利用其中的一半安排对象，相比能使用整个堆的GC算法而言，可以说这是复制法的一个重大缺陷

## 不兼容保守式GC算法

复制法必须移动对象重写指针，所以与要求不能移动对象的保守式GC不兼容

## 递归调用函数

在复制某个对象时要递归复制其子对象，容易产生栈溢出。为了解决这个问题有人提出了使用迭代进行复制的算法

# 多空间复制法

## 简介

复制法最大的缺点就是只能利用半个堆，为了解决这个问题可以把堆再作细分

例如：将堆分成10份，其中拿出2份分别作为From和To空间执行GC复制法，对剩下8份空间执行标记-清除法，也就是说将两种算法组合起来使用

## 执行过程

首先假设份数为4

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_6.png)



然后执行GC，对heap[0]和heap[1]执行复制法，对heap[2]和heap[3]执行标记-清除法

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_7.png)



并将To和From分别后移一个位置，设想再次没有了分块需要GC的情况

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_8.png)



此时再次执行GC

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes3/GCAlgorithmNotes3_9.png)

## 优点

多空间复制法相比普通复制法能更有效的利用堆空间

## 缺点

但因为对于其他空间使用了标记-清除法，因此就出现了标记-清除法固有的问题——分配耗时、分块碎片化等

