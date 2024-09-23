---
title: '一文了解 Mono 内存'
date: 2024-09-22 00:31:25
type: "tags"
tags: perf
comments: false
---


目录

- 什么是IL2CPP
- Mono内存分配流程
- BDWGC
- Mono内存监控

> 目录的主题之间存在相互依赖，前面三个主题是为了了解内存，第四个 Mono 内存监控是基于前三个主题的知识做的监控工具。
>
> 注：1. 本文的 Mono 内存全部指 Unity 开启 IL2CPP之后构建的手机包运行时的 Mono 内存。
>
> ​        2. BDWGC 指 Boehm-Demers-Weiser Garbage Collector，用在标题是全部为大写，用在文中时方便阅读我全部小写。

### 什么是IL2CPP

引用[Unity文档中]([IL2CPP Overview - Unity 手册](https://docs.unity3d.com/cn/current/Manual/IL2CPP.html))说的:

> The **IL2CPP** (Intermediate Language To C++) scripting backend is an alternative to the Mono backend. IL2CPP provides better support for applications across a wider range of platforms. The IL2CPP backend converts MSIL (Microsoft Intermediate Language) code (for example, C# code in scripts) into C++ code, then uses the C++ code to create a native binary file (for example, .exe, .apk, or .xap) for your chosen platform.

对Unity来说，就是在项目构建时，将从C#编译后的IL代码转换为Cpp代码，用来替代Mono。下面我们用一个例子来说明下。

首先我们在C#中实现了名字为 *BehaviourBase* 的类，如下：

```cs
// BehaviourBase.cs 注意：这里的MonoBehaviour是C#的类
public class BehaviourBase : MonoBehaviour
{
    public int m_guid = 0;
}
```

构建iOS包之后，我们可以从Xcode工程里面找到C# IL2CPP之后的代码，如下:

```cpp
// Assembly-CSharp.cpp  BehaviourBase
struct  BehaviourBase_t75F1477520D10A275C6158D6024AAD65240A0A12  : public MonoBehaviour_t4A60845CF505405AF8BE8C61CC07F75CADEF6429
{
public:
    // System.Int32 BehaviourBase::m_guid
    int32_t ___m_guid_4;
}

// 可以看到C#的MonoBehaviour变成了空壳
// 因为C#层的MonoBehaviour本身就是C++MonoBehaviour的壳子
struct  MonoBehaviour_t4A60845CF505405AF8BE8C61CC07F75CADEF6429  : public Behaviour_tBDC7E9C3C898AD8348891B82D3E345801D920CA8
{
public:
public:
};
```

再看下追溯下MonoBehaviour的基类继承关系，最终我们可以追溯到MonoBehaviour继承自Object类:

```cpp
// Assembly-CSharp.cpp UnityEngine.Object
struct  Object_tAE11E5E46CD5C37C9F3E8950C00CD8B45666A2D0  : public RuntimeObject
{
public:
    //这个 ___m_CachedPtr_0存储的就是C#绑定的C++对象的地址
    intptr_t ___m_CachedPtr_0;
    //省略
};
```

对应的C#中的UnityEngine的Object类:

```cs
// Object.cs
public partial class Object
{
    IntPtr   m_CachedPtr;
    //省略
}
```

对比可以看到，IL2CPP之后，Object类改为继承自了RuntimeObject，那么RuntimeObject是什么呢？我们可以从IL2CPP的代码中找到定义，定义如下：

```cpp
#if RUNTIME_MONO /* 是否是用mono,如果否，则用IL2CPP */
typedef MonoObject RuntimeObject;
#else
typedef Il2CppObject RuntimeObject;
#endif

//Il2CppObject保存了C#对象的类型信息, 虚函数等信息, 
//用于模拟C#运行时对象类型信息的查询、方法调用和多态性等操作。
typedef struct Il2CppObject
{
    union
    {
        // 保存类型信息，包含C#类的元数据(类名、方法等)
        Il2CppClass *klass;
        // 保存类的虚函数表，用于支持C#类的多态
        Il2CppVTable *vtable;
    };
    MonitorData *monitor;
} Il2CppObject;
```

根据继承关系我们可以得知，C#代码在Il2CPP之后，对象的内存应该是这样的：

```cs
+--------------------+
| Il2CppObject       |
| - klass/vtable 4b  |
| - monitor      4b  |
+--------------------+
| UnityCSharpClass   |
| - xxxField         |
+--------------------+
```

此外因为有了 *klass / vtable*  数据，对应的Cpp对象就可以获取对象的类型信息，可以实现C#的反射这类功能了。

总结下整个流程如下：

<img src="/images/mono_memory_intro/il2cpp_flow.png" alt="IL2CPP流程图" style="zoom:60%;" />

### Mono内存分配流程

了解了IL2CPP的过程，我们现在回到内存部分。我们都知道C#代码运行时内存是由mono运行时托管的，开发者在创建对象的时候不需要自己显示分配内存和释放内存，使用的是GC。IL2CPP之后，内存则由il2cpp运行时托管，使用的是BoehmGC。

接下来用一个例子来演示下il2cpp的内存托管过程，首先如果在C#层调用如下代码创建一个对象：

```cs
// Test_Il2Cpp.cs 
private static void CreateNode()
{
    //这里new Node对象, 内存是有mono的gc托管
    Node newNode = new Node();
    //do samething...
}
```

IL2CPP之后，我们看上面代码中的  *new Node()* 代码:

```cpp
// Assembly-CSharp.cpp 
IL2CPP_EXTERN_C IL2CPP_METHOD_ATTR void NodeFactory_CreateNode_m5E30EE5A6D660B15B3B6AADAAF00D416B84BFE12 (
{
    //省略
    Node_txxx * L_7 = (Node_txxx*)il2cpp_codegen_object_new(Node_txxx_il2cpp_TypeInfo_var);
    Node__ctor_m72145ADDB949F121912EEA2AAB5138ADEAF80EE4(L_7, /*hidden argument*/NULL);
}
```

可以看到 *new Node* 转换成了 *il2cpp_codegen_objec* 函数来创建，该函数最终会调用到*NewAllocSpecific* 再看下函数的实现：

```cpp
//il2cpp-codegen-il2cpp.cpp
Il2CppObject* Object::NewAllocSpecific(Il2CppClass *klass)
{
    Il2CppObject *o = NULL;
    //省略...
    if (!klass->has_references)
    {
        //调用 ALLOC_PTRFREE分配内存
        o = NewPtrFree(klass);
    }
//Unity2019这个宏是默认开启的, 宏是控制否启用 GC 描述符。
//述符的作用是帮助垃圾回收识别托管对象的引用，可以更准确的标记和回收对象
//在iOS平台测试是启用GC描述符的，其他平台没有测试。
#if IL2CPP_HAS_GC_DESCRIPTORS
    else if (klass->gc_desc != GC_NO_DESCRIPTOR)
    {
        //调用 ALLOC_TYPED 分配, 包含了klass这个Il2CppClass的Type信息
        o = AllocateSpec(klass->instance_size, klass);
    }
#endif
    else
    {
        //调用 ALLOC_OBJECT 分配,也包含了Il2CppClass的Type信息
        o = Allocate(klass->instance_size, klass);
    }
    //省略...
}
```

*NewAllocSpecific* 不同的分配函数最终都会调用到 Object.cpp中的内存分配宏定义(如下代码，内存分配接口收敛在这里对后面内存监控很有用)， 这三个宏定义的是 *bdwgc* 的分配接口。到这里我们可以看到，IL2CPP对象创建的内存分配直接是被 *bdwgc* 接管的。*bdwgc* 最终在在分配的是调用系统接口分配的是系统的虚拟内存，这个后面再讲。

```cpp
//Object.cpp

// NewPtrFree 调用, 定义在 malloc.c 中, 方法GC_malloc_kind_global
#define ALLOC_PTRFREE(obj, vt, size) do { (obj) = (Il2CppObject*)GC_MALLOC_ATOMIC ((size)); (obj)->klass = (vt); (obj)->monitor = NULL;  } while (0)

// Allocate 调用, 定义在 malloc.c 中, 方法GC_malloc_kind_global
#define ALLOC_OBJECT(obj, vt, size) do { (obj) = (Il2CppObject*)GC_MALLOC ((size)); (obj)->klass = (vt); } while (0)

// AllocateSpec调用, 定义在 gcj.mlc.c 中, 方法为 GC_core_gcj_malloc
#define ALLOC_TYPED(dest, size, type) do { (dest) = (Il2CppObject*)GC_gcj_malloc ((size),(type)); } while (0)
```

总结下整个流程图如下：

<img src="/images/mono_memory_intro/mem_alloc.png" style="zoom:60%;" />

### BDWGC

了解 *bdwgc* 的内存分配规则，我们才能知道Unity的Mono内存增长的规则。 *bdwgc* 指的是 Boehm-Demers-Weiser Garbage Collector，*bdwgc* 的作用主要是自动内存分配和释放的管理。下面讲内存的  '分配' 和 '释放' 两部分分别讲解下。


##### BDWGC 内存分配
bdwgc 的内存分配主要关注三个数据结构:

* GC_hblkfreelist : 维护了所有预分配或者可用的内存块，使用数组链表结构管理。这些内存是直接调用系统内存分配接口分配好的虚拟内存。
* ok_freelist : 维护了小块内存对象，使用数组链表结构来管理。这里维护的内存都是指向GC_hblkfreelist 的内存，这里只是做一个快速分配的管理。
* GC_obj_kinds：维护不同类型内存的数组，每个数组元素对象都包含一个ok_freelist。内存主要包括：
  * NORMAL：普通的内存性，包含指针和非指针的数据，比如IL2CPP 内存分配流程图中的 GC_malloc分配的内存。
  * PTRFREE：不包含指针的对象。 GC_malloc_atomic分配的内存，这些对象在垃圾回收的过程中不需要扫描，比如整形数据，元素是值类型。

看下 *ok_freelist*  和 *GC_hblkfreelist* 定义：

```c
//GC_obj_kinds的每个元素里面都定义了一个 ok_freelist 数组链表
GC_EXTERN struct obj_kind {
   void **ok_freelist;  /* Array of free list headers for this kind of  */
                        /* object.  Point either to GC_arrays or to     */
                        /* storage allocated with GC_scratch_alloc.     */
   // 省略...
} GC_obj_kinds[MAXOBJKINDS];

// N_HBLK_FLS 的大小为 60
struct hblk * GC_hblkfreelist[N_HBLK_FLS+1] = { 0 };
                             /* List of completely empty heap blocks */
                             /* Linked through hb_next field of      */
                             /* header structure associated with     */
                             /* block.  Remains externally visible   */
                             /* as used by GNU GCJ currently.        */
struct hblk {
    char hb_body[HBLKSIZE];
};
```

下面用一个分配流程来描述下内存分配的流程，流程不是很复杂：

<img src="/images/mono_memory_intro/alloc_flow.png" style="zoom:58%;" />

分配过程中有两点需要注意：

* 不管是 *ok_freelist* 还是  *GC_hblkfreelist*  内存都是 4 的整数倍，这个在 *bdwgc* 的规定，当业务创建对象分配内存的时候，如果不是 4 的整数倍，在 *bdwgc* 分配的时候，会先根据申请的大小向上匹配到 4 整数倍内存大小，这个大小就是 *bdwgc* 中实际分配的内存。

* GC_hblkfreelist 在分配一块内存的时候会选择性拆分几块，一块内存回收的时候会尝试合并下前序和后序内存块。

基于以上两点会导致我们常说的内存碎片问题，因为实际分配的内存大部分情况下是大于申请的内存的，这样就造成多余的这部分内存用不上。而且大内存块拆分成小内存块之后，造成内存池中大内存块逐步减少，业务层程序再次分配大内存块又不得不向系统请求分配，尽管现在池子当中还有很多可用但是不连续的内存。

#### BDWGC 内存释放

*bdwgc* 内存是释放有两种方式：

* 主动调用：业务层调用 *GC_gcollect(void)* 函数触发GC，Unity中对应C#接口的 *GC.Collect()*。
* 自动调用：在请求内存分配时，如果内存池的内存不够了触发 GC。

触发GC有两种模式：增量和全量。代码里面是通过 *GC_incrmental* 来区分的。可以看下请求内存分配时触发GC的区别：

```c
GC_INNER ptr_t GC_allocobj(size_t gran, int kind) {
    //1. 从ok_freelist中取
    void ** flh = &(GC_obj_kinds[kind].ok_freelist[gran]);
    //省略...
    //ok_freelist中获取失败
    while (*flh == 0) {
        //省略很多代码... (这里有次增量gc和重新分配)
        if (NULL == *flh) {
            //2. 去内存池 GC_hklbfreelist中取. 3.没有取不到从系统分配
            GC_new_hblk(gran, kind);
            if (GC_incremental && GC_time_limit == GC_TIME_UNLIMITED && !tried_minor && !GC_dont_gc) {
                //增量GC
                GC_collect_a_little_inner(1);
            } else {
                //全量GC
                if (!GC_collect_or_expand(1, FALSE, retry)) {/*省略*/}
          }}}
    return (ptr_t)(*flh);
}
```

上面降到的是调用层面的区别，对于增量和全量GC本身内存的区别在于，增量GC是把一次全量的GC分散到多帧执行，避免一次GC卡主太长的时间。全量GC就是执行一次，会从内存对象 GC_static_roots 做一次完成的标记扫描回收的过程。看下两者的代码区别：

```c
//alloc.c 增量回收
GC_INNER void GC_collect_a_little_inner(int n) {
    //省略...
    for (i = GC_deficit; i < 10 * n; i++) {
        //标记阶段
    	if (GC_mark_some(NULL))
        	break;
    }
    //清理回收
    GC_finish_collection();
}
```

```c
//alloc.c 
GC_INNER GC_bool GC_collect_or_expand(word needed_blocks, GC_bool ignore_off_page, GC_bool retry){
    //尝试回收内存。
    gc_not_stopped = GC_try_to_collect_inner(
                        GC_bytes_allocd > 0 && (!GC_dont_expand || !retry) ?
                        GC_default_stop_func : GC_never_stop_func);
    if (gc_not_stopped == TRUE || !retry) {
        // 执行成功的话直接返回, 注意 retry GC_allocobj调用过来都是为FALSE的
        return(TRUE);
    }
    //回收内存失败的话，直接扩容
    if (!GC_expand_hp_inner(blocks_to_get)
        && (blocks_to_get == needed_blocks
            || !GC_expand_hp_inner(needed_blocks))) {/*省略*/}
}
//如果是手动调用GC_gcollect,内部调用的接口就是此接口
GC_INNER GC_bool GC_try_to_collect_inner(GC_stop_func stop_func)
{
    //清除所有引用标记
    GC_clear_marks();

    //暂停所有线程(不包括GC自己)，遍历GC_static_roots标记所有的未引用对象(GC_mark_some),结束之后会恢复暂停的线程
    if (!GC_stopped_mark(stop_func)) {/*省略*/}

    //回收内存（会清空ok_freelist，重新构建ok_freelist）
    GC_finish_collection();
}
```

上面提到一个概念，在回收内存之前会先遍历 节点来标记未被引用的对象。这里解释下 *GC_static_roots*，先看下定义：

```cpp
//gc.priv.h GC_static_roots的定义
#define GC_static_roots GC_arrays._static_roots
struct roots _static_roots[MAX_ROOT_SETS];

struct roots {
	ptr_t r_start;/* multiple of word size */
	ptr_t r_end;  /* multiple of word size and greater than r_start */
    #if !defined(MSWIN32) && !defined(MSWINCE) && !defined(CYGWIN32)
    	struct roots * r_next;
	#endif
}
```

*GC_static_roots* 的构建是在库加载的时候，库加载的时候会将里面定义的全局变量和静态变量添加进来。回收时在 *GC_mark_same* 里，如果是全量GC，会将 GC_static_roots的所有对象添加到内存回收的标记栈里：

```cpp
//mark_rts.c
GC_INNER void GC_push_roots(GC_bool all, ptr_t cold_gc_frame GC_ATTR_UNUSED)
	/* Mark everything in static data areas.*/
	for (i = 0; i < n_root_sets; i++) {
        GC_push_conditional_with_exclusions(GC_static_roots[i].r_start, GC_static_roots[i].r_end, all);
	}
	//...
}
```

到这里 *bdwgc* 的分配和释放规则我们已经了解（*TODO: 这里补充一个流程图*）。

### Mono内存监控

> 注意： 本文说的mono内存都是指 IL2CPP模式下的mono内存。

Unity开发的程序，内存主要分为：托管内存（Mono内存），Native内存（引擎自己分配的内存，包括资源），C#非托管内存（比如Job中提供的NativeArray）。正常情况下我们只需要考虑托管mono内存和Native内存，C#非托管内存用的比较少，而且内存占用的比较少。

项目开发过程中，随着业务代码量增加，游戏逻辑越来越复杂，我们经常遇到性能测试出来的Mono内存过高的问题。针对这个问题，现有的开发条件下我们的定位手段：

* MemoryProfiler工具
* Profiler 工具

两个工具都是需要使用 development 包来采集数据，使用上有写差别。

#### MemoryProfiler

MemoryProfiler工具一次只能采集一帧数据，相当于把一帧的内存数据Dump出来，如下图（图片来自MemoryProfiler Package文档）：

<img src="/images/mono_memory_intro/memory-profiler-window.png" alt="img" style="zoom:50%;" />

##### 优点

* 对查某一帧内存问题的话还比较有效，可以很详细的看到这一帧中内存驻留了哪些对象，每个对象对象占用了多少内存，被什么对象引用了。

##### 缺点

* 不能方便查出整场单局的内存会分配的非常高的问题。因为撑高内存的对象很可能是分配了短期内又释放了这种，这种情况MemoryProfiler很难恰好Dump到这一帧。除非每帧一次Dump，但是MemoryProfiler本身一次Dump巨卡，所以每帧Dump非常不现实。

#### Profiler

Profiler工具是实时记录的每帧GC.Collect数据，这个数据可以用 ProfilerAnalyzer工具做个统计来看会更方便点，如下图（注：图片来自Unity ProfilerMemory docs）。

<img src="/images/mono_memory_intro/profiler-memory-simple-view.png" alt="The Simple view with some Profiler data loaded" style="zoom:67%;" />

##### 优点

可以监控到业务代码中所有触发GC.Alloc和GC.Collect。

##### 缺点

需要Profiler性能桩插的非常细，否则是定位不到具体函数的。性能桩插的足够细又会导致Profiler极度影响性能，甚至是影响游戏正常逻辑。

可以看到这两种方法都有很明显的弊端，很难在大规模的项目里面要很快速的解决问题。我们需要什么？

* 能够记录到任何一次Mono内存的分配，并且能够反定位到分配来源，类似于一个深层次并且带上行号的堆栈。可以方便的定位问题来源。
* 能够获取到每次分配的对象类型，这个便于统计我们内存分配情况，可以方便的确定优化点。
* 能够很轻松的对比多局数据，查看分配的差异。可以方便的对比不同版本相同单局的分配差异，定位内存增长部分的问题。

基于这些需求，现有的工具显然无法满足，那么我们就需要自己动手，打造一个更加 Powerful 的工具。

#### MemTracer

##### 方案思路
本文最开始的部分，我们介绍了Mono内存的分配流程，其实就是为这里做铺垫的。在Mono分配流程中分析到，Unity的Mono内存分配最终统一收敛到三个分配宏：ALLOC_PTRFREE、ALLOC_OBJECT 和 ALLOC_TYPED。

```cpp
//Object.cpp
#define ALLOC_PTRFREE(obj, vt, size) ...
#define ALLOC_OBJECT(obj, vt, size) ...
#define ALLOC_TYPED(dest, size, type) ...
```

如果我们在每次调用分配宏的时候记录下分配信息，就可以有完整的 Mono内存的分配列表了，流程是这样的（绿色部分是可以新增监听的流程）：

<img src="/images/mono_memory_intro/alloc_event.png" style="zoom:60%;" />

我们新增 *OnAllocEvent* 函数插入在 ALLOC 宏定义后面，每次 ALLOC分配完成自动调用 *OnAllocEvent*，我们就可以记录分配的信息了:

```c++
//Object.cpp
#define ALLOC_PTRFREE(obj, vt, size)  { ... OnAllocEvent(obj,  size);}
#define ALLOC_OBJECT(obj, vt, size)   { ... OnAllocEvent(obj,  size);}
#define ALLOC_TYPED(dest, size, type) { ... OnAllocEvent(dest, size);}
```

另外这里的记录我们先缓存下来，然后在业务层每帧触发把缓存输出到本地文件中，把一帧的数据输出作为一次输出。 

##### 分配采样

找到了记录内存分配的时机，那么我们接下来要考虑的是具体记录什么数据。首先根据我们的需求目标，我们需要的是：类型信息、分配大小和分配堆栈，我们一个一个来看。

###### 类型信息

类型信息其实很好获取，ALLOC的分配的对象类型都是Il2CppObject，前面已经分析了Il2CppObject结构，对象拥有类型为 Il2CppClass* 的成员变量 klass对象，klass 包含了类型名字：

```c++
typedef struct Il2CppClass
{
    const char* name;
    //...
}
```

###### 分配大小

ALLOC参数带了申请分配的大小 *size*，但是通过前面的 *bdwgc* 部分了解，申请的大小和实际的分配大小其实不一样，实际的分配大小会内存强制对齐到 4 的整数倍。这个 *bdwgc* 提供了一个根据对象地址获取对象内存的方法：

```c++
//gc.h
GC_API size_t GC_CALL GC_size(const void * /* obj_addr */) GC_ATTR_NONNULL(1);
```

###### 分配堆栈

获取堆栈可以直接用 c++里面 execinfo 的获取堆栈方法 *backtrace* , 保存堆栈的做法在 *bdwgc* 自己的调试中也有类似的功能，并且也是用的 *backtrace* 方法：

```c++
//os_dep.c
GC_INNER void GC_save_callers(struct callinfo info[NFRAMES])
```

三个数据都已经准备好了，接下来看下怎么保存这些数据。

首先提出保存这些数据的要求：

* 数据尽量足够小，因为我们要保证最终的输出文件尽量小，然后数据尽量小可以保证采集内存时本身的性能开销相对较小。
* 以一帧的数据为单位，这样我们在工具显示的时候可以按照帧的趋势来显示数据，便于阅读。

基于以上两点这里确定了下使用json紧凑的格式来存储采集的内存分配数据，下面是定义的记录内存分配数据的结构，一帧数据就用一个 *MonoMemorySnapshot* 来保存：

```csharp
//单个类型的对象，在一帧中分配的信息
[Serializable]
public class MonoMemoryRaw
{
    //类型
    public string t;
    //t申请内存的总大小
    public int s;
    //t真实分配的内存总大小
    public int r;
    //t类型所有分配的堆栈
    public List<string> st;
    //每个堆栈对应分配的内存大小
    public List<int> sz;
}
//一帧内所有对象分配的信息
[Serializable]
public partial class MonoMemorySnapshot
{
    //帧号
    public long f;
    //分配数据
    public List<MonoMemoryRaw> d;
}
```

定义好了数据结构，然后直接实现采样功能即可。

##### 地址解析

采样之后的数据是这样的：

![](/images/mono_memory_intro/mem_encode.png)

上面截取了两帧的数据，对于这个数据我们还需要做最后一步处理，就是把记录的地址解析成真实的堆栈。因为 *stacktrace* 记录的是堆栈地址（stack address），如果想要转换成代码的堆栈的话，需要转换成符号地址：

```c
symble address = stack address - slide address
```

这里的 *slide address* 是在程序加载的时候的基准地址，对于Unity工程我们直接获取UnityFramework的slide地址即可，获取出来可以保存再内存分配数据文件的头信息中，方便后面工具解析。然后解析直接用 macOS提供的 *atos* 指令即可。

```c++
atos -o 'dSYMs path' -arch arm64 -l 0x000000 {symble address}
```

这里有个点需要注意，如果项目规模比较大的情况并且内存分配的非常频繁，相应的内存分配记录文件也会比较大，记录的地址会非常多，会有几十万的地址需要解析。我们不要一个一个地址取解析，这将解析6小时+。我采取的策略是这样的：

* 因为同一个类型的对象，可能在不同地方分配，但是最终还会调用到公用的接口再触发底层分配，基于这种情况，肯定有很多记录的地址是相同的。我们可以先把记录的地址全部取出来，然后进行合并，合并之后数量会少很多。
* atos解析地址可以同时解析多个，因为本身启动atos程序就有开销，如果一次解析多个可以避免频繁启动atos。具体多少个，最好是直接用系统参数上限的个数，系统参数个数可以通过 *os.sysconf("SC_ARG_MAX")* 获取到。
* 由于解析地址本身是互相不依赖的，我们可以开启多线程解析。

采取以上步骤之后，我这边40万+的地址解析时间从原来的6小时+，减少到了60秒左右。

![](/images/mono_memory_intro/4.png)

到此数据都准备好了。

##### 展示工具

我们直接写个展示工具，针对展示数据的工具我们可以参考Profiler，ProfilerAnalyzer工具的布局。先细化下工具的要求：

* 能看到每帧分配的所有对象列表，每个对象类型的分配聚合在一起，能看到该对象的所有分配堆栈来源。
* 能看到整个单局分配的TOP30，这样便于全局知道我们内存是由哪些对象的分配造成膨胀的。
* 能够对比两局不同的内存采样数据，便于对比出内存差异，查出内存增长点。

基于以上需求，开发出来的工具如下：

###### Frame视图模式
MonoTracer统计了每一帧IL2CPP对象分配，每个类型的对象分配都会记录所有分配来源，每个来源都有详细的调用堆栈，根据这个堆栈可以定位到业务层的分配调用。

![](/images/mono_memory_intro/1.png)

###### TopN视图 
工具可以列出TopN的分配详情，根据这个分配排行可以来优化IL2CPP内存。

![](/images/mono_memory_intro/2.png)

###### Compare视图
工具可以列出TopN的分配详情，根据这个分配排行可以来优化Mono内存。

![](/images/mono_memory_intro/3.png)

有了工具之后，就可以很方便的定位出来内存过高的来源了。



---

引用:
1. [[Unity - Manual: Mono overview](https://docs.unity3d.com/Manual/Mono.html)]
2. [Unity3D托管堆BoehmGC算法学习-内存分配篇](https://juejin.cn/post/6966954993869914119)
3. [Unity3D托管堆BoehmGC算法学习-垃圾回收篇](https://juejin.cn/post/6968400262629163038)
4. [GitHub il2cpp各个版本的源码整理](https://github.com/4ch12dy/il2cpp)
5. [MemoryProfiler Package Manual](https://docs.unity3d.com/Packages/com.unity.memoryprofiler@1.1/manual/index.html)
6. [Unity Profiler Manual](https://docs.unity3d.com/Manual/Profiler.html)
7. [c/c++ backtrace打印函数调用栈](https://blog.csdn.net/jiangliuhuan123/article/details/130274179)
8. [iOS dSYM 文件 & 符号化](https://www.odszz.com/posts/ios-dsym-symbolicate/)
9. [利用dwarfdump从dsym文件中得到symbol](https://blog.csdn.net/xiaofei125145/article/details/50456614)
