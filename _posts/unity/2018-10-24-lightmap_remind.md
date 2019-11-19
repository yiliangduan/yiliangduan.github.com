---
title: Unity的Lightmap加载的内存问题
date: 2018-10-24 20:26:10
type: "tags"
tags: unity
comments: false
---

> unity 5.6.6f1

再检查项目的内存问题时发现Lightmap的加载会消耗大量堆内存(高达17M，总共差不多35张lightmap贴图)。抛开lightmap的资源内存分配理论上不应该消耗堆内存才对，查看加载的代码最终发现了问题。

* LightmapSettings.lightmaps 每次返回的是一个新创建的数组。在所有的lightmap加载完成后去查找单独每个lightmap数据的索引使用到这个API，这个是分配堆内存的主要原因。
* lightmapData.lightmapColor.name  每次都会创建一个新的string对象。项目里面使用查找每个lightmapData的时候使用了名字去查找对比，调用次数非常频繁导致消耗了不少堆内存

<!-- more --> 

下面利用代码来解释上面两个导致消耗堆内存的问题:

```csharp
public class TestLightmap : MonoBehaviour {

    private readonly List<string> mWillLoadLightmapNameList = new List<string>();
	
    //把lightmapData缓存起来，避免需要用到LightmapData的时候去LightmapSettings.lightmaps取
    private readonly Dictionary<string, LightmapData> mLoadedLightmapDataDict = new Dictionary<string, LightmapData>();
    //把名字缓存起来，缓存的顺序和LightmapSettings.lightmaps保持一致，确保查找索引时是相同的
    private readonly List<string> mLoadedLightmapNameList = new List<string>();

    public void Start()
    {
        LoadLightmap();
    }

    public void LoadLightmap()
    {
        mLoadedLightmapDataDict.Clear();

        LightmapData[] lightmapDataArray = new LightmapData[mWillLoadLightmapNameList.Count];

        for (int i=0; i<mWillLoadLightmapNameList.Count; ++i)
        {
            string tmpLightmapName = mWillLoadLightmapNameList[i];

            //使用Resources.Load仅仅为了demo测试，实际最好用AssetBundle加载
            Texture2D lightmapTex = Resources.Load<Texture2D>(tmpLightmapName);

            if (null != lightmapTex)
            {
                LightmapData lightmapData = new LightmapData
                {
                    lightmapColor = lightmapTex
                };

                lightmapDataArray[i] = lightmapData;

                mLoadedLightmapNameList.Add(tmpLightmapName);
                mLoadedLightmapDataDict.Add(tmpLightmapName, lightmapData);
            }
            else
            {
                Debug.LogError("Load lightmap texture error! " + tmpLightmapName);
            }
        }

        LightmapSettings.lightmaps = lightmapDataArray;
    }

    /// <summary>
    /// 一个错误的例子,造成堆内存消耗
    /// </summary>
    public int GetLightmapIndex_Error(string lightmapName)
    {
        // LightmapSettings.lightmaps每次调用都会创建一个新的lightmaps数组，会分配堆内存
        for (int i=0; i<LightmapSettings.lightmaps.Length; ++i)
        {
            LightmapData lightmapData = LightmapSettings.lightmaps[i];

            if (null != lightmapData && null != lightmapData.lightmapColor)
            {
                //lightmapData.lightmapColor.name 每次调用多会分配一个新的string导致分配堆内存
                if (lightmapName == lightmapData.lightmapColor.name)
                    return i;
            }
        }

        return -1;
    }


    /// <summary>
    /// 0堆内存消耗的获取方式
    /// </summary>
    /// <param name="lightmapName"></param>
    /// <returns></returns>
    public int GetLightmapIndex_Suggest(string lightmapName)
    {
        for (int i=0; i<mLoadedLightmapNameList.Count; ++i)
        {
            if (lightmapName == mLoadedLightmapNameList[i])
            {
                return i;
            }
        }
        return -1;
    }

    /// <summary>
    /// 0堆内存消耗的获取方式
    /// </summary>
    /// <param name="lightmapName"></param>
    /// <returns></returns>
    public LightmapData GetLightmapData_Suggest(string lightmapName)
    {
        LightmapData lightmapData;
        if (mLoadedLightmapDataDict.TryGetValue(lightmapName, out lightmapData))
        {
            return lightmapData;
        }
        return null;
    }
}

```

查出并修改问题之后看了下引擎源码的LightmapSettings.lightmaps API

```cpp
	// Lightmap array.
	CUSTOM_PROP static LightmapData[] lightmaps
	{
        return VectorToScriptingClassArray<LightmapData, LightmapDataMono> (GetLightmapSettings().GetLightmaps(), GetScriptingTypeRegistry().GetType("UnityEngine", "LightmapData"), LightmapDataToMono); 
	}
```
这是c++部分在返回给csharp时的处理。函数内调用了VectorToScriptingClassArray接口，也就是在返回给csharp层的时候把c++的lightmaps数组转换到新分配的csharp数组里然后返回给csharp层，相当于每次在csharp层调用LightmapSettings.lightmaps就会创建一个新的lightmaps数组了。

*End*