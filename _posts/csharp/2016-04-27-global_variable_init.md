---
title: 'C#全局变量的初始化过程'
date: 2016-04-27 18:35:35
type: "tags"
tags: csharp
comments: false
---

#### 前言
C#类型的成员变量如果没有在定义的时候赋初值都会被默认初始化，初始化的具体指可以参考[Default values(c#)][1]，那么当我一定一个成员变量的时候直接赋值给值这个变量的话，编译器是怎样处理这个初值的呢？下面我们通过一个简单的例子来说明。


#### 代码
```c#
using System;

public class TestMemberVariable
{
    public Int32 Id;
    public String Name;
    public Int32 Id2 = 1001;
    public String Name2 = "Microsoft";

    //ctor_1
    public TestMemberVariable()
    {
    }

    //ctor_2
    public TestMemberVariable(Int32 Id, String Name)
    {
        this.Id = Id;
        this.Name = Name;
    }
    //ctor_3
    public TestMemberVariable(Int32 Id, String Name, Int32 Id2, String Name2) : this(Id, Name)
    {
        this.Id2 = Id2;
        this.Name2 = Name2;
    }

    //ctor_4
    public TestMemberVariable(Int32 Id, String Name, Int32 Id2, String Name2)
    {
        this.Id2 = Id2;
        this.Name2 = Name2;
    }

    public override string ToString()
    {
        return String.Format("Id: {0}, Name: {1} - Id2: {2}, Name2: {3}", Id, Name, Id2, Name2);
    }

    public static void Main()
    {
        Console.WriteLine(new TestMemberVariable().ToString());

        Console.WriteLine(new TestMemberVariable(1, "Apple").ToString());
    }
}
```

 上面的代码运行起来输出的结果为:
Id: 0, Name:  - Id2: 1001, Name2: Microsoft
Id: 1, Name: Apple - Id2: 1001, Name2: Microsoft

这里可以看到不管我是调用ctor_1还是调用ctor_2创建的对象，成员变量Id2和Name2都会有定义的初值。那么初值是在什么时候赋给变量的呢？下面我们用ildasm来反编译一下TestMemberVariable.exe，看能不能通过IL，ctor_1和ctor_2的IL代码如下:


**ctor_1构造**
```c#
.method public hidebysig specialname rtspecialname 
    instance void  .ctor() cil managed
{
	// Code size       31 (0x1f)
	.maxstack  8
	IL_0000:  ldarg.0
	IL_0001:  ldc.i4     0x3e9
	IL_0006:  stfld      int32 TestMemberVariable::Id2 //这里把初值1001赋值给了Id2
	IL_000b:  ldarg.0
	IL_000c:  ldstr      "Microsoft"
	IL_0011:  stfld      string TestMemberVariable::Name2 //这里把初值"Microsoft"赋值给了Name
	IL_0016:  ldarg.0
	IL_0017:  call       instance void [mscorlib]System.Object::.ctor() //这里是ctor_1构造
	IL_001c:  nop
	IL_001d:  nop
	IL_001e:  ret
} // end of method TestMemberVariable::.ctor
```

**ctor_2构造**
```IL
.method public hidebysig specialname rtspecialname 
        instance void  .ctor(int32 Id,
                             string Name) cil managed
{
  // Code size       45 (0x2d)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  ldc.i4     0x3e9
  IL_0006:  stfld      int32 TestMemberVariable::Id2//这里把初值1001赋值给了Id2
  IL_000b:  ldarg.0
  IL_000c:  ldstr      "Microsoft"
  IL_0011:  stfld      string TestMemberVariable::Name2//这里把初值"Microsoft"赋值给了Name
  IL_0016:  ldarg.0
  IL_0017:  call       instance void [mscorlib]System.Object::.ctor()//这里是ctor_2构造
  IL_001c:  nop
  IL_001d:  nop
  IL_001e:  ldarg.0
  IL_001f:  ldarg.1
  IL_0020:  stfld      int32 TestMemberVariable::Id
  IL_0025:  ldarg.0
  IL_0026:  ldarg.2
  IL_0027:  stfld      string TestMemberVariable::Name
  IL_002c:  ret
} // end of method TestMemberVariable::.ctor
```
上面的ctor_1和ctor_2的IL代码我们可以看到，在CLR把c#编译成IL代码之后会在构造函数之前插入Id2和Name2的初始化代码，这样就保证了在构造函数里面或者创建对象之后都可以访问到赋值好的Id2和Name2变量。

#### 其他
看TestMemberVariable.exe反编译之后的IL代码我们可以看到，每个构造函数之前都会被CLR编译器插入已经赋值的成员变量的初始化的代码，那么怎样可以做到复用这一块代码呢？方法就是像ctor_3这样定义构造函数，通过this(Id, Name)可调用到自身的ctor_2构造函数，这样ctor_3在编译之后就不会生成初始化Id2和Name2的IL代码了。

**ctor_3构造**
```IL
.method public hidebysig specialname rtspecialname 
        instance void  .ctor(int32 Id,
                             string Name,
                             int32 Id2,
                             string Name2) cil managed
{
  // Code size       26 (0x1a)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  ldarg.1
  IL_0002:  ldarg.2
  IL_0003:  call       instance void TestMemberVariable::.ctor(int32,
                                                               string)
  IL_0008:  nop
  IL_0009:  nop
  IL_000a:  ldarg.0
  IL_000b:  ldarg.3
  IL_000c:  stfld      int32 TestMemberVariable::Id2
  IL_0011:  ldarg.0
  IL_0012:  ldarg.s    Name2
  IL_0014:  stfld      string TestMemberVariable::Name2
  IL_0019:  ret
} // end of method TestMemberVariable::.ctor

```

[1]:https://msdn.microsoft.com/en-us/library/aa691171(v=vs.71).aspx