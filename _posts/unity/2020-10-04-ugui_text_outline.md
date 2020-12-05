---
title: 利用着色器实现UGUI的文本描边
date: 2020-10-04 23:59:40
type: "tags"
comments: false
tags: unity,UGUI,unity2019.4.14f
---

> Unity2019.4.14f, UGUI

首先看UGUI自带Outline脚本实现的文本描边和利用着色器实现的文本描边效果对比（可以把图片放大点看，效果更明显一些）：

![](/images/ugui_text_outline/9.png)

在UGUI中，对文字的描边效果的实现原理其实很简单，类似于把文字复制多份叠加在多个偏移位置上。利用UGUI的Outline实现的效果如图：

![](/images/ugui_text_outline/2.png)

<center>图[1]</center>

可以看到，UGUI自带的Outline描边其实就是把原本一份Text顶点复制成四份，，四份顶点分别向四个轴偏移相同距离。具体实现是这样的：

```csharp
public override void ModifyMesh(VertexHelper vh)
{
     //...省略部分代码
     var start = 0;
     var end = verts.Count;
     //往右上角偏移。
     ApplyShadowZeroAlloc(verts, effectColor, start, verts.Count, effectDistance.x, effectDistance.y);
     start = end;
     end = verts.Count;
     //往右下角偏移
     ApplyShadowZeroAlloc(verts, effectColor, start, verts.Count, effectDistance.x, -effectDistance.y);
     start = end;
     end = verts.Count;
     //往左上角偏移
     ApplyShadowZeroAlloc(verts, effectColor, start, verts.Count, -effectDistance.x, effectDistance.y);
     start = end;
     end = verts.Count;
     //往左下角偏移
     ApplyShadowZeroAlloc(verts, effectColor, start, verts.Count, -effectDistance.x, -effectDistance.y);

     //...省略部分代码
}
```

这样实现的优点是原理简单，功能实现比较方便。不过有两个明显的缺点：

1. 描边的效果不是特别理想，可以看到图[1]中的 “这”这个字，在边缘的转角处的描边很多没有连贯起来。

   ![](利用着色器实现UGUI的文本描边.assets/2.png)

2. 因为这种描边相当于额外增加了3份的顶点数，所以描边之后带来的顶点渲染消耗也是成倍增加的。比如对于单个文字来说，描边和不描边的区别如下：

   |        | 顶点个数 | 三角形个数 |
   | :----: | :------: | :--------: |
   |  描边  |    30    |     10     |
   | 不描边 |    4     |     2      |

为了解决UGUI的Outline的第一个问题，我看到有些方法是增加重叠的三角形，让叠加的三角形能做不同的偏移来覆盖更多的边缘区域，但是这个方法会放大第二个问题的影响。因为顶点数是成倍增加的，而且一般UI界面的文字数量时比较多的，如果单个文字的顶点数成倍增加，势必会导致整个UI界面渲染的压力会极大增加。

既然增加顶点带来的收益不明显，那么我们可以考虑通过着色器来改进描边效果。因为问题的根源在于描边不连续的问题，也就是和字体边缘相同距离的像素有的有颜色，有的没有颜色。那么其实我可以不用生成多份顶点数据来做描边颜色，我可以直接在像素着色器中来根据原本的Text来计算每个文字的边缘多少个像素以内（描边宽度）的像素直接渲染为描边的颜色就可以了。

但是在像素着色器中，UGUI已经把Text处理成了一张纹理贴图，根据自己像素的坐标本身不能判断处于Text的什么位置。那么我们得换种思路，我们可以通过Text的纹理的像素来确定Text的位置，因为Text贴图有像素的地方必然是文字，没有文字的地方Alpha都是为0的。类似这样：

![](/images/ugui_text_outline/3.png)

<center>图[2]</center>

红色线框内就是类似于一张Text的纹理。有文字的地方才有像素（就是文字的颜色），没有文字的地方都是透明的（Alpha为0）。这样我们在像素着色器中可以根据当前像素的透明度来判断：

1. 如果当前渲染的像素Alpha>0，那么这个像素肯定是文字本身的像素。
2. 如果当前渲染的像素Alpha<=0，那么这个像素肯定不是文字本身的像素。
3. 如果当前渲染的像素的相邻像素Alpha>0，那么这个像素肯定是文字边缘像素。我们只需要把这种像素都渲染为描边颜色，那么自然得到字体的描边效果。

实现判断相邻像素需要几个条件：

