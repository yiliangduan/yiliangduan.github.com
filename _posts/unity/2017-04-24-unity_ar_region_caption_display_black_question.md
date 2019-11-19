---
title: Region_Capture截取的UV色彩变黑的问题
date: 2017-04-24 23:40:10
type: "tags"
comments: false
tags: unity
---

## 问题

使用Vuforia或者EasyAR的AR识别库我们经常用到*Region Capture*这个插件来得到我们摄像头拍摄的画面中的识别图，得到这张贴图我们就可以把它贴在Target模型上面来达到实时上色的效果。但是在*Region Capture*获取贴图的过程中(实时的)经常或出现我们移动摄像头偏离识别图的时候那么*Region Capture*得到贴图有部分是黑色的(这部分黑色的就是识别图没有被摄像头拍摄到的部分)。很显然这样的效果我们是无法接受的。

## 解决办法

我们读取*Region Capture*四个边角的pixel来判断是否是黑色。如果是黑色那么我们继续沿用上一次获取的贴图，知道下一次我们得到没有黑色的贴图再更新。下面是我的解决办法:

```cs

// CameraOutputTexture是Camera的Target Texture
// modelTextureCache是直接从CameraOuputTexture读区的贴图
// ModelTexture是最终使用的Texture

    private IEnumerator ClipDrawingTexture()
    {
        yield return new WaitForEndOfFrame();

        AllocModelTextureCacheIsNull();

        RenderTexture.active = CameraOutputTexture;

        modelTextureCache.ReadPixels(ClipRegion, 0, 0);

        RenderTexture.active = null;

        modelTextureCache.Apply();

        if (cameraFocusTargetCenter())
        {
            if (null != ModelTexture)
            {
                ModelTexture.LoadRawTextureData(modelTextureCache.GetRawTextureData());
                ModelTexture.Apply();
            }
        }
    }

    private bool cameraFocusTargetCenter()
    {
        int offsetStart = modelTextureCache.width / 10;

        int textureWidth = modelTextureCache.width;
        int textureHeight = modelTextureCache.height;

        Color leftTop = modelTextureCache.GetPixel(offsetStart, textureHeight - offsetStart);
        Color leftBottom = modelTextureCache.GetPixel(offsetStart, offsetStart);

        Color rightTop = modelTextureCache.GetPixel(textureWidth - offsetStart, textureHeight - offsetStart);
        Color rightBottom = modelTextureCache.GetPixel(textureWidth - offsetStart, offsetStart);

        return !isBlackColor(leftTop) && !isBlackColor(leftBottom) && !isBlackColor(rightTop) && !isBlackColor(rightBottom);
    }

    private bool isBlackColor(Color color)
    {
        return (color.r < 0.01f) && (color.g < 0.01f) && (color.b < 0.01f) && color.a > 0;
    }

```


