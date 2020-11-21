---
title: Unity 法线贴图总结
date: 2019-10-07 16:44:10
categories: graphics
tags: [graphics,unity]
comments: false
---

法线贴图是一种凹凸贴图（Bump Map），它可以让你将其纹理数据添加到模型，从而为模型增加表面细节（凹凸，凹槽和划痕）和光照效果。法线贴图分为两种：切线空间的法线贴图（Tangent-space normal map）和模型空间的法线贴图（Object-space normal map），两者的唯一区别正如命名那样法线所在的坐标系不同。通常情况下（游戏开发）我们使用的都是切线空间的法线贴图，所以这里介绍切线空间的法线贴图。

**切线空间（Tangent space）**

首先我们了解下什么是切线空间。切线空间 由Normal向量，Tangent向量和Binormal向量，三个互相垂直的向量组成。如图：

![](/images/normalmap/1.jpg)

图片来源:http://intcomputergraphics.blogspot.com/2013/04/tangent-space-normal-mapping.html

Tangent和Binormal只需要满足两者互相垂直并且与顶点所在的曲面相切即可，所以任意一个三维的顶点它的Tangent和Binormal的数量时无限多个的。为了方便我们对采样法线贴图，规定切线空间的Tangent向量保持和UV坐标系的U方向一致，Binormal向量保持和UV坐标系的V方向一致，Normal方向保持和Z方向（Up方向，在Unity中这点和模型坐标或者其他坐标系不同）一致。这样对于某个顶点来说，以它为原点的切线空间就是固定的了。

**法向量的存储**

法向量为什么要存储在切线空间中呢？我们在Unity中使用法线贴图的时候应该有过这样的经历，同一张法线贴图会使用在不同的Mesh上。不同的Mesh的顶点和面基本上是不同的，但是却可以使用同一张贴图，这就是切线空间好处，切线空间是建立在以顶点为原点的坐标系，它不需要关心你Mesh的结构。如果你的法向量存储在模型的坐标空间，那么采样出来的法向量对于其他模型来说没办法转换到自身的模型坐标了那就没办法使用。如果是存储的是切线空间，那么可以通过对应顶点的切线空间坐标系转换到自身的模型坐标来计算光照了，或者直接在切线空间计算光照即可。

对于切线空间还有个需要注意的地方，由于平台的差异（OpenGL平台UV的V轴坐标从下往上的值为[0, 1]，DirectX平台的UV的V轴坐标从上往下的值为[0, 1]）Binormal向量可能有两个方向。为了区分不同平台的这个差异，Unity在模型导入的时候计算出一个符号标记，存储在Tangent.w变量，在自己的着色器需要计算Binormal时需要用到（Mesh存储的是每个顶点的Binormal，如果需要精确每个Fragment的Binormal，则可以通过法向量和切线向量计算得来）。计算Tangent.w的方法：

```c++
//正交化切线向量
void OrthogonalizeTangent (TangentInfo& tangentInfo, Vector3f normalf, Vector4f& outputTangent)
{
    //tangentInfo里面带了tangent向量和binormal向量数据。
    TangentInfo::Vector3d tangent  = tangentInfo.tangent;
    TangentInfo::Vector3d binormal = tangentInfo.binormal;
    
    //1.此处省略了正交化tangent和binomal。因为导入的Mesh的tangent和binormal，normal和binormal可能不垂直。
    
    //2.此处省略tangent和binormal向量的长度异常检查。如果长度小于浮点型最小精度，则直接用标准化单位向量（如Up（0， 0，1））。

    //normalf, tangentf，binormalf是上面两步计算而来的长度大于0互相垂直向量
    //并且，这三个向量组成了切线坐标系。
    
    //先通过 normalf和tangentf计算出该平台的binormal向量
    Vector3f binormalPlatform = Cross(normalf, tangentf);
    
    //然后通过比较binormalPlatform和binormalf是否是相同方向
    float dp = Dot (binormalPlatform, binormalf);
    
    //如果两者的方向相同，则表示Mesh的binormal和当前平台采用的UV方向一致
    //就不需要flip，给tangent.w赋值为1。
    if ( dp > 0.0F)
        outputTangent.w = 1.0F;
    //反之，表示当前平台的V方向和Mesh数据里面的V方向是不同平台的标准
    //需要flip下，给tangent.w赋值为-1。
	else
		outputTangent.w = -1.0F;
}
```

