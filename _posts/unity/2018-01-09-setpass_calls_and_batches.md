---
title: 'Unity的SetPass Calls和Batches'
date: 2018-01-09 16:15:35
tags: Unity, Rendering
comments: false
---

在做性能优化的时候有两个指标是非常核心的。一个是SetPass Calls，另外一个是Batches。

> 优化的时候一直参考SetPass Calls和Batches这两个性能消耗，但是好像一直没有完全理解透这部分内容。特地花些时间阅读了关于这部分的Unity源码(4.7f1)和Manual，记录下对SetPass Calls和Batches的理解。

### SetPass calls

我发现不少人把Unity5.x版本后的setpass calls错误的理解成了draw call, setpass calls代表了什么? Unity的Manual中这么说的:

>The number of rendering passes. Each pass requires Unity runtime to bind a new shader which may introduce CPU overhead.

这个描述很简单就是说的渲染的pass数量，具体来讲这个pass就是我们MeshRenderer的materials的shader里面的pass。举个简单的列子;

<!-- more --> 

```c++
//这个一个对Panel的正反面渲染出不同的贴图的效果的shader
Shader "Custom/TestShader"
{
	Properties
	{
		_MainTex("Texture", 2D) = "white" {}
		_BackTex("BackTexture", 2D) = "white" {}
	}
	SubShader
	{
		Tags{ "Queue" = "Transparent"  "RenderType" = "Transparent" }
      
		//这个Pass渲染正面
		Pass
		{
			ZWrite Off
			Cull Back

			CGPROGRAM
			...
			ENDCG
		}
		//这个Pass渲染背面
		Pass
		{
			ZWrite Off
			Cull Front

			CGPROGRAM
			...
			ENDCG
		}
	}
}
```

这个shader包含了两个Pass，所以用这个shader渲染的对象会增加两个SetPass Calls。

### Batches

batches的数量其实和draw calls的数量是一致的，上面讲到setpass calls的数量，每个setpass call会渲染一个renderer的集合，这个集合首先会进行dynamic batching和static batching，之后得到一个最终要渲染的batches的集合。这个batches的集合的数量是大于等于1的。每个batche需要渲染的话都会产生一次draw  call。

一次setpass call至少会产生一次draw call，每次draw call都对应着一个渲染的batch。