1. 到底相邻几个像素，如果只判断相邻一个像素，那么渲染得到的就是在文字的边缘有一个像素的描边。由此得知，判断几个像素决定了描边有多少个像素，也就是描边宽度。
2. 描边的颜色。我们判断出需要描边的像素之后，我们需要有描边的颜色信息。

这两个条件是需要到像素着色器中有的值，类似于UGUI的Outline，我们可以通过MoifiedShadow组件来修改Text的顶点数据，这里我们不需要修改顶点数据，我们可以在修改顶点数据的函数里面传入我们需要的着色器参数值。

```csharp
[RequireComponent(typeof(Text))]
public class ShaderOutline : ModifiedShadow
{
    public override void ModifyMesh(VertexHelper vh)
    {
        //...省略很多代码
        //把描边需要的  1.描边宽度 2.描边颜色  传入到着色器中
        graphic.material.SetFloat("_OutlineWidth", Mathf.Max(effectDistance.x, effectDistance.y));
        graphic.material.SetVector("_OutlineColor", effectColor);
    }
}
```

在着色器中，我们可以参考高斯模糊的方式处理，在着色时都采样周围多个像素的值来决定当前像素的颜色。我们就采样周围8个像素的Alpha值来确定当前像素的Alpha，如果当前像素的Alpha>0那么证明在当前像素的相邻的8个像素中肯定有其中部分像素是文字本身的像素。类似于这样（这是示意图，真正采样的时候像素密度是根据屏幕分辨率来确定的）：

![](/images/ugui_text_outline/4.png)

当渲染像素 2 的时候，会采样到像素 1 。因为像素 1 是文字本身的像素，Alpha是大于0的。那么像素 2 最终也是Alpha大于0。计算代码如下：

```c++
//如果想要描边效果更佳平滑的话，升采样的像素点可以扩大到12或者更高，但是会带来更高的性能消耗
static const half2 UpSamplePixelCoord[8] =
{
	half2 (-1, 1),  half2 (0, 1),  half2 (1, 1),
	half2 (-1, 0),                 half2 (1, 0),
	half2 (-1, -1), half2 (0, -1), half2 (1, -1)
};

//升采样，每个像素根据周边8个像素的透明度来确定是否显示描边颜色
fixed UpSamplePixel(int index, v2f IN)
{
	half2 realOutlineWidth = _MainTex_TexelSize.xy * UpSamplePixelCoord[index] * _OutlineWidth;
	half2 pixelUV = IN.texcoord + realOutlineWidth;
	half4 pixelAlpha = (tex2D(_MainTex, pixelUV) + _TextureSampleAdd).w;
	return pixelAlpha;
}

fixed4 frag(v2f IN) : SV_Target
{
	//当前像素中心点的颜色
	fixed4 color = (tex2D(_MainTex, IN.texcoord) + _TextureSampleAdd) * IN.color;
	half4 outlineColor = half4(_OutlineColor.xyz, 0);
	int index = 0;
	for (; index < 8; ++index)
	{
		outlineColor.w += UpSamplePixel(index, IN);
	}
	outlineColor.w = clamp(outlineColor.w, 0, 1);
    //把文字本身的颜色和描边的颜色做一个过渡。
	color = lerp(outlineColor, color, color.a);
	return color;
}
```

我们现在可以看下利用Shader描边的效果：

![](/images/ugui_text_outline/5.png)

可以看到描边的效果有了，但是在字体的边缘都被截掉了。这是因为文字的顶点组成的矩形区域是确定的（每个文字是由两个三角形组成的矩形，UGUI处理之后到着色器中是Mesh数据），我们增加的描边相当于加宽了文字，但是原本文字的区域我们没有加大，自然描边超出的部分就会被截掉。所以接下来我们要处理的就是把组成单个文字的两个三角形加宽，同时我们得让文字本身的大小保持不变，这样文字四周的宽度留出来给描边。类似于这样：

![](/images/ugui_text_outline/6.png)

两个相同的文字，左边的文字的顶点矩形区域明显是比右边的是小的。右边的文字的四周空出了更多空白区域用于描边像素（空多少区域根据自己的需要描边的宽度来确定）。代码处理的方式还是通过MoifiedShadow组件来修改文字的顶点position和uv来实现：

