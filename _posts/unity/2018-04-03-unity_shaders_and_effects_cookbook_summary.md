---
title: Unity Shaders and Effects Cookbook总结
date: 2018-04-03 16:44:10
type: "tags"
tags: unity
comments: false
---



**Phong和BlinnPhong反射类型光照模型**

![](/images/unity_shader_effect_cookbook/phong_light.png)

*Phong*

Phong模型是基于光照(图中1 Light向量)的方向和用户的视角(图1 View向量)方向进行计算的。通过计算Light的反射向量与用户的视角方向向量的向量积作为光照的强度因子。

计算公式：

![](/images/unity_shader_effect_cookbook/phong_model_cal.png)

代码方法:

```cpp
//lightDir就是图1的Light向量，viewDir就是图1的View向量
fixed4 LightingPhong(SurfaceOutput s, fixed3 lightDir, half3 viewDir, fixed atten)
{
	//Reflection
  	float NdotL = dot(s.Normal - lightDir);
	float3 reflectionVector = normalize(2.0 * Normal * NdotL - Light);
    
  	//Specular
  	float spec = pow(max(0, dot(reflectionVector, viewDir)), _SpecPower);
  	float3 finalSpec = _SpecularColor.rgb * spec;
  
  	fixed4 color;
  	color.rgb = (s.Albedo * _LightColor0.rgb * max(0, NdotL) * atten) + (_LightColor0.rgb * finalSpec);
  	color.a = s.Alpha;
  
  	return color;
}
```

*BlinnPhong*

BlinnPhong可以看作是堆Phong模型的简化版，它是通过视角方向（View）和光照方向（Light）构成的半角向量（Half）来计算光照的。直接使用半角向量而不用计算光照的反射向量的方式更加高效。

计算公式：

![](/images/unity_shader_effect_cookbook/blinnphong_model_cal.png)

代码方法:

````cpp
//lightDir就是图1的Light向量，viewDir就是图1的View向量
fixed4 LightingPhong(SurfaceOutput s, fixed3 lightDir, half3 viewDir, fixed atten)
{
  	float NdotL = dot(s.Normal - lightDir);
  
    float3 halfVector = normalize(lightDir + viewDir);
  	float NdotH = max(0, dot(s.Normal, halfVector));
  	float spec = pow(NdotH, _SpecPower), _SpecularColor);
  
  	fixed4 color;
  	color.rgb = (s.Albedo * _LightColor0.rgb * max(0, NdotL) * atten) + (_LightColor0.rgb * finalSpec);
  	color.a = s.Alpha;
  
  	return color;
}
````



**挤压效果**

![](/images/unity_shader_effect_cookbook/extrusion_model.png)

这种挤压的效果的原理是<font color=red>将顶点沿发现方向进行投影</font>,用代码表示就是这样:

```c
v.vertex.xyz += v.normal * _Amount;
```
> _Amount 是挤压的因子

还可以通过额外添加一个纹理(或者使用主要纹理的alpha通道)来表示挤压的程度:

```cpp
sampler2D _ExtrusionTex;
void vert(inout appdata_full v)
{
	float4 tex = tex2Dlod(_ExtrusionTex, float4(v.texcoord.xy, 0, 0));;
	float extrusion = UnpackNormal(tex.r);
	
	v.vertex.xyz += v.normal * _Amount * extrusion;
}
```

> 从顶点修改器中采样一个纹理，应该使用texDlod而不是tex2D。

**抓取功能**

> 原理是结合vertex和fragment着色器以及抓取通行技术，然后对抓取纹理进行采样，在对其UV值做一点修改来制作出一些细微的变形效果。

```cpp
Pass {
  #pragma vertex vert
  #pragma fragment frag
  #include "UnityCG.cginc"
  
  //抓取通行会自动创建一个纹理，然后通过这个变量可以引用到
  sampler2D _GrabTexture;
  
  struct vertInput {
    float4 vertex : POSITION;
  };
  
  struct vertOutput {
    float4 vertex : POSITION;
    float4 uvgrab : TEXCOORD1;
  };
  
  vertInput vert(vertexInput v) {
    vertexOutput o;
    o.vertex = mul(UNITY_MATRIX_MVP, v.vertex);
    //获取到抓取的UV
    o.uvgrab = ComputeGrabScreenPos(o.vertex);
    return o;
  }
  
  half4 frag(vertexOutput i) : COLOR {
    //通过抓取的UV来采样抓取纹理的值
    fixed4 col = tex2Dproj(_GrabTexture, Unity_PROJ_COORD(i.uvgrab));
    return col + half4(0.5, 0, 0, 0);
  }
}
```

