---
title: 17 表面着色器
description: Unity Shader 入门精要 第十七章
date: 2024-09-05 22:12:30+0000
image: cover1.png
categories:
    - 技术笔记
tags:
    - Unity Shader
weight: 2036       # You can add weight to some posts to override the default sorting (date descending)
---

# 第17章 Unity的表面着色器探秘

在2009年的时候（当时Unity的版本是2.x），Unity的渲染工程师Aras（就是经常活跃在论坛和各种会议上的，大名鼎鼎的Aras Pranckevicius）连续发表了3篇名为《Shaders must die》的博客。在这些博客里，Aras认为，把渲染流程分为顶点和像素的抽象层面是错误的，是一种不易理解的抽象。目前，这种在顶点/几何/片元着色器上的操作是对硬件友好的一种方式，但不符合我们人类的思考方式。相反，他认为，应该划分成表面着色器、光照模型和光照着色器这样的层面。其中，表面着色器定义了模型表面的反射率、法线和高光等，光照模型则选择是使用兰伯特还是Blinn-Phong等模型。而光照着色器负责计算光照衰减、阴影等。这样，绝大部分时间我们只需要和表面着色器打交道，例如，混合纹理和颜色等。光照模型可以是提前定义好的，我们只需要选择哪种预定义的光照模型即可。而光照着色器一旦由系统实现后，更不会被轻易改动，从而大大减轻了Shader编写者的工作量。有了这样的想法，Aras在随后的文章中开始尝试把表面着色器整合到Unity中。最终，在2010年的Unity3中，Surface Shader被加入到Unity的大家族中了。

虽然Unity换了一个新的“马甲”，但表面着色器（Surface Shader）实际上就是在顶点/片元着色器之上又添加了一层抽象。按Aras的话来解释就是，顶点/几何/片元着色器是硬件能“理解”的渲染方式，而开发者应该使用一种更容易理解的方式。很多时候，使用表面着色器，我们只需要告诉Shader：“嘿，使用这些纹理去填充颜色，使用这个法线纹理去填充表面法线，使用兰伯特光照模型，其他的就不要来烦我了！”我们不需要考虑是使用前向渲染路径还是延迟渲染路径，场景中有多少光源，它们的类型是什么，怎样处理这些光源，每个Pass需要处理多少个光源等问题（正是因为有这些事情，人们总会抱怨写一个Shader是多么的麻烦……………）。这时，Unity说：“不要急，我来干！”

那么，表面着色器到底长什么样呢？它们又是如何工作的呢？这正是本章要学习的内容。

## 表面着色器的一个例子

在学习原理之前，我们首先来看一下一个表面着色器长什么样子。为此，我们需要做如下的准备工作。

1. 在Unity中新创建一个场景。在本书资源中，该场景名为Scene_17_1。在Unity5.2中，默认情况下场景将包含一个摄像机和一个平行光，并且使用了内置的天空盒子。在Window→Lighting→Skybox中去掉场景中的天空盒子。
2. 新创建一个材质。在本书资源中，该材质名为BumpedSpecularMat。
3. 新创建一个Unity Shader。在本书资源中，该Unity Shader名为Chapter17-BumpedDiffuse，把新的Unity Shader赋给第2步中创建的材质。
4. 在场景中创建一个胶囊体（capsule），并把第2步中的材质赋给该胶囊体。
5. 保存场景。

我们将使用表面着色器来实现一个使用了法线纹理的漫反射效果。这可以参考Unity内置的“Legacy Shaders/BumpedDiffuse”的代码实现（可以在官方网站的内置Shader包中找到）。打开Chapter17-BumpedDiffuse，删除原有的代码，把下面的代码粘贴进去：

```hlsl
Shader "Unity Shaders Book/Chapter 17/Bumped Diffuse" {
    Properties {
        _Color ("Main Color", Color) = (1,1,1,1)
        _MainTex ("Base (RGB)", 2D) = "white" {}
        _BumpMap ("Normalmap", 2D) = "bump" {}
    }
    SubShader {
        Tags { "RenderType"="Opaque" }
        LOD 300
        CGPROGRAM
        #pragma surface surf Lambert
        #pragma target 3.0
        sampler2D _MainTex;
        sampler2D _BumpMap;
        fixed4 _Color;
        struct Input {
            float2 uv_MainTex;
            float2 uv_BumpMap;
        };
        void surf (Input IN, inout SurfaceOutput o) {
            fixed4 tex = tex2D(_MainTex, IN.uv_MainTex);
            o.Albedo = tex.rgb * _Color.rgb;
            o.Alpha = tex.a * _Color.a;
            o.Normal = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap));
        }
        ENDCG
    }
    FallBack "Legacy Shaders/Diffuse"
}
```

