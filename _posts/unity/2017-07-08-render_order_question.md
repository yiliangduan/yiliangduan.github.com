---
title: 'Unity模型渲染层级顺序问题'
date: 2017-07-8 9:59:39
type: "tags"
tags: unity
comments: false
---

### 问题描述

项目中遇到一个问题，在Untiy中如果关闭ZTest的话，模型的子部件之间的层级可能会出现错乱的现象，模型是3ds max导出的模型。


### 原因思考
 既然是层级错误，根据Unity渲染管线的规则决定层级的有两个地方。一个是传入Unity的顶点数据的顺序(如果用了顶点数据索引，那么就是索引的顺序)，另外一个是Unity里面的DepthTest功能决定。那么两种情况的规则是怎样的呢？

> Unity中的shader有一个RenderQueue设置，使用设置了同一个RendererQueue的值的shader模型在同一批被渲染，比如RenderQueue设置了一般不透明物体设置为Geometry(对应的数值为2000)，透明物体物体设置为Transparent(对应的数值为3000)。Geometry的对象会先渲染，然后才会渲染Transparent的物体。本文我遇到的问题的模型都是用的同一个shader，所以RendererQueue不是问题的原因。

 首先看DepthTest的作用，在Unity中利用了深度缓冲来保证了渲染的层级问题，深度缓冲是根据RenderQueue和深度值(顶点的Z值)来确定的。渲染管线在光栅化之后会对顶点数据做一次DepthTest 。

 对于顶点数据的顺序的影响也是经过这个问题之后我才发现的。在DepthTest打开的情况下(Shader里面ZWrite&ZTest开启)，这个顺序也没有什么作用的，但是当我们关闭DepthTest的时候(比如渲染半透明的物体需要进行混合操作),Unity的渲染就直接根据这个数据的顺序去渲染了。

基于上面两种情况的分析，通过测试发现我们的模型在打开DepthTest的时候是没有层级的问题的，关闭DepthTest之后才出现层级混乱的问题。这样我们就把问题定位到原因了。那么接下来验证一下原因

