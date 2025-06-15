---
title: '高性能UI事件分发系统'
date: 2023-09-30 10:50:25
type: "tags"
tags: perf
comments: false
---


> 问题反馈或者讨论：yiliangduan@qq.com 

> 最近优化了下项目里面的UI事件系统，主要原因是随着业务量的膨胀，不管是从可维护的角度，还是从性能的角度上现有的UI设计结构已经承载不了现在的业务需求。

## 现有的UI事件方案

现有的UI事件分发系统以血量更新为例：

![](/images/unity_perf_ui_event/old_version_ui_event_system.png)

逻辑层数据变化了（血量）同步推送到表现层的UI系统来做UI刷新，伪代码如下：

```csharp
//ActorHpComponent.cs
public void Broadcast_HpUpdateEvent(long value)
{
	EventRouter.Instance.Broadcast(EventKey.HpUpdate, actorID, value);
}
```

````csharp
//HpBarView.UpdateHp.cs
public void OnHpUpdate(uint actorID, long value)
{
    //需要判断事件是否来自自己绑定的ActorID发来的
	if (actorID == selfActorID)
    {
        UpdateHp(value);
    }
}
````

这种模式存在的问题：

* View可维护性差 （View指代HPBarView这种具体的UI，下同）。
* 事件频次太高，UI刷新频次不可控。

#### View的可维护性差

UI的所有功能都在View中实现（HpBarView），包括事件处理、数据处理、UI更新显示。代码没有固定的结构，每个程序会在View中自由发挥，导致各种风格代码，各种造轮子。

#### 控件刷新频次不可控

UI控件的刷新完全取决于逻辑的数据更新，如果你数据更新一帧更新多次的话，每次都会通过事件分发到UI，最后触发UI刷新。整个链路触发频率高会导致消耗上涨，虽然UGUI的Rebuild是固定每帧刷新一次的频率，但是每次SetDirty也是不小的开销（比如Text更新text时）。

```csharp
//ugui Text.cs
public virtual string text
{
    set
    {
        //省略很多代码
        m_Text = value;
    	SetVerticesDirty();
        //这里会计算根据value长度来重新计算Layout,比较耗时.而且只要text更新就会计算。
    	SetLayoutDirty();
    }
}
```

##### 事件频次太高
如果HpBarView的数量非常多，比如当前场景有50个HpBarView需要显示，那就是50个HpBarView在监听 *EventKey.HpUpdate* 事件，如果有30个Actor正好在同一帧都触发了一次Hp的更新，那就是会抛出50次 *EventKey.HpUpdate* 事件，那HpBarView的OnHpUpdate函数实际触发的次数：

```csharp
30 (事件次数) * 50 (每个事件触发OnHpUpdate次数) = 150次
```

显然，Actor的数量越多，OnHpUpdate触发次数越高。但是我们也看到实际有效次数为 *if (actorID == selfActorID)*  成立的次数，也就是事件次数30次才对。

## 改进方案

#### 视图View层的改进

首先需要调整的是View层的结构。我们可以参考UE5的 UMG Viewmodel系统来改造下View层结构。首先看下Viewmodel设计结构：

![](/images/unity_perf_ui_event/ue5_viewmodelworkflow.png)

> UMG Widget：可以看作是Unity里面的Prefab，并且是挂在了MonoBehaviour的UI脚本。
>
> Object in Application: 可以看作是我们的逻辑层的对象。

可以看到Object 有变量更新，会先发送到ViewModels层，ViewModels层然后发到Widget，最后Widget刷新显示。相比我们自己的旧的方案，这里对UMG Widget增加了一个Viewmodel，Viewmodel的工作可以概括以下几点：

* Viewmodel解耦了UMG Widget和 Object in Application，Object in Application层数据变化，首先是发送消息发到Viewmodel，Viewmodel负责管理这些数据，UMG Widget本身不需要有数据相关的内容，相当于把数据拆分出来单独管理。
* Viewmodel维护了UMG Widget需要变化的组件的列表，并且把UI 组件和对应的数据变量做了绑定，数据刷新直接触发UI刷新，简化了整个UI刷新的流程。

把UE5 的Viewmodel结构应用到自己的结构中，调整了如下：

