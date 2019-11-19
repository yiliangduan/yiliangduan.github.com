---
title: 'Unity中shader代码的条件分支的性能问题'
date: 2017-08-20 17:03:34
type: "tags"
tags: unity
comments: false
---

### 优化方案

GPU对条件分支语句的执行性能比较敏感似乎是一个比较普遍的认知了，那么到底影响有多大呢？这里做了一次测试，测试针对Shader中的条件分支语句的性能情况。在有if语句和没有if语句时候的程序不同写法的:

> 针对if语句的优化方法，参考的[这里](http://answers.unity3d.com/questions/442688/shader-if-else-performance.html), 目前没有找到更好的替换if语句的办法，希望有更好办法的同学评论里面提出来，非常感谢！

```
fixed4 col = tex2D(_MainTex, i.uv);
fixed4 col2 = tex2D(_SubTex, i.uv);
 
//1. if语句写法
if (col.r < 0.1)
{
	col.r = col2.r*0.1;
}
 
//2. 对if优化之后的写法
fixed r_factor = max(0, sign(col.r - 0.1));
col.r = r_factor * col.r + (1 - r_factor) * col2.r*0.1; 
 
```

为了放大测试效果，我在测试的shader里面添加了30个if语句，对应的在替换if语句计算的公式也添加了30个。在空场景上渲染500个14.4k面的模型，并且对每个模型的material更改了颜色，保证500个模型渲染不batching，没有关照及其他因素影响。

>测试环境: macOS 10.12.6,  unity 5.6.3f1,  xcode 8.3.3， iOS 10.3.3,  iPhone7
>测试的结果主要针对iphone7的设备，其他设备测试结果可能不同，和GPU的型号直接关系。

###  结果验证

#### 测试场景

![](/images/shader_performance/test.jpg)

#### 测试结果

1.	没有if的测试情况

| 程序执行1min           | 程序执行5min            |
| ------------------ | ------------------- |
| FPS: 11            | FPS: 7              |
| CPU: 94ms ～105ms   | CPU: 140ms  ~ 144ms |
| GPU: 65ms  ~  71ms | GPU: 90ms  ~ 98ms   |

5min时刻的性能截图:

![](/images/shader_performance/performance_different1.jpg)

2. 存在if语句的测试情况


| 程序执行1min         | 程序执行5min            |
| ---------------- | ------------------- |
| FPS: 11          | FPS: 7              |
| CPU: 90ms ~ 98ms | CPU: 138ms ~160ms   |
| GPU: 66ms ~ 71ms | GPU: 90ms  ~  101ms |

5min时刻的性能截图:

![](/images/shader_performance/performance_different2.jpg)

总结: 通过测试发现两种写法的shader性能基本相近, 数值一直在波动，但是波动范围和数值频率差别比较微小, 没有哪种情况能够从数值上面看出绝对的优势。也就是相比使用if语句的代码，优化之后的代码并没有执行效率的明显提升，所以这里所谓的优化就不存在了。