### 问题解决
首先我直接在Unity中通过代码生成一个Mesh来渲染，目的是为了可以方便的更改顶点数据的顺序。(创建Mesh的代码参考了[ProceduralMesh](https://github.com/mortennobel/ProceduralMesh)，根据自己测试需求做了一些改动)

```cs

public class CreateMesh : MonoBehaviour {

    public bool UseRightTrianglesOrder;
    private MeshFilter m_MeshFilter;

    private void Start()
    {
        gameObject.AddComponent<MeshRenderer>();
        m_MeshFilter = gameObject.AddComponent<MeshFilter>();
        m_MeshFilter.mesh = new Mesh();

        BuildMesh();
    }

    public void OnGUI()
    {
        if(GUI.Button(new Rect(20, 20, 200, 80), "ReBuild"))
        {
            BuildMesh();
        }
    }

    public void BuildMesh()
    {
        Vector3 p0 = new Vector3(0, 0, 0);
        Vector3 p1 = new Vector3(1, 0, 0);
        Vector3 p2 = new Vector3(0.5f, 0, Mathf.Sqrt(0.75f));
        Vector3 p3 = new Vector3(0.5f, Mathf.Sqrt(0.75f), Mathf.Sqrt(0.75f) / 3);

        Mesh mesh = m_MeshFilter.sharedMesh;
        mesh.Clear();

        mesh.vertices = new Vector3[]{
            p0,p1,p2,
            p0,p2,p3,
            p2,p1,p3,
            p0,p3,p1
        };

        if (UseRightTrianglesOrder)
        {
            //这是正确的顺序
            mesh.triangles = new int[]{
                9,10,11, //蓝色的那个三角形
                0,1,2,
                3,4,5,
                6,7,8
            };
        }
        else
        {
            //这是错误的顺序
            mesh.triangles = new int[]{
                0,1,2,
                3,4,5,
                6,7,8,
                9,10,11 //蓝色的那个三角形,如果ZTest关闭，那么这个三角形在最后被渲染，会覆盖前面的三个三角形部分区域
            };
        }

        Vector2 uv3a = new Vector2(0, 0);
        Vector2 uv1 = new Vector2(0.5f, 0);
        Vector2 uv0 = new Vector2(0.25f, Mathf.Sqrt(0.75f) / 2);
        Vector2 uv2 = new Vector2(0.75f, Mathf.Sqrt(0.75f) / 2);
        Vector2 uv3b = new Vector2(0.5f, Mathf.Sqrt(0.75f));
        Vector2 uv3c = new Vector2(1, 0);

        mesh.uv = new Vector2[]{
            uv0,uv1,uv2,
            uv0,uv2,uv3b,
            uv0,uv1,uv3a,
            uv1,uv2,uv3c
        };

        mesh.RecalculateNormals();
        mesh.RecalculateBounds();
    }
}

```

在使用正确的triangles的数据并且关闭ZTest的时候，显示的效果如下:

![](/images/3ds_max_fbx_export_model_layer/opengl_model_render_order_1.png)

这种效果是正确的。在使用乱序的triangles的索引并且关闭ZTest的时候，显示效果如下:

![](/images/3ds_max_fbx_export_model_layer/opengl_model_render_order_2.png)

这看起来很怪异，再看看3D空间视图里面的顶部视图:

![](/images/3ds_max_fbx_export_model_layer/opengl_model_render_order_3.png)

通过这个图我们可以看到，其实乱序的triangles数据渲染出来的效果是因为蓝色的那个三角形被渲染在前面了，然而他本身的位置是在红色和绿色的后面的。说明他比红色和绿色的面片要后渲染出来。

上面的两个测试证明了在关闭ZTest的时候，Unity的渲染顺序其实就是根据triangles数据的顺序来渲染的。如果我们开启ZTest，那么上面两个triangles数据渲染出来的模型我们会发现都是没有问题的。现在证明了是triangles顺序的问题但是我们模型的triangles的数据来自于3dsMax，说明这应该是在3dsMax导出模型的一些设置问题，或者说是模型本身有问题。

### 3DS Max导出模型的问题

现在问题定位到模型导出的问题，由于没用过3dsMax。为了测试需要我直接摸索创建了一个包含3个Cube的模型，然后合并三个模型的Mesh。导出FBX之后再在Unity观察三个Cube的层级关系，结果重现了问题模型的问题。那么到底是哪一步出错了呢？经过无数次的测试，最终发现了triangles顺序是由合并Mesh的顺序决定的。如果想得到triangles数据的顺序正好是对应顶点的层级由低到高的顺序，那么在3dsMax中在合并Mesh(Attach子模型)的时候需要从层级最低的模型逐个Attach到层级最高的模型。


最后测试的Shader(其实就是官方给的Unlit-Alpha改了一点点)

```
// Unlit alpha-blended shader.
// - no lighting
// - no lightmap support
// - no per-material color

Shader "Custom/TestRenderOrder" {
Properties {
	_MainTex ("Base (RGB) Trans (A)", 2D) = "white" {}
}

SubShader {
	Tags {"RenderType"="Opaque"}
	LOD 100

	//ZWrite Off
	ZTest Off
	Cull Off

	Pass {  
		CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			#pragma target 2.0
			#pragma multi_compile_fog

			#include "UnityCG.cginc"

			struct appdata_t {
				float4 vertex : POSITION;
				float2 texcoord : TEXCOORD0;
				UNITY_VERTEX_INPUT_INSTANCE_ID
			};

			struct v2f {
				float4 vertex : SV_POSITION;
				float2 texcoord : TEXCOORD0;
				UNITY_FOG_COORDS(1)
				UNITY_VERTEX_OUTPUT_STEREO
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;

			v2f vert (appdata_t v)
			{
				v2f o;
				UNITY_SETUP_INSTANCE_ID(v);
				UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.texcoord = TRANSFORM_TEX(v.texcoord, _MainTex);
				UNITY_TRANSFER_FOG(o,o.vertex);
				return o;
			}

			fixed4 frag (v2f i) : SV_Target
			{
				fixed4 col = tex2D(_MainTex, i.texcoord);
				UNITY_APPLY_FOG(i.fogCoord, col);
				return col;
			}
		ENDCG
	}
}
}
```
