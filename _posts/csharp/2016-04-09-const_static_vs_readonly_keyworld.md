---
title: 'C# 的const, static 和 readonly 关键字的理解'
date: 2016-04-09 16:47:36
type: "tags"
tags: csharp
comments: false
---

#### 基本含义
关于c#的const, static, readyonly关键字修饰成员变量时的区别,下面通过一个简单的例子来说明:
 


```cs
//SomeType.cs

using System;

public sealed class SomeType
{
    public Int32 Id = 1;

    public const Int32 constId = 100; //IL  .field public static literal int32 constId = int32(0x00000032)
    
    public static Int32 staticId = 1000; //IL .field public static int32 staticId

    public readonly Int32 readonlyId = 2000; //IL .field public initonly int32 readonlyId

    public readonly Int32[] readonlyArrayId = new Int32[]{1, 2, 3};//readonlyArrayId本身不能再次write, 但是指向的对象可以改变

    public static readonly Int32 staticReadOnlyId = 5000; //IL .field public static initonly int32 staticReadOnlyId

    public SomeType()
    {
        readonlyId = 20001;
    }
}

```


```cs
//TestSomeType.cs

using System;

public class TestConst 
{
    public static void Main(String[] args)
    {
        SomeType someType = new SomeType();

        //test1
        int result = someType.Id + SomeType.constId + SomeType.staticId + someType.readonlyId + SomeType.staticReadOnlyId;

        //test2
        someType.readonlyArrayId[1] = 4;

        Console.WriteLine(someType.readonlyArrayId[1]);//output 4

        //test3
        Console.WriteLine("constId: " + SomeType.constId);//

        Console.WriteLine("staticId: " + SomeType.staticId);
    }
}

```

 一. 结论

1. readonly

　　(1) 修饰成员变量的时候,该变量只能通过变量定义时或者早构造方法里面赋值。

　　(2) 修饰的成员变量是实例字段，内存是在构造函数类型的实例时分配的。

　　(3) 修饰的成员变量是引用类型的时候，不可改变的是引用，引用指向的对象是可以改变的。

2. static

　　(1) 修饰的成员变量是类型资源，内存是在类型对象中分配的，和具体的实例无关。

3. const

　　(1) 编译成IL代码后由static literal 修饰，由此可见const修饰的变量最终表示成static类型同时被literal修饰( literal修饰的变量必须在变量声明的时候赋值，编译器编译成IL代码时会直接把这个变量的值插入到引用这个变量的位置替代之前的变量，从而程序在运行时不需要再为此变量分配内存)。

 

#### const与static的区别

1. 编译SomeType.cs 和 TestSomeType.cs

//生成SomeType.netmodule
csc /t:module SomeType.cs
 
//生成TestSomeType.exe
csc TestSomeType.cs /addmodule:SomeType.netmodule
 运行TestSomeType.exe 可以看到test3中的输出为:

constId: 100
staticId: 1000
现在改变SomeType.cs类型的constId和staticId的值分别为50, 500,然后重新生成SomeType.netmodule

接着再次运行TestSomeType.exe可以看到test3中输出为:

constId: 100
staticId: 500
结果发现只有staticId的值引用的新值。再来看看TestSomeType.cs类的Main函数的IL代码:

 
```cs
.class public auto ansi beforefieldinit TestConst
    extends [mscorlib]System.Object
{
    // Methods
    .method public hidebysig static 
        void Main (
            string[] args
        ) cil managed 
    {
       ...

        IL_0000: nop
        IL_0001: newobj instance void SomeType::.ctor()
        IL_0006: stloc.0
        IL_0007: ldloc.0
        IL_0008: ldfld int32 SomeType::Id
        IL_000d: ldc.i4.s 100 //（1）
        IL_000f: add
        IL_0010: ldsfld int32 SomeType::staticId //（2）
     	 
     	 ...
     	 
        IL_003b: ldstr "constId: "
        IL_0040: ldc.i4.s 100 //（3）
        
        ...
        
        IL_0052: ldstr "staticId: "
        IL_0057: ldsfld int32 SomeType::staticId //（4）
        IL_005c: box [mscorlib]System.Int32
        IL_0061: call string [mscorlib]System.String::Concat(object, object)
        IL_0066: call void [mscorlib]System.Console::WriteLine(string)
        IL_006b: nop
        IL_006c: ret
    } // end of method TestConst::Main

    ...

} // end of class TestConst
```
 

可以看到上图的四个标记。标记（1）和（3）是constId编译成IL代码的表示，标记（2）和（4）是staticId编译成IL代码的表示。

对于constId 我们可以发现constId其实已经替换成其对应的值了,所以改变SomeType.cs的constId重新编译影响不到已经编译好的TestSomeType程序集。

对于staticId变量标明的是所在模块的信息，当程序运行起来时CLR会加载其对应的netmodule来获取staticId的值。所以改变SomaeType.cs的staticId的值并重新编译，在TestSomeType程序集中即可获取最新的值。

 