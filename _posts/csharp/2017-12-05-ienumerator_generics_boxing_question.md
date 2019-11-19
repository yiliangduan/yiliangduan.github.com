---
title: 'C# 泛型集合的Boxing问题'
date: 2017-12-05 20:16:35
tags: csharp
comments: false
---


今天检查项目中代码的Boxing问题的时候。有一个点当时让我困惑了不少时间。如下:

```csharp
public IEnumerator<float> Func0()
{
  yield return 0.1f;
}

public IEnumerator Func1()
{
  yield return 0;
}
```

这里在实际代码运行过程中Func1会产生Boxing而Func0没有产生Boxing，按照自己浏览的C#的文档对Boxing的理解这里应该都是会产生Boxing的才对。下面是[C#的Boxing文档说明](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/boxing-and-unboxing)的：

>Boxing is the process of converting a value type to the type object or to any interface type implemented by this value type. When the CLR boxes a value type, it wraps the value inside a System.Object and stores it on the managed heap. 

查看了IEnumerator<T>的定义:

```csharp
public interface IEnumerator<T> : IEnumerator, IDisposable
{
  T Current {get;}
}
```

是interface类型，那么上面的函数中*yield return 0.1f*中的0.1f这个float类型的变量转换为IEnumerator<float>为什么不会Boxing呢? 先查看下两个方法的IL代码，对比下IL代码的区别。如下：

```csharp
//对于Func0
.class nested private auto ansi sealed beforefieldinit '<Func0>c__Iterator0'
	...
{
	...
	// Fields
	.field assembly float32 $current //current编译成为了我们指定的float类型
	.field assembly bool $disposing
	.field assembly int32 $PC	
	...
}

//对于Func1
.class nested private auto ansi sealed beforefieldinit '<Func1>c__Iterator1'
	...
{
	...
	// Fields
	.field assembly object $current//current编译成为了默认的object类型
	.field assembly bool $disposing
	.field assembly int32 $PC
}

```

那么问题就很清楚了，Func0其实每次迭代的时候接收值的变量就是float类型(对应于IL的float32类型)的所以根本不需要转换类型。但是对于Func1函数，编译成IL代码存储当前值的变量是object类型的，所以当我们Func1中返回值为0的int类型时，这个int型的变量会被转换成object类型而导致Boxing。

其实正确的操作是先看[C#关于泛型的文档](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/generics/)才对。不过现在看下也不迟。其中第一段就有如下描述:

> by using a generic type parameter T you can write a single class that other client code can use without incurring the cost or risk of runtime casts or boxing operations

到这儿疑问就比较明确的解决了。