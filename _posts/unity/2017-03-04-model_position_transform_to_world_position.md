---
title: Unity模型坐标转换到屏幕坐标
date: 2017-03-04 14:16:40
type: "tags"
comments: false
tags: unity
---

项目中遇到一个需求，需要添加一个引导，引导用户从UI层拖拽一个物品到三维的一个模型上面，引导只需在UI层表现即可。这里需要把模型的坐标转换到屏幕的坐标，这里记录一下。

首先要清楚转换的步骤:

> 模型坐标 -> 世界坐标 -> 屏幕坐标 -> Canvas坐标


需要注意的地方:

* 模型坐标到世界坐标这一步不需要我们自己转换，因为Transform的position属性就是模型的世界坐标

* 屏幕坐标到Canvas坐标这一步很容易被忽略，在设备的分辨率和我们**CanvasScaler**设置的
*Reference Resolution*设置的分辨率不是一致的话，那么*CanvasScaler*会进行缩放操作，所以我们需要考虑缩放之后的值

下面是转换过程:

```
Vector3 modelScreenPos = modelCamera.WorldToScreenPoint(model.transform.position);

Vector2 canvasRect = canvasScaler.GetComponent<RectTransform>().rect;

float factorX = canvasRect.width / Screen.width;
float factorY = canvasRect.height / Screen.height;

Vector3 modelCanvasPos = new Vector3(modelScreenPos.x * factorX, modelScreenPos.y * factorY, 0);

```