保存程序后，返回Unity中查看。在BumpedDiffuseMat的面板上，我们把本书资源中的Assets/Textures/Chapter17/MudDiffuse.tif和Assets/Textures/Chapter17/MudNormal.tif分别拖拽到MainTex和_BumpMap属性上，就可以得到类似图17.1中左图的结果。我们还可以向场景中添加一些点光源和聚光灯，并改变它们的颜色，就可以得到类似图17.1中右图的结果。注意，在这个过程中，我们不需要对代码做任何改动。

从上面的例子可以看出，相比之前所学的顶点/片元着色器技术，表面着色器的代码量很少（只需要三十多行），图17.1表面着色器的例子左边：在一个平行光下的效果。如果我们使用顶点/片元着色器来实现上述的功能，大概需要150多行代码（参考本书资源中的“UnityShadersBook/Common/BumpedDiffuse”）！而且，我们可以非常轻松地实现常见的光照模型，甚至不需要和任何光照变量打交道，Unity就帮我们处理好了每个光源的光照结果。

读者可以在Unity官方手册的表面着色器的例子一文（http://docs.unity3d.com/Manual/SL-SurfaceShaderExamples.html）中找到更多的示例程序。下面，我们将具体学习表面着色器的特点和工作原理。

和顶点/片元着色器需要包含到一个特定的Pass块中不同，表面着色器的CG代码是直接而且也必须写在SubShader块中，Unity会在背后为我们生成多个Pass。当然，可以在SubShader一开始处使用Tags来设置该表面着色器使用的标签。在Chapter17-BumpedDiffuse中，我们还使用LOD命令设置了该表面着色器的LOD值（详见16.8.1节）。然后，我们使用CGPROGRAM和ENDCG定义了表面着色器的具体代码。

一个表面着色器中最重要的部分是两个结构体以及它的编译指令。其中，两个结构体是表面着色器中不同函数之间信息传递的桥梁，而编译指令是我们和Unity沟通的重要手段。

## 编译指令

我们首先来看一下表面着色器的编译指令。编译指令是我们和Unity沟通的重要方式，通过它可以告诉Unity：“嘿，用这个表面函数设置表面属性，用这个光照模型模拟光照，我不要阴影和环境光，不要雾效！”只需要一行代码，我们就可以完成这么多事情！

编译指令最重要的作用是指明该表面着色器使用的表面函数和光照函数，并设置一些可选参数。表面着色器的CG块中的第一句代码往往就是它的编译指令。编译指令的一般格式如下：

```hlsl
#pragma surface surfaceFunction lightModel [optionalparams]
```

其中，`#pragma surface`用于指明该编译指令是用于定义表面着色器的，在它的后面需要指明使用的表面函数（surfaceFunction）和光照模型（lightModel），同时，还可以使用一些可选参数来控制表面着色器的一些行为。

### 表面函数

我们之前说过，表面着色器的优点在于抽象出了“表面”这一概念。与之前遇到的顶点/片元抽象层不同，一个对象的表面属性定义了它的反射率、光滑度、透明度等值。而编译指令中的surfaceFunction就用于定义这些表面属性。surfaceFunction通常就是名为surf的函数（函数名可以是任意的），它的函数格式是固定的：

```hlsl
void surf (Input IN, inout SurfaceOutput o)
void surf (Input IN, inout SurfaceOutputStandard o)
void surf (Input IN, inout SurfaceOutputStandardSpecular o)
```

其中，后两个是Unity5中由于引入了基于物理的渲染而新添加的两种结构体。SurfaceOutput、SurfaceOutputStandard和SurfaceOutputStandardSpecular都是Unity内置的结构体，它们需要配合不同的光照模型使用，我们会在下一节进行更详细地解释。

在表面函数中，会使用输入结构体Input IN来设置各种表面属性，并把这些属性存储在输出结构体SurfaceOutput、SurfaceOutputStandard或SurfaceOutputStandardSpecular中，再传递给光照函数计算光照结果。读者可以在Unity手册中的表面着色器的例子一文（http://docs.unity3d.com/Manual/SL-SurfaceShaderExamples.html）中找到更多的示例表面函数。

### 光照函数

除了表面函数，我们还需要指定另一个非常重要的函数——光照函数。光照函数会使用表面函数中设置的各种表面属性，来应用某些光照模型，进而模拟物体表面的光照效果。Unity内置了基于物理的光照模型函数Standard和StandardSpecular（在UnityPBSLighting.cginc文件中被定义），以及简单的非基于物理的光照模型函数Lambert和BlinnPhong（在Lighting.cginc文件中被定义）。例如，在Chapter17-BumpedDiffuse中，我们就指定了使用Lambert光照函数。

