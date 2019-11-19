---
title: 'UGUI的Graphic Rebuild问题'
date: 2017-12-09 20:16:35
tags: unity
comments: false
---

今天在项目中发现了一个UI的性能消耗比较异常的问题。在Unity的Profiler里面我看到一项*Canvas.SendWillRenderCanvases*CPU消耗持续比较高，但是查看游戏的UI界面好像动态变化的UI并没有。在逐个关闭UI才定位到有一张图片一直在做渐变效果，所以才导致UI一直在重建。这里写个简单的例子模拟下，在场景中创建一张Image，然后挂载一个测试脚本，在这个脚本里面一直更新这个Image的alpha通道就可以看到这种现象。

```csharp
public class TestGraphicRebuild : MonoBehaviour
{
    public Image FadeImage;
    private float _mAlpha = 0;

	void Update ()
	{
	    _mAlpha += 0.01f;
	    _mAlpha = _mAlpha > 1 ? 0 : _mAlpha;
	    
        FadeImage.color = new Color(
                              FadeImage.color.r, 
                              FadeImage.color.g, 
                              FadeImage.color.b, 
                              _mAlpha);
	}
}
```

<!-- more --> 

Profiler的情况如下(开启了Deep Profile模式)：

![](/images/ugui_graphics_rebuild/ugui_graphic_rebuild_profiler.png)

如果能避免UGUI的Graphic Rebuild又能让这张图片有渐变效果，目前想到一个办法是把这个Image使用自定义的material来渲染。这样我可以在shader中来改变这张图片的alpha通道达到渐变的效果。

我们create一个material，然后实现一个UI 图片渐变效果的shader。把material的shader选择我们自己实现的这个shader。最后把material拖到目标Image的Material属性上就可以了。shader的实现很简单，如下:



```c
Shader "Unlit/FadeShader"
{
	Properties
	{
		[PerRendererData] _MainTex ("Texture", 2D) = "white" {}
	}
	SubShader
	{
		Tags 
		{
			"RenderType"="Transparent" 
			"IgnoreProjector"="true"
			"RenderType"="Transparent"
		}

		Lighting Off
		Cull Off
		ZTest [unity_GUIZTestMode]
		ZWrite Off
		Blend SrcAlpha OneMinusSrcAlpha

		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			
			#include "UnityCG.cginc"

			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float2 uv : TEXCOORD0;
				float4 vertex : SV_POSITION;
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;
			
			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target
			{
				fixed4 col = tex2D(_MainTex, i.uv);
				if (col.a > 0.001f) //图片透明的地方渐变
				{
					col.a = col.a * 0.5f + _SinTime.z*_SinTime.z;//根据自己需要的效果调
				}
				return col;
			}
			ENDCG
		}
	}
}
```

这样操作之后运行起来就可以看到渐变的效果，再通过Profiler看看性能(同样开启Deep Profiler)：

![](/images/ugui_graphics_rebuild/ugui_graphic_rebuild_profiler_perfomance.png)

可以看到*Canvas.SendWillRenderCanvases*的消耗降下来了。