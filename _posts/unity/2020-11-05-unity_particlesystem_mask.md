---
title: UI上的特效的裁剪问题
date: 2020-11-05 23:59:40
type: "tags"
comments: false
tags: unity,UGUI,unity2019.4.14f
---

> Unity2019.4.14f, UGUI

在UI（UGUI）界面的制作中，我们会遇到这种需求：在一个 *ScrollRect* 内显示若干个子物件UI，并且子物件UI需要显示粒子特效。这种情况我们会遇到超出 *ScrollRect* 需要裁剪粒子特效的问题。如：

![](/images/unity_particlesystem_mask/a.png)

<center>图[1]</center>

可以看到，超出 ScrollRect 的 SubItem 背景的 Image 裁剪掉了，但是粒子特效没有被裁减。不管是是用 RectMask2D还是 Mask 都是这样的结果。

要知道其中的原因我们先了解下 RectMask2D 和 Mask 这两个功能脚本。

**RectMask2D**  正如其名它只能对2D的对象进行裁剪，它是通过计算每个子节点对象的RectTransform的size来判定是否超出了设定的mask区域的，如果超出了则重新计算这个对象的大小并且调整顶点，把超出的部分丢弃掉。大概的操作如下：

```csharp
//m_clipTargets存储的是当前RectMask2D的所有可以被裁减的子节点。
//凡是继承了IClipable的控件都是可以被裁减的，比如Image
foreach (IClippable clipTarget in m_ClipTargets)
{
    if (clipRectChanged || forceClip)
    {
        clipTarget.SetClipRect(clipRect, validRect);
    }
    var maskable = clipTarget as MaskableGraphic;
    if (maskable != null && !maskable.canvasRenderer.hasMoved && !clipRectChanged)
        continue;
 
	//只有在RectMask2D区域内的子节点才会显示。如果RectMask2D本身被裁减了，
    //那么当前RectMask2D的所有子节点也会跟着一同被裁减掉。
    clipTarget.Cull(
        maskIsCulled ? Rect.zero : clipRect,
        maskIsCulled ? false : validRect);
}

```

正如图[1]中的第一个item，在超出 Scrollrect 的区域背景被裁减掉了。了解了 RectMask2D的工作原理，那么很显然 RectMask2D对粒子特效裁减不了，因为 ParticleSystem 不是 UGUI 模块的代码，不可能继承了 IClippable。

**Mask**的实现相比于 RectMask2D 更加低层，它是利用 模板测试 (*Stencil Test*) 来进行裁剪的。模版测试是渲染管线（Render Pipeline）中的流程，在GPU中执行的， RectMask2D 则是在CPU中执行。模版测试的原理其实很简单：根据屏幕像素的密度给定一个对应大小的 Stencil Buffer ，每个元素只需要1byte字节，也就是最大255的精度。然后在渲染的时候 Mask（UGUI会把UI的对象转换成Mesh去渲染）对 Stencil Buffer 指定的区域（Mask设定的裁剪区域）写入指定的值（UGUI里面是写入的1），当渲染在这个区域（*Stencil Buffer* 的值为1）的Mask子节点的时候会进行模版测试，比较Stencil Buffer值，满足条件就继续渲染，否则丢弃掉不渲染。例如一个4x4像素的buffer，然后mask的区域是中间四个像素，假设在渲染Mask节点之前没有任何材质对Stencil Buffer进行修改，那么Stencil Buffer在渲染Mask的时候的变化是这样的：

![](/images/unity_particlesystem_mask/b.png)

<center>图[2]</center>

接着渲染Mask的子节点的时候，如果对应的像素的 Stencil值为1，则通过测试继续渲染，如果对应的像素的 Stencil值为0，则丢弃掉（相当于裁减掉了）。

既然在Mask区域只要通过测试就可以渲染，粒子特效本身也是一个Render，有单独的渲染材质，那么我们是不是可以在粒子特效的材质里添加模版测试，并且把通过测试的值设置成UGUI的Mask设置的值一致就可以了。于是在例子特效的着色器中添加模版测试的代码：

```c++
Properties{
    //省略其他属性
	[Header(Stencil)]
	[Enum(UnityEngine.Rendering.CompareFunction)]_StencilComp("Stencil Comparison", Float) = 8
	_Stencil("Stencil ID", Float) = 0
	[Enum(UnityEngine.Rendering.StencilOp)]_StencilOp("Stencil Operation", Float) = 0
	_StencilWriteMask("Stencil Write Mask", Float) = 255
	_StencilReadMask("Stencil Read Mask", Float) = 255
}

Stencil {
	Ref[_Stencil]
	Comp[_StencilComp]
	Pass[_StencilOp]
	ReadMask[_StencilReadMask]
	WriteMask[_StencilWriteMask]
}
```

