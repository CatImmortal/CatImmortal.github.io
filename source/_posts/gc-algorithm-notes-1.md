---
title: 《垃圾回收的算法与实现》1.标记-清除法
date: 2022-03-13 13:36:42
tags: GC
categories: 
  - 程序
  - 学习笔记
  - 垃圾回收的算法与实现
description: 《垃圾回收的算法与实现》的读书笔记
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_Head.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_Head.png
---

# 前言

垃圾回收（Garbage Collection，后文简称GC）是一种程序运行时的自动内存管理技术

其作用主要在于防止因手动管理内存导致的诸如**内存泄露**，**悬垂指针**等问题的发生，减少程序员在开发时的心智负担

GC的基础算法有3种：

1. 标记-清除法
2. 引用计数法
3. 复制法

后世的诸多GC算法都是在这3种基础算法上衍生而来的



# 基本概念说明

## 对象/头/域

在GC的世界中，对象表示的是”数据的集合“，可理解为GC进行回收的基本单位，即GC管理的是一个个对象

对象由头(header)和域(field)组成

### 头

对象中保存对象本身信息的部分称为头，包括：对象大小，对象种类，此外还会根据GC算法的不同保存不同的额外信息（比如标记-清除法会在对象头部设置1个flag记录此对象是否已标记）

### 域

域可简单理解为对象中的字段，分为指针和非指针

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_1.png)

## 指针

GC根据指针去搜寻到其他对象，对非指针不进行任何操作

需要注意2点：

1. GC是否能判别指针和非指针
2. 指针指向对象的哪个部分。如果指向对象首地址以外的部分，GC就会变得非常复杂，因此在大多数语言中指针都默认指向对象首地址

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_2.png)

## 堆

堆(heap)指的是用于动态存放对象的内存空间，GC管理的即是堆中的对象，当堆被对象占满时GC就会启动，从而分配可用空间，如果GC后空间还是不够，就需要扩大堆

活动对象/非活动对象

活动对象指的是那些被引用着的对象，反之已经没被引用的对象就是非活动对象，即“垃圾”

GC会保留活动对象，销毁非活动对象

## 分块

分块(chunk)指事先准备的堆内存空间

初始状态下，堆被一个大的分块占据，程序会把这个分块分割成合适的大小，作为活动对象使用

活动对象不久后会变成垃圾被回收，此时这部分被回收的内存空间再次成为分块，为下次被利用做准备

内存里的各个分块都重复着分块→活动对象→垃圾→分块→......这样的过程

## 根

根(root)指的是指向对象的指针的起点部分

一般而言，静态变量，局部变量，寄存器都是根对象

## 评价标准

评价GC算法性能时，有以下4个标准

1. 吞吐量（堆内存大小/总GC耗时）
2. 最大暂停时间（单次GC最大的耗时）
3. 堆使用效率（对堆的额外占用与管理范围）
4. 访问的局部性（缓存命中率）

# 标记-清除法

## 整体流程

标记-清除法由标记阶段和清除阶段组成

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_3.png)



标记阶段从根从发，将所有活动对象做上标记

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_4.png)



清除阶段则会遍历堆，将没有标记到的那些非活动对象回收（放入空闲链表）

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_5.png)



因为需要遍历整个堆，所以标记-清除法花费的时间与堆大小成正比

### 分配

在清除阶段已经将所有非活动对象回收进空闲链表了，那么分配就是搜索空闲链表寻找大小合适的分块

搜索策略有以下3种：

1. First-fit（返回第一个大小等于目标大小的分块，若找到的分块大于目标大小，将其分割为目标大小和剩余大小）
2. Best-fit（返回大小等于目标大小的最小分块）
3. Worst-fit（找出空闲链表中最大的分块，将其分割为目标大小和剩余大小，目的是将分割后剩余的分块最大化，但是容易生成大量小分块）

### 合并

在进行数次的分配与回收后，可能会产生大量小分块，如果它们是连续的，就能把所有小分块连在一起形成大分块，这就是合并，合并是在清除阶段进行的

## 优点

### 简单

