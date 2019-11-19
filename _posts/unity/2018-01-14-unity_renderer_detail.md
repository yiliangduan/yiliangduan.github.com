---
title: Unity渲染流程简述
date: 2018-01-14 16:22:10
type: "tags"
comments: false
tags: unity
---

> unity 4.7.1f1

之前了解Unity的setpass call和batches时把顺道把渲染这块的代码(4.7.f1)也阅读了下，先利用一个思维导图把渲染的几个大的步骤记下来，以便有个整体的脉络(这里只记录Forward渲染)。

![](/images/unity_renderer/1.png)

> 注1：由Camera的投影矩阵和对象的Z轴向量的积得出

<!-- more --> 

**RenderManager**

* RenderCameras
* RenderOffscreenCameras



**RenderCameras和RenderOffscreenCameras**

相同点:

* 遍历所有的渲染的相机(或离屏渲染的相机)，先让每个相机做裁剪操作，裁剪完之后得到裁剪之后的对象集合，然后让相机渲染这个集合。

不同点:

* RenderCameras是即时渲染的，即处理一个相机的Render就会马上渲染到场景上，接着再处理第二个相机。
* RenderOffscreenCameras处理所有的相机Render之后才渲染到场景上的。



**Cull**

现获取相机可见的场景上的所有Render，然后通过裁剪得到一个可见的对象集合。裁剪里面有个比较重要的参数就是相机使用了OcclusionCulling这个功能。使用了OnOcclusionCulling，则会调用Umbra进行Render在场景中的位置关系来处理。

* 范围检测
  * 没有开启OnOcclusionCulling, 根据相机的Plane和每个Render的Bounds做AABB检测。
  * 开启了OnOcclusionCulling, 使用Umbra的API对每个Render的Bounds做可见检测。

* 可见检测
  * 对象的renderer是否为空了。
  * 对象是否是disable状态，如果是则不需要渲染了。
  * 对象是否被Camera的lod功能剔除了。
  * 对象是否被Camera的LayerDistance剔除了。

经过Cull处理之后得到了一个渲染的CullResults的数据对象。CullResults包含了对应的一个相机的需要渲染的Render集合、灯光信息、阴影等信息。



**RenderLoop**

这个步骤就处理上一步得到的CullResults数据，遍历CullResults里面的Render集合，对每个Render的上使用的material信息(Material,RenderQueue,subShaderIndex,lightmap,staticBatchIndex,renderer等)进行收集。得到一个需要渲染的已material为基础的RenderObjectData对象集合。

有了这个数据，就可以开始进行渲染的操作了，接下来考虑的是使用Forward渲染还是Deferred模式渲染。不管那种渲染流程基本一致，这个考虑Forward渲染的方式。步骤如下:

* 渲染Opaque物体
* 渲染天空盒
* 处理ImageFilters
* 渲染透明物体

这几个步骤渲染完成之后真个一帧的Render渲染工作就完成了。



**DoForwardVertexRenderLoop**

处理RenderLoop中得到的渲染对象数据(RenderObjectData)集合。处理之后把得到需要渲染的一个数据集合PlainRenderPasses，每个元素的数据对应一个需要渲染的Pass和渲染这个Pass需要的其他数据(subshader、light、material等数据)。



**SortRenderPassData**

处理PlainRenderPasses，把每个Pass都按照导图里面列出的元素的顺序优先级排序，排序之后的顺序就是接下来要渲染的顺序了。这里面的排序规则还是比较多的，这也关乎到场景上的模型被渲染之后的显示问题。比如两个Renderer A和B，两者使用的都是透明贴图，需要做透明混合处理。A的RenderQueue设置为3000，B的RenderQueue设置为30001，那么排序之后A在B的前面渲染，这样A的Renderer渲染出来的物体不会遮挡住B。排序的规则大致和导图里面的是一致的。



**PerformRendering**

这个方法里遍历在DoForwardVertexRenderLoop中处理得到的PlainRenderPasses集合，上面提到这个集合的的每个元素是一个pass需要渲染的内容。遍历这个数据集合对需要的渲染的mesh做动态合批，处理不同的pass里面的阴影。做好这些之后所有要在CPU处理的事情都做好了。



**BatchRenderer**

PerformRendering里遍历PlainRenderPasses，会对每个pass需要渲染的renderer用BatchRenderer处理，把可以合并在一个pass中渲染的renderer记录在一个动态数组中，当接下来遍历的元素不可以合并到当前的pass的时候，这个数组的所有renderer马上进行一次合批操作，然后再记录记下来这个元素。重复操作，一直到PlainRenderPasses的所有元素遍历完成。合并的具体操作在Renderer(MeshRenderer、ParticleSystemRenderer和SpriteRenderer)的RenderMultiple操作里面完成。

**RenderMultiple**

这个函数会处理每个需要合并的的renderer集合。首先会判断是动态合并还是静态合并，然后对renderer集合做相应的动态或者静态合批处理。



**DrawVBO**
把合批之后的数据提交到GPU渲染。