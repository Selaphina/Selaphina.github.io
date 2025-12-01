---
title: 13 深度和法线纹理
description: Unity Shader 入门精要 第十三章
date: 2024-12-20 22:10:30+0000
image: cover1.png
categories:
    - 技术笔记
tags:
    - Unity Shader
weight: 2036       # You can add weight to some posts to override the default sorting (date descending)
---

在第12章中,我们学习的屏幕后处理效果都只是在屏幕颜色图像上进行各种操作来实现的。然而,很多时候我们不仅需要当前屏幕的颜色信息,还希望得到深度和法线信息。例如,在进行边缘检测时,直接利用颜色信息会使检测到的边缘信息受物体纹理和光照等外部因素的影响,得
到很多我们不需要的边缘点。

一种更好的方法是,我们可以在深度纹理和法线纹理上进行边缘检测,这些图像不会受纹理和光照的影响,而仅仅保存了当前渲刻染物体的模型信息,通过这样的方式检测出来的边缘更加可靠。

## 1.获取深度和法线纹理

### 1）背后的原理

深度纹理实际就是一张渲染纹理，只不过它里面存储的像素值不是颜色值，而是一个高精度的深度值。由于被存储在一张纹理中，深度纹理里的深度值范围是[0,1]，而且通常是非线性分布的。

那么，这些深度值是从哪里得到的呢?要回答这个问题，我们需要回顾在第4章学过的顶点变换的过程。总体来说，这些深度值来自于顶点变换后得到的归一化的设备坐标(NormalizedDevice Coordinates ，NDC)。回顾一下，一个模型要想最终被绘制在屏幕上，需要把它的顶点从模型空间变换到齐次裁剪坐标系下，这是通过在顶点着色器中乘以MVP变换矩阵得到的。在变换的最后一步，我们需要使用一个投影矩阵来变换顶点，当我们使用的是透视投影类型的摄像机时，这个投影矩阵就是非线性的，具体过程可回顾4.6.7小节。

图13.1显示了4.6.7小节中给出的Unity 中透视投影对顶点的变换过程。图13.1中最左侧的图显示了投影变换前，即观察空间下视锥体的结构及相应的顶点位置，中间的图显示了应用透视裁剪矩阵后的变换结果，即顶点着色器阶段输出的顶点变换结果，最右侧的图则是底层硬件进行了透视除法后得到的归一化的设备坐标。

需要注意的是，这里的投影过程是建立在Unity 对坐标系的假定上的，也就是说，我们针对的是观察空间为右手坐标系，使用列矩阵在矩阵右侧进行相乘，且变换到 NDC后z分量范围将在[-1,1]之间的情况。而在类似DirectX 这样的图形接口中，变换后z分量范围将在[0,1]之间。如果需要在其他图形接口下实现本章的类似效果，需要对一些计算参数做出相应变化。关于变换时使用的矩阵运算，读者可以参考4.6.7小节。

![unity的坐标系](image-20251201173138827.png)

![image-20251201173202845](image-20251201173202845.png)

图13.2显示了在使用正交摄像机时投影变换的过程。同样，变换后会得到一个范围为[-1,1]的立方体。正交投影使用的变换矩阵是线性的。

![image-20251201173226154](image-20251201173226154.png)

在得到 NDC后,深度纹理中的像素值就可以很方便地计算得到了,这些深度值就对应了NDC中顶点坐标的z分量的值。由于NDC中z分量的范围在[-1,1]，为了让这些值能够存储在一张图像中，我们需要使用下面的公式对其进行映射:
$$
d = 0.5 \cdot z_{ndc} + 0.5
$$
其中，d对应了深度纹理中的像素值，z-ndc-对应了NDC坐标中的z分量的值。那么 Unity 是怎么得到这样一张深度纹理的呢?

在 Unity 中，深度纹理可以直接来自于真正的深度缓存，也可以是由一个单独的 Pass 渲染而得，这取决于使用的渲染路径和硬件。通常来讲,当使用延迟渲染路径(包括遗留的延迟渲染路径)时，深度纹理理所当然可以访问到，因为延迟渲染会把这些信息渲染到 G-bufer 中。