![](/images/unity_perf_ui_event/ui_event_system_v2.png)


View层把空间的赋值刷新绑定到ViewModel的delegate，自己不做任何逻辑，被动响应刷新控件就行，代码规则非常明确，代码可以自动实现：

```csharp
// HpBarView.cs 这种View代码可以全自动实现,规则非常明确  => View层
public class HpBarView : MonoBehaviour
{
    public Text hpValueText;
    //这个函数写个工具自动生成
    public void BindVariables(HpBarViewmodel viewmodel)
    {
        //绑定好ViewModel的接口。
        viewmodel.SetHpValueUI = SetHpValueUI;
    }
    public void SetHpValueUI(long hpValue)
    {
        hpValueText.text = hpValue.ToString();
    }
}
```

ViewModel层负责接收事件数据，并且调用绑定好的控件delegate来触发UI空间更新到最新数据：

```csharp
//HpBarViewmodel.cs  => ViewModel层
public class HpBarViewmodel : ViewmodelBase
{
    //这种Delegate代码可以写个工具,在Prefab上通过用户选择绑定的变量和变量类型 自动生成好
    public SetHpValueUIDelegate SetHpValueUI;
    
    public void OnHpUpdate(uint actorID, long hpValue)
    {
        //Viewmodel这里维护selfActorID,HpVarView就不需要有Actor归属这类依赖数据的代码了。
        if (actorID == selfActorID)
            SetCurrentHpValue(hpValue);
    }
    public void SetCurrentHpValue(long hpValue)
    {
        SetHpValueUI(hpValue);
    }
}
```

至此解决了View维护性差的问题，按照规则View的代码完全是自动生成，完全是表现层代码。

#### 更新频次改进

要控制UI刷新频次，机制上需要做的就是控制事件触发到刷新UI控件的频次。理论上来说，从源头事件发送者不发送那么多次就行，但是这对编写代码的要求和繁琐度就高了。举个例子：

受击扣血，当一个角色同一帧被10个怪攻击到主角时，会触发10次扣血操作，也就是会发送10次扣血时间到UI层，UI层如果不对事件的频次做限制的话，那么这一帧主角的血条会刷新十次，十次刷新血量从10000扣到5000，从表现上来看一帧刷新十次由10000刷到5000和一帧刷新一次由10000刷到5000肯定是一样的。不过之类也有特殊情况，就是刷新血量可能本身附带逐步刷见的动画效果，这种情况理论上来说也是触发UI一次刷新：当前值10000到目标值5000的动画是最合适的。那么怎么来处理一帧事件次数过多的问题，这里我的处理方式是两种：

* 主动轮询获取数据方式：事件消息不直接抛给UI，而是放到一个数据池里面，UI定时轮询来获取数据。
* 事件合并分发：事件消息不直接抛给UI，放到一个消息管理池中，每帧定时发送到对应的监听着，如果一帧收到多个事件，这合并事件的数据。

这里可能会有个疑问，为什么从逻辑层触发的UI数据，必须需要事件，是否可以直接调用UI刷新？ 对于小项目来说，这种都是无所谓的，甚至不需要考虑这方面的性能，但是对于大项目来说，逻辑和表现的分离非常有必要，而且当逻辑非常重度的时候，很多游戏逻辑直接放在单独的子线程里面跑，这种情况逻辑层的到表现层的事件可能还是作为不同线程数据交换的一种方式，关于逻辑子线程这块内容比较多，后面我单独会写几篇来介绍下。

##### 主动轮询获取数据

这种方式的好处是每个UI可以根据自己需要的频率获取数据来刷新UI，但是缺点也很明显：

* 很多UI更新的次数比较低频，如果一直在Tick里面轮询获取数据，本身是一种开销浪费。
* 如果每个UI都自己写一套定时在Tick里面轮询获取数据的代码，这个代码不可控也不好维护。

##### 事件合并分发

这种方式是基于普通的逻辑层的事件发送直接到表现层的事件响应的一个改进，先看设计结构：

![](/images/unity_perf_ui_event/ui_event_combine_and_dispatch.png)








---

引用:

1. [UMG Viewmodel](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/umg-viewmodel)

