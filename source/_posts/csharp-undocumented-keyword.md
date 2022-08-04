---
title: C#高级编程：未公开关键字研究
date: 2022-08-04 14:25:48
tags: 
  - C#
categories: 
  - 程序
description: 被IDE藏起来的禁忌技术
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpLogo.jpeg
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpLogo.jpeg
---

# 简介

C#语言中存在4个未被文档记录(**Undocumented**)的关键字，在VS、Rider这样的IDE中试图敲出它们时并不会给你任何智能提示，它们分别是：

1. **__makeref**
2. **__refvalue**
3. **__reftype**
4. **__arglist**

这4个关键字都涉及对IL代码栈数据的直接操作，并且都与`TypedReference`有关

# TypedReference

那么`TypedReference`又是什么东西呢？

简而言之，`TypedReference`就是一个包装了对象指针和对象类型的结构体

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_1.png)



# __makeref

此关键字用于使用指定对象创建引用此对象的`TypedReference`

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_2.png)

对应的IL代码则是直接使用`mkrefany`指令创建`TypedReference`对象

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_3.png)

# __refvalue

此关键字用于从`TypedReference`获取对象(类型必须一致或可转换)

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_4.png)

对应的IL代码则是直接使用`refanyval`指令获取对象

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_5.png)



因为`TypedReference`保存的是对象指针，所以可以通过对`__refvalue`赋值来修改**tf**所引用变量的值，以实现引用传递的效果

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_6.png)



# __reftype

此关键字用于从`TypedReference`中获取对象类型对应的Type

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_7.png)

对应的IL代码则是直接使用`refanytype`指令获取Type对象![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_8.png)



结合`__makeref`,`__reftype`,`__refvalue`即可实现无拆装箱开销的通用参数处理

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_9.png)



# __arglist

此关键字将不定数量的参数打包传递给方法

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_11.png)

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_10.png)

通过`__arglist`创建专门用于枚举变长参数列表的`ArgIterator`，即可对所有参数进行操作



对应的IL代码则是直接通过`vararg`将参数打包传入方法中

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CSharpUndocumentedKeyword/CSharpUndocumentedKeyword_12.png)

相比使用`params`关键字传递变长参数，`__arglist`不对类型有限制，不会产生额外的装箱开销，对应的IL指令也更少



# 注意事项

不过比较遗憾的是，**Unity IL2CPP**并不支持以上关键字与`TypedReference`