在着色器中计算每个Fragment的Binormal的方法为：

```c++
//如果Mesh的Scale值为（-1， 1， 1）值，那么需要对这个Mesh以zy组成的平面做镜像，那么计算出来的binormal也是需要做镜像的，unity_WorldTransformParams.w存储的就是是否需要做镜像的符号标记
//tagent.w 是由于不同平台UV坐标轴的差异建立的符号标记
float3 binormal = cross(normal, tangent.xyz) * (tangent.w * unity_WorldTransformParams.w);
```

在法线贴图中，法向量的XYZ值存储在贴图的颜色通道中。Unity会分别处理贴图是否使用了压缩格式（PC平台）的情况：

* 不压缩的情况下法线贴图对法向量的XYZ值分别存储在RGB中，值的格式为（X，Y，Z，1）。
* 压缩的情况下又会分别根据压缩格式做不同的处理，Unity中有两种针对贴图的压缩格式：
  * DXT5，法向量的XY值分别存储在贴图的WG通道中，值的形式为（1，Y，1，X）
  * BC格式（BC5和BC7），这种格式法向量的XY值分别存储在贴图的RG通道，值的形式为（X， Y， 1， 1）。

使用这两种压缩格式法线贴图都不会存储Z值，因为法线贴图存储的法向量是单位向量，单位向量的长度为1，所以Z值可以通过XY分量计算出来（减小了贴图的存储大小，但是牺牲了GPU的性能）。计算公式如下：

$$
Z = \sqrt{1 - X^2 - Y^2} 

\tag{1}
$$

此外由于贴图通道的值的范围为[0, 1]，而标准化的法向量的值为[-1, 1]（有正负方向），所以为了把向量值作为颜色值存储在贴图中，需要把向量的值的经过变换：

$$
f(N) = \frac{N+1}{2}

\tag{2}
$$

**法向量的读取**

现在来看下Unity内置着色器中解码法向量值的代码，结合之前讲的存储的方式来看解码的代码还是比较清晰的。


```c++
half3 UnpackScaleNormalRGorAG(half4 packednormal, half bumpScale)
{
    #if defined(UNITY_NO_DXT5nm)
  		 //未压缩
  		 //按照存储时的公式（2），把值还原成[-1, 1]的范围。
        half3 normal = packednormal.xyz * 2 - 1;
        #if (SHADER_TARGET >= 30)
            normal.xy *= bumpScale;
        #endif
        return normal;
    #else
  			//DXT5，BCX执行这里
        // 把w值转换到x值上，即（1，Y，1，X）变成（X，Y，1，X）
        packednormal.x *= packednormal.w;

        half3 normal;
        normal.xy = (packednormal.xy * 2 - 1);
        #if (SHADER_TARGET >= 30)
            normal.xy *= bumpScale;
        #endif
        normal.z = sqrt(1.0 - saturate(dot(normal.xy, normal.xy)));
        return normal;
    #endif
}

//bumpScale是缩放值
half3 UnpackScaleNormal(half4 packednormal, half bumpScale)
{
    return UnpackScaleNormalRGorAG(packednormal, bumpScale);
}
```

**法线贴图为什么主要是蓝色**

还有最后一个问题，为什么Unity编辑器中法线贴图看起来一般呈现的主要是是蓝色的（Unity使用的是切线空间的法线贴图，实际上模型空间的法线贴图表现没有特别的呈现主要为蓝色的现象），如图：

![](/images/normalmap/2.jpeg)

一个不对顶点像素产生任何作用（没有实际影响该顶点的光照计算，没有对该像素产生任何额外的表面效果）的切线空间的法向量是这样的（0， 0， 1），也就是正Up方向的单位向量。法向量的XYZ值存储到法线贴图的RGB话需要经过公式(2)转换下值的范围，转换之后为[（0.5， 0.5， 1）](https://rgb.to/rgb/128,128,255),这个颜色值显示出来是蓝色。也就是说如果一张法线贴图每个像素都是存储的这个颜色值，那么这张法线贴图呈现出纯蓝色的。一个模型按照实际表现出来的真实细节来看，大部分情况下法向量都是接近于Up值的，少部分细节（比如脸上的皱纹边缘，地板的缝隙处）法向量的值有相对比较大点的差异，如上法线贴图就是用在墙面模型的，可以看出来砖块的交界处的颜色明显不一样。