当然，我们也可以定义自己的光照函数。例如，可以使用下面的函数来定义用于前向渲染中的光照函数：

```hlsl
// 用于不依赖视角的光照模型，例如漫反射
half4 Lighting<Name> (SurfaceOutput s, half3 lightDir, half atten)
// 用于依赖视角的光照模型，例如高光反射
half4 Lighting<Name> (SurfaceOutput s, half3 lightDir, half3 viewDir, half atten);
```

读者可以在Unity手册的表面着色器中的自定义光照模型一文（http://docs.unity3d.com/Manual/SL-SurfaceShaderLighting.html）中找到更全面的自定义光照模型的介绍。而一些例子可以参见手册中的表面着色器的光照例子一文（http://docs.unity3d.com/Manual/SL-SurfaceShaderLightingExamples.html)，这篇文档展示了如何使用表面着色器来自定义常见的漫反射、高光反射、基于光照纹理等常用的光照模型。

### 其他可选参数

在编译指令的最后，我们还可以设置一些可选参数（optionalparams）。这些可选参数包含了很多非常有用的指令类型，例如，开启/设置透明度混合/透明度测试，指明自定义的顶点和颜色修改函数，控制生成的代码等。下面，我们选取了一些比较重要和常用的参数进行更深入地说明。读者可以在Unity官方手册的编写表面着色器一文（http://docs.unity3d.com/Manual/SL-SurfaceShaders.html）中找到更加详细的参数和设置说明。

- **自定义的修改函数**。除了表面函数和光照模型外，表面着色器还可以支持其他两种自定义的函数：顶点修改函数（`vertex:VertexFunction`）和最后的颜色修改函数（`finalcolor:ColorFunction`）。顶点修改函数允许我们自定义一些顶点属性，例如，把顶点颜色传递给表面函数，或是修改顶点位置，实现某些顶点动画等。最后的颜色修改函数则可以在颜色绘制到屏幕前，最后一次修改颜色值，例如实现自定义的雾效等。
- **阴影**。我们可以通过一些指令来控制和阴影相关的代码。例如，`addshadow`参数会为表面着色器生成一个阴影投射的Pass。通常情况下，Unity可以直接在FallBack中找到通用的光照模式为ShadowCaster的Pass，从而将物体正确地渲染到深度和阴影纹理中（详见9.4节）。但对于一些进行了顶点动画、透明度测试的物体，我们就需要对阴影的投射进行特殊处理，来为它们产生正确的阴影，正如我们在13.3节中看到的一样。`fullforwardshadows`参数则可以在前向渲染路径中支持所有光源类型的阴影。默认情况下，Unity只支持最重要的平行光的阴影效果。如果我们需要让点光源或聚光灯在前向渲染中也可以有阴影，就可以添加这个参数。相反地，如果我们不想对使用这个Shader的物体进行任何阴影计算，就可以使用`noshadow`参数来禁用阴影。
- **透明度混合和透明度测试**。我们可以通过`alpha`和`alphatest`指令来控制透明度混合和透明度测试。例如，`alphatest:VariableName`指令会使用名为VariableName的变量来剔除不满足条件的片元。此时，我们可能还需要使用上面提到的`addshadow`参数来生成正确的阴影投射的Pass。
- **光照**。一些指令可以控制光照对物体的影响，例如，`noambient`参数会告诉Unity不要应用任何环境光照或光照探针（light probe）。`novertexlights`参数告诉Unity不要应用任何逐顶点光照。`noforwardadd`会去掉所有前向渲染中的额外的Pass。也就是说，这个Shader只会支持一个逐像素的平行光，而其他的光源会按照逐顶点或SH的方法来计算光照影响。这个参数通常会用于移动平台版本的表面着色器中。还有一些用于控制光照烘焙、雾效模拟的参数，如`nolightmap`、`nofog`等。
- **控制代码的生成**。一些指令还可以控制由表面着色器自动生成的代码，默认情况下，Unity会为一个表面着色器生成相应的前向渲染路径、延迟渲染路径使用的Pass，这会导致生成的Shader文件比较大。如果我们确定该表面着色器只会在某些渲染路径中使用，就可以`exclude_path:deferred`、`exclude_path:forward`和`exclude_path:prepass`来告诉Unity不需要为某些渲染路径生成代码。

从上述可以看出，表面着色器支持的编译指令参数很多，为我们编写表面着色器提供了很大的方便。之前在顶点/片元着色器中需要耗费大量代码来完成的工作，在表面着色器中可能只需要一个参数就可以了。当然，相比于顶点/片元着色器，表面着色器也有它自身的限制，我们会在17.6节中对比它们的优缺点。

## 两个结构体

