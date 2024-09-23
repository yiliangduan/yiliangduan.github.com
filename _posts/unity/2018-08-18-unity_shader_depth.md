---
title: Unity渲染的深度值获取
date: 2018-08-18 16:22:10
type: "tags"
comments: false
tags: unity,shader

---

> unity 5.6.6f1

深度值是对象距离摄像机距离的一个参考值，在制作一些场景效果时会经常用得到。深度值是在顶点着色器之后的像素着色器之前的深度测试阶段生成的，所以获取深度值就得在像素着色器中获取。下面创建了一个简单的场景来显示深度信息

<!-- more --> 

```c
Pass
{
    CGPROGRAM
    #pragma target 3.0
    #pragma vertex vert
    #pragma fragment frag
    
    #include "UnityCG.cginc"
    
    struct appdata
    {
    	float4 vertex : POSITION;
    	float2 uv : TEXCOORD0;
    };
    
    struct v2f {
    	float4 clipPos : SV_POSITION;
    	float4 scrPos : TEXCOORD0;
    };
    
    uniform sampler2D _CameraDepthTexture;
    
    v2f vert (appdata_base v)
    {
  	v2f o;
    	//裁剪坐标
        //裁剪坐标的xy的范围是[-v.vertex.w, -v.vertex.w]
    	o.clipPos = UnityObjectToClipPos(v.vertex);
    	//屏幕坐标
        //这里的屏幕坐标的范围是[0, o.clipPos.w]
        //o.clipPos的zw和v.vertex.zw值是相同的
    	o.scrPos = ComputeScreenPos(o.clipPos);
    	return o;
    }
    
    fixed4 frag (v2f i) : COLOR
    {
    	//处理平台差异，一般直接返回输入的值
    	float4 projCoord = UNITY_PROJ_COORD(i.scrPos);
    
    	//SAMPLE_DEPTH_TEXTURE_PROJ(sampler, uv) = tex2Dproj(sampler, uv).r
    	float4 depthTexProj = SAMPLE_DEPTH_TEXTURE_PROJ(_CameraDepthTexture, projCoord);
    
    	//把depthTexProj的值转换到0~1区间
    	float sceneZ = Linear01Depth(depthTexProj);
    
    	//直接显示深度值信息
    	return float4(sceneZ, sceneZ, sceneZ, 1);
    }
    ENDCG
}
```

显示的颜色限制在0~1之间，也就是如果深度值越大颜色就越接近于黑色，反之深度值越小就越接近于白颜色。效果如下(底部片没有显示深度颜色):

![](/images/unity_depth/1.png)

通过上面Pass里面的操作即可获取到深度值，这里详细解释下里面操作的一些步骤：

1. ComputeScreenPos

```c
o.scrPos = ComputeScreenPos(o.clipPos);
```

 裁剪坐标转换到屏幕坐标要处理的工作就是根据屏幕的宽高来转换到对应的坐标,计算公式如下：

![](/images/unity_depth/2.png)

利用这个公式即可根据裁剪坐标计算出屏幕坐标，但是当查看Unity内置的ComputeScreenPos函数时发现里面并不是这样计算的:

```c
inline float4 ComputeNonStereoScreenPos(float4 pos) {
    float4 o = pos * 0.5f;
    //_ProjectionParams.x的值为1或者-1
    o.xy = float2(o.x, o.y*_ProjectionParams.x) + o.w;
    o.zw = pos.zw;
    return o;
}

inline float4 ComputeScreenPos(float4 pos) {
    float4 o = ComputeNonStereoScreenPos(pos);
#if defined(UNITY_SINGLE_PASS_STEREO)
    o.xy = TransformStereoScreenSpaceTex(o.xy, pos.w);
#endif
    return o;
}
```



主要看ComputeNonStereoScreenPos函数发现这个计算和我们上面列出的计算屏幕坐标的方式不同，仔细推导发现这个计算方式是这样得来的:

![](/images/unity_depth/3.png)

再结合像素着色器中计算深度的接口SAMPLE_DEPTH_TEXTURE_PROJ就明白其中的原因了。从标准的计算屏幕坐标的公式计算好的screen坐标的区间范围是[0, pixelwidth],这个区间就是屏幕像素大小的区间。而通过ComputeScreenPos计算得到的newScreen的坐标的区间范围是[0, w]的范围，这个不是真正的屏幕坐标，但是不要紧，在使用到这个坐标的SAMPLE_DEPTH_TEXTURE_PROJ中是这样的：

```c
#define SAMPLE_DEPTH_TEXTURE_PROJ(sampler, uv) (tex2Dproj(sampler, uv).r)

half4 tex2Dproj(sampler2D s, in half4 t) { return tex2D(s, t.xy / t.w); }
```

tex2D 里面会再把t.xy转换成[0, 1]的区间来采样纹理。这样ComputeScreenPos计算的结果正好用作采样_CameraDepthTexture的UV值。

