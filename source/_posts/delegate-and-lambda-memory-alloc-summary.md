---
title: C#委托与匿名方法内存分配总结
date: 2021-11-13 23:50:38
tags: 
  - C#
categories: 
  - 程序
  - 性能优化
description: 关于在C#中使用委托与匿名方法的各种情况下内存分配的总结
---

# 前言

使用委托与匿名方法在C#编程中是一件很常见的事

但在使用不当的情况下会造成大量的额外内存分配，因此笔者便试图以分类讨论的形式，总结以不同的方式去使用委托与匿名方法会造成的内存分配情况



# 委托

考虑一个在循环中对委托赋值的场景：

```csharp
    public void Test()
    {
        for (int i = 0; i < 100; i++)
        {
            Action act = ???
        }
    }
```



有3种可能的赋值方式：

1. 使用Action对象赋值
2. 使用方法名赋值
3. 使用匿名方法赋值



## 使用Action对象赋值

通过将一个方法赋值给一个Action对象a，然后再将a赋值给act

```csharp
    public void Test()
    {
        Action a = ActionMethod;
        for (int i = 0; i < 100; i++)
        {
            Action act = a;
        }
    }

    public void ActionMethod()
    {

    }
```



然后通过SharpLab查看编译后的C#代码：

```csharp
	public void Test()
    {
        Action action = new Action(ActionMethod);
        int num = 0;
        while (num < 100)
        {
            Action action2 = action;
            num++;
        }
    }

    public void ActionMethod()
    {
    }
```

可以看到本质上是通过调用Action的构造方法，使用ActionMethod构造了一个Action对象然后在循环进行赋值

**整个过程产生了1次内存分配**



## 使用方法名赋值

直接用ActionMethod赋值给act

```csharp
	public void Test()
    {
        for (int i = 0; i < 100; i++)
        {
            Action act = ActionMethod;
        }
    }

    public void ActionMethod()
    {

    }
```



然后通过SharpLab查看编译后的C#代码：

```csharp
	public void Test()
    {
        int num = 0;
        while (num < 100)
        {
            Action action = new Action(ActionMethod);
            num++;
        }
    }

    public void ActionMethod()
    {
    }
```

在循环中每一次赋值时都构造了新的Action对象

**整个过程产生了100次内存分配**



## 使用匿名方法赋值

通过使用Lambda表达式产生出一个匿名方法赋值给act

```csharp
    public void Test()
    {
        for (int i = 0; i < 100; i++)
        {
            Action act = () => { };
        }
    }
```



然后通过SharpLab查看编译后的C#代码：

```csharp
	[Serializable]
    [CompilerGenerated]
    private sealed class <>c
    {
        public static readonly <>c <>9 = new <>c();

        public static Action <>9__0_0;

        internal void <Test>b__0_0()
        {
        }
    }

    public void Test()
    {
        int num = 0;
        while (num < 100)
        {
            Action action = <>c.<>9__0_0 ?? (<>c.<>9__0_0 = new Action(<>c.<>9.<Test>b__0_0));
            num++;
        }
    }
```

编译器会生成一个匿名类，将匿名方法保存到其中的静态Action字段中

**整个过程产生了1次内存分配**



## 总结

因此在对于需要重复赋值Action变量的情况

**Action对象赋值(1)=匿名方法赋值(1)>方法名赋值(100)**



# 匿名方法

以上的讨论中涉及的匿名方法没有捕获到外部变量（无闭包）

但一般在使用匿名方法时都会对外部变量进行捕获（有闭包）

而被捕获的外部变量有3种可能的情况：

1. 捕获到静态字段
2. 捕获到实例字段
3. 捕获到外部方法的局部变量



## 捕获到静态字段

在匿名方法中声明一个int变量x，去捕获静态字段staticInt

```csharp
public class TestClass
{
    private static int staticInt;

    public void Test()
    {
        for (int i = 0; i < 100; i++)
        {
            Action act = () => {
                int x = staticInt;
            };
        }
    }

}

```



然后通过SharpLab查看编译后的C#代码：

```csharp
public class TestClass
{
    [Serializable]
    [CompilerGenerated]
    private sealed class <>c
    {
        public static readonly <>c <>9 = new <>c();

        public static Action <>9__1_0;

        internal void <Test>b__1_0()
        {
            int staticInt = TestClass.staticInt;
        }
    }

    private static int staticInt;

    public void Test()
    {
        int num = 0;
        while (num < 100)
        {
            Action action = <>c.<>9__1_0 ?? (<>c.<>9__1_0 = new Action(<>c.<>9.<Test>b__1_0));
            num++;
        }
    }
}
```