在上一节我们已经讲过，表面着色器支持最多自定义4种关键的函数：表面函数（用于设置各种表面性质，如反射率、法线等），光照函数（定义表面使用的光照模型），顶点修改函数（修改或传递顶点属性），最后的颜色修改函数（对最后的颜色进行修改）。那么，这些函数之间的信息传递是怎么实现的呢？例如，我们想把顶点颜色传递给表面函数，添加到表面反射率的计算中，要怎么做呢？这就是两个结构体的工作。

一个表面着色器需要使用两个结构体：表面函数的输入结构体Input，以及存储了表面属性的结构体SurfaceOutput（Unity5新引入了另外两个同种的结构体SurfaceOutputStandard和SurfaceOutputStandardSpecular）。

### 数据来源：Input结构体

Input结构体包含了许多表面属性的数据来源，因此，它会作为表面函数的输入结构体（如果自定义了顶点修改函数，它还会是顶点修改函数的输出结构体）。Input支持很多内置的变量名，通过这些变量名，我们告诉Unity需要使用的数据信息。例如，在Chapter17-BumpedDiffuse中，Input结构体中包含了主纹理和法线纹理的采样坐标`uv_MainTex`和`uv_BumpMap`。这些采样坐标必须以“uv”为前缀（实际上也可用“uv2”为前缀，表明使用次纹理坐标集合），后面紧跟纹理名称。以主纹理`_MainTex`为例，如果需要使用它的采样坐标，就需要在Input结构体中声明`float2 uv_MainTex`来对应它的采样坐标。表17.1列出了Input结构体中内置的其他变量。

**表17.1**

| 变量               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| float3 viewDir     | 包含了视角方向，可用于计算边缘光照等                         |
| float4 COLOR       | 使用COLOR语义定义的float4变量，包含了插值后的逐顶点颜色      |
| float4 screenPos   | 包含了屏幕空间的坐标，可以用于反射或屏幕特效                 |
| float3 worldPos    | 包含了世界空间下的位置                                       |
| float3 worldRefl   | 包含了世界空间下的反射方向。前提是没有修改表面法线o.Normal。如果修改了表面法线o.Normal，需要使用该变量告诉Unity要基于修改后的法线计算世界空间下的反射方向。在表面函数中，我们需要使用`WorldReflectionVector(IN, o.Normal)`来得到世界空间下的反射方向 |
| float3 worldNormal | 包含了世界空间的法线方向。前提是没有修改表面法线o.Normal。如果修改了表面法线o.Normal，需要使用该变量告诉Unity要基于修改后的法线计算世界空间下的法线方向。在表面函数中，我们需要使用`WorldNormalVector(IN, o.Normal)`来得到世界空间下的法线方向 |

需要注意的是，我们并不需要自己计算上述的各个变量，而只需要在Input结构体中按上述名称严格声明这些变量即可，Unity会在背后为我们准备好这些数据，而我们只需要在表面函数中直接使用它们即可。一个例外情况是，我们自定义了顶点修改函数，并需要向表面函数中传递一些自定义的数据。例如，为了自定义雾效，我们可能需要在顶点修改函数中根据顶点在视角空间下的位置信息计算雾效混合系数，这样我们就可以在Input结构体中定义一个名为`half fog`的变量，把计算结果存储在该变量后进行输出。

### 表面属性：SurfaceOutput结构体

有了Input结构体来提供所需要的数据后，我们就可以据此计算各种表面属性。因此，另一个结构体就是用于存储这些表面属性的结构体，即SurfaceOutput、SurfaceOutputStandard和SurfaceOutputStandardSpecular，它会作为表面函数的输出，随后会作为光照函数的输入来进行各种光照计算。相比于Input结构体的自由性，这个结构体里面的变量是提前就声明好的，不可以增加也不会减少（如果没有对某些变量赋值，就会使用默认值）。SurfaceOutput的声明可以在Lighting.cginc文件中找到：

```hlsl
struct SurfaceOutput {
    fixed3 Albedo;
    fixed3 Normal;
    fixed3 Emission;
    half Specular;
    fixed Gloss;
    fixed Alpha;
};
```

而SurfaceOutputStandard和SurfaceOutputStandardSpecular的声明可以在UnityPBSLighting.cginc中找到：

```hlsl
struct SurfaceOutputStandard {
    fixed3 Albedo;              // base (diffuse or specular) color
    fixed3 Normal;              // tangent space normal, if written
    half3 Emission;
    half Metallic;              // 0=non-metal, 1=metal
    half Smoothness;            // 0=rough, 1=smooth
    half Occlusion;             // occlusion (default 1)
    fixed Alpha;                // alpha for transparencies
};

struct SurfaceOutputStandardSpecular {
    fixed3 Albedo;              // diffuse color
    fixed3 Specular;            // specular color
    fixed3 Normal;              // tangent space normal, if written
    half3 Emission;
    half Smoothness;            // 0=rough, 1=smooth
    half Occlusion;             // occlusion (default 1)
    fixed Alpha;                // alpha for transparencies
};
```

