---
title: Unity中内存资源释放总结
date: 2016-06-18 12:03:10
type: "tags"
comments: false
tags: unity
---

**Unity内存释放** 问题主要考虑一下几个方面:
  (1)  Resources 加载的资源需要释放的时候调用 Resources.UnloadUnusedAssets();
  

  (2) WWW 加载的任何资源<font color=#FF0000>必须</font>调用WWW的Dispose接口

  (3) 创建的RenderTexture<font color=#FF0000>必须</font>调用Release接口

  (4) 必须用Xcode联机调试确定C++层有没有内存泄漏

  (5) AssetBundle资源在Unity Profile中的Scene Memory不会显示，可以通过Xcode联机调试确定有没有释放完全
  
  (6) 加载资源和卸载资源的异步处理,避免同一帧内加载过多资源导致内存瞬间增大从而导致Crash(低配机器上表现更明显)

一般通过WWW加载一张Texture来通过Image显示的话，一种的方法是使用Texture通过Sprite.Create来创建一个Sprite对象赋值给Image.overrideSprite完成显示。但是需要注意一个问题，Sprite.Create创建对象对拷贝Texture中数据，并生成一个新的Sprite对象，这样会造成耗费将近2倍Texture大小的内存，而且这个内存占用不会随着Image的销毁而马上销毁，知道你调用Resources.UnloadUnusedResources()接口才会销毁。如果在一个场景中有大量的图片需要动态加载的话，那么必须注意内存问题了。所以建议不要使用显示Image的方式，有一种方式是显示RawImage，按照官方文档的描述:

>The Raw Image control displays a non-interactive image to the user. This can be used for decoration, icons, etc, and the image can also be changed from a script to reflect changes in other controls.

RawImage是一个没有和用户交互的Image，所以如果我们不需要比如事件这种交互的操作的话，还是建议用RawImage来显示Texture。RawImage的方式会高效很多，如果从AssetBundle里面加载了一张Texture，不管是赋值给多少个RawImage，Texture是公用的，只会有一分内存。