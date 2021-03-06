---
title: 'Unity游戏内存回收的优化(译)'
date: 2017-11-18 20:58:35
tags: csharp
comments: true

---

> 原文[Optimizing garbage collection in Unity games](https://unity3d.com/cn/learn/tutorials/topics/performance-optimization/optimizing-garbage-collection-unity-games)。这里对这篇文章的粗略翻译，当作自己的笔记。
>
> Garbage Collector在这里被翻译成名词GC。对于Garbage Collection这里翻译成GC内存回收。
>
> 内存垃圾: 代码中销毁了(disposed)但是GC还没有清理的内存。

# Unity的托管内存的简单介绍

为了理解GC（本文GC指代Garbage Collector）在内存分配和回收时是怎样工作的，我们必须首先了解Unity的引擎代码和我们自己编写的脚本是怎么利用内存来工作的。

Unity引擎代码运行时管理内存的方式叫做手动管理内存(manual memory management)。也就是说引擎代码必须显示的声明内存是怎么使用的。手动管理内存没有用到GC，这部分内容本文不做介绍。

当运行我们自己编写的脚本时Unity管理这部分内存的方式叫做自动内存管理(automatic memory management).这意味这我们自己编写的代码不需要很详细的去告诉Unity怎样去管理这部分内存。Unity都帮我们做好了。

简而言之，Unity的自动管理内存工作方式如下：

* Unity有两个可以访问的内存池：栈(stack)和堆 (heap)(也被叫做托管堆。栈内存被用在短期存储一些小片段的数据，堆则用来存储一些长期并且比较大的片段数据)。
* 当一个变量创建好了，Unity会在栈或者堆中申请一个内存块。
* 只要这个创建的变量在作用范围之内(我们的代码一直可以访问到它)，为它分配的相应的那块内存就一直是保持可用状态。我们把这块内存成为已经被分配的(allocated)。我们把在栈中分配内存的变量叫做栈上的对象，把内存分配在堆上的变量成为堆上的对象。
* 当这个变量已经脱离了它的作用区域，相应的这个变量所分配的内存也就不再需要了，这块内存随之就会把返回给分配时的那个内存池。我们称这块内存叫做已回收(deallocated)。栈上的内存一旦离开了作用区域就会立即被回收。如果是堆上的内存即便是它的引用已经超出了作用区域也不会被立即回收，它还会保持它的分配状态。
* GC会标记并且回收堆上没有被使用的内存。它会周期性的对堆上的内存进行清理。

现在我们了解了内存使用的流程，让我们来更深入的来了解栈和堆上的内存的分配和释放。

<!-- more --> 

## 栈在分配和释放内存的时候发生了什么？

栈内存的分配和回收比较快速和简单。这是因为栈仅仅用于存储一些小片段短周期的数据。分配和回收的操作基本上是按照预期的顺序和规模(多少量的内存)发生的。

栈的工作原理类似与数据结构类型的栈: 它是一个简单的元素收集器，在这种情况下栈的内存块只能按照严格的顺序来进行添加和删除。简单和严格的规则使得这个操作非常的快,当一个变量需要存储到栈上，内存会为它在栈底简单的分配一块内存。当一个栈的变量已经超出了作用域，存储这个变量的这块内存会立即释放掉变成可用内存。

## 堆在分配和释放内存的时候发生了什么？

堆的内存分配要比栈的内存分配负责很多。这是因为堆既能够用来存储长周期的的数据也能够用来存储短周期的数据。并且这些数据可以是各种不同类型和不同大小。分配和回收不会按照预期的顺序经常发生，并且这个分配和回收的内存有可能是大小差异比较大的内存块。

当创建了一个堆内存的变量，它会顺序执行一下这些步骤:

* 首先, Unity必须检查在堆上是否有足够的剩余可以分配的内存。如果足够了则会为这个变量分配相应大小的内存
* 如果在堆上的内存不够分配这个变量所需的内存。 Unity的GC触发器会去尝试释放堆上的没有被使用的内存。这将会是一个比较慢的操作。如果释放之后有了足够的剩余内存，则为这个变量分配相应大小的内存。
* 如果GC在释放内存之后堆上的剩余内存还是不够这个变量所需要的内存，Unity将会增加堆内存的大小。这也是一个比较慢的操作。这个操作之后将会为这个变量分配到相应大小的内存空间。

堆内存分配比较慢，尤其是触发了GC运行和堆内存的扩展。

## GC进行内存回收的时候发生了什么？

当一个堆内存变量已经超出了它的作用范围(或者说生命周期)时，存储这个变量的那块内存不会立即被回收，这块内存可以被称作未被使用的内存。未被使用的内存只有当GC运行的时候才会被回收。

每当GC运行一次，它会顺序执行一下步骤:

* GC会检查每个在堆上的对象
* GC会查找所有当前对象的引用情况来决定这个对象是否还在作用范围之内。
* 任何对象只要不在作用范围之内就会被GC标记为需要删除的对象。
* 删除被标记的对象然后回收这些变量所占用的内存。

GC回收内存是一个比较昂贵的操作。堆上的对象越多，对象的引用也会越多，GC需要处理的工作随之也会更多。

## 什么情况下会导致GC的运行

有三种情况会导致GC的运行:
* 不管什么时候只要堆的内存已经不能满足用户申请的内存的时候。
* GC会自己不定时的执行（这个频率是根据平台来定的）。
* 用户主动强制调用GC回收的时候

GC回收内存是一个比较频繁的操作。不管何时只要堆内存不能满足变量申请的内存的大小GC回收内存操作将会被触发，这也意味着频繁的堆内存的申请和释放操作会导致GC的频繁运行。

# GC回收内存的问题

现在我们已经了解了GC回收内存在Unity的内存管理中所扮演的角色，接下来我们可以考虑下在GC回收内存的时候可能发生的各种问题。

一个很明显的问题是GC会花费很多的时间。如果GC需要检查大量的堆内存对象或者是大量的对象引用关系，这个检查的过程将会非常的慢，这也会导致我们的游戏出现卡顿(stutter)或者运行缓慢。

另外一个问题是GC可能在不恰当的时间执行。如果CPU在执行我们游戏的性能关键部分已经承受了非常承重的负担了，这个时候即使想CPU添加一个很小的额外负担都会造成我们游戏的帧率下降并且性能会明显变差。

还有个不太明显的问题是堆内存的碎片化。当从堆中分配内存的时候，需要根据存储数据的大小从剩余的内存块从分配，当这些被分配的内存重新回收后，堆内存就会得到大量的被分割成很多小的不连续的可用的内存块。这意味着GC没有进行内存回收的情况下即时总共的可用内存会比较多，但是我们却不能分配一个比较大的一块连续内存。

这里我们总结了两个内存碎片化将会导致的问题。一方面我们的游戏内存使用比我们需要的会更高，另一方面GC内存回收操作将会更加频繁。想要堆内存碎片化更详细的讨论, 可以看这篇文章 [this Unity best practice guide on performance](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity4-1.html?_ga=2.83231508.342435300.1511170133-741569329.1509691291)

## 寻找堆内存的分配点

如果我们知道在我们游戏中导致GC运行的原因，我们需要知道哪一部分代码产生了内存垃圾。内存垃圾产生于超出了作用域的堆内存变量，因此我们首先需要知道变量把内存分配在堆中的原因。

下面列出的代码是一个在栈上分配内存的例子，变量localInt是局部的同时也是值类型的变量(value-type)这个变量的内存将会在这个函数运行之后被立即回收到栈中。

```csharp
void ExampleFunction()
{
  int localInt = 5;
}
```
下面列出的代码是一个在堆上分配内存的例子，变量localList是局部变量但是是引用类型的(reference-type)。这个变量的内存回收是在GC运行的时候(函数执行完并不会立即释放)。

```csharp
void ExampleFunction()
{
  List localList = new List();
}
```

## 使用Unity的Profile窗口可以找到堆内存的分配

我们可以通过Profiler窗口看到我们代码创建堆内存的地方

![](/images/unity_performance_ways/window_with_data.png)

在CPU使用情况的分析器中，我们可以选择任何一帧然后在Profiler窗口的底部部分查看这一帧的CPU的数据使用情况，其中有一列数据叫做 **GC Alloc** 。这一列显示了这一帧正在进行的内存分配情况。如果我们选择这一列的头部我们可以对这些统计好的数据进行排序，以便更能轻松的看出来在我们游戏中是哪些函数产生最大的堆内存分配。一旦我们知道是哪个函数产生分配堆内存，我们可以检查这个函数。

当我们知道了在这个函数中引起内存分配的那段代码，我们可以决定怎样去解决这个问题使得这里的分配的内存量最小化。

## 减少GC回收内存带来的影响

概括来说，我们可以通过三种方式来降少GC回收内存给游戏带来的影响:

- 我们可以减少GC运行的时间
- 我们可以减少个GC运行的频率
- 我们可以谨慎的在性能压力比较小的时候触发GC回收内存，例如在loading界面

基于这几个想法，这里有三个策略可以帮助我们：

- 我们可以构造好我们的游戏以便我们有更少的堆内存分配和更少的对象引用。堆上对象较少并且检查的对象引用更少意味着当GC被触发，GC回收内存需要更少的时间。
- 我们可以减少堆内存的分配和释放的频率，尤其是当性能压力比较大的时候。更少的内存分配和释放就会有更少场景触发GC进行垃圾回收。这也较少了堆内存碎片的风险
- 我们可以尝试在预期的在方便的时间时主动调用GC垃圾回收进行内存的拓展。这样做比较困难且比较难达到预期，但是当这个方式作为整个内存管理策略的一部分时可以减少GC垃圾回收的影响。

## 减少垃圾内存创建的数量

让我们来做一些技术上的测试来帮助我们减少我们自己的代码中产生的垃圾内存。

### 缓存

如果我们的代码在重复的调用一个函数，这个函数会导致堆内存分配而且每次我们调用完这个函数之后会丢弃这个函数的计算结果，这里会产生不必要的内存垃圾。取而代之，我们应该存储这个这些对象来重复使用。这个技术叫做缓存。

在下面列出的例子中，这段代码每次调用都会产生堆内存分配。这是因为会创建一个新的数组。

```csharp
void OnTriggerEnter(Collider other)
{
  Renderer[] allRenderers = FindObjectsOfType<Renderer>()
  ExampleFunction(allRenderers);
}
```
下面列出的代码是修改之后的，现在无论调用OnTriggerEnter多少次都只会造成一次堆内存分配，这个数组被创建好并赋值然后缓存起来。这个缓存数组可以被重复使用并不会产生额外的内存。

```csharp
private Renderer[] allRenderers;

void start()
{
  allRenderes = FindObjectsOfType<Renderer>()  
}

void OnTriggerEnter(Collider other)
{
  ExampleFunction(allRenderers);
}
```

### 不要在频繁调用的函数里面分配内存

如果我们必须在MonoBehaviour脚本里面分配堆内存，最糟糕的是我们把这个操作放在频繁运行的函数里。例如Update()和LateUpdate, 这两个函数每帧都会调用。所以如果我们会分配堆内存的代码放在这这种函数里面，内存非常快的增加。这种情况我们尽可能的考虑让这个对象在Start()或者Awake()函数中创建好然后使用引用来保存这个对象,或者确保这段代码只有在需要的时候才分配内存。

来看个非常简单的例子，这个例子稍作改动使得函数只有当条件改变的时候才运行。下面列出的代码，有个会产生内存分配的函数每帧都会被Update()调用，会频繁的创建内存垃圾。

```csharp
void Update()
{
  ExampleGarbageGeneratingFunction(transform.position.x);
}
```

做一个简单的改动，我们现在确保只有当transform.position.x的值发生改变的时候才调用ExampleGarbageGeneratingFunction。改动之后只有当我们需要的时候才会分配内存而不是每帧都会分配一次。

```csharp
private float previousTransformPositionX;

void Update()
{
  float transformPositionX = transform.position.x;
  if (transformPositionX != previousTransformPositionX)
  {
    ExampleGarbageGeneratingFunction(transformPositionX);
    previousTransformPositionX = transformPositionX;
  }
}
```

另外一个可以减少生成内存垃圾的方法是在Update()里面使用计时器。这个方法比较适合用于我们的这个会产生内存垃圾的函数ExampleGarbageGeneratingFunction必须有规律的运行，但是不需要每帧那么频繁。

下列例子中的代码，函数ExampleGarbageGeneratingFunction会每帧都会被调用，这个函数内部会生成内存垃圾。

```csharp
void Update()
{
  ExampleGarbageGeneratingFunction();
}
```

我们使用计时器的方式来修改下这个这段代码，确保这个会生成内存垃圾的函数每秒才调用一次。

```csharp
private float timeSinceLastCalled;
private float delay = 1f;

void Update()
{
  timeSinceLastCalled += Time.deltaTime;
  if (timeSinceLastCalled > delay)
  {
    ExampleGarbageGeneratingFunction();
    timeSinceLastCalled = 0f;
  }
}
```

当我们的代码运行的非常频繁的时候，只做一点小的改动就能够大幅度较少内存内存垃圾量。

## 清理容器

创建一个新的容器(List、Dictionary等)会申请分配堆上的内存。如果我们发现在代码中有超过一次的创建新的容器，那么我们应缓存起来这个指向这个容器的引用，然后使用Clear()来清空这个容器而不是重复调用New来重新创建。

下面这个例子中，每次调用new关键字都会引起分配一次堆内存。

```csharp
void Update()
{
  List myList = new List();
  PopulateList(myList);
}
```

下面这个例子，只有当容器创建或者容器内部必须改变存储大小的时候才会发生内存分配。这能够大幅减少产生内存垃圾的量。

```csharp
private List myList = new List();

void Update()
{
  myList.Clear();
  PopulateList(myList);
}
```

### 对象池

即使我们减少我们脚本中的内存分配，如果我们在游戏运行时大量的创建和销毁对象那么GC回收内存的问题任然存在。对于重复使用的对象，我们没有必要每次使用到都去重复的创建和销毁，利用**Object pooling**可以减少对于重复利用的对象的内存的分配和释放。Object pooling被广泛的应用在游戏中频繁生成和销毁的小对象的场合。例如，从枪中射出的子弹。

本文不讨论完整的object pooling使用规范，但是这是一项非常有用且值得学习的技术。[This tutorial on object pooling on the Unity Learn site](https://unity3d.com/learn/tutorials/topics/scripting/object-pooling) 是一个非常好的针对于Unity的object pooling系统的实现规范。



## 常见的导致不必要的堆内存分配的一些原因

我们已经理解了值类型的变量是分配的栈内存，引用类型的变量分配的堆内存。然而有很多分配堆内存的地方会让我们自己感到惊讶，因为根本就没必要分配堆内存。下面让我们来看一些常见的不必要的堆内存分配情况并且想出最好的办法去减少这种情况。

### String

在C#中, [string是引用类型](https://msdn.microsoft.com/en-us/library/362314fe.aspx)而不是值类型，尽管它看上去像是持有了字符串的值。这意味着string创建和丢弃之后会产生内存垃圾。作为一个在大量代码中常用的类型，这些内存垃圾会一直增长。

C#中的String类型同时也是不可变类型，意思就是在它创建之后就不能改变它的值了。当我们每次操作一个字符串(例如，使用重载运算符+来连接两个string)。Unity如果需要更新一个字符串的值，它会创建一个新的字符串来保存新值然后丢弃当前这个字符串。这会产生内存垃圾。

我们可以遵循一些简单的规则让string产生的内存垃圾最小化。先考虑下这些规则，然后在例子中看看它是怎么被应用的。

* 我们应该减少那些不必要的string创建。如果我们使用同一个string的值超过一次，那么我们应该创建好这个string然后缓存起来。
* 我们应该减少哪些不必要的string的值的更改。例如，如果我们有个频繁更新的Text组件，Text组件显示的值是每次两个string的连接成一个新string得到的。我们应该考虑把把这个Text组件分成两个Text组件，这样分别显示两个string的值，就不需要每次会创建一个新的string了。
* 如果我们在游戏运行时需要构建一个string(如多个字符串值连接成一个字符串)。我们应该使用[StringBuilder类](https://msdn.microsoft.com/en-us/library/system.text.stringbuilder(v=vs.110).aspx)。StringBuilder类是专门为了构建string而设计的，它不会产生内存分配而且在我们连接一个比较复杂的string时会避免产生大量的内存垃圾。
* 一旦我们不需要调试项目时我们应该把项目里面的Debug.Log()全部去掉。在我们游戏中调用Debug.Log()会持续的构建string，即使它们不在输出任何调试日志。每调用一次Debug.Log()会创建和销毁至少一个string对象，所以如果我们游戏里面包含了大量的这种调用，那么内存垃圾会增加很多。

让我们来写个例子来测试下，这个例子包含无效率的string而导致产生了不必要的内存垃圾的代码。在下面这段代码中，我们创建了一个记录分数值的string对象，它在Update()里面和值为“TIME:”的float类型对象*timer*进行合并。这里产生了不必要的内存垃圾。

```csharp
public Text timerText;
private float timer;

void Update()
{
  timer += Time.deltaTime;
  timerText.text = "TIME:" + timer.ToString();
}
```

再来看看下面列出的代码，我们对一些地方做了比较有效的改进。我们把"TIME:"字符串独立出来在Start()中赋值给一个单独Text组件。这意味着在Update()函数中我们不再需要并合字符串了。这非常有效的减少了大量的内存垃圾的产生。

```csharp
public Text timerHeaderText;
public Text timerValueText;
private float timer;

void Start()
{
  timerHeaderText.text = "TIME:";
}

void Update()
{
  timerValueText.text = timer.toString();
}
```



### Unity的函数调用

当我们调用不是自己实现的代码的时候一定要非常的注意，这些代码包括Unity自己的引擎代码或者是第三方的插件代码，这种情况可能会产生内存垃圾。调用一些Unity的函数会分配堆内存，所以我们应该比较细心去避免一些不必要的内存垃圾产生。

这里没有一个应该禁止调用的函数的清单。每个函数在一些情形中有非常有用，但是在其他的情况就用处不大了。还是那句话，最好是仔细分下分析我们的游戏，找出内存垃圾创建的地方，仔细考虑下怎么处理它比较合适。在一些情况中，缓存函数的结果是比较明智的；但是另一些情况中，最明智的方式是减少这个函数的调用频率；还有些情况，最好的方式是重构这段代码把按照不同的情况来分成不同的函数。老方法，让我们来看两个Unity中常见的函数，这两个函数会造成堆内存分配。我么考虑下怎么去最好的处理这两个函数。

每个我们访问Unity的函数都会放返回一个数组，这个函数内存会创建一个新的数组作为返回值返回给我们。这种行为并不总是明显或者可以意料到的，尤其是这个函数是被定义为[*get*形式访问的变量](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/get)(例如，Mesh.normals)。

 下面列出的代码，在这个迭代循环中，每循环一次都会创建一个新的数组。

```csharp
void ExampleFunction()
{
  for (int i=0; i<meMesh.normals.Length; i++)
  {
    Vector3 normal = myMesh.normals[i];
  }
}
```

在这个例子中很容易采取这些方法来减少内存分配: 我们可以简单的缓存一个指向这个数组的引用。当我们要做代码中事情的时候只需要创建一个数组即可，这样就减少了大量的内存分配从而会导致大量内存垃圾的产生。

下面的代码就是经过这个改动的一个演示。在这段代码中，我们在循环运行之前首先调用Mesh.normals来缓存了的数组的引用以便只有一个数组呗创建。

```csharp
void ExampleFunction()
{
  Vector3[] meshNormals = myMesh.normals;
  for (int i=0; i<meshNormals.Length; i++)
  {
    Vector3 normal = meshNormals[i];
  }
}
```

堆内存分配的另外一个比较意外的原因可以在*GameObject.name*或者*GameObject.tag*中找到。这两个函数也是*get类型访问的变量*返回一个新的字符串，这意味着调用这些函数将会产生内存垃圾。一个比较有效的方法是缓存这些变量值，但在一些情况中我们可以使用Unity的函数来代替调用这些会产生内存垃圾的方法。要在不生成内存垃圾的前提下去检查GameObject的tag值，我们可以使用*[GameObject.CompareTag()](https://docs.unity3d.com/ScriptReference/GameObject.CompareTag.html?_ga=2.97011549.316260478.1511584816-1497166853.1503199345)*。

在下列代码中，调用GameObject.tag创建了内存垃圾。



```csharp
private string palyerTag = "Player";

void OnTriggerEnter(Collider other)
{
  bool isPlayer = other.gameObject.tag == playerTag;
}
```

如果我们使用*GameObject.CompareTag()*，这个函数就不再会生成任何内存垃圾了。

```csharp
private playerTag = "Player";

void OnTriggerEnter(Collider other)
{
  bool isPlayer = other.gameObject.CompareTag(playerTag);
}
```

*GameObject.CompateTag*不是Unity唯一的这类函数；很多Unity的函数都有不会引起对内存分配的版本。比如，我们可以使用*[Input.GetTouch()](https://docs.unity3d.com/ScriptReference/Input.GetTouch.html?_ga=2.99577052.316260478.1511584816-1497166853.1503199345)*和*[Input.touchCount](https://docs.unity3d.com/ScriptReference/Input-touchCount.html?_ga=2.99577052.316260478.1511584816-1497166853.1503199345)*来代替*[Input.touches](https://docs.unity3d.com/ScriptReference/Input-touchCount.html?_ga=2.99577052.316260478.1511584816-1497166853.1503199345)*，可以使用*[Physics.SpereCastNonAlloc()](https://docs.unity3d.com/ScriptReference/Physics.SphereCastNonAlloc.html?_ga=2.159211768.316260478.1511584816-1497166853.1503199345)*来代替*[Physics.SphereCastAll()](https://docs.unity3d.com/ScriptReference/Physics.SphereCastAll.html?_ga=2.159211768.316260478.1511584816-1497166853.1503199345)*


### 装箱
装箱(Boxing)是值类型变量被用做引用类型变量的地方发生的。它通常发生在我们传入一个值类型的变量给一个参数是引用类型的函数。比如给Object.Equals()传入的值是int或者float值，但是Object.Equals()的参数类型是一个引用类型的。

举个例子，在函数*Stirng.Format()*使用了字符串和对象参数，当我们传入一个string和一个int类型的参数，这个int类型的变量就会被装箱。下列的代码包含了一个装箱的例子:

```csharp
void ExampleFunction()
{
  int cost = 5;
  string displayString = String.Format("Price:{0}gold",cost);
}
```

装箱在内部会创建内存垃圾。当一个值类型的变量被装箱，Unity会在堆上创建一个零时的System.Object对象包包裹这个值类型的变量。System.Object时引用类型的变量，所以当这个零时的变量被销毁时产生了内存垃圾。

装箱是一个非常常见的引起不必要的堆内存分配的原因。即使我们在自己的代码中去避免这种变量的装箱操作，但是我们可能使用了第三方的插件会造成装箱或者是插件的代码内部实现存在装箱。最好的习惯是避免任何可能出现装箱的代码，同时去掉所有的导致装箱的函数。

### 协程
调用*StartCoroutine()*会创建少量的内存垃圾，因为Unity必须创建一个实例来管理*StartCoroutine()*所返回的coroutine实例。考虑到这点，尽量减少对*StartConoroutine*的调用，因为我们游戏的交互和性能是一个核心所在。为了减少协程所带来的内存垃圾，任何在性能负载高的时期运行的协程应该提前执行同时我们应该特别注意那些嵌套的协程，这些协程里面可能也包含延迟调用*StartCoroutine*。

协程内的yield生命不会自己创建堆内存。然而，我们传递的yield状态的值可能会导致不必要的堆内存分配。例如，下面的代码会产生内存垃圾:

```csharp
yield return 0;
```

这段代码因为值0会装箱(Boxing)会导致内存垃圾。在这个例子中，如果我们想不会产生任何堆内存实现简单的等待一帧效果，最好的方式是这样做:

```csharp
yiled return null;
```

另外一个比较常见的错误是在协程里面使用new操作多次创建一个值相同的yield状态变量。例如,下面的代码每次循环都会创建和销毁一个WaitForSeconds对象。

```csharp
while(!isComplete)
{
  yield return new WaitForSeconds(1f);
}
```

如果我们缓存WaitForSeconds这个变量然后重复使用，这回极大的减少内存垃圾。下面的代码展示了修改之后的方式：

```csharp
WaitForSeconds delay = new WaitForSeconds(1f);

while (!isComplete)
{
  yield return delay;
}
```

如果我们的代码在协程运行期间会产生大量的内存垃圾，我们可能希望考虑下使用非协程的方式来重构这部分代码。重构代码是一个复杂的主题同时每个项目都是唯一的，但是这里也提供两个常用的协程的替代方式我们希望能够记住。例如，如果我们使用协程主要是用来管理时间的话，我们可以在*Update()*函数里面保持简单的时间记录。如果我们使用协程主要是用来控制我们游戏中一些逻辑业务执行顺序时，我们可以创建一些有序的消息发送系统来允许对象间的通信。没有一些方法能够适合所有的情况，但是请记住，在代码中往往不止一种方式可以实现相同的目的。

### foreach 循环

> 译: 这个问题的具体分析可以看[C# foreach带来的内存问题](http://www.elangduan.com/2016/10/29/Mono%20V2.0.5%E7%9A%84foreach%E5%86%85%E5%AD%98%E9%97%AE%E9%A2%98/)这篇文章

在Unity5.5之前的版本中，一个foreach遍历出了数组之外的任何集合，每次循环结束之后都会产生内存垃圾。这是由于在foreach内部发生了装箱操作。当一个循环开始的时候会在创建一个System.Object堆内存对象，当循环结束的时候这个对象会被销毁，从而产生了一个内存垃圾。这个问题在Unity5.5的版本中修复了。

举个例子，在Unity5.5之前的版本中，下面这段吗的循环会产生内存垃圾:

```csharp
void ExampleFunction(List listOfInts)
{
  foreach(int currentInt in listOfInts)
  {
    DoSomething(currentInt);
  }
}
```

如果我们不能够升级我们的Unity版本，这里又一个简单的解决方法。*for*和*while*循环不会在内部导致装箱操作因此也不会产生堆内存垃圾。当我们要遍历的集合不是数组的时候，我们应该更偏向于使用这两个方式。

下面的代码循环遍历不会导致产生内存垃圾:

```csharp
void ExampleFunction(List listOfInts)
{
  for (int i=0; i<listOfInts.Count; ++i)
  {
    int currentInt = listOfInts[i];
    DoSomething(currentInt);
  }
}
```

### 函数的引用

不管是[匿名函数](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/anonymous-methods)还是普通函数，在Unity中都是引用类型的变量。他们都会产生堆内存分配。把一个匿名方法转换为闭包(匿名函数内部访问了外部变量)会增加很多内存开销同时创建多个堆内存对象。

更加精确和详细的函数引用和闭包的内存分配变化依赖于所在的平台和编译器的设置。但是如果GC内存回收是一个比较需要注意的问题，那么在游戏运行过程中最好减少函数的引用和闭包的使用。[[This Unity best practice guide on performance](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity4-1.html?_ga=2.69215250.342435300.1511170133-741569329.1509691291)]针对这个主题有更加深入的技术细节的介绍。

### LINQ和正则表达式

LINQ和正则表达式内部都会因为装箱产生内存垃圾。最好的做法是在性能消耗的集中点禁止使用它们。还是这篇文章，[this Unity best practice guide on performance](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity4-1.html) 提供了非常好的关于这个主题的技术细节

## 重构我们的代码把GC内存回收的影响减小到最小

我们代码的架构方式会影响GC的内存回收。即使我们代码没有创建任何堆内存，它也会增加GC的负担。

我们的代码没必要增加GC的负担的一种情况是它需要去检查(译:判断这个对象是否引用情况来决定是否回收这个对象)本该不需要检查的事情。结构体是一个值类型的变量，但是当我们有一个结构体包含了一个引用类型的变量的时候GC必须去检查这整个结构体了。如果我们有个很大这种结构体的数组，这样就给GC造成了非常多的额外工作。

在这个例子中，这个结构体包含了一个string类型的对象，它是引用类型的。当这段代码运行时GC必须对这整个结构体数组都进行检查。

```csharp
public strct ItemData
{
  public string name;
  public int cost;
  public Vector3 position;
}
```

```csharp
private ItemData[] itemData;
```

在下面这个例子中，我们将数据存储在单独的数组中。当GC运行起来的时候，它只需要检查string类型的数组同时忽略调用其他值类型的数组。这使得GC只需要做必要的工作。

```csharp
private string[] itemNames;
private int[] itemCosts;
private Vector3[] itemPositions;
```

另外一种在我们代码中给GC增加了不必要的负担的情况是只有了不必要的内存引用。当GC查找堆内存上的对象的引用的时候，它必须检查当前我们代码每个对象的引用，即使我们不需要减少总的堆内存上的对象的数量，代码中更少的对象引用意味着GC需要做的工作就更少。

下面这个例子中，我们实现了个对话框的类。当用户查看这个对话时，将会显示一个对话框。我们的代码包含了指向下一个应该显示的对话框数据实例的引用。这意味GC必须去检查这部分操作的引用:

```csharp
public class DialogData
{
  private DialogData nextDialog;
  
  public DialogData GetNextDialog()
  {
    return nextDialog;
  }
}
```

这里我们重构下这段代码，让这个函数返回一个指向像个对话数据实例的标识符来代替返回实例本身。修改之后没有引用，这就不会增加GC的时间开销了。

```csharp
public class DialogData
{
  private int nextDialogID;
  
  public int GetNextDialogID()
  {
    return nextDialogID;
  }
}
```

这是一个非常简单的例子。然而，如果我们的游戏中包含了很多持有了其他对象的引用的对象。那么我们需要考虑使用这种方式来重构我们的代码，减少堆内存对象的复杂度。

## 定时进行GC内存回收

### 手动强制进行内存回收

最终，我们可能希望自己去触发GC回收内存。如果我们知道存在堆内存已经分配了但是不再使用了(例如, 如我们的代码在加载资源时产生了内存垃圾)同时我们知道GC回收内存操作不会影响到玩家(例如，loading界面一直显示的时候)，我们可以通过下面的代码来要求GC执行内存回收操作。

```csharp
System.GC.Collect();
```

这段代码会强制GC运行，方便我们在这个时间点释放那些没有任何引用的对象的的内存。

## 总结

我们学习了Unity的GC内存回收的工作原理，引起性能问题的原因是什么? 怎么去减少这些问题对我们的游戏的影响。使用本文介绍的知识和我们的性能优化工具，我们可以修正GC内存回收的性能问题和重构我们的代码使得他们更有效率的管理内存。