而当无法直接获取深度缓存时，深度和法线纹理是通过一个单独的 Pass 渲染而得的。具体实现是，Unity会使用着色器替换 ( Shader Replacement ) 技术选择那些渲染类型 (即 SubShader的RenderType 标签 ) 为Opaque 的物体，判断它们使用的渲染队列是否小于等于2500 ( 内置的Background、Geometry和AlphaTest 渲染队列均在此范围内 )，如果满足条件，就把它渲染到深度和法线纹理中。因此，要想让物体能够出现在深度和法线纹理中，就必须在 Shader 中设置正确的 RenderType 标签
在 Unity 中，我们可以选择让一个摄像机生成一张深度纹理或是一张深度+法线纹理。当选择前者，即只需要一张单独的深度纹理时，Unity会直接获取深度缓存或是按之前讲到的着色器替换技术，选取需要的不透明物体，并使用它投射阴影时使用的Pass(即LightMode被设置为ShadowCaster 的Pass，详见9.4节)来得到深度纹理。如果 Shader 中不包含这样一个Pass，那么这个物体就不会出现在深度纹理中(当然，它也不能向其他物体投射阴影)。深度纹理的精度通常是24位或16位，这取决于使用的深度缓存的精度。如果选择生成一张深度+法线纹理，Unity会创建一张和屏幕分辨率相同、精度为32位(每个通道为8位)的纹理，其中观察空间下的法线信息会被编码进纹理的R和G通道，而深度信息会被编码进B和A通道。法线信息的获取在延迟渲染中是可以非常容易就得到的，Unity只需要合并深度和法线缓存即可。而在前向渲染中，默认情况下是不会创建法线缓存的，因此 Unity底层使用了一个单独的 Pass 把整个场景再次渲染一遍来完成。这个Pass被包含在Unity内置的一个UnityShader中，我们可以在内置的builtin shaders-xxx/DefaultResources/Camera-DepthNormalTexture.shader 文件中找到这个用于渲染深度和法线信息的 Pass。

### 2）获取深度纹理

在 Unity 中，获取深度纹理是非常简单的，我们只需要告诉 Unity:“嘿，把深度纹理给我!”然后再在 Shader 中直接访问特定的纹理属性即可。这个与Unity 沟通的过程是通过在脚本中设置摄像机的 depthTextureMode 来完成的，例如我们可以通过下面的代码来获取深度纹理:

```
camera,depthTextureMode = DepthTextureMode.Depth;
```

一旦设置好了上面的摄像机模式后，我们就可以在Shader中通过声明CameraDepthTexture变量来访问它。这个过程非常简单，但我们需要知道这两行代码的背后，Unity为我们做了许多工作(见13.1.1节)。

同理，如果想要获取深度+法线纹理，我们只需要在代码中这样设置:

```
camera.depthTextureMode = DepthTextureMode.DepthNormals;
```

然后在 Shader中通过声明 CameraDepth变量来访问它。TaSexture我们还可以组合这些模式，让一个摄像机同时产生一张深度和深度+法线纹理:

```
camera.depthTextureMode=DepthTextureMode.Depth;
camera.depthTextureMode=DepthTextureMode.DepthNormals;
```

在 Unity5中，我们还可以在摄像机的Camera 组件上看到当前摄像机是否需要渲染深度或深度+法线纹理。当在 Shader 中访问到深度纹理CameraDepthTexture后，我们就可以使用当前像素的纹理坐标对它进行采样。绝大多数情况下，我们直接使用tex2D函数采样即可，但在某些平台(例如PS3和PSP2)上，我们需要一些特殊处理。Uniy为我们提供了一个统一的宏SAMPLE DEPTH TEXTURE，用来处理这些由于平台差异造成的问题。而我们只需要在 Shader中使用 SAMPLE DEPTH TEXTURE 宏对深度纹理进行采样，例如:

```
float d=SAMPLE DEPTH TEXTURE( CameraDepthTexture,i.uv);
```

其中，i.uv是一个foat2类型的变量，对应了当前像素的纹理坐标。类似的宏还有SAMPLE DEPTH TEXTURE PROJ和 SAMPLE DEPTH TEXTURE LOD。 SAMPLE DEPTHTEXTURE PROJ宏同样接受两个参数--深度纹理和一个 foat3 或 foat4 类型的纹理坐标,它的内部使用了 tex2Dproj这样的函数进行投影纹理采样，纹理坐标的前两个分量首先会除以最后一个分量，再进行纹理采样。如果提供了第四个分量，还会进行一次比较，通常用于阴影的实现中。SAMPLE DEPTH TEXTUREPROJ的第二个参数通常是由顶点着色器输出插值而得的屏幕坐标，例如:

```
float d=SAMPLE DEPTH TEXTURE PROJ( CameraDepthTexture,UNITY PROJ COORD(i.scrPos));
```

其中，i.scrPos 是在顶点着色器中通过调用 ComputeScreenPos(o.pos)得到的屏幕坐标。上述这
些宏的定义，读者可以在 Unity内置的HSLSupport.cginc 文件中找到。当通过纹理采样得到深度值后，这些深度值往往是非线性的，这种非线性来自于透视投影使用的裁剪矩阵。然而，在我们的计算过程中通常是需要线性的深度值，也就是说，我们需要把投影后的深度值变换到线性空间下，例如视角空间下的深度值。那么，我们应该如何进行这个转换呢?实际上，我们只需要倒推顶点变换的过程即可。下面我们以透视投影为例，推导如何由深度纹理中的深度信息计算得到视角空间下的深度值。
由4.6.7节可知，当我们使用透视投影的裁剪矩阵P"对视角空间下的一个顶点进行变换后，裁剪空间下顶点的z和w分量为:
$$
z_{\text{clip}} = -z_{\text{visw}} \frac{ \text{Far} + \text{Near} }{ \text{Far} - \text{Near} } - \frac{2 \cdot \text{Near} \cdot \text{Far} }{ \text{Far} - \text{Near} }
$$

$$
w_{\text{clip}} = -z_{\text{visw}}
$$



其中，Far 和Near 分别是远近裁剪平面的距离。然后，我们通过齐次除法就可以得到 NDC下的z分量:
$$
z_{\text{ndc}} = \frac{z_{\text{clip}}}{w_{\text{clip}}} = \frac{ \text{Far} + \text{Near} }{ \text{Far} - \text{Near} } + \frac{2 \cdot \text{Near} \cdot \text{Far} }{ (\text{Far} - \text{Near}) \cdot z_{\text{visw}} }
$$
在 13.1.1节中我们知道，深度纹理中的深度值是通过下面的公式由NDC计算而得的:

$$
d = 0.5 \cdot z_{ndc} + 0.5
$$
由上面的这些式子，我们可以推导出用d表示而得的z的表达式:
$$
z'_{\text{visw}} = \frac{ 1 }{ \frac{ \text{Far} - \text{Near}}{\text{Near} \cdot \text{Far} } d - \frac{1}{\text{Near}}}
$$
由于在 Unity 使用的视角空间中，摄像机正向对应的z值均为负值，因此为了得到深度值的正数表示，我们需要对上面的结果取反，最后得到的结果如下:
$$
z_{01} = \frac{ 1 }{ \frac{ \text{Near} - \text{Far}}{\text{Near}} d + \frac{Far}{\text{Near}}}
$$
幸运的是，Unity 提供了两个辅助函数来为我们进行上述的计算过程一LinearEyeDepth和Linear01Depth。

LinearEyeDepth负责把深度纹理的采样结果转换到视角空间下的深度值，也就是我们上面得到的Z'visw。

而Linear01Depth 则会返回一个范围在[0,1]的线性深度值，也就是我们上面得到的 Z01。这两个函数内部使用了内置的 ZBuferParams 变量来得到远近裁剪平面的距离。如果我们需要获取深度+法线纹理，可以直接使用tex2D数对CameraDepthNormalsTexture进行采样，得到里面存储的深度和法线信息。Unity 提供了辅助函数来为我们对这个采样结果进行解码，从而得到深度值和法线方向。这个函数是DecodeDepthNormal，它在UnityCG.cginc 里被定义：

```
inline void DecodeDepthNormal( float4 enc,out float depth, out float3 normal)
{
    depth =DecodeFloatRG(enc.zw);
    normal=DecodeViewNormalStereo(enc);
}
```

