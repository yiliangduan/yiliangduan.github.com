---
title: Unity Object缓存遇到的一个问题
date: 2018-11-29 20:26:10
type: "tags"
tags: unity
comments: false
---

> unity 5.6.6f1

今天在项目里面遇到一个BUG，一个脚本对一个Prefab的引用偶现丢失的情况。查证之后发现这个问题还比较特殊，记录一下。

问题是这样的: 有两个Prefab A和B，在场景中A和B分别 Instantiate出来a和b对象，a的父节点是其他的Prefab Instantiate的对象，但是b的父节点就是a。在a使用完成之后直接将a放进了ObjectPool里面回收了，然后对b也用ObjectPool对其同样进行回收。过一段时间之后重新用到a对象，然后直接从ObjectPool里面取出来。但是这个时候Instantiate b的时候发现a的Instantiate对象上已经存在b(b在回收的时候没有断开与父节点的关联，a在之后回收的时候也没有删除b子节点)，这个时候就继续使用已经存在的b，过了一段时间(正好ObjectPool里面的缓存对象的生存周期结束)ObjectPool把B给回收了，造成了对b引用的脚本空指针了。结构如图:

<!-- more --> 

![](/images/unity_objectpool_problem/1.png)

这个问题暴露了这个模块的代码的几个问题:
1. 对象a在ObjectPool回收的时候没有删除相对于A的多余的对象
2. 对象b在ObjectPool回收的时候没有断开与其他对象的引用
3. ObjectPool里面存在对象相互引用但是没有做出相应的警告或者处理

一个模块的代码健壮稳定必须得反复推敲反复测试。

*End*