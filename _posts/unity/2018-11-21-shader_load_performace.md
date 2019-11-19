---
title: Unity Shader加载性能消耗问题
date: 2018-11-21 23:30
type: "tags"
comments: false
tags: unity
---


>unity 5.6.6f1

在做性能优化时，发现Shader在每次加载使用了Shader的Prefab资源的时候都会调用*Shader.Parse*，*Shader.CreateGPUProgram*两个函数，这个函数的调用消耗了大量的CPU性能。这里写个[demo](https://github.com/elangduan/unity-shaderpreload)测试下，测试的demo创建了几个prefab，每个Prefab都挂在material，然后没隔0.5s执行一次随机创建一个prefab，再隔0.5s销毁，依次重复。主要如下:

<!-- more --> 

```csharp
   private void LoadObjects()
    {
        string resAssetName = "Cube_" + Random.Range(1, 7);
        string resABName = resAssetName;

        GameObject loadObject = ABLoader.Instance.LoadAsset<GameObject>(resAssetName, resABName);

        if (null != loadObject)
        {
            GameObject newObject = Instantiate(loadObject);
            newObject.transform.localPosition = Vector3.zero;
            newObject.transform.localScale = Vector3.one;

            mLoadObjectList.Add(newObject);

            bLoaded = true;
        }

        ABLoader.Instance.UnLoadAsset(resAssetName, resABName);
    }

    private void ClearObjects()
    {
        for (int i = mLoadObjectList.Count - 1; i >= 0; --i)
        {
            Destroy(mLoadObjectList[i]);
        }

        mLoadObjectList.Clear();
        Resources.UnloadUnusedAssets();

        bLoaded = false;
    }

    private void Update()
    {
        mCurWaitingTime += Time.deltaTime;
        if (mCurWaitingTime > RefreshTimeInterval)
        {
            mCurWaitingTime = 0;
            if (bLoaded)
                ClearObjects();
            else
                LoadObjects();
        }
    }
```

测试结果如图:

![](/images/unity_shader_load_performance/1.png)

*Shader.CreateGPUProgram*具体是做什么工作的呢？在[Unity论坛](https://forum.unity.com)上找到了Unity开发作者的[解答](https://forum.unity.com/threads/profiling-shader-creategpuprogram.249362/)

>It's the submission of a shader to the graphics card, which includes the graphics card driver compiling it into a card-specific format.

>You may want to look into ShaderVariantCollection.

其实就是把shader从CPU提交到GPU，这其中包含了显卡驱动把这些shader编译成GPU特定的格式操作。按照作者的建议我查阅了ShaderVariantCollection，原来Unity提供了预编译Shader的接口。查看了ShaderVariantCollection的API文档，Unity在运行过程中会自动搜集使用的Shader以及这些Shader用到的Variant。然后我们可以在GraphicsSettings菜单里保存这些搜集的数据成ShaderVariantCollection 格式的Asset文件当项目运行起来时加载这个Asset，然后调用ShaderVariantCollection的WarmUp方法即可，这些Shader就会加载解析并编译好，下次加载的资源如果引用到了这个Shader的话不需要再次去解析编译了，避免了重复的CPU开销。ShaderVariantCollection具体工作原理[文档描述](https://docs.unity3d.com/ScriptReference/ShaderVariantCollection.WarmUp.html)如下:
>Calling this function will perform dummy one-invisible-triangle rendering for the shaders and their variants in this ShaderVariantCollection.

也就是说当我们调用WarmUp时，Unity会利用这些Shader去渲染一个不可见的三角形，这样的话Unity会调用*Parse*和*CreateGPUProgram*，然后一直引用这些Shader不释放(自己根据Profiler观察的)。

那么就使用下ShaderVariantCollection，按照*Unity Manual*的步骤，在*GraphicsSettings*中保存当前的ShaderVariantCollection。然后在在加载测试的Prefab之前加载好ShaderVariantCollection(这里Build成了AssetBundle方式加载):

![](/images/unity_shader_load_performance/2.png)

```csharp
ShaderVariantCollection loadObject = ABLoader.Instance.LoadAsset<ShaderVariantCollection>(ShaderVariantsCollectionABName, ShaderVariantsCollectionABName);
if (null != loadObject)
{
    loadObject.WarmUp();
}
```

测试之后发现一个奇怪的问题，在加载使用了项目中自己编写的shader时不会调用*Shader.Parse*和*Shader.CreateGPUProgram*, 但是使用了Unity内置的Shader时任然会调用这两个函数造成CPU开销。由于是Unity内置Shader我们自己没办法来改变它的加载和缓存情况，这里想到的一个办法是直接下载一个Unity的内置Shader放到工程里面然后用同样的Shader替换掉内置Shader，这里主要做的工作就是替换工作，其实很简单直接利用*AssetDatabase.FindAssets*查找所有的Material，然后替换Material的Shader即可:

```csharp
var guids = AssetDatabase.FindAssets("t:Material");
foreach (var guid in guids)
{
    var path = AssetDatabase.GUIDToAssetPath(guid);
    if (path.ToLower().EndsWith("mat"))
    {
        var mat = AssetDatabase.LoadAssetAtPath<Material>(path);
        if (mat && mat.shader)
        {
        	//Shader.Find在查找的时候，假如Unity工程中的Shader和内置Shader同名，则返回的是工程中的Shader
            mat.shader = Shader.Find(mat.shader.name);
        }
    }
}
```

在替换了使用的内置Shader之后再运行demo测试，在预加载了ShaderVariantsCollection之后再加载这些Shader不再有*Shader.Parse*和*Shader.CreateGPUProgram*开销了。

![](/images/unity_shader_load_performance/3.png)

在Unity的官方文档里面查不到可以保存或者获取到ShaderVariantCollection的数据的接口，很奇怪的是GraphicsSetting是通过什么接口来保存的，想到GraphicsSetting里面显示了*Currently tracked: x shaders x total variants*,那么这个获取这个数据的接口一定写在引擎的csharp层的编辑器代码中可以查的到，于是下载了[Unity的csharp引擎代码](https://github.com/Unity-Technologies/UnityCsReference/tree/2017.1)，搜索关键字*Currently tracked: *，果然在ShaderUtil中有个三个Private的方法分别是SaveCurrentShaderVariantCollection, GetCurrentShaderVariantCollectionShaderCount和GetCurrentShaderVariantCollectionVariantCount。这三个方法的作用方法名已经写得很清楚了，这里利用反射的方式调用到保存ShaderVariantsCollection数据来实现一个脚本生成ShaderVariantsCollection的工具。

```csharp
private const string SVCSavePath = "Assets/Res/GameShaderVariants.shadervariants";

    /// <summary>
    /// 生成ShaderVariantsCollection
    /// </summary>
    [MenuItem("Assets/SaveShaderVariantCollection")]
    public static void SaveShaderVariantCollection()
    {
        BindingFlags bindingFlag = BindingFlags.Static | BindingFlags.NonPublic;

        MethodInfo saveCurrentSVCMethod = typeof(ShaderUtil).GetMethod("SaveCurrentShaderVariantCollection", bindingFlag, null, new System.Type[] { typeof(string)}, null);
        if (null != saveCurrentSVCMethod)
        {
            saveCurrentSVCMethod.Invoke(null, new object[] { SVCSavePath});

            int shaderCount = 0;
            MethodInfo getCurrentShaderCountMethod = typeof(ShaderUtil).GetMethod("GetCurrentShaderVariantCollectionShaderCount", bindingFlag);
            if (null != getCurrentShaderCountMethod)
            {
                shaderCount = (int)getCurrentShaderCountMethod.Invoke(null, null);    
            }

            int variantCount = 0;
            MethodInfo getCurrentVariantsCountMethod = typeof(ShaderUtil).GetMethod("GetCurrentShaderVariantCollectionVariantCount", bindingFlag);
            if (null != getCurrentVariantsCountMethod)
            {
                variantCount = (int)getCurrentVariantsCountMethod.Invoke(null, null);
            }

            EditorUtility.DisplayDialog("提示", "保存ShaderVariantCollection成功," + shaderCount + "个Shader. "+ variantCount + "个变量.", "确定");
        }
    }
```
*END*