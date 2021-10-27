---
title: ILRuntime类型系统概述
date: 2021-10-25 01:08:51
tags: ILRuntime
categories: 程序
description: 关于ILRuntime的反射与泛型中类型系统的简要概述
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/ILRuntimeTypeSystem/ilruntime-type-sysytem.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/ILRuntimeTypeSystem/ilruntime-type-sysytem.png
---



# 前言

笔者在前段时间为[CatJson](https://github.com/CatImmortal/CatJson)进行ILRuntime适配时，因为涉及到反射与泛型的跨域使用，故而进行了一番对ILRuntime类型系统的研究。

本文将从三个方面对ILRuntime类型系统进行讨论：

1. 热更层往主工程传递实例
2. 热更层调用主工程泛型方法
3. 热更层往主工程传递Type对象



# 热更层往主工程传递实例

```C#
/// <summary>
/// 主工程类型
/// </summary>
public class MainClass { 
    
}

/// <summary>
/// 主工程工具类
/// </summary>
public static class Util
{
    public static void Test(object obj)
    {
        //打印出obj的类型
        Debug.Log(obj.GetType());

    }
}

/// <summary>
/// 热更层类型
/// </summary>
public class HotfixClass
{

}
```

```C#
//热更层调用
MainClass mc = new MainClass();
HotfixClass hc = new HotfixClass();

Util.Test(mc);  //输出MainClass
Util.Test(hc);  //输出ILTypeInstance
```

在从热更层往主工程传递对象时

- 如果此对象类型定义在主工程，那么就是通常情况，obj.GetType将得到一个正确的Type对象
- 如果此对象类型定义在热更层，那么此对象就会统一处理为ILTypeInstance类型的对象，ILTypeInstance包装了热更层类型实例对象



# 热更层调用主工程泛型方法

```C#
/// <summary>
/// 主工程工具类
/// </summary>
public static class Util
{
    public static void Test<T>()
    {
        //打印出T的类型
        Debug.Log(typeof(T));

    }
}
```

```c#
//热更层调用
Util.Test<MainClass>();  //输出MainClass
Util.Test<HotfixClass>();  //输出ILTypeInstance
```

在热更层调用主工程泛型方法的情况与传递对象实例类似

- 如果此泛型类型定义在主工程，那么就是通常情况
- 如果此泛型类型定义在热更层，那么该泛型就会统一变成ILTypeInstance

# 热更层往主工程传递Type对象

先对ILRuntime类型系统主要涉及到的类进行说明：

- System.Type：Type基类
- System.RuntimeType：一般情况下，使用typeof和GetType得到的Type对象都是此类型
- ILRuntimeType：主要包装了ILType
- ILRuntimeWrapperType：主要包装了CLRType
- IType：热更层Type接口
- ILType：热更层类型的Type
- CLRType：热更层中的主工程类型的Type



![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/ILRuntimeTypeSystem/ilruntime-type-sysytem.png)



```c#
/// <summary>
/// 主工程工具类
/// </summary>
public static class Util
{
    public static void Test(Type type)
    {
        //打印出Type的类型
        Debug.Log(type.GetType());
    }
}
```

```c#
//热更层调用
MainClass mc = new MainClass();
HotfixClass hc = new HotfixClass();

Util.Test(mc.GetType());  //输出System.RuntimeType
Util.Test(hc.GetType());  //输出ILRuntime.Reflection.ILRuntimeType
```

- 如果此Type是主工程类型的Type，那么就是通常情况

- 如果此Type是热更层类型的Type，那么此Type的类型就是ILRuntimeType

  

## 对于ILRuntimeType的说明

在反射编程中，往往需要使用得到的Type对象进行一系列反射操作来读取类型元数据，比如获取字段信息、获取属性信息、获取方法信息等

ILRuntimeType都对相关操作进行了对应的重写，以返回ILRuntimeFieldInfo、ILRuntimePropertyInfo、ILRuntimeMethodInfo

```c#
public class ILRuntimeType : Type
{
    //...
    ILRuntimeFieldInfo[] fields;
    ILRuntimePropertyInfo[] properties;
    ILRuntimeMethodInfo[] methods;
    //...
}
```



需要注意的是，这些ILRuntime****Info所提供的Type是与通常情况不同的，以ILRuntimeFieldInfo的FieldType获取到的结果为例：

- 如果此字段是热更层类型，那么Type就是ILRuntimeType
- 如果此字段是主工程类型，那么Type就是ILRuntimeWrapperType



## 对于ILRuntimeWrapperType的说明

ILRuntimeWrapperType包装了CLRType，CLRType表示的是主工程类型的Type



ILRuntimeWrapperTypeObj.RealType可以获取到主工程类型对应的 System.RuntimeType对象，这和ILRuntimeWrapperTypeObj.CLRType.TypeForCLR等价



如果ILRuntimeWrapperType包装的是如List<T>这样的主工程泛型类，可以通过ILRuntimeWrapperTypeObj.CLRType.GenericArguments[0].Value.ReflectionType获取到T的Type对象，它可能是ILRuntimeType，也可能是ILRuntimeWrapperType，具体根据T的定义位置决定