```csharp
public override void ModifyMesh(VertexHelper vh)
{
    //...省略很少一部分代码
    List<UIVertex> vertexStream = ListPool<UIVertex>.Get();
    vh.GetUIVertexStream(vertexStream);
    float expandWidth = Mathf.Abs(effectDistance.x * 0.5f);
    float expandHeight = Mathf.Abs(effectDistance.y * 0.5f);
    //   v1----v2
    //   | \    |
    //   |   \  |  
    //   |     \|
    //   v4----v3
    //  顺序按照vertex来确定的，一个文字由两个triangle组成，并且任意文字的vertex顺序都是相同的
    //  但是不同文字的uv的顺序不一样
    int length = vertexStream.Count;
    for (int i = 0; i < length; i += 6)
    {
        Vector2 expandPositionSize = GetExpandPositionSize(expandWidth, expandHeight);
        Vector2 shrinkUvSize = GetShrinkUvSize(vertexStream, expandWidth, expandHeight, i);
        RePackTextVertex(vertexStream, i, expandPositionSize, shrinkUvSize);
    }
    vh.Clear();
    vh.AddUIVertexTriangleStream(vertexStream);
    ListPool<UIVertex>.Release(vertexStream);
        //...省略很少一部分代码
}
```

代码中的 **GetExpandPositionSize** 比较好处理，因为文字的顶点顺序是确定的，按照固定的顺序去增加或者减去expand值，保证每个顶点的坐标向外扩即可。**GetShrinkUvSize**这个处理起来要稍微注意下，因为文字的UV的坐标不确定，我没有法线任何规律，也就是说两个文字的同一个顶点处，uv坐标顺序可能会不一样。这就需要自己根据每个uv的值来判断了，保证每个顶点的uv是向内缩即可，实现的原理如图（图中假设内缩大小为(0.2, 0.2)）：

![](/images/ugui_text_outline/11.png)

我这里的代码是这样处理的：

```csharp
private Vector2 ShrinkUvSize(Vector2 uv, Vector2 leftVertexUv, Vector2 rightVertexUv, Vector2 shrinkSize)
{
    float x = ShrinkValue(uv.x, leftVertexUv.x, rightVertexUv.x, shrinkSize.x);
    float y = ShrinkValue(uv.y, leftVertexUv.y, rightVertexUv.y, shrinkSize.y);
    return new Vector2(x, y);
}
/// <summary>
///   left  三个uv点关系。value：当前uv点，left：value的左边uv点，right：value的右边UV点
///   |
///   |
///  value --- right
/// </summary>
private float ShrinkValue(float value, float left, float right, float shrinkValue)
{
    if (value < left)
    {
        value -= shrinkValue;
    }
    else if (value == left)
    {
        value = value < right ? value-shrinkValue : value +shrinkValue;
    }
    else
    {
        value += shrinkValue;
    }
    return value;
}
```

处理了文字描边被裁减的问题之后我们再看下效果：

![](/images/ugui_text_outline/7.png)

每个文字的边缘出现了不需要的颜色像素，因为我们修改了uv，把文字的顶点的uv值放大了，所以放大的那部分uv对应的像素原本是其他文字的像素现在被采样近来了。所以我们需要剔除掉这部分本不是当前文字的像素。这个可以直接在着色器中处理：

```c++
//判断像素是否在三角形中：像素点依次和三角形的两个顶点的叉积的方向同向。
fixed IsPixelInTriangle(float3 pixelPos, float3 a, float3 b, float3 c)
{
	float z1 = cross(pixelPos - a, a - b).z;
	float z2 = cross(pixelPos - b, b - c).z;
	float z3 = cross(pixelPos - c, c - a).z;
	return z1 * z2 > 0 && z2 * z3 > 0;
}

//判断像素是否在一个Rect中，由于Rect是由两个Triangle组成，所以只需要判断顶点是否在任意一个Triangle中即可。
fixed IsPixelInRect(float3 pixelPos, float3 a, float3 b, float3 c, float3 d)
{
	return IsPixelInTriangle(pixelPos, a, b, c) || IsPixelInTriangle(pixelPos, c, d, a);
}
```

最后我们再看下渲染的结果：

![](/images/ugui_text_outline/8.png)

现在已经没有问题了，而且效果还是不错的，描边比较柔和并不会出现UGUI的outline那种不连续的情况。我们对比下两种描边的效果：

![](/images/ugui_text_outline/9.png)

再来看下两种描边的三角形数量：

![](/images/ugui_text_outline/10.png)

总体来说利用着色器来进行描边的最终效果对于UGUI的outline的改进还是比较明显的。利用着色器来描边的方式把CPU的性能消耗转移到了GPU，因为我们需要对每个像素做8次的采样，自然就增加了GPU的性能消耗。两种方式哪种合适还得结合自己的项目需求来定。

---

参考：[https://gameinstitute.qq.com/community/detail/114969](https://gameinstitute.qq.com/community/detail/114969)

