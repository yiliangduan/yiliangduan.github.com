---
title: Unity 草地效果
date: 2019-09-03 21:26:10
type: "tags"
tags: graphics, unity
comments: true
mathjax:  true
---



> 参考: [GPUGems3 Chapter16](https://developer.nvidia.com/gpugems/GPUGems3/gpugems3_ch16.html) 。
>
> 如果不想看英文版本的话，我阅读的时候根据自己的理解翻译下来，放在[这里](https://github.com/yiliangduan/grass_effect_lab/blob/master/gpu_gems3_ch16.pdf)，需要的可以参考。



首先看效果（文件比较大，需要加载一会）:

![](/images/graphics/grass/grass2.gif)<!-- more --> 



**草地的Mesh**



草的Mesh为了适应各个角度，做法是用两个面片交叉形成一个十字面片。如下图:

![](/images/graphics/grass/1.png)

这样不管从哪个角度看，至少可以看到一个面（需要在着色器中开启双面渲染）。要实现这种效果其实还有另外一种方法，就是BillBoard方法。之前写过一篇介绍BillBoard的文章，可以参考[这里](https://www.duanyiliang.com/2019/04/08/graphics/billboard_learning_note/)。



**相位变化曲线**



草的摆动效果通过顶点的相位变化来形成，主要相位变化的函数如下：

```c
//平滑 曲线
float4 SmoothCurve( float4 x )
{
    return x * x * (3.0 - 2.0 * x);
}

//波
float4 TriangleWave( float4 x )
{
    return abs (frac( x + 0.5 ) * 2.0 - 1.0);
}

//波平滑
float4 SmoothTriangleWave( float4 x )
{
    return SmoothCurve(TriangleWave( x ));
}
```



可以将函数画出来，看看计算出来的数值曲线。画图的工具我是用的[geogebra.](https://www.geogebra.org/graphing)



![](/images/graphics/grass/2.png)



草的摆动相位函数有了，还有最后一个问题是现实当中草在被风吹动的时候摆动的效果从草的根部到草尖的效果是不一样的。正常情况下，草最根部是不动，然后自下往上至叶尖摆动幅度是越来越大的。在GPU Gems中介绍的方法是给模型上顶点色，把振幅因子存储到顶点色中，然后渲染的时候读出这个数值作为草的摆动的时候的振幅因子。我这里直接简单一点处理，我是用一张渐变的的贴图，如下:

![](/images/graphics/grass/3.png)

这样，在计算草的摆动的时候，采样这张贴图，然后把贴图的其中一个通道（我这里取的R通道，其他这种灰色的图取RGB通道值都是差不多的）作为振幅因子（程序中的phaseWeight变量）去和计算好的草的相位做积。就可以得出草的摆动幅度自草根到草尖是越来越强的效果。具体实现代码:

```c
//草的风吹摇摆效果的相位变化
float3 GrassSwingPhase(float4 vertex, fixed phaseWeight)
{
	float4 worldPos = mul(unity_ObjectToWorld, vertex);

	fixed branchPhase = dot(worldPos, 1);

	float vtxPhase = dot(vertex.xyz, branchPhase);
	float wavesIn = _Time.y + vtxPhase;

	float4 waves = (frac(wavesIn.xxxx * float4(1.975, 0.793, 0.375, 0.193)) * 2.0 - 1.0) * _Speed * phaseWeight;
	waves = SmoothTriangleWave(waves);

	float2 wavesSum = waves.xz + waves.yw;

	return wavesSum.xxy * _Speed;
}
```





**物体与草地的交互**

草地上的物体移动时，在物体周围的草自然都会散开来。实现的方法是，把物体的世界坐标传入到草的着色器中，然后自己配置一个半径，这个半径根据物体的大小来自己配置。草地着色器渲染时根据每个顶点去计算和草地的距离，如果这个顶点在物体的半径范围内，则做一定量的偏移，偏移的值根据这个草的顶点和物体的距离来确定，距离越远偏移越小。具体实现代码如下：

```c
//角色的周围草地挤压的效果
float distanceToRole = distance(worldPos.xyz, _RolePos);
float reactDistance = 1 - saturate(distanceToRole / _Radius); 

float2 offset = float2((sign(worldPos.x - _RolePos.x))*reactDistance, sign(worldPos.z - _RolePos.z)*reactDistance);

phase.xz += clamp(offset * (phaseWeight), -_MaxOffset, _MaxOffset);
```





**顶点着色器代码**



```c
v2f vert (appdata input)
{
    v2f o;

    o.uv = TRANSFORM_TEX(input.uv, _MainTex);

    //相位变化权重，草的根部近似0，越往草尖，值越大
    fixed phaseWeight = max(0, tex2Dlod(_SwingTex, fixed4(TRANSFORM_TEX(input.uv, _SwingTex), 0, 0)).r);

    float4 vertexPhase = float4(input.vertex.xyz + GrassSwingPhase(input.vertex, phaseWeight), input.vertex.w);
    
    o.vertex = UnityObjectToClipPos(vertexPhase);

    return o;
}
```



*END*