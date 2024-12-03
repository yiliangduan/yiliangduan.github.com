---
title: 'Unity内存（一）了解Mono内存'
date: 2024-09-22 21:31:25
type: "tags"
tags: perf
comments: false
---


目录

- 什么是IL2CPP
- Mono内存分配流程
- BDWGC

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

到这里 *bdwgc* 的分配和释放规则我们已经了解。

---

引用:

1. [[Unity - Manual: Mono overview](https://docs.unity3d.com/Manual/Mono.html)](https://docs.unity3d.com/Manual/Mono.html)

2. [Unity3D托管堆BoehmGC算法学习-内存分配篇](https://juejin.cn/post/6966954993869914119)

3. [Unity3D托管堆BoehmGC算法学习-垃圾回收篇](https://juejin.cn/post/6968400262629163038)

4. [GitHub il2cpp各个版本的源码整理](https://github.com/4ch12dy/il2cpp)

5. [MemoryProfiler Package Manual](https://docs.unity3d.com/Packages/com.unity.memoryprofiler@1.1/manual/index.html)

6. [Unity Profiler Manual](https://docs.unity3d.com/Manual/Profiler.html)

7. [c/c++ backtrace打印函数调用栈](https://blog.csdn.net/jiangliuhuan123/article/details/130274179)

8. [iOS dSYM 文件 & 符号化](https://www.odszz.com/posts/ios-dsym-symbolicate/)

9. [利用dwarfdump从dsym文件中得到symbol](https://blog.csdn.net/xiaofei125145/article/details/50456614)