当捕获到的是静态字段时，编译器的处理就和无闭包时差不多，会直接在匿名方法中去访问静态字段

**整个过程产生了1次内存分配**



## 捕获到实例字段

在匿名方法中声明一个int变量x，去捕获实例字段instanceInt

```csharp
public class TestClass
{
    private  int instanceInt;

    public void Test()
    {
        for (int i = 0; i < 100; i++)
        {
            Action act = () => {
                int x = instanceInt;
            };
        }
    }

}
```

然后通过SharpLab查看编译后的C#代码：

```csharp
public class TestClass
{
    private int instanceInt;

    public void Test()
    {
        int num = 0;
        while (num < 100)
        {
            Action action = new Action(<Test>b__1_0);
            num++;
        }
    }

    [CompilerGenerated]
    private void <Test>b__1_0()
    {
        int num = instanceInt;
    }
}
```

捕获的是实例字段时，编译器将不会生成匿名类，而是直接用匿名方法构造临时Action

**整个过程产生了100次内存分配**



## 捕获到外部方法的局部变量

在匿名方法中声明一个int变量x，去捕获循环中声明的局部变量instanceInt

```csharp
public class TestClass
{


    public void Test()
    {
        for (int i = 0; i < 100; i++)
        {
            int localInt = 0;
            Action act = () => {
                int x = localInt;
            };
        }
    }

}

```

然后通过SharpLab查看编译后的C#代码：

```csharp
public class TestClass
{
    [CompilerGenerated]
    private sealed class <>c__DisplayClass0_0
    {
        public int localInt;

        internal void <Test>b__0()
        {
            int num = localInt;
        }
    }

    public void Test()
    {
        int num = 0;
        while (num < 100)
        {
            <>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
            <>c__DisplayClass0_.localInt = 0;
            Action action = new Action(<>c__DisplayClass0_.<Test>b__0);
            num++;
        }
    }
}
```

编译器会为每一次捕获的局部变量创建匿名类对象来保存该局部变量，然后使用匿名方法去创建Action对象并赋值给act

**整个过程产生了200次内存分配**



## 总结

在使用匿名方法时

**无闭包(1)=捕获静态字段(1)>捕获实例字段(100)>捕获局部变量(200)**



## 优化建议

但在实际开发中使用匿名方法捕获循环中的局部变量可以说是非常常见的情况了，那么应该如何优化？



笔者给出的建议是：**通过增加参数数量，使用方法参数去传递要捕获的变量，避免掉对外部变量的捕获**



原本的写法，直接捕获局部变量：

```csharp
public class TestClass
{
    public void Test()
    {
        
        for (int i = 0; i < 100; i++)
        {
            int localInt = 0;
            CallAction(() =>
            {
                int x = localInt;
            });
        }
    }

    public void CallAction(Action action)
    {
        Action act = action;
        act();
    }
}
```



优化的写法，将局部变量作为方法参数传递：

```csharp
public class TestClass
{
    public void Test()
    {
       
        for (int i = 0; i < 100; i++)
        {
            int localInt = 0;
            CallAction(localInt,(param) =>
            {
                int x = param;
            });
        }
    }

    public void CallAction(int param, Action<int> action)
    {
        Action<int> act = action;
        act(param);
    }
}
```



然后通过SharpLab查看优化的写法编译后的C#代码：

```csharp
public class TestClass
{
    [Serializable]
    [CompilerGenerated]
    private sealed class <>c
    {
        public static readonly <>c <>9 = new <>c();

        public static Action<int> <>9__0_0;

        internal void <Test>b__0_0(int param)
        {
        }
    }

    public void Test()
    {
        int num = 0;
        while (num < 100)
        {
            int param = 0;
            CallAction(param, <>c.<>9__0_0 ?? (<>c.<>9__0_0 = new Action<int>(<>c.<>9.<Test>b__0_0)));
            num++;
        }
    }

    public void CallAction(int param, Action<int> action)
    {
        action(param);
    }
}
```

可以看到优化后就和无闭包一样了

**整个过程产生的内存分配从200次降低到了1次**