在一个表面着色器中，只需要选择上述三者中的其一即可，这取决于我们选择使用的光照模型。Unity内置的光照模型有两种，一种是Unity5之前的、简单的、非基于物理的光照模型，包括了Lambert和BlinnPhong；另一种是Unity5添加的、基于物理的光照模型，包括Standard和StandardSpecular，这种模型会更加符合物理规律，但计算也会复杂很多。如果使用了非基于物理的光照模型，我们通常会使用SurfaceOutput结构体，而如果使用了基于物理的光照模型Standard或StandardSpecular，我们会分别使用SurfaceOutputStandard或SurfaceOutputStandardSpecular结构体。其中，SurfaceOutputStandard结构体用于默认的金属工作流程（Metallic Workflow），对应了Standard光照函数；而SurfaceOutputStandardSpecular结构体用于高光工作流程（Specular Workflow），对应了StandardSpecular光照函数。更多关于基于物理的渲染内容，我们会在第18章中讲到。

在本节，我们着重介绍一下SurfaceOutput结构体中的变量和含义。在表面函数中，我们需要根据Input结构体传递的各个变量计算表面属性。在SurfaceOutput结构体，这些表面属性包括了：

- fixed3 Albedo：对光源的反射率。通常由纹理采样和颜色属性的乘积计算而得。
- fixed3 Normal：表面法线方向。
- fixed3 Emission：自发光。Unity通常会在片元着色器最后输出前（如果在最后的顶点函数被调用前，如果定义了的话），使用类似下面的语句进行简单的颜色叠加：`c.rgb += o.Emission;`
- half Specular：高光反射中的指数部分的系数，影响高光反射的计算。例如，如果使用了内置的BlinnPhong光照函数，它会使用如下语句计算高光反射的强度：`float spec = pow (nh, s.Specular*128.0) * s.Gloss;`
- fixed Gloss：高光反射中的强度系数。和上面的Specular类似，计算公式见上面的代码。一般在包含了高光反射的光照模型里使用。
- fixed Alpha：透明通道。如果开启了透明度的话，会使用该值进行颜色混合。

尽管表面着色器极大地减少了我们的工作量，但它带来的一个问题是，我们经常不知道为什么会得到这样的渲染结果。如果你不是一个“好奇宝宝”的话，你可以高高兴兴地使用表面着色器来方便地实现一些不错的渲染效果。但是，一些好奇的初学者往往会提出这样的问题：“为什么我的场景里没有灯光，但物体不是全黑的呢？为什么我把光源的颜色调成黑色，物体还是有一些渲染颜色呢？”这些问题都源于表面着色器对我们隐藏了实现细节。而想要更加得心应手地使用表面着色器，我们就需要学习它的工作流水线，并了解Unity是如何为一个表面着色器生成对应的顶点/片元着色器的（时刻记着，表面着色器本质上就是包含了很多Pass的顶点/片元着色器）。

## Unity背后做了什么

在前面的内容中，我们已经了解到如何利用编译指令、自定义函数（表面函数、光照函数，以及可选的顶点修改函数和最后的颜色修改函数）和两个结构体来实现一个表面着色器。我们一直强调，Unity实际会在背后为表面着色器生成真正的顶点/片元着色器。那么，表面着色器中的各个函数、编译指令和结构体与顶点/片元着色器之间有什么关系呢？这正是本节要学习的内容。

我们之前说过，Unity在背后会根据表面着色器生成一个包含了很多Pass的顶点/片元着色器。这些Pass有些是为了针对不同的渲染路径，例如，默认情况下Unity会为前向渲染路径生成LightMode为ForwardBase和ForwardAdd的Pass，为Unity5之前的延迟渲染路径生成LightMode为PrePassBase和PrePassFinal的Pass，为Unity5之后的延迟渲染路径生成LightMode为Deferred的Pass。还有一些Pass是用于产生额外的信息，例如，为了给光照映射和动态全局光照提取表面信息，Unity会生成一个LightMode为Meta的Pass。有些表面着色器由于修改了顶点位置，因此，我们可以利用`addshadow`编译指令为它生成相应的LightMode为ShadowCaster的阴影投射Pass。这些Pass的生成都是基于我们在表面着色器中的编译指令和自定义的函数，这是有规律可循的。