DecodeDepthNormal的第一个参数是对深度+法线纹理的采样结果，这个采样结果是 Unity 对深度和法线信息编码后的结果,它的xy分量存储的是视角空间下的法线信息,而深度信息被编码进了 zw 分量。通过调用 DecodeDepthNormal 函数对采样结果解码后，我们就可以得到解码后的深度值和法线。这个深度值是范围在[0.11的线性深度值(这与单独的深度纹理中存储的深度值不同)，而得到的法线则是视角空间下的法线方向。同样，我们也可以通过调用DecodeFloatRG 和DecodeViewNormalStereo 来解码深度+法线纹理中的深度和法线信息。

至此，我们已经学会了如何在 Unity 里获取及使用深度和法线纹理。下面，我们会学习如何使用它们实现各种屏幕特效。

### 3）查看深度和法线纹理

很多时候，我们希望可以查看生成的深度和法线纹理，以便对Shader 进行调试。Unity5提供了一个方便的方法来查看摄像机生成的深度和法线纹理，这个方法就是利用帧调试器(FrameDebugger)。图13.3 显示了使用帧调试器査看到的深度纹理和深度+法线纹理。

![image-20251202005807184](image-20251202005807184.png)

使用帧调试器查看到的深度纹理是非线性空间的深度值,而深度+法线纹理都是由Unity 编码后的结果。有时，显示出线性空间下的深度信息或解码后的法线方向会更加有用。此时，我们可以自行在片元着色器中输出转换或解码后的深度和法线值，如图13.4所示。输出代码非常简单，我们可以使用类似下面的代码来输出线性深度值:

```
float depth = SAMPLE DEPTH TEXTURE( CameraDepthTexture,i.uv);
float linearDepth = Linear01Depth(depth);
return fixed4(linearDepth,linearDepth,linearDepth，1.0);
```

或是输出法线方向:

```
fixed3 normal = DecodeViewNormalStereo(tex2D(_CameraDepthNormalsTexture, i.uv).xy);
return fixed4(normal*0.5+0.5, 1.0);
```

在查看深度纹理时，读者得到的画面有可能几乎是全黑或全白的。这时候读者可以把摄像机的远裁剪平面的距离(Unity默认为1000)调小，使视锥体的范围刚好覆盖场景的所在区域。这是因为，由于投影变换时需要覆盖从近裁剪平面到远裁剪平面的所有深度区域，当远裁剪平面的距离过大时，会导致离摄像机较近的距离被映射到非常小的深度值，如果场景是一个封闭的区域(如图13.4所示)，那么这就会导致画面看起来几乎是全黑的。相反，如果场景是一个开放区域，且物体离摄像机的距离较远，就会导致画面几乎是全白的。

![image-20251202010103493](image-20251202010103493.png)

## 2.再谈运动模糊

在12.6节中，我们学习了如何通过混合多张屏幕图像来模拟运动模糊的效果。但是，另一种应用更加广泛的技术则是使用速度映射图。速度映射图中存储了每个像素的速度，然后使用这个速度来决定模糊的方向和大小。速度缓冲的生成有多种方法，一种方法是把场景中所有物体的速度渲染到一张纹理中。但这种方法的缺点在于需要修改场景中所有物体的 Shader 代码，使其添加计算速度的代码并输出到一个渲染纹理中。

《GPU Gems3》在第27章(http://http.developer,nvidia.com/GPUGems3/gpugems3 ch27.html)中介绍了一种生成**速度映射图**的方法。这种方法利用深度纹理在片元着色器中为每个像素计算其在世界空间下的位置，这是通过使用当前的视角*投影矩阵的逆矩阵对NDC下的顶点坐标进行变换得到的。当得到世界空间中的顶点坐标后，我们使用前一帧的视角*投影矩阵对其进行变换，得到该位置在前一帧中的 NDC坐标。然后，我们计算前一帧和当前帧的位置差，生成该像素的速度。这种方法的优点是可以在一个屏幕后处理步骤中完成整个效果的模拟，但缺点是需要在片元着色器中进行两次矩阵乘法的操作，对性能有所影响。为了使用深度纹理模拟运动模糊，我们需要进行如下准备工作。

为了使用深度纹理模拟运动模糊，我们需要进行如下准备工作。(1)新建一个场景。在本书资源中，该场景名为Scene132。在 Unity5.2中，默认情况下场景将包含一个摄像机和一个平行光,并且使用了内置的天空盒子。在Window→ Lighting→Skybox中去掉场景中的天空盒子。
(2)我们需要搭建一个测试运动模糊的场景。在本书资源的实现中，我们构建了一个包含3面墙的房间，并放置了4个立方体，它们都使用了我们在95节中创建的标准材质。同时，我们把本书资源中的 Translating.cs脚本拖曳给摄像机，让其在场景中不断运动。(3)新建一个脚本。在本书资源中，该脚本名为MotionBlurWithDepthTexture.cs。把该脚本拖曳到摄像机上。
(4)新建一个Unity Shader。在本书资源中，该Shader 名为Chapter13-MotionBlurWithDepthTexture。

### MotionBlurWithDepthTexture.cs

我们首先来编写 MotionBlurWithDepthTexture.cs脚本。打开该脚本，并进行如下修改。

(1)首先，继承12.1节中创建的基类

```
public class MotionBlurWithDepthTexture :PostEffectsBase {
```

(2)声明该效果需要的Shader，并据此创建相应的材质:

```
public Shader motionBlurShader;
private Material motionBlurMaterial = null;
```

```
	public Material material {  
		get {
			motionBlurMaterial = CheckShaderAndCreateMaterial(motionBlurShader, motionBlurMaterial);
			return motionBlurMaterial;
		}  
	}
```

3)定义运动模糊时模糊图像使用的大小:

```
[Range(0.0f, 1.0f)]
	public float blurSize = 0.5f;
```


(4)由于本节需要得到摄像机的视角和投影矩阵，我们需要定义一个Camera 类型的变量,以获取该脚本所在的摄像机组件:

```
	private Camera myCamera;
	public Camera camera {
		get {
			if (myCamera == null) {
				myCamera = GetComponent<Camera>();
			}
			return myCamera;
		}
	}
```

(5)我们还需要定义一个变量来保存上一帧摄像机的视角*投影矩阵:

```
private Matrix4x4 previousViewProjectionMatrix;
```

(6)由于本例需要获取摄像机的深度纹理，我们在脚本的 OnEnable 函数中设置摄像机的状态:

```
void onEnable(){
camera.depthTextureMode= DepthTextureMode .Depth;
}
```

(7)最后，我们实现了OnRenderlmage 函数:

```c++
void OnRenderImage (RenderTexture src, RenderTexture dest) {
		if (material != null) {
			material.SetFloat("_BlurSize", blurSize);

			material.SetMatrix("_PreviousViewProjectionMatrix", previousViewProjectionMatrix);
			Matrix4x4 currentViewProjectionMatrix = camera.projectionMatrix * camera.worldToCameraMatrix;
			Matrix4x4 currentViewProjectionInverseMatrix = currentViewProjectionMatrix.inverse;
			material.SetMatrix("_CurrentViewProjectionInverseMatrix", currentViewProjectionInverseMatrix);
			previousViewProjectionMatrix = currentViewProjectionMatrix;

			Graphics.Blit (src, dest, material);
		} else {
			Graphics.Blit(src, dest);
		}
	}
```

上面的 OnRenderImage 函数很简单，我们首先需要计算和传递运动模糊使用的各个属性。才例需要使用两个变换矩阵——`前一帧的视角\*投影矩阵`以及`当前帧的视角\*投影矩阵`的逆矩阵。因此，我们通过调用 camera.worldToCameraMatrix和 camera.projectionMatrix来分别得到当前摄像机的视角矩阵和投影矩阵。对它们相乘后取逆，得到`当前帧的视角*投影矩阵的逆矩阵`，并传递给材质。然后，我们把取逆前的结果存储在 previousViewProjectionMatrix 变量中，以便在下一帧时传递给材质的 PreviousViewProjectionMatrix属性。

### Chapter13-MotionBlurWithDepthTexture

下面，我们来实现 Shader 的部分。打开 Chapter13-MotionBlurWithDepthTexture，进行如下修改。

(1)我们首先需要声明本例使用的各个属性:

```
Properties{
	_MainTex("Base(RGB)"2D)="white"{}
	_BlurSize ("Blur Size"Float)=1.0
}

```

MainTex 对应了输入的渲染纹理，BlurSize 是模糊图像时使用的参数。我们注意到，虽然在脚本里设置了材质的 PreviousViewProjectionMatrix和CurrentViewProjectionInverseMatrix属性，但并没有在Properties块中声明它们。这是因为Unity 没有提供矩阵类型的属性，但我们仍然可以在CG代码块中定义这些矩阵，并从脚本中设置它们。


(2)在本节中,我们使用 CGINCLUDE来组织代码。我们在 SubShader块中利用CGINCLUDE和 ENDCG 语义来定义一系列代码:

```
SubShader
{
	CGINCLUDE
	……
	ENDCG
	……
    }
```

(3)声明代码中需要使用的各个变量:

```
sampler2DMainTex;
half4 MainTex TexelSize;
sampler2DCameraDepthTexture;
float4x4 CurrentViewProjectionInverseMatrix;
float4x4PreviousViewProjectionMatrix;
half Blursize;
```

在上面的代码中，除了定义在 Properties 声明的 MainTex和 BlurSize 属性，我们还声明了其他三个变量。CameraDepthTexture 是Unity 传递给我们的深度纹理，而 CurrentViewProjectionInverseMatrix和PreviousViewProiectionMatrix是由脚本传递而来的矩阵。除此之外，我们还声明了MainTex TexelSize 变量，它对应了主纹理的纹素大小，我们需要使用该变量来对深度纹理的采样坐标进行平台差异化处理(详见5.6.1节)。

(4)顶点着色器的代码和之前使用多次的代码基本一致，只是增加了专门用于对深度纹理采样的纹理坐标变量:

```
struct v2f {
	float4 pos : SV_POSITION;
	half2 uv : TEXCOORD0;
	half2 uv_depth : TEXCOORD1;
};

v2f vert(appdata_img v) {
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	
	o.uv = v.texcoord;
	o.uv_depth = v.texcoord;
	
	#if UNITY_UV_STARTS_AT_TOP
	if (_MainTex_TexelSize.y < 0)
		o.uv_depth.y = 1 - o.uv_depth.y;
	#endif
			 
	return o;
}
```

由于在本例中，我们需要同时处理多张渲染纹理，因此在DirectX这样的平台上，我们需要处理平台差异导致的图像翻转问题。在上面的代码中，我们对深度纹理的采样坐标进行了平台差异化处理，以便在类似 Directx的平台上，在开启了抗锯齿的情况下仍然可以得到正确的结果,

(5)片元着色器是算法的重点所在:

```
		fixed4 frag(v2f i) : SV_Target {
			// Get the depth buffer value at this pixel.
			float d = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth);
			// H is the viewport position at this pixel in the range -1 to 1.
			float4 H = float4(i.uv.x * 2 - 1, i.uv.y * 2 - 1, d * 2 - 1, 1);
			// Transform by the view-projection inverse.
			float4 D = mul(_CurrentViewProjectionInverseMatrix, H);
			// Divide by w to get the world position. 
			float4 worldPos = D / D.w;
			
			// Current viewport position 
			float4 currentPos = H;
			// Use the world position, and transform by the previous view-projection matrix.  
			float4 previousPos = mul(_PreviousViewProjectionMatrix, worldPos);
			// Convert to nonhomogeneous points [-1,1] by dividing by w.
			previousPos /= previousPos.w;
			
			// Use this frame's position and last frame's to compute the pixel velocity.
			float2 velocity = (currentPos.xy - previousPos.xy)/2.0f;
			
			float2 uv = i.uv;
			float4 c = tex2D(_MainTex, uv);
			uv += velocity * _BlurSize;
			for (int it = 1; it < 3; it++, uv += velocity * _BlurSize) {
				float4 currentColor = tex2D(_MainTex, uv);
				c += currentColor;
			}
			c /= 3;
			
			return fixed4(c.rgb, 1.0);
		}
```

我们首先需要利用深度纹理和当前帧的视角*投影矩阵的逆矩阵来求得该像素在世界空间下的坐标。过程开始于对深度纹理的采样，我们便用内置的SAMPLE DEPTH TEXTURE宏和纹理坐标对深度纹理进行采样，得到了深度值 d。由 13.1.2节可知,d是由 NDC下的坐标映射而来的。我们想要构建像素的 NDC 坐标 ，就需要把这个深度值重新映射回 NDC。这个映射很简单，只需要使用原映射的反函数即可，即d*2-1。同样，NDC的x分量可以由像素的纹理坐标映射而来(NDC下的xyz分量范围均为[-1,1])。当得到NDC下的坐标H后，我们就可以使用`当前帧的视角*投影矩阵的逆矩阵`对其进行变换，并把结果值除以它的w分量来得到世界空间下的坐标表示 worldPos.

一旦得到了世界空间下的坐标，我们就可以使用`前一帧的视角*投影矩阵`对它进行变换，得到前一帧在 NDC下的坐标 previousPos。然后，我们计算前一帧和当前帧在屏幕空间下的位置差,得到该像素的速度 velocity。
当得到该像素的速度后，我们就可以使用该速度值对它的邻域像素进行采样，相加后取平均值得到一个模糊的效果。采样时我们还使用了_BlurSize 来控制采样距离。

(6)然后，我们定义了运动模糊所需的 Pass:

```
Pass {      
	ZTest Always Cull Off ZWrite Off
	    	
	CGPROGRAM  
	
	#pragma vertex vert  
	#pragma fragment frag  
	  
	ENDCG  
}
```

(7)最后，我们关闭了shader的Fallback:

```
Fallback Off
```

完成后返回编辑器，并把 Chapter13-MotionBlurWithDepthTexture 拖曳到摄像机的 MotionBlurWithDepthTexture.cs脚本中的 motionBlurShader 参数中。当然，我们可以在 MotionBlurWithDepthTexture.cs的脚本面板中将 motionBlurShader 参数的默认值设置为 Chapter13-MotionBlurWithDepthTexture，这样就不需要以后使用时每次都手动拖曳了。

本节实现的运动模糊适用于场景静止、摄像机快速运动的情况，这是因为我们在计算时只考虑了摄像机的运动。因此，如果读者把本节中的代码应用到一个物体快速运动而摄像机静止的场景，会发现不会产生任何运动模糊效果。如果我们想要对快速移动的物体产生运动模糊的效果，就需要生成更加精确的速度映射图。读者可以在 Unity 自带的ImageEfect包中找到更多的运动模糊的实现方法。
本节选择在片元着色器中使用逆矩阵来重建每个像素在世界空间下的位置。但是，这种做法往往会影响性能，在13.3节中，我们会介绍一种更快速的由深度纹理重建世界坐标的方法。

## 3.全局雾效

雾效(Fog)是游戏里经常使用的一种效果。Unity内置的雾效可以产生基于距离的线性或指数雾效。然而，要想在自己编写的顶点/片元着色器中实现这些雾效，我们需要在Shader 中添加#pragma multi compile fog指令，同时还需要使用相关的内置宏，例如 UNITY FOG COORDS、UNITY TRANSFERFOG和UNITY APPLY FOG等。这种方法的缺点在于，我们不仅需要为场景中所有物体添加相关的渲染代码，而且能够实现的效果也非常有限。当我们需要对雾效进行些个性化操作时，例如使用基于高度的雾效等，仅仅使用Unity内置的雾效就变得不再可行。

在本节中，我们将会学习一种基于屏幕后处理的全局雾效的实现。使用这种方法，我们不需要更改场景内渲染的物体所使用的Shader代码，而仅仅依靠一次屏幕后处理的步骤即可这种方法的自由性很高，我们可以方便地模拟各种雾效，例如均匀的雾效、基于距离的线性指数雾效、基于高度的雾效等。在学习完本节后，我们可以得到类似图13.5中的效果。

![image-20251202012321171](image-20251202012321171.png)

基于屏幕后处理的全局雾效的关键是，根据深度纹理来重建每个像素在世界空间下的位置。尽管在 13.2节中，我们在模拟运动模糊时已经实现了这个要求，即构建出当前像素的NDC坐标，再通过当前摄像机的视角*投影矩阵的逆矩阵来得到世界空间下的像素坐标，但是，这样的实现需要在片元着色器中进行矩阵乘法的操作，而这通常会影响游戏性能。

在本节中，我们将会学习一个快速从深度纹理中重建世界坐标的方法。这种方法首先对图像空间下的视锥体射线(从摄像机出发，指向图像上的某点的射线)进行插值，这条射线存储了该像素在世界空间下到摄像机的方向信息。然后，我们把该射线和线性化后的视角空间下的深度值相乘，再加上摄像机的世界位置，就可以得到该像素在世界空间下的位置。

当我们得到世界坐标后，就可以轻松地使用各个公式来模拟全局雾效了。