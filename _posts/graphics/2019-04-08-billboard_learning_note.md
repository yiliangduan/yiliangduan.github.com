---
title: Billboard 效果实现原理
date: 2019-04-08 20:26:10
type: "tags"
tags: unity,graphics
comments: false
---

在[ShadowgunSample](https://files.unity3d.com/assetstore/ShadowgunSampleLevel.zip)中学到了用shader实现一个广告牌（Billboard）效果，这里记录下。在ShadowgunSample的场景中，这个模型两边的光晕永远是对着移动中的摄像机，如下图:

<!-- more --> 

![](/images/opengl_shader_billboard/3.png)

其实你如果看这个模型，它就是一个面片。然后在顶点着色器里面根据摄像机的当前的视角来修正每个顶点位置从而达到整个面片一直正向对着相机的效果。单独提取这个光晕模型显示的效果:

![](/images/opengl_shader_billboard/3.gif)

下面来介绍下这个效果是怎样实现过程。

首先我们分析这个模型，模型的顶点色存储了每个对应的每个顶点规范之后的坐标，即把模型坐标规范到[0, 1]范围，这个可以直接查看模型的vertices和colors值对比下就可以知道（这里只取顶点0，1， 3，2）:

```cpp
//vertices:           //colors                             //UV
(-4.0, 0.0, 0.0)      RGBA(0.000, 0.000, 0.000, 1.000)     (7.9, 8.4)
(-2.5, 0.0, 0.0)      RGBA(0.188, 0.000, 0.000, 1.000)     (7.9, 8.4)
(-2.5, 7.9, 0.0)      RGBA(0.188, 1.000, 0.000, 0.000)     (7.9, 8.4)
(-4.0, 7.9, 0.0)      RGBA(0.000, 1.000, 0.000, 0.000)     (7.9, 8.4)
```

所以这里求出来的centerOffs是当前的顶点坐标距离模型的中心点的偏移值。

```cpp
//float(0.5).xx - v.color.rg把v.color.rg又转换到[-0.5, 0.5]的范围，然后再乘以模型的宽和高得
//了当前顶点坐标距离模型的中心点的偏移值。其中v.texcoor1.xyy相当于(width, height, height)，第
//三个值取height仅仅是为了得到float3类型
float3	centerOffs = float3(float(0.5).xx - v.color.rg, 0) * v.texcoord1.xyy;
```

这一步的计算过程: 

* 把存储在v.color中模型的规范坐标转换到[-0.5, 0.5]范围模型坐标
* 把上一步得到的[-0.5, 0.5]的模型坐标与存储在uv2中的模型的宽高相乘的得到当前顶点距离模型中心点的偏移值

接着用顶点的模型坐标加上该顶点距离模型中心点的偏移，得到centerLocal是模型中心点的坐标

```cpp
//centerLocal是模型的中心点坐标
float3	centerLocal = v.vertex.xyz + centerOffs.xyz;
```

得到模型中心点的坐标之后计算摄像机对模型的视角的方向向量:

```cpp
// _WorldSpaceCameraPos是Unity Shader内置变量，存储了当前摄像机的世界坐标，unity_WorldObject
//是世界坐标转模型坐标的矩阵，这里把摄像机的世界坐标转换到了当前模型的局部坐标
float3 viewerLocal = mul(unity_WorldToObject,float4(_WorldSpaceCameraPos,1));
//把摄像机的局部坐标减去中心点坐标，得到摄像机相对于模型中心点的方向向量
float3 localDir = viewerLocal - centerLocal;
//_VerticalBillboarding是自定义变量，控制localDir的y轴的值的范围
localDir[1] = lerp(0,localDir[1], _VerticalBillboarding);
```

然后到最重要的计算模型随着摄像机的视角的变化而变化的偏移:

```cpp
//计算出摄像机对模型视角的方向向量的长度
float localDirLength = length(localDir);

//方向向量localDir转换到单位向量
float3	localDirUnit = localDir / localDirLength;

//这里upLocal, rightLocal实际是建立一个三维坐标系，upLocal就是向上方向的，rightLocal
//是向右方向的，这里并不对应x，y，z。因为不同的角度的时候upLocal可能代表y，也可能代表z

//当摄像机的y轴坐标localDirUnit.y大于0.999时相当于摄像机看向模型的顶部或者底部
//此时upLocal向上的坐标其实是z轴，因为摄像机的正前方向上的坐标就是z轴
//当摄像机的y轴坐标localDirUnit.y小于等于0.999时，相当于摄像机正前方向上的坐标是y轴
//此时upLocal向上的坐标就是y轴了
float3 upLocal = abs(localDirUnit.y) > 0.999f ? float3(0,0,1) : float3(0,1,0);

//通过upLocal和localDirUnit做向量的叉积得到垂直于这两个向量的rightLocal
float3 rightLocal = normalize(cross(upLocal, localDirUnit));

//有了rightLocal和localDirUnit就可以计算得到垂直与这两个向量的向上的向量upLocal
//注意这里的upLocal适合localDirUnit,rightLocal两两垂直的，构成了一个坐标系
//但是上面也求得一个upLocal，两者是不一样的。上面的upLocal是相对于摄像机的视角方向
//的正上方向量
upLocal = cross(localDirUnit, rightLocal);

//localDirUnit是摄像机相对模型的视角的模型向量，只要摄像机移动，这个向量就会变
//构建好的upLocal,rightLocal和localDirUnit摄像机视角坐标系就会跟着变化，而模型需要
//Billboard效果(正面一直对着摄像机)也就需要随着摄像机的移动而移动，那么移动的量
//正好是这个坐标系随相机变化的偏移量。

//根据摄像机的距离和顶点颜色透明度来计算缩放值，v.color.a其实只配置了0和1
//只有可见和不可见
float	distScale = CalcDistScale(localDirLength) * v.color.a;

//把当前顶点法线坐标偏移到摄像机视角的坐标系
float3	BBNormal = rightLocal * v.normal.x + upLocal * v.normal.y;

//把当前顶点到中心点的偏移值的x和y分别与rightLocal和upLocal做积，相当于计算cetnerOffs偏移
//之后的值。然后用中心点centerLocal减去这个偏移值，得到的是顶点偏移之后的值。最后加上法线和缩放
//积的分量，相当于显示的一个因子分量。
float3	BBLocalPos = centerLocal - (rightLocal * centerOffs.x + upLocal * centerOffs.y) + BBNormal * distScale;

//_ViewerOffset自定义的偏移分量,BBLocalPos就是随着相机变化之后顶点位置。
BBLocalPos += _ViewerOffset * localDir;
```

完整的shader如下:

```cpp
v2f vert (appdata_full v)
{
	v2f o;
	
	float3	centerOffs = float3(float(0.5).xx - v.color.rg, 0) * v.texcoord1.xyy;
	float3	centerLocal = v.vertex.xyz + centerOffs.xyz;
	
	
	float3	viewerLocal = mul(unity_WorldToObject,float4(_WorldSpaceCameraPos,1));
	float3	localDir = viewerLocal - centerLocal;
	
	localDir[1] = lerp(0,localDir[1],_VerticalBillboarding);
	
	float localDirLength = length(localDir);
	float3	localDirUnit = localDir / localDirLength;
	
	float3 upLocal = abs(localDirUnit.y) > 0.999f ? float3(0, 0, 1) : float3(0, 1, 0);
	
	float3 rightLocal = normalize(cross(upLocal, localDirUnit));
	upLocal = cross(localDirUnit, rightLocal);
	
	float	distScale = CalcDistScale(localDirLength) * v.color.a;
	float3	BBNormal = rightLocal * v.normal.x + upLocal * v.normal.y;
	float3	BBLocalPos = centerLocal - (rightLocal * centerOffs.x + upLocal * centerOffs.y) + BBNormal * distScale;
	
	BBLocalPos += _ViewerOffset * localDir;
	
	o.uv = v.texcoord.xy;
	o.pos = UnityObjectToClipPos(float4(BBLocalPos,1));
	o.color =  _Color;
	
	return o;
}
```



如果看到这里理解还是有些模糊，那么接下来我们用C#来模拟下shader运算的过程，用C#的好处是每个数据都可以实时显示出来或者输出Log，更加直观。

我们用C#在运行时刻创建一个Mesh面片，自己自定义6个顶点。顶点的分布如下:

```cpp
2           3            5
--------------------------
|           |            |
|           |            | 
--------------------------
0           1            4
```

创建Mesh的代码:

```csharp
public Mesh BBMesh;

private Vector3[] BBMeshVertexs = new Vector3[6];
private int[] BBMeshTriangles = new int[12];
private Vector2[] BBMeshUVs = new Vector2[6];
private Color[] BBMeshColors = new Color[6];

private MeshFilter mMeshFilter;
private MeshRenderer mMeshRenderer;

private readonly Vector3[] OriginVertexs = new Vector3[] {
                                     new Vector3(-2, -1, 0) ,
                                     new Vector3(0, -1, 0) ,
                                     new Vector3(-2, 1, 0) ,
                                     new Vector3(0, 1, 0) ,
                                     new Vector3(2, 1, 0) ,
                                     new Vector3(2, 1, 0) };

public void Start()
{
    BBMesh = new Mesh
    {
        name = "billboardmesh"
    };

    mMeshFilter = gameObject.AddComponent<MeshFilter>();
    mMeshFilter.mesh = BBMesh;

    mMeshRenderer = gameObject.AddComponent<MeshRenderer>();

    //配置了顶点颜色的话，需要用到支持顶点色的Shader
    Material mat = new Material(Shader.Find("Custom/VertexColored"));
    mat.color = Color.white;

    mMeshRenderer.material = mat;

    SetMesh();
}

public void SetMesh()
{
    /*
    *   2           3            5
    *   --------------------------
    *   |           |            |
    *   |           |            | 
    *   --------------------------
    *   0           1            4
    */

    //vertex
    for (int i=0; i<BBMeshVertexs.Length; ++i)
    {
        BBMeshVertexs[i] = OriginVertexs[i];
    }

    BBMesh.vertices = BBMeshVertexs;

    //triangles
    BBMeshTriangles[0] = 0;
    BBMeshTriangles[1] = 2;
    BBMeshTriangles[2] = 1;

    BBMeshTriangles[3] = 2;
    BBMeshTriangles[4] = 3;
    BBMeshTriangles[5] = 1;

    BBMeshTriangles[6] = 1;
    BBMeshTriangles[7] = 3;
    BBMeshTriangles[8] = 4;

    BBMeshTriangles[9] = 3;
    BBMeshTriangles[10] = 5;
    BBMeshTriangles[11] = 4;
    BBMesh.triangles = BBMeshTriangles;

    //uv
    BBMeshUVs[0] = new Vector2(0, 0);
    BBMeshUVs[1] = new Vector2(0.5f, 0);
    BBMeshUVs[2] = new Vector2(0, 1);
    BBMeshUVs[3] = new Vector2(0.5f, 1);
    BBMeshUVs[4] = new Vector2(1, 0);
    BBMeshUVs[5] = new Vector2(1, 1);

    BBMesh.uv = BBMeshUVs;

    BBMeshColors[0] = Color.red;
    BBMeshColors[2] = Color.red;
    BBMeshColors[1] = Color.yellow;
    BBMeshColors[3] = Color.yellow;
    BBMeshColors[4] = Color.green;
    BBMeshColors[5] = Color.green;

    BBMesh.colors = BBMeshColors;
}
```

用到的支持顶点色的Shader（这里用的是Surface Shader）:

```csharp
Shader "Custom/VertexColored" {
    Properties{
    }
    
    SubShader{
        Pass {
            ColorMaterial AmbientAndDiffuse
        }
    }
}
```

创建的Mesh显示如下:

![](/images/opengl_shader_billboard/4.png)

然后照着ShadowgunSample的Shader计算每个顶点随着相机的移动来变换，从而实现这个Mesh面片的Billboard效果:

```csharp
    public void Update()
    {
        UpdateBillBoard();
    }

    public void UpdateBillBoard()
    {
         //模型的顶点随着摄像机移动而更新顶点位置
        for (int i=0; i< OriginVertexs.Length; ++i)
        {
            BBMeshVertexs[i] = SimulateVertexShaderCal(OriginVertexs[i]);
        }

         BBMesh.vertices = BBMeshVertexs;

        BBMesh.RecalculateNormals();
    }

    public Vector3 SimulateVertexShaderCal(Vector3 vertex)
    {
        //这里的计算省去中心点的计算
        Vector3 cameraPos = Camera.main.transform.position;
        Vector3 cameraLocalPos = transform.InverseTransformPoint(cameraPos);
        Vector3 localDir = -cameraLocalPos;

        Vector3 normalizeLocalDir = Vector3.Normalize(localDir);

        Vector3 upLocal = Mathf.Abs(normalizeLocalDir.y) > 0.999f ? new Vector3(0, 0, 1) : new Vector3(0, 1, 0);
        Vector3 right = Vector3.Normalize(Vector3.Cross(normalizeLocalDir, upLocal));
        upLocal = Vector3.Cross(normalizeLocalDir, right);

        return vertex.x * right + vertex.y * upLocal;
    }
```

运行Editor然后移动相机，可以看到这个面片的Billboard效果了。


![](/images/opengl_shader_billboard/1.gif)

现在如果对Shader里面哪里不明白的地方，可以利用这个C#模拟的例子运行起来断点看每一步计算的值。这样就很容易理解其中的每个步骤了。

*END*