Unity提供了一个功能，让那些“好奇宝宝”可以对表面着色器自动生成的代码一探究竟：在每个编译完成的表面着色器的面板上，都有一个“Show generated code”的按钮，如图17.2所示，我们只需要单击一下它就可以看到Unity为这个表面着色器生成的所有顶点/片元着色器。

通过查看这些代码，我们就可以了解到Unity到底是如何根据表面着色器生成各个Pass的。以Unity生成的LightMode为ForwardBase的Pass（用于前向渲染）为例，它的渲染计算流水线如图17.3所示。从图17.3中我们可以看出，4个允许自定义的函数在流水线中的位置。

Unity对该Pass的自动生成过程大致如下。

1. 直接将表面着色器中CGPROGRAM和ENDCG之间的代码复制过来，这些代码包括了我们对Input结构体、表面函数、光照函数（如果自定义了的话）等变量和函数的定义。这些函数和变量会在之后的处理过程中被当成正常的结构体和函数进行调用。
2. Unity会分析上述代码，并据此生成顶点着色器的输出——v2f_surf结构体，用于在顶点着色器和片元着色器之间进行数据传递。Unity会分析我们在自定义函数中所使用的变量，例如，纹理坐标、视角方向、反射方向等。如果需要，它就会在v2f_surf中生成相应的变量。而且，即使有时我们在Input中定义了某些变量（如某些纹理坐标），但Unity在分析后续代码时发现我们并没有使用这些变量，那么这些变量实际上是不会在v2f_surf中生成的。这也就是说，Unity做了一些优化。v2f_surf中还包含了一些其他需要的变量，例如阴影纹理坐标、光照纹理坐标、逐顶点光照等。
3. 接着，生成顶点着色器。
   - 如果我们自定义了顶点修改函数，Unity会首先调用顶点修改函数来修改顶点数据，或填充自定义的Input结构体中的变量。然后，Unity会分析顶点修改函数中修改的数据，在需要时通过Input结构体将修改结果存储到v2f_surf相应的变量中。
   - 计算v2f_surf中其他生成的变量值。这主要包括了顶点位置、纹理坐标、法线方向、逐顶点光照、光照纹理的采样坐标等。当然，我们可以通过编译指令来控制某些变量是否需要计算。
   - 最后，将v2f_surf传递给接下来的片元着色器。
4. 生成片元着色器。
   - 使用v2f_surf中的对应变量填充Input结构体，例如，纹理坐标、视角方向等。
   - 调用我们自定义的表面函数填充SurfaceOutput结构体。
   - 调用光照函数得到初始的颜色值。如果使用的是内置的Lambert或BlinnPhong光照函数，Unity还会计算动态全局光照，并添加到光照模型的计算中。
   - 进行其他的颜色叠加。例如，如果没有使用光照烘焙，还会添加逐顶点光照的影响。
   - 最后，如果自定义了最后的颜色修改函数，Unity就会调用它进行最后的颜色修改。

其他Pass的生成过程和上面类似，在此不再赘述。

## 表面着色器实例分析

为了帮助读者更加深入地理解表面着色器背后的原理，我们在本节以一个表面着色器为例，分析Unity为它生成的代码。

读者可以在本书资源中的Scene_17_4中找到相应的测试场景。它实现的效果是对模型进行膨胀，如图17.4所示。这种效果的实现非常简单，就是在顶点修改函数中沿着顶点法线方向扩张顶点位置。为了分析表面着色器中4个允许自定义函数（顶点修改函数、表面函数、光照函数和最后的颜色修改函数）的原理，在本例中我们对这4个函数全部采用了自定义的实现。读者可以在Chapter17-NormalExtrusion文件中找到该表面着色器，它的代码如下：

