---
title: 'Unity内存（二） 内存快照'
date: 2024-11-15 00:31:25
type: "tags"
tags: perf,unity
comments: true
---



Unity的Mono内存有两个指标很关键，一个是 *Used Total* 另外一个是 *Reserved Memory*。 这个可以从 Profiler 的 Memory 项中可以看到：

<img src="/images/perf/unity_memory_2/1.png" alt="1" style="zoom: 67%;" />

> Unity2019.4.18f1 版本的Profiler截图

从脚本里面我们可以通过Unity提供的API可以获取到这些内存大小：

* Mono Used Total ：[GetMonoUsedSizeLong](https://docs.unity3d.com/ScriptReference/Profiling.Profiler.GetMonoUsedSizeLong.html) 返回当前正在使用的Mono内存总和，包括存活的对象内存和已经不被引用但是还没有被GC回收的对象内存。
* Mono Reseved Total :  [GetTotalReservedMemoryLong](https://docs.unity3d.com/ScriptReference/Profiling.Profiler.GetTotalReservedMemoryLong.html) Reserved内存表示当前Unity向系统总共申请分配的内存，其中包括正在使用的内存（Mono Used Total）、预申请的内存、已经使用过并且回收的内存。

Mono内存问题原因很简单就是分配的太多了，但是出现问题主要有两个原因导致的：

* 释放的太少，导致内存占用过高
* 虽然释放的很多，但是因为内存碎片的问题（上一篇已经介绍），释放的内存用不上，导致内存占用过高。

针对以上两个不同的原因，我们分别需要用不同的方式来定位解决：

* 释放的太少问题  -  当前内存占用过高，Dump出当前的内存，查看内存中的对象分配，就可以定位出来哪些对象分配的太多，并且没有释放了。
* 内存碎片的问题  -  内存碎片问题就需要追踪历史的内存分配，需要关注一些频繁分配对象的代码，尤其是数量多但是单个对象内存占用小的情况。这里Dump显然不合适，Dump功能只能看某一帧内存详情，并且Dump一下内存本身会比较耗时导致游戏会卡住，所以我们不可能把单局的每一帧都Dump出来。所以我们需要记录下每帧内存的情况来源以及分配大小，类似 Profiler中的CPU消耗列表一样。

下面我们针对两种方式做详细的介绍。

#### Memory Dump

Unity提供了一个API用来Dump内存：

```csharp
public static void TakeSnapshot(string path, Action<string,bool> finishCallback, Unity.Profiling.Memory.CaptureFlags captureFlags);

[Flags]
public enum CaptureFlags : uint
{
  ManagedObjects = 1, //托管内存(Mono内存)
  NativeObjects = 2,
  NativeAllocations = 4,
  NativeAllocationSites = 8,
  NativeStackTraces = 16, // 0x00000010
}
```

根据指定的枚举类型，可以Dump出对应的内存数据，我们常用的HeapExplore（最新的还有MemoryProfiler）内存Dump工具就是用的这个API。Dump Native内存比较简单，因为Native对象都在引擎里面自己分配自己维护的，只需要把分配对象列表输出即可（看HeapExplore解析的代码也很简单）。Dump Mono内存就相对复杂了，因为il2cpp之后，设备上跑的是C++，内存申请是业务层申请的，引擎层是一层类似中间层的组织管理，最终的分配是通过 bdwgc（这个流程上一篇已经介绍）的，分配内存由Mono托管，所以Mono虚拟机里面才有完整的内存对象列表。

问：既然HeapExplore可以满足Dump的需求了，那是不是直接用HeapExplore工具，不用去管具体怎么Dump的。

答：如果你需要用到Dump的频率很少，并且对里面的技术细节不感兴趣，那是的。 但是一般项目中会涉及到版本内存的日常监控，开发过程中内存监控，就需要更加高效，更加自动化的流程。比如自动化流水线直接输出Mono内存的差异列表（版本间的对比数据），直接提单到对应的程序，整个自动化过程不需要人工干预。自己了解了Dump的工作原理和细节之后就可以定制化这种功能。

关于TakeSnapshot输出的内存Dump文件，在Unity Documents基本上搜不到有价值的信息（也可能是我没找到）。所以最直接的方式是直接看HeapExplore代码，通过HeapExplore代码来反推Dump文件的信息，了解Dump的细节（这里有个奇怪的点是HeapExplore的作者是怎么了解到Dump文件的数据内容的，莫不是有源码）。那就先从HeapExplore工具展示的内容，再到HeapExplore的代码，最后了解Dump的原理。首先我们看下HeapExplore里面Mono内存的统计：

<img src="/images/perf/unity_memory_2/2.png" alt="image-20241118210547014" style="zoom:80%;" />

可以看到HeapExplorer可以把当前帧每种类型的对象都列出来，并且每种类型的对象分配的数量和大小都有统计。了解功能之后我们直接看解析TakeSnapshot文件的代码，看这个C#对象列表是怎么解析出来的。首先我们分析下 TakeSnapshot出来的文件对象（只列出Mono内存相关）：

```csharp
//PackedMemorySnapshot.cs
public class PackedMemorySnapshot : IDisposable
{
    /// All GC handles in use in the memorysnapshot.
    public GCHandleEntries gcHandles {get; internal set;}
    
    // Descriptions of all the managed types that were known to the virtual machine when the snapshot was taken.
    public PackedManagedType[] managedTypes = new PackedManagedType[0];
}
```

下面逐步解析下 PackMemorySnapshot 的成员变量所存储的数据。

##### gcHandles

GChandle提供用于从非托管内存访问托管对象的方法。实现的方式是通过GCHandle给指定对象分配句柄，分配句柄之后的对象可以阻止GC收集此对象，避免对象没有被引用时被GC回收，直到调用Free接口才会释放。这样非托管对象引用托管对象时，控制托管对象的生命周期，防止托管对象被GC回收。可以做个简单的测试来看下GCHandle的作用：

```csharp
public class GCHandleTest : MonoBehaviour
{
    private void Start()
    {
        FreeGCHandleObject();
    }
    public class TestGCHandleObject
    {
        public string Name;
        ~TestGCHandleObject()
        {
            Debug.LogError("~TestGCHandleObject " + Name);
        }
    }
    //有GCHandle,有Free，new的对象在Free之后的GC会立即销毁，不会等到函数调用结束。
    private void FreeGCHandleObject()
    {
        Debug.LogError("STEP0.BEGIN----------");
        var ptr = NewGCHandleObject();
        CallGCCollect();
        //备注1. 可以通过地址获取到Handle，并且可以获取到对象 handle.Target
        var handle = GCHandle.FromIntPtr(ptr);
        Debug.LogError("STEP1.FREE-----------");
        //备注2
        handle.Free();
        CallGCCollect();
        Debug.LogError("STEP2.END-------------");
    }
	//创建一个GCHandle Pinned的对象
    private IntPtr NewGCHandleObject()
    {
        var newObj = new TestGCHandleObject();
        newObj.Name = "Hello GCHandle!";
        var gcHandle = GCHandle.Alloc(newObj);
        //备注3.可以直接获取到对象的地址
        IntPtr newObjectPtr = GCHandle.ToIntPtr(gcHandle);
        return newObjectPtr;
    } 
}
```

输出结果：

```csharp
STEP0.BEGIN----------
STEP1.FREE-----------
~TestGCHandleObject Hello GCHandle!
STEP2.END-------------
```

可以看到，GCHandle可以直接获取到对象的地址（代码备注3），根据地址可以再次获取到对象（代码备注1）。这样为托管对象访问非托管对象提供了便利，Int32类型的地址在托管内存和非托管内存是可以直接传递的，不需要拷贝，内存是对齐的。

怎么获取到内存中所有的 GCHandle对象的？既然Unity是用的 il2cpp，会不会是il2cpp已经提供的接口，不然Unity引擎本身理论上也是不会存储这些数据的。看了下il2cpp的源码，确实il2cpp提供了获取接口：

```c++
void GCHandle::WalkStrongGCHandleTargets(WalkGCHandleTargetsCallback callback, void* context)
{
    lock_handles(handles);
    const GCHandleType types[] = { HANDLE_NORMAL, HANDLE_PINNED };

    for (int gcHandleTypeIndex = 0; gcHandleTypeIndex < 2; gcHandleTypeIndex++)
    {
        const HandleData& handles = gc_handles[types[gcHandleTypeIndex]];

        for (uint32_t i = 0; i < handles.size; i++)
        {
            if (handles.entries[i] != NULL)
                callback(static_cast<Il2CppObject*>(handles.entries[i]), context);
        }
    }
    unlock_handles(handles);
}
```

代码在 [GCHandle.cpp](https://github.com/MlgmXyysd/libil2cpp/blob/master/libil2cpp/Unity_2019.4/2019.4.0f1/gc/GCHandle.cpp)，可以看到GCHandle对象存储在数组链表结构的 gc_handles中， 根据这个函数可以遍历出来 所有GCHandle的对象。

**总结下，GCHandle 持有的托管对象是不会被释放的，直到调用Free。那么PackedMemorySnapshot里面的 gcHandles 数据即在内存中的所有GCHandle勾住的托管对象。**



##### managedTypes

managedTypes数组的元素类型为 PackedManagedType，这个对象保存了一个非常重要的信息 "对象类型地址"，看下定义：

```c++
public struct PackedManagedType
{
		// An array containing descriptions of all fields of this type.
        public PackedManagedField[] fields;
	 	
	 	// Size in bytes of an instance of this type. If this type is an arraytype, this describes the amount of bytes a single element in the array will take up.
        public System.Int32 size;

        // The address in memory that contains the description of this type inside the virtual machine.
        // This can be used to match managed objects in the heap to their corresponding TypeDescription, as the first pointer of a managed object points to its type description.
        public System.UInt64 typeInfoAddress;

        // The typeIndex of this type. This index is an index into the PackedMemorySnapshot.typeDescriptions array.
        public System.Int32 managedTypesArrayIndex;
        
        //...省略很多代码
}
```

*typeInfoAddress* 就是保存的对象类型地址。C# 代码在 il2cpp 之后，代码的类型、名称、父类、方法等等这些信息都会保存一个叫做 Il2CppClass的对象中，这使得编写的C#代码时使用的反射，泛型等C#语言的特性能在 il2cpp 之后的C++代码得以继承。可以看到 Il2CppClass对象的定义（上一篇已经讲过）:

```c++
typedef struct Il2CppObject
{
    union
    {
        //类型信息
        Il2CppClass *klass;
        Il2CppVTable *vtable;
    };
    MonitorData *monitor;
} Il2CppObject;
```

类型信息存放在对象的首地址，这也使得只要我们有类型信息对象 klass，这样我们就可以获取到对象的地址了。managedTypes 保存了所有类型信息，但是不是说直接遍历 managedTypes 能获取到所有对象。因为Il2CppObject中的klass对象是共享的，也就是只要类型相同，创建出来的Il2CppObject对象的 klass指向的是同一个对象，这个可以从Il2Cpp中的代码看到：

```c++
//MetadataCache.cpp
Il2CppClass* il2cpp::vm::MetadataCache::GetTypeInfoFromTypeIndex(TypeIndex index, bool throwOnError)
{
    if (index == kTypeIndexInvalid)
        return NULL;

    IL2CPP_ASSERT(index < s_Il2CppMetadataRegistration->typesCount && "Invalid type index ");

    //如果已经存在，则直接返回类型对象
    if (s_TypeInfoTable[index])
        return s_TypeInfoTable[index];

    const Il2CppType* type = s_Il2CppMetadataRegistration->types[index];
    Il2CppClass *klass = il2cpp::vm::Class::FromIl2CppType(type, throwOnError);
    if (klass)
    {
        il2cpp::vm::ClassInlines::InitFromCodegen(klass);
        s_TypeInfoTable[index] = klass;
    }

    return s_TypeInfoTable[index];
}
```

读了HeapExplorer的解析代码，可以通过 managedTypes 获取到所有的成员(fields)，然后再遍历静态成员（PackedManagedType的 fields）来索引引用的对象，这样一直遍历完所有的对象，相当于把所有被引用的对象都能遍历出来，这些被引用的对象就是内存中还存活的对象。这里我们可以总结出来：

```c++
托管对象集 = GCHandle对象集 + 静态对象及其被引用的所有对象集
```

那么我们就有了所有的托管对象了，这个也是HeapExplorer显示的托管对象列表的数据集。同样的，我在 il2cpp 的代码中找到了获取所有类型对象的接口：

```c++
//MemoryInfomation.cpp
void ReportIL2CppClasses(ClassReportFunc callback, void* context)
{
    const AssemblyVector* allAssemblies = Assembly::GetAllAssemblies();
    for (AssemblyVector::const_iterator it = allAssemblies->begin(); it != allAssemblies->end(); it++)
    {
        const Il2CppImage& image = *(*it)->image;
        for (uint32_t i = 0; i < image.typeCount; i++)
        {
            Il2CppClass* type = MetadataCache::GetTypeInfoFromTypeDefinitionIndex(image.typeStart + i);
            if (type->initialized)
                callback(type, context);
        }
    }
	//...省略一些代码
}
```

**总结下，拥有managedTypes数据，可以遍历到所有静态对象以及被静态对象引用的对象，再加上 GCHandle对象集就是完整的 托管对象集合了。**

到这里我们已经了解了 TakeSnapshot采集的托管对象数据结构，也了解了HeapExplorer怎么解析的托管内存Snapshot数据了。我们也可以自定定义采集数据做自动化监控功能：

<img src="/images/perf/unity_memory_2/3.png" alt="image-20241118210547014" style="zoom:70%;" />

---

引用：

1. [Total System Memory](https://discussions.unity.com/t/memory-profiling-question-used-total-vs-total-system-memory-usage/659501/2)

2. [The Truth About GCHandles](https://learn.microsoft.com/zh-cn/archive/blogs/clyon/the-truth-about-gchandles)

3. [Universal Windows Platform: Debugging on IL2CPP Scripting Backend](https://docs.unity3d.com/2019.4/Documentation/Manual/windowsstore-debugging-il2cpp.html)

