---
title: Unity Texture合并工具
date: 2018-12-06 21:26:10
type: "tags"
tags: unity
comments: false
---

> Unity 2018.3.5f1

这里实现了一个简单的合并贴图的工具，在Unity的Project窗口中，选中一个存放图片的文件夹。右键菜单中可以看到 [Pack Texture], [Pack Texture(All Relayout)] 选项, 即可合并该文件夹下的所有贴图。具体如图所示:

选中一个存放了图片的文件夹:

![](/images/unity_atlas_tool/2.png)

执行合图操作之后，将会合并贴图并将结果生成在事先在AtlasConfig.cs中配置好的路径中(这里测试配置的是Assets/Res目录)。结果包含两类文件Asset和Atlas，Asset文件保存了Atlas文件中每张贴图在图集中的偏移，缩放和对原始图片的引用。如果选中的目录下既有带Alpha的图片又有不带Alpha的图片，工具会根据是否带Alpha进行归类并且结果输出到不同的目录。如下图:

![](/images/unity_atlas_tool/1.png)

生成的Asset文件保存的信息如图:

![](/images/unity_atlas_tool/3.png)

由于这些合并后的贴图是用于Mesh的贴图，并且MeshRenderer会根据这些贴图再图集中的偏移而要改变自己的UV，所以为了避免每次合图之后MeshRenderer的都需要重写UV的操作，这里的合图有两个选项*Pack Texture*和*Pack Texture(All Relayout)*。*Pack Texture*选中执行过程中不会对之前合并好的贴图进行改变，保持其在图集中的位置偏移，而*Pack Texture(All Relayout)*相当于重新排列每张图片在图集中的位置。所以需要注意这两点:

- Pack Texture:  如果之前合并过贴图，该种模式下会继承上次贴图的合并后在图集中的偏移，不会改动之前已经合过贴图的位置。
- Pack Texture(All Relayout): 全部重新排列合图，这种情况下可能改变之前已经合过贴图的在图集中的偏移。



<!-- more --> 

合并贴图的工具的实现还是比较简单的，这里列出以下主要的步骤:

```csharp
/// <summary>
/// 合并贴图
/// </summary>
private static List<TextureAtlas> PackAtlas(List<Texture2D> textureList, string atlasName, bool isTransparent, int atlasWidth, int atlasHeight)
{
    int atlasIndex = 0;
    int textureCount = textureList.Count;

    textureList.Sort((a, b) => { return a.width * a.height - b.width * b.height; });

    List<TextureAtlas> atlasList = new List<TextureAtlas>();

    string fileName = atlasName + "_" + atlasIndex;
    string assetPath = TextureAtlas.GetAssetPath(isTransparent, fileName);

    //加载已经存在的Asset文件
    while (File.Exists(assetPath))
    {
        TextureAtlas atlas = AssetDatabase.LoadAssetAtPath<TextureAtlas>(assetPath);

        if (null != atlas)
        {
            atlas.Layout();
            atlasList.Add(atlas);
        }

        atlasIndex++;

        fileName = atlasName + "_" + atlasIndex;
        assetPath = TextureAtlas.GetAssetPath(isTransparent, fileName);
    }

    List<Texture2D> newlyTextures = new List<Texture2D>();

    //找出新增的图片
    for (int i=0; i<textureList.Count; ++i)
    {
        bool found = false;

        for (int j=0; j<atlasList.Count; ++j)
        {
            TextureAtlas atlas = atlasList[j];

            if (null != atlas)
            {
                TextureAtlasElement element = atlas.GetElement(textureList[i]);
                if (null != element)
                {
                    found = true;
                    break;
                }
            }
        }

        if (!found)
            newlyTextures.Add(textureList[i]);
    }

    //Asset中删除图片已经删除的记录
    for (int i=0; i<atlasList.Count; ++i)
    {
        TextureAtlas atlas = atlasList[i];

        for (int j=atlas.ElementList.Count-1; j>=0; --j)
        {
            Texture2D elementTex = atlas.ElementList[j].Tex;

            bool found = false;
            for (int m=0; m<textureList.Count; ++m)
            {
                if(elementTex == textureList[m])
                {
                    found = true;
                    break;
                }
            }
            if (!found)
            {
                atlas.RemoveElementAt(j);
            }
        }
    }

    //排列新增的图片
    for (int i=0; i<newlyTextures.Count; ++i)
    {
        Texture2D newlyTexture = newlyTextures[i];

        if (null != newlyTexture)
        {
            bool added = false;

            for (int j=0; j<atlasList.Count; ++j)
            {
                if(atlasList[j].AddTexture(newlyTexture))
                {
                    added = true;
                    break;
                }
            }

            if (!added)
            {
                fileName = atlasName + "_" + atlasList.Count;
                assetPath = TextureAtlas.GetAssetPath(isTransparent, fileName);

                TextureAtlas atlas = ScriptableObject.CreateInstance<TextureAtlas>();
                if (null != atlas)
                {
                    atlas.Init(atlasWidth, atlasHeight, false, isTransparent, fileName);
                    atlas.AddTexture(newlyTexture);

                    atlasList.Add(atlas);
                }
                else
                {
                    Debug.Log("Create atlas instance failed.");
                }
            }
        }
    }

    for (int i = 0; i < atlasList.Count; ++i)
    {
        EditorUtility.DisplayProgressBar("", "Pack atlas ", (i + 1) / atlasList.Count);

        atlasList[i].Pack();
    }

    EditorUtility.ClearProgressBar();

    return atlasList;
}
```