```hlsl
Shader "Unity Shaders Book/Chapter 17/Normal Extrusion" {
    Properties {
        _ColorTint ("Color Tint", Color) = (1,1,1,1)
        _MainTex ("Base (RGB)", 2D) = "white" {}
        _BumpMap ("Normalmap", 2D) = "bump" {}
        _Amount ("Extrusion Amount", Range(-0.5, 0.5)) = 0.1
    }
    SubShader {
        Tags { "RenderType"="Opaque" }
        LOD 300
        CGPROGRAM
        // surf - which surface function.
        // CustomLambert - which lighting model to use.
        // vertex:myvert - use custom vertex modification function.
        // finalcolor:mycolor - use custom final color modification function
        // addshadow - generate a shadow caster pass. Because we modify the vertex position,
        // the shader needs special shadows handling.
        // exclude_path:deferred / exclude_path:prepass - do not generate passes for
        // deferred / legacy deferred rendering path.
        // nometa - do not generate a "meta" pass (that's used by lightmapping & dynamic
        // global illumination to extract surface information).
        #pragma surface surf CustomLambert vertex:myvert finalcolor:mycolor addshadow exclude_path:deferred exclude_path:prepass nometa
        #pragma target 3.0
        fixed4 _ColorTint;
        sampler2D _MainTex;
        sampler2D _BumpMap;
        half _Amount;
        struct Input {
            float2 uv_MainTex;
            float2 uv_BumpMap;
        };
        void myvert (inout appdata_full v) {
            v.vertex.xyz += v.normal * _Amount;
        }
        void surf (Input IN, inout SurfaceOutput o) {
            fixed4 tex = tex2D(_MainTex, IN.uv_MainTex);
            o.Albedo = tex.rgb;
            o.Alpha = tex.a;
            o.Normal = UnpackNormal(tex2D(_BumpMap, IN.uv_BumpMap));
        }
        half4 LightingCustomLambert (SurfaceOutput s, half3 lightDir, half atten) {
            half NdotL = dot(s.Normal, lightDir);
            half4 c;
            c.rgb = s.Albedo * _LightColor0.rgb * (NdotL * atten);
            c.a = s.Alpha;
            return c;
        }
        void mycolor (Input IN, SurfaceOutput o, inout fixed4 color) {
            color *= _ColorTint;
        }
        ENDCG
    }
    FallBack "Legacy Shaders/Diffuse"
}
```

在顶点修改函数中，我们使用顶点法线对顶点位置进行膨胀；表面函数使用主纹理设置了表面属性中的反射率，并使用法线纹理设置了表面法线方向；光照函数实现了简单的Lambert漫反射光照模型；在最后的颜色修改函数中，我们简单地使用了颜色参数对输出颜色进行调整。注意，除了4个函数外，我们在`#pragma surface`的编译指令一行中还指定了一些额外的参数。由于我们修改了顶点位置，因此，要对其他物体产生正确的阴影效果并不能直接依赖FallBack中找到的阴影投射Pass，`addshadow`参数可以告诉Unity要生成一个该表面着色器对应的阴影投射Pass。默认情况下，Unity会为所有支持的渲染路径生成相应的Pass，为了缩小自动生成的代码量，我们使用`exclude_path:deferred`和`exclude_path:prepass`来告诉Unity不要为延迟渲染路径生成相应的Pass。最后，我们使用`nometa`参数取消对提取元数据的Pass的生成。

当在该表面着色器的导入面板中单击“Show generated code”按钮后，我们就可以看到Unity生成的顶点/片元着色器了。由于代码比较多，为了节省篇幅我们不再把全部代码粘贴到这里。因此，在往下阅读之前，请读者先打开生成的代码文件，以便明白我们接下来的分析。

在这个将近600行代码的文件中，Unity一共为该表面着色器生成了3个Pass，它们的LightMode分别是ForwardBase、ForwardAdd和ShadowCaster，分别对应了前向渲染路径中的处理逐像素平行光的Pass、处理其他逐像素光的Pass、处理阴影投射的Pass。这些Pass的原理可以回顾9.1.1节和9.4节中的相关内容。读者可以在这些代码中看到大量的#ifdef和#if语句，这些语句可以判断一些渲染条件，例如，是否使用了动态光照纹理、是否使用了逐顶点光照、是否使用了屏幕空间的阴影等，Unity会根据这些条件来进行不同的光照计算，这正是表面着色器的魅力之一——把这些烦人的光照计算交给Unity来做！

需要注意的是，不同的Unity版本可能生成的代码有少许不同。在本书中，我们以Unity5.2.1中的结果为准。下面，我们来分析Unity生成的ForwardBase Pass。

1. Unity首先指明了一些编译指令：
    ```hlsl
    Pass {
        Name "FORWARD"
        Tags { "LightMode"="ForwardBase" }
        CGPROGRAM
        // compile directives
        #pragma vertex vert_surf
        #pragma fragment frag_surf
        #pragma target 3.0
        #pragma multi_compile_fwdbase
        #include "HLSLSupport.cginc"
        #include "UnityShaderVariables.cginc"
    ```
    顶点着色器`vert_surf`和片元着色器`frag_surf`都是自动生成的。
2. 之后出现的是一些自动生成的注释，这些注释表明了Unity的分析过程和它的分析结果。
3. 随后，Unity定义了一些宏来辅助计算，例如：
    ```hlsl
    #define INTERNAL_DATA half3 internalSurfaceTtoW0; half3 internalSurfaceTtoW1; half3 internalSurfaceTtoW2;
    #define WorldReflectionVector(data,normal) reflect(data.worldRefl, half3(dot(data.internalSurfaceTtoW0, normal), dot(data.internalSurfaceTtoW1, normal), dot(data.internalSurfaceTtoW2, normal)))
    #define WorldNormalVector(data,normal) fixed3(dot(data.internalSurfaceTtoW0, normal), dot(data.internalSurfaceTtoW1, normal), dot(data.internalSurfaceTtoW2, normal))
    ```
    实际上，在本例中上述宏并没有被用到。这些宏是为了在修改了表面法线的情况下，辅助计算得到世界空间下的反射方向和法线方向。
