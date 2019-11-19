---
title: 'C#多线程下数据同步处理'
date: 2016-06-05 23:33:59
type: "tags"
tags: csharp
comments: false
---

本文是阅读[CLR via C#](https://book.douban.com/subject/26285940/)和 [Theading in C#](http://www.albahari.com/threading/part4.aspx#_Reader_Writer_Locks)做的个人总结



### 为什么要进行多线程的同步

多个线程同时访问共享数据时有可能对数据造成损坏，线程同步的目的就是保证多线程访问共享数据时数据的安全性。在C#中能够对一个变量中的所有字节都一次性读取或写入，我们就认为这个变量类型具有原子性，CLR保证对以下类型变量的读写是原子性的: Boolean, Char, (S)Byte, (U)Int16, (U)Int32, (U)IntPtr,Single。

不具备原子性的类型变量的读取或者写入操作会被分割，也就是说对x变量的读取或者写入操作需要多条指令完。对于变量x来说，赋值操作需要两个MOV指令来完成即:

```cpp
Int64 x = 0x0123456789abcdef;  // Step1: 0x0123456700000000  Step2: 0x000000089abcdef
```

那么在多线程的情况下会出现另外一个读取该值的操作取出来的值是0x0123456700000000,所以<font color=#FF4040>我们要对非原子性的类型变量在多线程访问的情况下要进行同步操作(类似于自己构造变量的原子性)，以保证我们获取的数据的完整性</font>。


### 需要进行多线程同步的数据具体有哪些

* static 数据
* new 操作符分配的对象，并且这个对象传递给了其他线程

这两类数据都是作为共享数据，可以让多个线程同时访问的。所以我们要对其进行同步操作。但是当多个线程同时对共享数据进行只读操作的时候是不需要进行任何数据同步的。

### 多线程同步的几种常用接口


* <small>**Monitor (lock)**</small>


Monitor为static类，它的工作原理是操作对象的同步快索引。CLR在初始化的时候会在堆中分配一个同步块数组，这个数组的元素用来关联new构造器构造出来的对象的同步索引。一个对象在构造时它的同步块索引初始化为-1，表示不引用任何同步索引块。然后调用Monitor.Enter的时候，CLR在数组中找到一个空白的同步块来关联当前Monitor传入的对象，对象的同步块索引将会自增，对象处于锁定状态。当调用Monitor.Exit的时候如果没有其他线程等待这个对象，如果没有那么Exit将对象的同步块索引设置回-1，对象处于自由状态。


* <small>**Mutex**</small>


```cpp
    public sealed class Mutex : WaitHandler {
      public Mutex();
      public void ReleaseMutex();
    }
```

Mutex类强制线程标识，只能由获得它的线程可以释放互斥体。Mutex内部维护着一个递归计数，拥有互斥锁的线程可以递归调用相同互斥体（互斥对象）WaitOne而不会阻止线程执行。但是必须调用相同次数的ReleaseMutex来释放互斥体。


* <small>**Semaphore**</small>


```cpp
    public sealed class Semaphore : WaitHandler {
        public Semaphore(Int32 initialCount, Int32 maximumCount);
        public Int32 Release();
        public Int32 Release(Int32 releaseCount);
    }
```

内部维护者一个Int32变量。信号量为0时在信号量上等待的线程会阻塞，信号量大于0时解除阻塞。在信号量上等待的线程解除阻塞时，内核自动从信号量的计数中减1。


* <small>**ReaderWriterLockSlim**</small>

    在Monitor、Mutex和Semaphore方式来同步线程时就算某一时刻多个线程来同时读取数据也会造成只有一个线程工作其他线程等待的情况，这样就在成了资源的浪费。ReaderWriterLockSlim基于多个线程对共享数据同时读取不需要同步这个点来做的优化。它的工作方式如下:

    (1) 一个线程向数据写入时，请求访问的其他所有线程都被阻塞

    (2) 一个线程从数据读取时，请求读取的其他线程能够继续执行，但请求写入的线程仍然被阻塞

    (3) 写入的线程结束后，要么解除一个写入线程的阻塞，要么解除所有的读取线程的阻塞。如果没有线程被阻塞则锁就进入自由状态

    (4) 读取线程结束后，一个写入的线程被解除阻塞（如果有）。如果没有线程被阻塞则锁进入自由状态


### 测试例子

<!-- more --> 

```cs
public class TestThreadSync
{
    private Int32 id = 1;

    private const Int32 threadNum = 200;

    private const Int32 testReadWriteCount = 10;

    public static void Main(String[] args)
    {
        var testThreadSync = new TestThreadSync();

        testThreadSync.TestMonitor();

        Console.ReadLine();
    }

    public void TestMonitor()
    {
        new Thread(() =>
        {
            for (int i = 0; i < testReadWriteCount; i++)
            {
                CalculateId(1);
            }
        }).Start();

        new Thread(() =>
        {
            for (int i = 0; i < testReadWriteCount; i++)
            {
                CalculateId(5);
            }
        }).Start();
    }

    //-----TestMonitor------
    public void CalculateId(int value)
    {
        Monitor.Enter(this);

        try
        {
            Console.WriteLine("pre: " + id);

            int tmpValue = id;//step1

            //step2
            for (int i=0; i<=value; ++i)
            {
                tmpValue += i;
            }

            Console.WriteLine("tmpValue: " + tmpValue);

            id = tmpValue;//step3

            Console.WriteLine("aft: " + id);

            //如果进行线程同步的话,在多线程访问这个函数的情况下
            //有可能在step1和step3操作之前的id值会出现不一致的情况
        }
        finally
        {
            Monitor.Exit(this);
        }
    }
    //-----TestMonitor------

    
    //-----TestMutex------

    private static Mutex mutex = new Mutex();

    public void CalculateIdMutex(int value)
    {
        mutex.WaitOne();

        Console.WriteLine("pre: " + id);

        int tmpValue = id;

        for (int i = 0; i <= value; ++i)
        {
            tmpValue += i;
        }

        Console.WriteLine("tmpValue: " + tmpValue);

        id = tmpValue;

        Console.WriteLine("aft: " + id);

        mutex.ReleaseMutex();
    }
    //-----TestMutex------


    //-----TestSemaphore-----

    private static Semaphore semaphore = new Semaphore(0, 1);

    public void CaculateIdSemaphore(int value)
    {
        semaphore.WaitOne();

        Console.WriteLine("pre: " + id);

        int tmpValue = id;

        for (int i = 0; i <= value; ++i)
        {
            tmpValue += i;
        }

        Console.WriteLine("tmpValue: " + tmpValue);

        id = tmpValue;

        Console.WriteLine("aft: " + id);

        semaphore.Release();
    }

    //-----TestSemaphore-----
}
```


### 项目中遇到的多线程案例

最近在项目中用到多线程的地方是图片上传这个模块上,图片上传用的是C#的WebClient API。WebClient上传图片会在内部开启一个子线程去处理图片，这里需要处理的问题就是图片上传完成时WebClient对调用者设置的回掉函数是在子线程回掉的,由于不能在子线程处理UI逻辑，所以我们得通知下主线程来处理下载完成的逻辑。最简单的办法就是子线程和主线程共享一个对象，当子线程完成的时候版数据直接存储在这个对象的成员变量里，而主线程自开启子线程之后就开始判断这个对象是否已经存储了数据，如果存储了数据说明子线程的任务完成了。流程大概是这样子:

![](/images/cs_multi_thread_data_sync/multi_thread_data_sync.png)