在Unity的Inpsector中对该着色器设置值如下：

![](/images/unity_particlesystem_mask/c.png)

<center>图[3]</center>

设置Stencil ID为1，正好是UGUI的Mask对Stencil Buffer设置的模板通过值。Stencil Comparsion值为为Equal，让当前粒子特效渲染在模版测试时 深度值等于1的时候通过，并且Stencil Operation为Keep，因为不需要改变Stencil Buffer的值。这个设置和Mask的其它UI节点一致。

修改之后发现粒子特效不可见了，也就是没有渲染出来了。间接证明了，Stencil Buffer里在粒子特效对应的像素的位置值不为1了，说明之前Mask写入的Stencil Buffer的值可能被重置掉了。这个就很奇怪了，到底哪一步做了清理工作呢？我在Unity的Frame Debugger中找到了答案。下图是 图[1]中的对象渲染时候的Frame Debug信息：

![](/images/unity_particlesystem_mask/d.png)

<center>图[4]</center>

可以看到UGUI渲染的最后一个Mesh的Stencil设置，相当于把Buffer里面为1的值重置为0了。而我们的粒子特效时在UI渲染之后渲染的（图中的Draw Dynamic），这个Mesh不渲染任何对象，纯粹做重置Stencil Buffer用的。进一步阅读了下Mask的代码，原理就清楚了。主要代码如下：

```csharp
 public virtual Material GetModifiedMaterial(Material baseMaterial)
 {
     //...省略部分代码
     //如果原先的buffer里面的值为0， 就简便处理，直接设置Buffer的值为1，然后利用另外一个Material来直接重置Buffer的值。
     if (desiredStencilBit == 1)
     {
         //设置Buffer为1的Material
         var maskMaterial = StencilMaterial.Add(baseMaterial, 1, StencilOp.Replace, CompareFunction.Always, m_ShowMaskGraphic ? ColorWriteMask.All : 0);
         StencilMaterial.Remove(m_MaskMaterial);
         m_MaskMaterial = maskMaterial;

         //重置Buffer为0的Material，这个Material就是我们在Frame Debugger中看到的UGUI渲染对象的最后一个Mesh
         var unmaskMaterial = StencilMaterial.Add(baseMaterial, 1, StencilOp.Zero, CompareFunction.Always, 0);
         StencilMaterial.Remove(m_UnmaskMaterial);
         m_UnmaskMaterial = unmaskMaterial;
         graphic.canvasRenderer.popMaterialCount = 1;
         graphic.canvasRenderer.SetPopMaterial(m_UnmaskMaterial, 0);

         return m_MaskMaterial;
     }

     //这里就考虑了在此之前Buffer的值不为0的情况，直接根据Buffer的值，然后计算出比现有Buffer值大的素数值（3，7 ...）
     //...省略

     return m_MaskMaterial;
 }
```

原来Mask中会创建两个Material，maskMaterial用来设置Stencil Buffer值，对应的unmaskMaterial是用来还原Stencil Buffer值的，也就是图[4]中看到的UGUI渲染的时候最后一个渲染的Mesh。到这里我们明白了为什么不管是RectMask2D和Mask都不能裁剪特效了。那么怎样可以裁剪特效呢?

第一种方法是让Mask的unmaskMaterial不重置Stencil Buffer值，而是在粒子特效渲染之后再重置Stencil Buffer的值。unmaskMaterial对Stencil的设置为StencilOp.Keep，保持不变。这种方法可以正确的裁剪掉粒子特效，测试下来也是可行的。不过需要我们修改UGUI的源码，另外一个问题是在ParticleSystem之后需要自己重置下Stencil Buffer的值。

第二种方法是使用RectMask2D来裁剪UI，我们自己来对RectMask2D区域写入指定的Stencil Buffer 值，然后对ParticleSystem的Render材质设置Stencil测试，通过测试才显示。设置如下：

![](/images/unity_particlesystem_mask/e.png)

<center>图[5]</center>

ParticleSystem的材质的设置保持和图[3]的设置一样。通过这样的设置可以看到粒子特效同样被顺利裁剪掉了。和第一种方法一样带来的问题是需要自己重置下Stencil Buffer的值。不过重置本身比较简单，可以让被裁减的 ParticleSystem 带第二个材质，第二个材质不渲染任何实际的对象，只修改Stencil Buffer值即可。

