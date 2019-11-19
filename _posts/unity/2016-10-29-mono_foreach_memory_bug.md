---
title: 'C# foreach带来的内存问题'
date: 2016-10-29 21:16:35
tags: csharp
comments: false
---

#### 简介
在使用Unity工具开发游戏的时候经常会被建议不要使用foreach,究其原因说是会产生额外的heap内存,既然存在这种问题, 那我必须得自己搞清楚下。本文是自己查阅资料和试验的结果总结。

本文参考了[Memory allocation when using foreach loops in C#](https://stackoverflow.com/a/26110503)

*测试环境: Unity5.4.1f1, MonoV2.0.5*

#### 剖析
首先来搞明白使用foreach和for的区别,下面是一个简单的例子:

```cs
//Sample1.0
{
    List<string> list = new List<string>(){"hello", "world"};

    foreach(string item in list)
    {
        UnityEngine.Debug.Log(item);
    }
}
	
```

<!-- more --> 

编译之后查看对应的IL代码, 发现foreach编译之后通过获取List的内部类Enumerator来遍历List的数据,其中List<string>.Enumerator内部类的定义为:

```cs
public struct Enumerator : IEnumerator<T>, IDisposable, IEnumerator{}

```

Enumerator为struct类型, 在c#中struct类型为[Value Types](https://msdn.microsoft.com/en-us/library/s1ax56ch.aspx),根据c#对[Value and Reference Types](https://msdn.microsoft.com/en-us/library/4d43ts61(v=vs.90).aspx)的描述,其中说到:

A value type stores its contents in memory allocated on the stack

也就是说Enumerator的内存是分配在stack区域的, 再仔细看看Enumerator发现其实现了IDisposable接口,而IDisposable接口有什么特性呢? 从[.Net Framework文档](https://msdn.microsoft.com/en-us/library/system.idisposable(v=vs.110).aspx)中我们可以看到:

Implement IDisposable only if you are using unmanaged resources directly. If your app simply uses an object that implements IDisposable, don't provide an IDisposable implementation. Instead, you should call the object's IDisposable.Dispose implementation when you are finished using it. Depending on your programming language, you can do this in one of two ways:

* By using a language construct such as the using statement in C# and Visual Basic.
* By wrapping the call to the IDisposable.Dispose implementation in a try/catch block.


也就是说继承了IDisposable接口之后得使用using来声明或者使用try/catch块来包裹我们代码。那我们根据这些特性可以通过对应的IL代码来还原成C#的代码:

*使用using*
```cs
using(List<string>.Enumerator enumerator = list.GetEnumerator())
{
    while(enumerator.MoveNext())
    {
        UnityEngine.Debug.Log(enumerator.Current);
     }
 }
```

*使用try/catch包裹*
```	cs
List<string>.Enumerator enumerator = list.GetEnumerator();
    
try
{
    while (enumerator.MoveNext())
    {
        UnityEngine.Debug.Log(enumerator.Current);
    }
}
finally
{
    ((IDisposable)enumerator).Dispose();
}
```

从使用try/catch包裹这个代码中我们可以看出问题了，在finally块中编译器对enumerator进行的类型的强者转换,转换称了IDisposable接口类型的,我们知道在C#中接口类型属于reference type。reference type的内存是在heap上分配的, 分析到这里其实答案基本上已经知道了。另一方面其实在分析看IL代码就可以发现问题, 在IL中的的finally块是这样的:

```cs
	finally  { // 0
	  IL_004d:  ldloc.2 
	  IL_004e:  box valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<string>
	  IL_0053:  callvirt instance void class [mscorlib]System.IDisposable::Dispose()
	  IL_0058:  endfinally 
	} // end handler 0
	
```
其中第三行有一个很扎眼的关键子*box*。那么box是的作用是什么呢？从上面IL代码其实可以看出Boxing把Enumerator<string>这个valuetype转换成了IDisposable这个interface类型的对象。具体的过程参照[Boxing官方的文档](https://msdn.microsoft.com/en-us/library/yz2be5wk.aspx)有这样的描述:

>Boxing is used to store value types in the garbage-collected heap. Boxing is an implicit conversion of a value type to the type object or to any interface type implemented by this value type. Boxing a value type allocates an object instance on the heap and copies the value into the new object.

也就是说把一个value type对象转换成其实现的Reference type对象, 然而转换之后的这个对象的内存是分配在heap区域的。现在我们终于知道foreach会产生heap内存的原因了。

总结:基于使用foreach会在heap上分配额外内存的问题。在Unity使用foreach就要变得谨慎了,尤其是Update这种每帧执行的函数里面还是尽量避免使用foreach,尽管它有良好的代码可读性和使用上的方便等特性。



---

在*Unity5.5.3f1*上面测试了一下发现这个问题已经解决了, 看下在*5.5.3f1*上foreach反编译的finally的代码:

```
finally
{
	// 0x02B9: 12 03
	IL_0049: ldloca.s 3
	// 0x02BB: FE 16 02 00 00 1B
	IL_004b: constrained. valuetype [mscorlib]System.Collections.Generic.List`1/Enumerator<string>
	// 0x02C1: 6F 07 00 00 0A
	IL_0051: callvirt instance void [mscorlib]System.IDisposable::Dispose()
	// 0x02C6: DC
	IL_0056: endfinally
}

```

对List的GetEnumerator已经没有进行box操作了。