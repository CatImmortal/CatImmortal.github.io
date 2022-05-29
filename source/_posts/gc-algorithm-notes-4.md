---
title: 《垃圾回收的算法与实现》4.标记-压缩法
date: 2022-05-28 16:51:34
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

标记-压缩法是将标记-清除法和复制法结合起来的产物，由**标记阶段**和**压缩阶段**组成

- 标记阶段：此阶段与标记-清除法提到的标记阶段完全一样
- 压缩阶段：此阶段通过数次搜索堆来重新装填活动对象

# Lisp2算法

## 对象结构

Lisp2算法在对象头中定义了一个forwarding指针，其用法和复制法中的用法一致([《垃圾回收的算法与实现》3.复制法](http://cathole.top/2022/04/17/gc-algorithm-notes-3/))

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes4/GCAlgorithmNotes4_1.png)

## 执行过程

假设以下图的堆空间开始GC：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes4/GCAlgorithmNotes4_2.png)



首先是**标记阶段**，标记完成后堆空间如下图所示：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes4/GCAlgorithmNotes4_3.png)



**压缩阶段**结束后的堆空间：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes4/GCAlgorithmNotes4_4.png)

在Lisp2算法中，压缩阶段并不会改变对象的排列顺序，只是将它们变得更紧凑了



压缩阶段由以下3个步骤组成：

1. 设定forwarding指针
2. 更新指针
3. 移动对象



## 设定forwarding指针

在此步骤中，会搜索整个堆，为活动对象设定forwarding指针，将其指向

预计要移动到的地址

此步骤结束后的堆空间如下：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes4/GCAlgorithmNotes4_5.png)

## 更新指针

此步骤首先会更新根节点的指向，使所有root指向root.forwarding，然后会搜索整个堆，使所有指向活动对象的指针都改为指向此指针的forwarding

此步骤结束后的堆空间如下：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes4/GCAlgorithmNotes4_5.png)

## 移动对象

此步骤会将所有活动对象移动到forwarding指向的地址处（注意：此时已经是第3次搜索整个堆了）

此步骤结束后的堆空间如下：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes4/GCAlgorithmNotes4_6.png)

## 优点

### 可有效利用堆

标记-压缩法和其他GC算法相比，对堆空间的利用率更高，在防止出现标记-清除法可能导致的内存碎片的情况下，不会出现复制法那样只能利用一半堆空间的情况

## 缺点

### 压缩花费成本高

Lisp2算法的压缩中，必须对整个堆进行3次搜索，因此吞吐量要劣于其他算法
