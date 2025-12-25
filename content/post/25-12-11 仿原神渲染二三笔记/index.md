---
title: 仿原神渲染
description: 记录
date: 2025-12-11 20:32:30+0000
image: CUC兑换点2.png
categories:
    - 技术笔记
weight: 2019       # You can add weight to some posts to override the default sorting (date descending)

---

## 1 前期准备工作

1.新建URP 3D项目

2.点击资产面板的URP资产，此时右侧面板高亮，可以Add Render Objects

![设置](image-20251211220748882.png)

![此时可以添加render feature](image-20251211220808619.png)

添加Render Objects

![image-20251211220951165](image-20251211220951165.png)

添加3个pass。

![image-20251211220916403](image-20251211220916403.png)

mask改为everything。

![image-20251211220842140](image-20251211220842140.png)

> TODO: 有仙人曾言： // 我自己试下来，在角色身上 LowQuality 比 Medium 和 High 好
>  // Medium 和 High 采样数多，过渡的区间大，在角色身上更容易出现 Perspective aliasing
>
> 等到时候自己验证一下在说 。

FaceLightmap

![FaceLightmap](image-20251211221505443.png)

Body_Diffuse

![image-20251211221635125](image-20251211221635125.png)

Body_lightmap

![image-20251211221733472](image-20251211221733472.png)

法线图

![image-20251211222108578](image-20251211222108578.png)

shadow ramp

![image-20251211222233237](image-20251211222233237.png)

Face_Diffuse

![image-20251211222320885](image-20251211222320885.png)

Hair_Diffuse

![头发漫反射](image-20251211222358382.png)

hair_lightmap

![image-20251211222446504](image-20251211222446504.png)

MetalMap

![image-20251211222623823](image-20251211222623823.png)

贴图设置完毕。

值得注意的是 杜林的翅膀贴图是和头发放在一起，观察一下贴图纹理的对应。

![image-20251221163249828](image-20251221163249828.png)



## 2 代码结构

### 1.整体结构

![整体的结构](image-20251220215926595.png)

一开始不太习惯HLSL的代码结构，和CG有一定的区别，注意。

```
Shader "EXAM1/EXAM_Shader"
{
    Property
    {
        ……
    }

    Subshader
    {
        Pass
        {……}

        Pass
        {……}
    }
}

```

### 2.Subshader结构

![](image-20251220220214291.png)

```
 SubShader
 {
     HLSLINCLUDE
     #include "../../ShaderLibrary/…….hlsl"
     ENDHLSL
     
     Tags
     {……}
     
     Pass
     {……}
     
     Pass
     {……}
 }
```

### Subshader结构

**1.HLSLINCLUDE & ENDHLSL**

```
 SubShader
 {
     HLSLINCLUDE
     //导入库
     #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"  //默认库
     #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl"  //光照库        
     ENDHLSL

     Pass {  
         };
 }
```

**HLSLINCLUDE** 和 **ENDHLSL** 是Unity ShaderLab中的指令，它们之间的代码会被**自动包含到该着色器的所有Pass中**。

相当于一个“公共头文件”，写在这里的代码（如变量声明、函数、宏定义）对所有Pass都可见。

**为什么这样设计？**

- **减少重复**：如果不使用HLSLINCLUDE，每个Pass都需要单独声明这些变量和贴图，代码会冗长且难以维护。
- **保持一致性**：所有Pass使用同一套参数和贴图，确保渲染结果统一。

**2 HLSLPROGRAM**

![](image-20251221023828695.png)

在Unity URP Shader中，每个Pass通常需要将顶点和片元着色器代码包裹在`HLSLPROGRAM`和`ENDHLSL`块中。

### Pass结构

Pass块中，需要在`Tags`之后添加`HLSLPROGRAM`：

```
Pass {
    Tags { "LightMode" = "head" }
    
    HLSLPROGRAM
    #pragma vertex vert
    #pragma fragment frag
    
    // 您的a2v和v2f结构体定义
    // 您的vert和frag函数定义
    
    ENDHLSL
}
```

**3 在 `frag` 函数内部定义函数（HLSL 不允许）**

在 `frag` 里写大量函数定义，例如：

```
half4 frag (v2f i) : SV_TARGET {

    float3 shadow_ramp(...) { ... }
    float3 Spec(...) { ... }
    float3 Metal(...) { ... }
    float3 edgeLight(...) { ... }
    float3 light(...) { ... }
    float3 Body(...) { ... }
    float3 Face(...) { ... }

    ...
}

```

这是**绝对错误**的

- **HLSL / ShaderLab 不支持函数嵌套定义**
- 函数**必须定义在全局作用域**
- DX11 编译器会直接报错

**正确写法：**

```
float3 shadow_ramp(float4 lightmap, float NdotL) { ... }
float3 Spec(...) { ... }
float3 Metal(...) { ... }
float3 edgeLight(...) { ... }
float3 light(...) { ... }
float3 Body(...) { ... }
float3 Face(...) { ... }

half4 frag(v2f i) : SV_TARGET
{
    ...
}

```

## 3 正式开始

1）a2v定义从应用阶段（CPU）到顶点着色器的顶点数据。

```
struct a2v{
	float4 vertex : POSITION;//顶点坐标
	float2 texcoord0 : TEXCOORD0;//纹理坐标UV0
	float3 normal : NORMAL;//顶点法线
	float4 tangent : TANGENT;//顶点切线
};
```

为什么`normal`是`float3`而`tangent`是`float4`？

> 1. **法线（normal）**：`float3` 法线是一个三维向量（x, y, z），表示顶点表面的朝向。 它只需要三个分量就能完整描述方向，因此使用`float3`足够。
> 2. **切线（tangent）**：`float4` 切线通常用于法线贴图（normal mapping）计算，需要与法线和副切线（binormal/bitangent）构成切线空间（tangent space）。 前三个分量（x, y, z）表示切线的方向向量。 **第四个分量（w）是一个符号值**，通常为+1或-1，用于指示副切线的方向。
>
> **关键区别**：
>
> 1. **法线的方向是绝对的**： 法线向量(x, y, z)本身就完整定义了方向 例如，法线(0, 1, 0)明确表示"向上"，没有歧义
> 2. **切线需要确定副切线方向**： 给定法线和切线后，副切线可以通过叉积计算：`副切线 = 叉积(法线, 切线)` **但叉积有两种可能方向**：左手系或右手系 `tangent.w`（通常±1）就是用来指定这个方向的： 如果`tangent.w = 1`：副切线 = 叉积(法线, 切线) 如果`tangent.w = -1`：副切线 = 叉积(切线, 法线)
> 3. **几何意义**： 法线、切线、副切线构成**切线空间基** 法线是"主方向"，切线和副切线是"辅助方向" 切线的w分量确保整个坐标系的一致性（避免镜像翻转）



## 附录：



一些常见问题：

报错：INVALID UTF8 STRING

### Visual Studio 设置默认编码格式为 UTF-8 或 GB2312-80

[Visual Studio 设置默认编码格式为 UTF-8 或 GB2312-80 与文件没有高级保存选项怎么显示_visual studio 不使用简体中文gb2312编码加载文件-CSDN博客](https://blog.csdn.net/qq_41868108/article/details/105750175)