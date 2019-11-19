---
title: 'C# 内存管理之对象循环引用问题'
date: 2016-06-29 21:06:34
type: "tags"
tags: csharp
comments: false
---

**C#的内存管理**用到了引用计数， 但是和以往我们认识的Java、Objective-c的内存管理有非常大的不同之处。不同之处在于C#的引用计数只有在GC Root对象引用的对象身上才生效，也就是说如果一个对象被GC Root对象引用了，那么在这个GC Root对象没有销毁之前这个对象是不会被销毁的。反之，如果一个对象没有被GC Root对象引用，即便这个对象被再多非GC Root对象引用，也会再下一次内存回收的时候被销毁。

GC Roots:

* 正在运行的局部变量
* 静态变量
* 重写了object的finalizer方法的变量
* 正在调用的函数传递的参数


下图演示了几种对象引用的情况:

![](/images/cs_object_reference_each_other/1.png)

根据我们前面的描述，我们很容易判断object A和object B在RootA 被释放之后随之会被释放，object C和object D 虽然循环引用，但是没有被Root引用，所以在下一次内存回收的时候就会被释放,RootB和RootC造成了循环引用所以释放不了。