标记-清除法的实现简单，只有标记和清除两个阶段，容易与其他GC算法相结合

### 与保守式GC算法兼容

保守式GC算法对象是不能被移动的，因此与需要移动对象的复制法以及标记-压缩法不兼容，但标记-清除法不需要移动对象

## 缺点

### 碎片化

在标记-清除法的使用过程中会产生大量被细化的分块，不久就会导致无数小分块散步在堆的各处，这就是碎片化

如果发生碎片化，那么即使堆中分块总大小够用，也会因为分块都太小导致内存分配失败

为了避免碎片化，可以采用**内存压缩**，或者**BiBOP**法

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_6.png)

### 分配速度

因为分块不连续，所以每次分配都要遍历空闲链表，找到足够大的分块，最糟的情况就是每次分配都要遍历链表到最后一个节点

而在复制法和标记-压缩法中，分块是连续的内存空间，没必要遍历空闲链表，分配就能高速进行，而且能在堆范围内分配很大的对象

后面会提到的**多个空闲链表**和**BiBOP**法都是为了能在标记-清除法中高速分配而想出的办法

### 与写时复制技术不兼容

写时复制技术(copy on write)简单来说就是在复制数据时不真正的进行复制，而是假装复制了，实际上是用的共享数据，到了需要写入数据时才复制一份数据作为自己的数据然后进行写入操作

标记-清除法下，即使没有重写对象，也会设置所有活动对象的标志位，这样就会频繁发生本不应该发生的赋值，压迫到内存空间

为了处理这个问题，需要采用**位图标记**法(bitmap marking)

## 多个空闲链表

标记-清除法在只用一个空闲链表的情况下分配速度太慢，为了提高速度，需要根据分块大小创建多个空闲链表，比如创建只连接大分块的和只连接小分块的

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_7.png)

这里有一个问题就是到底创建多少个空闲链表比较好？

通常会给分块大小设定一个上限，超出此上限的统一扔进一个空闲链表里处理（虽然这样会导致对于非常大的分块的搜索效率很低，但因为分配这种大小的情况很罕见，所以也是可以接受的，更重要的是怎么去更快的搜索需要频繁分配的小分块）

## BiBOP法

BiBOP法(Big Bag Of Pages)用一句话概括就是“将大小相近的对象整理成固定大小的块进行管理的做法”

因为标记-清除法会产生碎片化，导致堆上杂乱散布着大小各异的对象，对此就可以使用此方法：把堆分割成固定大小的块，让每个块只能配置同样大小的分块

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_8.png)

这样配置对象能提高内存的使用效率，因为每个块中只能配置同样大小的对象，所以不可能出现大小不均的分块

但是BiBOP法并不能完全消除碎片化，比如在全部用于2个字的块中，只有1到2个活动对象，就不算是有效利用了堆

## 位图标记

用于标记的位是被分配到各个对象头中的，也就是说是把对象和头一并处理的，这与写时复制技术不兼容

对此，可以只收集各个对象的标志位并表格化，不与对象一起管理，在标记时不在对象头中置位，而是在表格中的特定场所置位，这样的表格就是“位图表格，利用这个表格进行标记的行为就是”位图标记“

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/GCAlgorithmNotes1/GCAlgorithmNotes1_9.png)

### 优点

与写时复制技术兼容

清除操作更高效（可以快速消去标志位）

### 要注意的地方

在进行位图标记时，必须注意对象地址和位图表格的对应，在堆有多个，对象地址不连续的时候，一般会为每个堆都准备一个位图表格

## 延迟消除法

在前面提到过，标记-清除法在清除操作所花费的时间和堆大小成正比，延迟清除法是在标记结束后，不立马进行清除，而是通过延迟防止主线程长时间暂停

那么什么时候进行清除呢？

在分配对象内存时会进行延迟清除，以找到一个适合的分块，如果此时找不到会进行一遍标记操作，然后再进行第二次延迟清除，这次如果再找不到适合的分块就意味着内存分配失败了

因为延迟清除法不是一下遍历整个堆，而只在分配时执行必要的遍历，所以可以压缩因清除操作导致的主线程暂停时间
