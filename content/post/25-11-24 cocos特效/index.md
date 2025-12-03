---
title: cocos复现特效（其二）
description: 粒子特效的笔记。
date: 2026-11-14 10:12:30+0000
image: CUC兑换点2.png
categories:
    - 工具指南
tags:
    - Note
weight: 2040       # You can add weight to some posts to override the default sorting 
---



结构

```
test

​	-3

​		-q1

​		-q2
```



## 3 中心光辉径向渐变

Material(在Renderer设置)

![image-20251202113217034](image-20251202113217034.png)

![glow01](glow01.png)

![image-20251202115042320](image-20251202115042320.png)

### 1）Start lifetime

Start lifetime：`0.2~0.5`

把生命周期改短，不会出现粒子下雨的长尾状。而是浮在表面上的一两个粒子。（因为很快就消失嘞）

![image-20251202111730112](image-20251202111730112.png)

2）

Start sizee：`2 ~ 3`

![image-20251202112735389](image-20251202112735389.png)

3）

Start Color

![image-20251202112903663](image-20251202112903663.png)

4）后续默认
![image-20251202112931978](image-20251202112931978.png)

### Emission默认

### Shape取消

### Color over Lifetime颜色

### ![image-20251202113735050](image-20251202113735050.png) 

### Size over Lifetime

![image-20251202114203926](image-20251202114203926.png)

## q1 圆环光圈1

![image-20251202115216375](image-20251202115216375.png)

![quan76](quan76.png)

![image-20251202115245723](image-20251202115245723.png)

### Start lifetime

0.2~0.5

### Start Speed

0

> 设置为0就不会向下掉落了

### Start Size

2

### Start Rotation

0-360之间随机

![image-20251202120541312](image-20251202120541312.png)

### Start Color

![image-20251202120649511](image-20251202120649511.png)

### Shape模块 取消

### Emission模块 默认

### Color over lifetime模块

![image-20251202121223303](image-20251202121223303.png)

### Size over lifetime模块

![image-20251202121335070](image-20251202121335070.png)

### Rotation over Lifetime模块

![image-20251202142140626](image-20251202142140626.png)

## q2 半圆弧描边2

同上，只是改个贴图，并保证start size略微大于q1的圆环。

![image-20251202165821735](image-20251202165821735.png)

![quan16](quan16.png)

> PS：内置的particle shader有自带的广告牌效果，根据摄像机视角自动旋转角度。涉及旋转有关的特效，最好先用纯色的贴图进行代替，观测好适合的初始角度再进行后续调整。

![image-20251202181722815](image-20251202181722815.png)

![image-20251202181737355](image-20251202181737355.png)
