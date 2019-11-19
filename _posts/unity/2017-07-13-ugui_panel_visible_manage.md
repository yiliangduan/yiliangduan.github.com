---
title: Unity3D的UIPanel隐藏的处理方式
date: 2017-07-13 23:40:10
type: "tags"
comments: false
tags: unity
---

1. 隐藏的方式

   1.1 设置Active为false方式。这种方式UIPanel的所有脚本逻辑、Rendering、Canvas.BuildBatch都会停止。构建的渲染的Mesh也会被清理。

   1.2 设置不被渲染的Camera的Layer层上。这种方式相当于只是停掉了Rendering，其他的逻辑照常执行。

2. 优缺点比较

   2.1 方式1隐藏Panel的时候构建的Mesh会被清理掉，会节省这部分内存。而且在动态Panel的时候(Panel上包含移动变化的GameObject)，Canvas的BuildBatch不会再调用，这会节省CPU(节省的程度根据Panel的复杂度决定的)。但是这种方式在隐藏和重新显示的时候由于需要重新构建渲染的Mesh，会消耗一定的CPU和产生GC.Allocate。

   2.2 方式2的好处在于Panel隐藏和重新显示的时候不需要重新构建渲染的Mesh，不会产生额外的CPU消耗和GC.Allocate。但是这种方式在隐藏的时候Panel的所有逻辑任然在执行(Update等)，在动态Panel的话Canvas的BuildBatch也会照常执行，这会一直消耗CPU资源。