怎样把一张贴图合并到一张大的Atlas中，这个是关键所在。上述代码的*AddTexture*函数实现的就是这个功能。我们需要记录Atlas哪些区域是已经填充了贴图的，哪些区域是空闲的（空闲的空间可能是分散成一块一块的）。当我们需要再往这个Atlas插入一张贴图时，我们就根据这张准备插入的贴图的宽和高在Atlas的空闲中间中寻找一个合适的位置，什么位置比较合适呢？这里有几种策略（这里参考自这个开源项目[RectangleBinPack](https://github.com/juj/RectangleBinPack)）:

* 依据贴图的width，height短的这一边，在空闲空间中找到一块填充率最高的空闲空间。
* 依据贴图的width，height长的这一边，在空闲空间中找到一块填充率最高的空闲空间。
* 依据贴图的width，height找到在每块可以填充的空间空间中的填充率的空闲空间
* 在空闲空间中按照找到的剩余空间块的左下角坐标来计算，加上高度之后取y值越小得那个空闲块
* 在空闲空间中根据长，宽的比例相加结果为因子来计算最优

找到一块空闲空间之后填充新的贴图，然后再对这个空闲空间中生下来的部分进行分割，填充好的和剩余的空间分别记录到记录Atlas填充信息和空闲信息中区。比如我新插入的贴图的大小为600x256，但是我再Atlas的空闲空间中找到的一块合适的空闲空间的大小为1024x256。那么新插入贴图填充之后剩余424x256的大小记录到空闲空间信息中。用一张图来描述:

![](/images/unity_atlas_tool/4.jpg)

> 填充区域为:   
>
> （0，0，512，512）
>
> （512，0，256，512）
>
> （0，512，640，256）
>
> 空闲区域为:   
>
> （768，0，256，1024）
>
> （640，512，484，512）
>
> （0，768，640，256）
>
> （0，768，1024，256）

插入一张600x256的图片之后

![](/images/unity_atlas_tool/5.jpg)

> 填充区域为:   
>
> （0，0，512，512）
>
> （512，0，256，512）
>
> （0，512，640，256）
>
> （0，768，600，256）//新增
>
> 空闲区域为:   
>
> （768，0，256，1024）
>
> （640，512，484，512）
>
> （0，768，640，256）
>
> （600，768，1024，256）//修改

整个贴图合并就是这样一张一张的排列进去的，所有的贴图排列完成之后完后把所有贴图的像素写入到一张新的Texture2D图片中。

```csharp
private void WriteTexture()
{
    Texture2D atlas = new Texture2D(Width, Height, TextureFormat.RGBA32, false);

    Color[] defaultColors = atlas.GetPixels();

    //设置默认的颜色为黑色

    Color defaultColor = new Color(0, 0, 0, 0);
    for (int i = 0; i < defaultColors.Length; ++i)
    {
        defaultColors[i] = defaultColor;
    }

    //把每张图片的像素写入atlas中.
    for (int i = 0; i < ElementList.Count; ++i)
    {
        TextureAtlasElement element = ElementList[i];

        if (null != element && null != element.Tex)
        {
            Color[] colors = element.Tex.GetPixels();

            int offsetX = (int)element.Offset.x;
            int offsetY = (int)element.Offset.y;

            for (int column = 0; column < element.Tex.width; ++column)
            {
                for (int row = 0; row < element.Tex.height; ++row)
                {
                    int atlasColorIndex = (column + offsetX) + (row + offsetY) * atlas.width;
                    int texColorIndex = column + row * element.Tex.width;

                    defaultColors[atlasColorIndex] = colors[texColorIndex];
                }
            }
        }
    }

    atlas.SetPixels(defaultColors);
    atlas.Apply();

    Atlas = atlas;

    File.WriteAllBytes(AtlasPath, atlas.EncodeToPNG());

    AssetDatabase.Refresh(ImportAssetOptions.ForceUpdate);
    AssetDatabase.SaveAssets();
}
```

然后把这些排列的数据用序列化保存，下次需要新增或者删减图片时必须根据上一次合并的排列数据来排列。

*End*
