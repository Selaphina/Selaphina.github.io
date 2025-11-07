---
title: 8 透明效果
description: Unity Shader 入门精要 第八章
slug: hello-world
date: 2022-03-06 00:00:00+0000
image: cover.jpg
categories:
    - 技术笔记
tags:
    - Unity Shader
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 8 透明效果

unity中常用两种实现透明效果的方法：

1. 透明度测试（Alpha Test）

   > 一种极端霸道的机制。一个片元的透明度判定是否满足条件，如果不满足，则直接舍弃，否则就留下，即按普通的不透明物体处理。要么完全透明（看不到），要么完全不透明。所以透明度测试是不需要关闭深度写入的。

2. 透明度混合（Alpha Blending）

   > 这种方法可以得到真正的半透明效果。它会使用当前偏远的透明度作为混合因子，与已经存在颜色缓冲中的颜色值进行混合，得到新颜色。
   >
   > 注意：透明度混合只关闭了深度写入，没有关闭深度测试。（即深度缓冲对于透明度混合来说，是只读不写的）因为：当透明度混合渲染一个片元时，还是会比较它的深度值和当前深度缓冲中的深度值，如果它的深度距离摄像机比较远，而近处有一个不透明的物体，那么它的颜色就不会在进行混合操作。

## 8.1 渲染顺序

渲染顺序很重要。既然如此重要，那为什么需要关闭深度写入呢？

```
原因：

如果不关闭深度写入，在一个半透明表面背后的表面本来是可以透过它被我们看见的，但是，在深度测试环节的判定结果是：该半透明表面距离摄像机近。导致该表面背后的所有面都会被剔除。导致我们无法透过半透明表面看到后面的物体了。
```



Unity 提前定义5个渲染队列

| 名称        | 队列索引号 | 描述                                                         |
| ----------- | ---------- | ------------------------------------------------------------ |
| Background  | 1000       | 这个渲染队列会在任何其他队列之前被渲染，我们通常使用该队列来渲染那么些需要绘制在背景上的物体 |
| Geometry    | 2000       | 默认的渲染队列。大多数物体都用这个队列，不透明物体用这个队列。 |
| Alpha Test  | 2450       | 透明度测试的物体用的队列。Unity 5以后把他从geometry队列中分出，因为在所有不透明物体渲染之后再渲染它会更高效。 |
| Transparent | 3000       | 这个队列中的物体会在所有Geometry和Alphatest队列之后，按照**从后往前**的顺序渲染。所有使用了透明度混合（关闭深度写入）的物体都应该使用该队列。 |
| Overlay     | 4000       | 这个队列用于实现一些叠加效果。任何需要在最后渲染的物体都应该使用该队列。 |



## 8.7 双面渲染的透明效果

### 1. 透明度测试的双面渲染

透明度测试的双面渲染：

![image-20251025125910175](.assets/image-20251025125910175.png)

代码见：Chapter8-AlphaTest.shader

![image-20251025130050435](.assets/image-20251025130050435.png)

在透明度测试的基础上加上Cull Off即可。



### 2. 透明度混合的双面渲染

![image-20251025130804715](.assets/image-20251025130804715.png)

代码：Chapter8-AlphaBlendBothSided.shader

因为透明度混合没有深度写入，为了防止渲染顺序出错，用两个Pass，先渲染背面，后渲染前面，把控一下正确的混合顺序。

```
Shader "Unity Shaders Book/Chapter 8/Alpha Blend" {
	Properties {
		_Color ("Color Tint", Color) = (1, 1, 1, 1)
		_MainTex ("Main Tex", 2D) = "white" {}
		_AlphaScale ("Alpha Scale", Range(0, 1)) = 1
	}
	SubShader {
		Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
		
		Pass {
			Tags { "LightMode"="ForwardBase" }
			//先渲染背面（剔除前面）
			Cull Front
			// …………
			// 和之前一样的代码
		}

		Pass {
			Tags { "LightMode"="ForwardBase" }
			//后渲染前面（剔除后面）
			Cull Back
            // …………
			// 和之前一样的代码
	} 
	FallBack "Transparent/VertexLit"
}
```

