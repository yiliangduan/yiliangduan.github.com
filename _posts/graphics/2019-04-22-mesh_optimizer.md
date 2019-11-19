---
title: Unity Mesh合并工具
date: 2019-04-22 20:26:10
type: "tags"
tags: unity
comments: true
---

> Unity  2018.3.5f1

项目中经常存在很多Material只是贴图不一样的情况，不同的Material是需要单独一个DrawCall来提交数据渲染这个对象的，比如下面场景中五个Cube对象，每个Cube对象使用了一个单独的Material，其中各自的Material只有贴图不一样。因为每个Cube使用了独立的Material，渲染花费了五个DrawCall：

![](/images/unity_mesh_tool/4.png)

如果我们把这些贴图合并到一张大的贴图中去（参考[Unity贴图合并工具](https://www.duanyiliang.com/2018/12/06/graphics/texture_pack_tool/)），然后让那些仅仅是贴图不一样的Material都使用这个合并后的大贴图。针对每个模型修改自己的UV，让Mesh的UV匹配到合并后的大贴图。这样这些模型就可以使用相同的Material了，Unity对使用相同的Material渲染的Mesh会进行[动态合批（dynamic batching）](https://docs.unity3d.com/Manual/DrawCallBatching.html)。这样游戏运行时这五个Cube只需要一个DrawCall消耗了。进一步思考，dynamic batching可以在运行时刻开辟一块内存把这些分散的Mesh合并到一个新的Mesh中去，我们干脆直接在资源层面将这些Mesh合并，这样还节省运行时刻的内存。这办法可行，下面介绍下一个简单的Mesh合并工具的实现过程。

需要合并Mesh我们需要先了解其数据结构，一个Mesh文件基本组成部分：

* 顶点（vertex）：表示位置的坐标
* 面（triangles）：顶点连接起来形成的面
* UV ：贴图的坐标
* 颜色（color）：模型的附带颜色
* 法线（normal）：每个面的垂直线

两个独立的Mesh文件，我们只需要把它们这些信息合并就可以得到一个新的Mesh。这个新的Mesh包含了两者的信息。第一步获取这些信息:

<!-- more --> 

```csharp
/// <summary>
/// 获取SubMesh的数据
/// </summary>
/// <param name="mesh"></param>
/// <param name="subMeshIndex"></param>
/// <returns></returns>
public static MeshData GetSubMeshData(MeshFilter meshFilter, int subMeshIndex)
{
    Mesh mesh = meshFilter.sharedMesh;

    if (null == mesh)
    {
        Debug.LogError("Mesh missing.");
        return null;
    }

    int[] triangles = mesh.GetTriangles(subMeshIndex);

    Vector3[] vertices = mesh.vertices;

    Matrix4x4 matrix4x4 = meshFilter.gameObject.transform.localToWorldMatrix;

    for (int i=0; i< vertices.Length; ++i)
    {
        //转换到世界坐标。因为涉及到多个Mesh合并，每个Mesh的vertices都是自己的标准化坐标，
        //所以转到到世界坐标
        vertices[i] = matrix4x4.MultiplyPoint(vertices[i]);
    }

    MeshData meshData = new MeshData
    {
        triangles = triangles,

        uv = mesh.uv,
        uv2 = mesh.uv2,

        vertices = vertices,

        colors = mesh.colors,

        normals = mesh.normals,
    };

    return meshData;
}
```

或者了所有的Mesh信息之后，正如文章开头所说，这些Mesh使的Material仅仅是贴图不一样的Material。这里事先利用[Unity 贴图合并工具](<https://www.duanyiliang.com/2018/12/06/graphics/texture_pack_tool/>)把这些需要合并的Mesh的所用到的渲染其的Material的贴图合并到一张大贴图中去。

![](/images/unity_mesh_tool/5.png)

现在需要的是渲染合并之后的Mesh用到的一个Material，这个Material的的贴图用的是合并之后的大贴图，Material的其他属性和未合并的Mesh所用的Material的属性是相同的，创建一个这样的Material供之后生成合并的Mesh使用：

```csharp
/// <summary>
/// 生成合并之后的Material
/// </summary>
/// <param name="meshData">每个SubMesh的数据</param>
/// <param name="materials">已经创建的合并之后的Mesh所用的Material</param>
/// <returns></returns>
public static Material GeneratorCombineMaterial(MeshData meshData, List<Material> materials)
{
    //属性使用未合并的Mesh的Material的属性
    Material outMaterial = new Material(meshData.material);

    //贴图替换成合并之后的大贴图
    Texture2D atlas = meshData.texData.Atlas.Atlas;
    Vector2 atlasSize = new Vector2(meshData.texData.Atlas.Width, meshData.texData.Atlas.Height);

    outMaterial.mainTexture = atlas;
    outMaterial.mainTextureOffset = Vector2.zero;
    outMaterial.mainTextureScale = new Vector2(atlas.width / atlasSize.x, atlas.height / atlasSize.y);

    bool foundSameMaterial = false;
    for (int i = 0; i < materials.Count; ++i)
    {
        if (MaterialTool.Compare(outMaterial, materials[i]))
        {
            outMaterial = materials[i];
            foundSameMaterial = true;
            break;
        }
    }

    //如果没有创建，则创建个Asset
    if (!foundSameMaterial)
    {
        AssetDatabase.CreateAsset(outMaterial, GeneratorMaterialPath(meshData));
    }

    //添加到
    if (!materials.Contains(outMaterial))
        materials.Add(outMaterial);

    return outMaterial;
}
```

然后合并Mesh生成一个新的Mesh即可:

```csharp
public static Mesh DoCombine(List<MeshData> meshDatas, string newMeshName)
{
    Mesh combineMesh = new Mesh{name = newMeshName};

    int vertexCount = 0;
    int triangleCount = 0;

    //只要有一个元素没有值，则设为没有值,代码省略
    bool hasUV2Data = true;
    bool hasNormalData = true;
    bool hasColorData = true;
    //...

    Vector3[] vertices = new Vector3[vertexCount];
    int[] triangles = new int[triangleCount];

    Vector2[] uvs = new Vector2[vertexCount];
    Vector2[] uv2s = new Vector2[vertexCount];

    Color[] colors = new Color[vertexCount];

    Vector3[] normals = new Vector3[vertexCount];

    int vertexArrayIndex = 0;
    int triangleArrayIndex = 0;

    for (int i=0; i<meshDatas.Count; ++i)
    {
        MeshData meshData = meshDatas[i];

        if (null != meshData)
        {
            meshData.vertices.CopyTo(vertices, vertexArrayIndex);

            for (int j=0; j<meshData.triangles.Length; ++j)
            {
                triangles[triangleArrayIndex + j] = meshData.triangles[j] + vertexArrayIndex;
            }
		    //这里需要注意的是替换成大贴图之后由于原先使用的小图只是大贴图的一部分，
            //所以Mesh的UV肯定是要根据原先小贴图在大贴图中的位置和大小比例重新计算的。
            IntVector2 elementOffset = meshData.texData.Element.Offset;

            TextureAtlas atlas = meshData.texData.Atlas;

            int atlasWidth = atlas.Width;
            int atlasHeight = atlas.Height;

            Vector2 uvOffset = new Vector2(elementOffset.x/(float)atlasWidth, elementOffset.y/(float)atlasHeight);

            float atlasRealWidth = atlas.Atlas.width;
            float atlasRealHeight = atlas.Atlas.height;

            Vector2 atlasScale = new Vector2(atlasRealWidth/ atlasWidth, atlasRealHeight/ atlasHeight);

            IntVector2 texSize = meshData.texData.Element.Size;
            Vector2 texLocalOffset = new Vector2(texSize.x/(float)atlasWidth, texSize.y/(float)atlasHeight);

            for (int j=0; j<meshData.uv.Length; ++j)
            {
                uvs[vertexArrayIndex + j] = new Vector2(meshData.uv[j].x * texLocalOffset.x, meshData.uv[j].y * texLocalOffset.y) + 
                                            new Vector2(uvOffset.x * atlasScale.x, uvOffset.y * atlasScale.y);
            }

            if (hasUV2Data)
                meshData.uv2.CopyTo(uv2s, vertexArrayIndex);

            if (hasColorData)
                meshData.colors.CopyTo(colors, vertexArrayIndex);

            if (hasNormalData)
                meshData.normals.CopyTo(normals, vertexArrayIndex);

            vertexArrayIndex += meshData.vertices.Length;
            triangleArrayIndex += meshData.triangles.Length;
        }
    }

    combineMesh.vertices = vertices;
    combineMesh.SetTriangles(triangles, 0);
    combineMesh.uv = uvs;
    combineMesh.uv2 = uv2s;
    combineMesh.colors = colors;
    combineMesh.RecalculateNormals();

    return combineMesh;
}
```

上面代码第60行有个处理需要注意下，就是合并之后的Mesh使用合并之后的Atlas之后，Mesh对应的UV需要重新计算下。比如说一张256x256的贴图TexA合并到了一张1024x1024的图集中，那么使用TexA的Mesh需要修改下UV。如下图所示：

![](/images/unity_mesh_tool/8.png)

最后，合并之后的Mesh的效果:

![](/images/unity_mesh_tool/7.png)

现在渲染这五个Cube只需要一个DrawCall即可。