4. 接着，Unity把我们在表面着色器中编写的CG代码复制过来，作为Pass的一部分，以便后续调用。
5. 然后，Unity定义了顶点着色器到片元着色器的插值结构体（即顶点着色器的输出结构体）v2f_surf。在定义之前，Unity使用#ifdef语句来判断是否使用了光照纹理，并为不同的情况生成不同的结构体。
6. 随后，Unity定义了真正的顶点着色器。顶点着色器首先会调用我们自定义的顶点修改函数来修改一些顶点属性，然后计算v2f_surf中各个变量的值，例如，计算经过MVP矩阵变换后的顶点坐标、使用`TRANSFORM_TEX`内置宏计算两个纹理的采样坐标、计算从切线空间到世界空间的变换矩阵、判断是否使用了光照映射和动态光照映射、判断是否开启了逐顶点光照等。最后，计算阴影坐标并传递给片元着色器。
7. 在Pass的最后，Unity定义了真正的片元着色器。Unity首先利用插值后的结构体v2f_surf来初始化Input结构体中的变量，然后声明了一个SurfaceOutput结构体的变量，并对其中的表面属性进行了初始化，再调用了表面函数`surf`来填充这些表面属性。之后，Unity进行了真正的光照计算，包括计算光照衰减和世界空间下的法线方向、判断是否关闭了光照映射、是否需要使用自定义的光照模型计算光照结果、是否开启了动态光照映射等。最后，Unity调用自定义的颜色修改函数，对输出颜色c进行最后的修改，并使用内置宏`UNITY_OPAQUE_ALPHA`来重置片元的透明通道。

至此，ForwardBase Pass就结束了。接下来的ForwardAdd Pass和上面的ForwardBase Pass基本类似，只是代码更加简单了，Unity去掉了对逐顶点光照和各种判断是否使用了光照映射的代码，因为这些额外的Pass不需要考虑这些。

最后一个重要的Pass是ShadowCaster Pass。相比于之前的两个Pass，它的代码比较简单短小，它的生成原理很简单，就是通过调用自定义的顶点修改函数来保证计算阴影时使用的是和之前一致的顶点坐标。正如我们在11.3.3节和15.1节中看到的一样，这个自定义的阴影投射的Pass同样使用了内置的`V2F_SHADOW_CASTER`、`TRANSFER_SHADOW_CASTER_NORMALOFFSET`和`SHADOW_CASTER_FRAGMENT`来计算阴影投射。

## Surface Shader的缺点

从上面的内容中我们可以看出，表面着色器给我们带来了很大的便利。那么，我们之前为什么还要花那么久的时间学习顶点/片元着色器？直接与表面着色器就好了嘛。

正如我们一直强调的那样，表面着色器只是Unity在顶点/片元着色器上面提供的一种封装，是一种更高层的抽象。但任何在表面着色器中完成的事情，我们都可以在顶点/片元着色器中重现。但不幸的是，这句话反过来并不成立。

这世上任何事情都是有代价的，如果我们想要得到便利，就需要以牺牲自由度为代价。表面着色器虽然可以快速实现各种光照效果，但我们失去了对各种优化和各种特效实现的控制。因此使用表面着色器往往会对性能造成一定的影响，而内置的Shader，例如Diffuse、BumpedSpecular等都是使用表面着色器编写的。尽管Unity提供了移动平台的相应版本，例如Mobile/Diffuse和Mobile/BumpedSpecular等，但这些版本的Shader往往只是去掉了额外的逐像素Pass、不计算全局光照和其他一些光照计算上的优化。但要想进行更多深层的优化，表面着色器就不能满足我们的需求了。

除了性能比较差以外，表面着色器还无法完成一些自定义的渲染效果，例如10.2.2节中透明玻璃的效果。表面着色器的这些缺点让很多人更愿意使用自由的顶点/片元着色器来实现各种效果，尽管处理光照时这可能难度更大些。

因此，我们给出一些建议供读者参考：

- 如果你需要和各种光源打交道，尤其是想要使用Unity中的全局光照的话，你可能更喜欢使用表面着色器，但要时刻小心它的性能；
- 如果你需要处理的光源数目非常少，例如只有一个平行光，那么使用顶点/片元着色器是一个更好的选择；
- 最重要的是，如果你有很多自定义的渲染效果，那么请选择顶点/片元着色器。

