---
title: 13 深度纹理
description: Unity Shader 入门精要 第十三章
date: 2024-12-20 22:10:30+0000
image: cover1.png
categories:
    - 技术笔记
tags:
    - Unity Shader
weight: 2036       # You can add weight to some posts to override the default sorting (date descending)
---

在本章中,我们将会学习如何使用噪声来模拟各种看似“神奇”的特效。

在 15.1节中，我们将使用一张噪声纹理来模拟火焰的消融效果。

15.2节则把噪声应用在模拟水面的波动上，从而产生波光粼粼的视觉效果。

在15.3 节中，我们会回顾13.3节中实现的全局雾效，并向其中添加噪声来模拟不均匀的飘渺雾效。



## 1.消融效果

消融(dissolve)效果常见于游戏中的角色死亡、地图烧毁等效果。在这些效果中，消融往往从不同的区域开始，并向看似随机的方向扩张，最后整个物体都将消失不见。

消融原理非常简单。概括来说就是<u>噪声纹理+透明度测试。</u>

我们使用对噪声纹理采样的结果和某个控制消融程度的阈值比较，如果小于阈值，就使用 clip 函数把它对应的像素裁剪掉，这些部分就对应了图中被“烧毁”的区域。而镂空区域边缘的烧焦效果则是将两种颜色混合，再用 pow函数处理后，与原纹理颜色混合后的结果。

(1)首先，声明消融效果需要的各个属性:

```
	Properties {
		_BurnAmount ("Burn Amount", Range(0.0, 1.0)) = 0.0
		_LineWidth("Burn Line Width", Range(0.0, 0.2)) = 0.1
		_MainTex ("Base (RGB)", 2D) = "white" {}
		_BumpMap ("Normal Map", 2D) = "bump" {}
		_BurnFirstColor("Burn First Color", Color) = (1, 0, 0, 1)
		_BurnSecondColor("Burn Second Color", Color) = (1, 0, 0, 1)
		_BurnMap("Burn Map", 2D) = "white"{}
	}
```

BurnAmount属性用于控制消融程度，当值为0时，物体为正常效果，当值为1时，物体会完全消融。 LineWidth 属性用于控制模拟烧焦效果时的线宽，它的值越大，火焰边缘的蔓延范围越广。 MainTex和 BumpMap 分别对应了物体原本的漫反射纹理和法线纹理。BurnFirstColor 和BurnSecondColor 对应了火焰边缘的两种颜色值。 BurnMap 则是关键的噪声纹理。

(2)我们在SubShader 块中定义消融所需的Pass:

```
		Pass {
			Tags { "LightMode"="ForwardBase" }

			Cull Off
			
			CGPROGRAM
			
			#include "Lighting.cginc"
			#include "AutoLight.cginc"
			
			#pragma multi_compile_fwdbase
```

为了得到正确的光照，我们设置了Pass的LightMode和 multi compile fwdbase 的编译指令。值得注意的是，我们还使用Cu命令关闭了该Shader 的面片剔除，也就是说，模型的正面和背面都会被渲染。这是因为，消融会导致裸露模型内部的构造，如果只渲染正面会出现错误的结果。

(3)定义顶点着色器

```
struct v2f {
				float4 pos : SV_POSITION;
				float2 uvMainTex : TEXCOORD0;
				float2 uvBumpMap : TEXCOORD1;
				float2 uvBurnMap : TEXCOORD2;
				float3 lightDir : TEXCOORD3;
				float3 worldPos : TEXCOORD4;
				SHADOW_COORDS(5)
			};
			
			v2f vert(a2v v) {
				v2f o;
				o.pos = UnityObjectToClipPos(v.vertex);
				
				o.uvMainTex = TRANSFORM_TEX(v.texcoord, _MainTex);
				o.uvBumpMap = TRANSFORM_TEX(v.texcoord, _BumpMap);
				o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);
				
				TANGENT_SPACE_ROTATION;
  				o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
  				
  				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
  				
  				TRANSFER_SHADOW(o);
				
				return o;
			}
			
```

顶点着色器的代码很常规。我们使用宏TRANSFORM TEX计算了三张纹理对应的纹理坐标再把光源方向从模型空间变换到了切线空间。最后，为了得到阴影信息，计算了世界空间下的顶点位置和阴影纹理的采样坐标(使用了TRANSFERSHADOW宏)。具体原理可参见9.4节。

(4)我们还需要实现片元着色器来模拟消融效果:

```
fixed4 frag(v2f i) : SV_Target {
				fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;
				
				clip(burn.r - _BurnAmount);
				
				float3 tangentLightDir = normalize(i.lightDir);
				fixed3 tangentNormal = UnpackNormal(tex2D(_BumpMap, i.uvBumpMap));
				
				fixed3 albedo = tex2D(_MainTex, i.uvMainTex).rgb;
				
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * albedo;
				
				fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));

				fixed t = 1 - smoothstep(0.0, _LineWidth, burn.r - _BurnAmount);
				fixed3 burnColor = lerp(_BurnFirstColor, _BurnSecondColor, t);
				burnColor = pow(burnColor, 5);
				
				UNITY_LIGHT_ATTENUATION(atten, i, i.worldPos);
				fixed3 finalColor = lerp(ambient + diffuse * atten, burnColor, t * step(0.0001, _BurnAmount));
				
				return fixed4(finalColor, 1);
			}
```

我们首先对噪声纹理进行采样,并将采样结果和用于控制消融程度的属性 BurnAmount 相减,传递给 clip函数。

当结果小于0时，该像素将会被剔除，从而不会显示到屏幕上。如果通过了测试，则进行正常的光照计算。我们首先根据漫反射纹理得到材质的反射率albedo，并由此计算得到环境光照，进而得到漫反射光照。

然后，我们计算了烧焦颜色burnColor。我们想要在宽度为LineWidth 的范围内模拟一个烧焦的颜色变化，第一步就使用了 smoothstep 函数来计算混合系数4。当t值为1时，表明该像素位于消融的边界处，当值为0时，表明该像素为正常的模型颜色而中间的插值则表示需要模拟一个烧焦效果。我们首先用来混合两种火焰颜色BurnFirstColor和 BurnSecondColor，为了让效果更接近烧焦的痕迹，我们还使用pow函数对结果进行处理。然后,我们再次使用1来混合正常的光照颜色(环境光+漫反射)和烧焦颜色。我们这里又使用了 step函数来保证当 BurnAmount为0时,不显示任何消融效果。

最后,返回混合后的颜色值 finalColor。

(5)与之前的实现不同，我们在本例中还定义了一个用于投射阴影的Pass。正如我们在 9.4.5节中的解释一样，使用透明度测试的物体的阴影需要特别处理，如果仍然使用普通的阴影Pass,那么被剔除的区域仍然会向其他物体投射阴影，造成“穿帮”。为了让物体的阴影也能配合透明度测试产生正确的效果，我们需要自定义一个投射阴影的Pass:

```
// Pass to render object as a shadow caster
		Pass {
			Tags { "LightMode" = "ShadowCaster" }
			
			CGPROGRAM
			
			#pragma vertex vert
			#pragma fragment frag
			
			#pragma multi_compile_shadowcaster
```

在 Unity 中，用于投射阴影的Pass 的LightMode 需要被设置为 ShadowCaster，同时，还需要使用#pragma multi compile shadowcaster指明它需要的编译指令。

顶点着色器和片元着色器的代码很简单:

```
v2f vert(appdata_base v) {
				v2f o;
				
				TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
				
				o.uvBurnMap = TRANSFORM_TEX(v.texcoord, _BurnMap);
				
				return o;
			}
			
			fixed4 frag(v2f i) : SV_Target {
				fixed3 burn = tex2D(_BurnMap, i.uvBurnMap).rgb;
				
				clip(burn.r - _BurnAmount);
				
				SHADOW_CASTER_FRAGMENT(i)
			}
```

阴影投射的重点在于我们需要按正常 Pass的处理来剔除片元或进行顶点动画，以便阴影可以和物体正常渲染的结果相匹配。

在自定义的阴影投射的Pass中，我们通常会使用 Unity 提供的内置宏V2F SHADOW CASTER、TRANSFER SHADOW CASTER NORMALOFFSET(旧版本中会使用TRANSFERSHADOWCASTER)和SHADOWCASTERFRAGMENT来帮助我们计算阴影投射时需要的各种变量，而我们可以只关注自定义计算的部分。

在上面的代码中，我们首先在v2f结构体中利用V2FSHADOWCASTER来定义阴影投射需要定义的变量。随后，在顶点着色器中，我们使用TRANSFERSHADOWCASTERNORMALOFFSET来填充V2FSHADOWCASTER 在背后声明的一些变量，这是由Unity 在背后为我们完成的。

我们需要在顶点着色器中关注自定义的计算部分，这里指的就是我们需要计算噪声纹理的采样坐标uvBurnMap。

在片元着色器中，我们首先按之前的处理方法使用噪声纹理的采样结果来剔除片元，最后再利用SHADOWCASTERFRAGMENT来让Unity为我们完成阴影投射的部分，把结果输出到深度图和阴影映射纹理中。

![image-20251125001949202](image-20251125001949202.png)

通过 Unity 提供的这3个内置宏(在 UnityCGcginc 文件中被定义)，我们可以方便地自定义需要的阴影投射的Pass，但由于这些宏需要使用一些特定的输入变量，因此我们需要保证为它们提供了这些变量。例如，TRANSFERSHADOWCASTERNORMALOFFSET会使用名称v作为输入结构体,v中需要包含顶点位置v.vertex和顶点法线v.normal的信息，我们可以直接使用内置的appdata base 结构体，它包含了这些必需的顶点变量。如果我们需要进行顶点动画，可以在顶点着色器中直接修改v.vertex,再传递给 TRANSFER SHADOWCASTER NORMALOFFSET即可(可参见11.3.3节)。

在本例中，我们使用的噪声纹理(对应本书资源的Assets/Textures/Chapter15/Burn Noise.png)如图15.2所示。把它拖曳到材质的BurnMap属性上，再调整材质的BurnAmount性，就可以看到木箱逐渐消融的效果。在本书资源的实现中，我们实现了一个辅助脚本，用来随时间调整材质的BurnAmount值，因此，当读者单击运行后，也可以看到消融的动画效果。使用不同的噪声和纹理属性(即材质面板上纹理的Tiling和 Ofset值)都会得到不同的消融效果。因此，要想得到好的消融效果，也需要美术人员提供合适的噪声纹理来配合。

## 2.水波效果

在模拟实时水面的过程中，我们往往也会使用噪声纹理。此时，噪声纹理通常会用作一个高度图，以不断修改水面的法线方向。为了模拟水不断流动的效果，我们会使用和时间相关的变量来对噪声纹理进行采样，当得到法线信息后，再进行正常的反射+折射计算，得到最后的水面波动效果。
在本节中，我们将会使用一个由噪声纹理得到的法线贴图，实现一个包含菲涅耳反射(详见10.1.5节)的水面效果，如图15.3所示。

![image-20251125003541111](image-20251125003541111.png)

我们曾在10.2.2节介绍过如何使用反射和折射来模拟一个透明玻璃的效果。本节使用的Shader 和10.2.2节中的实现基本相同。我们使用一张立方体纹理(Cubemap)作为环境纹理，模拟反射。

为了模拟折射效果，我们使用GrabPass来获取当前屏幕的渲染纹理，并使用切线空间下的法线方向对像素的屏幕坐标进行偏移，再使用该坐标对渲染纹理进行屏幕采样，从而模拟近似的折射效果。与 10.2.2节中的实现不同的是，水波的法线纹理是由一张噪声纹理生成而得，而且会随着时间变化不断平移，模拟波光粼粼的效果。

除此之外，我们没有使用一个定值来混合反射和折射颜色，而是使用之前提到的菲涅耳系数来动态决定混合系数。我们使用如下公式来计算菲涅耳系数:

![image-20251125004800025](image-20251125004800025.png)

其中，v和n分别对应了视角方向和法线方向。它们之间的夹角越小，fesnel值越小，反射越弱，折射越强。菲涅耳系数还经常会用于边缘光照的计算中。
为此，我们需要做如下准备工作。
(1)新建一个场景。在本书资源中，该场景名为Scene152。在 Unity5.2中，默认情况下场景将包含一个摄像机和一个平行光,并且使用了内置的天空盒子。在 Window->Lighting->Skybox，
中去掉场景中的天空盒子。

(2)新建一个材质。在本书资源中，该材质名为WaterWaveMat。

(3)新建一个 Unity Shader。在本书资源中，该Shader名为Chapter15-WaterWave。把新的Shader赋给第2步中创建的材质。

(4)构建一个测试水波效果的场景。

在本书资源的实现中，我们构建了一个由6面墙围成的封闭房间，它们都使用了我们在9.5节中创建的标准材质。我们还在房间中放置了一个平面来模拟水面。把第2步中创建的材质赋给该平面。

们使用了10.1.2节中实现的创建立方体纹理的脚本(通过 Gameobject->Render into Cubemap 打开编辑窗口)来创建它，如图 15.4所示。在本书资源中，该Cubemap名为Water Cubemap。

![image-20251125004915458](image-20251125004915458.png)

1）声明

```
Properties {
	_Color ("Main Color", Color) = (0, 0.15, 0.115, 1)
	_MainTex ("Base (RGB)", 2D) = "white" {}
	_WaveMap ("Wave Map", 2D) = "bump" {}
	_Cubemap ("Environment Cubemap", Cube) = "_Skybox" {}
	_WaveXSpeed ("Wave Horizontal Speed", Range(-0.1, 0.1)) = 0.01
	_WaveYSpeed ("Wave Vertical Speed", Range(-0.1, 0.1)) = 0.01
	_Distortion ("Distortion", Range(0, 100)) = 10
}
```

其中，Color 用于控制水面颜色;MainTex是水面波纹材质纹理默认为白色纹理；WaveMap是一个由噪声纹理生成的法线纹理;Cubemap 是用于模拟反射的立方体纹理;Distortion 则用于控制模拟折射时图像的扭曲程度;WaveXSpeed和WaveYSpeed 分别用于控制法线纹理在X和Y方向上的平移速度。

(2)定义相应的渲染队列，并使用GrabPass来获取屏幕图像:

```
SubShader {
	// We must be transparent, so other objects are drawn before this one.
	Tags { "Queue"="Transparent" "RenderType"="Opaque" }
	
	// This pass grabs the screen behind the object into a texture.
	// We can access the result in the next pass as _RefractionTex
	GrabPass { "_RefractionTex" }
```

我们首先在 SubShader 的标签中将渲染队列设置成Transparent,并把后面的 RenderType 设置为Opaque。把Queue 设置成 Transparent 可以确保该物体渲染时，其他所有不透明物体都已经被渲染到屏幕上了，否则就可能无法正确得到“透过水面看到的图像”。而设置RenderType 则是为了在使用着色器替换(Shader Replacement)时，该物体可以在需要时被正确渲染。这通常发生在我们需要得到摄像机的深度和法线纹理时,这在第13 章中介绍过。随后,我们通过关键词 GrabPass定义了一个抓取屏幕图像的Pass。

在这个 Pass中我们定义了一个字符串，该字符串内部的名称决定了抓取得到的屏幕图像将会被存入哪个纹理中(可参见1022节)。


3) 定义渲染水面所需的Pass。为了在Shader 中访问各个属性，我们首先需要定义它们对应的变量:

```
		fixed4 _Color;
		sampler2D _MainTex;
		float4 _MainTex_ST;
		sampler2D _WaveMap;
		float4 _WaveMap_ST;
		samplerCUBE _Cubemap;
		fixed _WaveXSpeed;
		fixed _WaveYSpeed;
		float _Distortion;	
		sampler2D _RefractionTex;
		float4 _RefractionTex_TexelSize;
```

需要注意的是，我们还定义了_RefractionTex和RefractionTex_TexelSize 变量，这对应了在使用 GrabPass时，指定的纹理名称。RefractionTex TexelSize 可以让我们得到该纹理的纹素大小,例如一个大小为 256x512 的纹理，它的纹素大小为(1/256,1/512)。我们需要在对屏幕图像的采样坐标进行偏移时使用该变量。

(4)定义顶点着色器，这和10.2.2节中的实现完全一样:

```
struct v2f {
	float4 pos : SV_POSITION;
	float4 scrPos : TEXCOORD0;
	float4 uv : TEXCOORD1;
	float4 TtoW0 : TEXCOORD2;  
	float4 TtoW1 : TEXCOORD3;  
	float4 TtoW2 : TEXCOORD4; 
};

v2f vert(a2v v) {
	v2f o;
	o.pos = UnityObjectToClipPos(v.vertex);
	
	o.scrPos = ComputeGrabScreenPos(o.pos);
	
	o.uv.xy = TRANSFORM_TEX(v.texcoord, _MainTex);
	o.uv.zw = TRANSFORM_TEX(v.texcoord, _WaveMap);
	
	float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;  
	fixed3 worldNormal = UnityObjectToWorldNormal(v.normal);  
	fixed3 worldTangent = UnityObjectToWorldDir(v.tangent.xyz);  
	fixed3 worldBinormal = cross(worldNormal, worldTangent) * v.tangent.w; 
	
	o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);  
	o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);  
	o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);  
	
	return o;
}
```

在进行了必要的顶点坐标变换后，我们通过调用ComputeGrabScreenPos来得到对应被抓取屏幕图像的采样坐标。读者可以在UnityCGcginc文件中找到它的声明，它的主要代码和ComputeScreenPos基本类似，最大的不同是针对平台差异造成的采样坐标问题(见5.6.1节)进行了处理。

接着,我们计算了 MainTex和 BumpMap的采样坐标,并把它们分别存储在一个 foat4类型变量的xy 和zw 分量中。由于我们需要在片元着色器中把法线方向从切线空间(由法线纹理采样得到)变换到世界空间下，以便对Cubemap进行采样，因此，我们需要在这里计算该顶点对应的从切线空间到世界空间的变换矩阵,并把该矩阵的每一行分别存储在TtoW0、TtoW1和TtoW2的x”z分量中。

这里面使用的数学方法就是，得到切线空间下的3个坐标轴(x、y、z轴分别对应了切线、副切线和法线的方向)在世界空间下的表示，再把它们依次按列组成一个变换矩阵即可TtoW0 等值的w分量同样被利用起来，用于存储世界空间下的顶点坐标。

(5)定义片元着色器:

```
fixed4 frag(v2f i) : SV_Target {
	float3 worldPos = float3(i.TtoW0.w, i.TtoW1.w, i.TtoW2.w);
	fixed3 viewDir = normalize(UnityWorldSpaceViewDir(worldPos));
	float2 speed = _Time.y * float2(_WaveXSpeed, _WaveYSpeed);
	
	// Get the normal in tangent space
	fixed3 bump1 = UnpackNormal(tex2D(_WaveMap, i.uv.zw + speed)).rgb;
	fixed3 bump2 = UnpackNormal(tex2D(_WaveMap, i.uv.zw - speed)).rgb;
	fixed3 bump = normalize(bump1 + bump2);
	
	// Compute the offset in tangent space
	float2 offset = bump.xy * _Distortion * _RefractionTex_TexelSize.xy;
	i.scrPos.xy = offset * i.scrPos.z + i.scrPos.xy;
	fixed3 refrCol = tex2D( _RefractionTex, i.scrPos.xy/i.scrPos.w).rgb;
	
	// Convert the normal to world space
	bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
	fixed4 texColor = tex2D(_MainTex, i.uv.xy + speed);
	fixed3 reflDir = reflect(-viewDir, bump);
	fixed3 reflCol = texCUBE(_Cubemap, reflDir).rgb * texColor.rgb * _Color.rgb;
	
	fixed fresnel = pow(1 - saturate(dot(viewDir, bump)), 4);
	fixed3 finalColor = reflCol * fresnel + refrCol * (1 - fresnel);
	
	return fixed4(finalColor, 1);
}
```

我们首先通过 TtoW0 等变量的w分量得到世界坐标，并用该值得到该片元对应的视角方向。

除此之外，我们还使用内置的Time.y变量和WaveXSpeed、WaveYSpeed 属性计算了法线纹理的当前偏移量,并利用该值对法线纹理进行两次采样(这是为了模拟两层交叉的水面波动的效果),对两次结果相加并归一化后得到切线空间下的法线方向。

然后，和10.2.2节中的处理一样，我们使用该值和 Distortion属性以及 RefactionTex TexelSize 来对屏幕图像的采样坐标进行偏移，模拟折射效果。 Distortion 值越大，偏移量越大，水面背后的物体看起来变形程度越大。在这里，我们选择使用切线空间下的法线方向来进行偏移，是因为该空间下的法线可以反映顶点局部空间下的法线方向。

需要注意的是，在计算偏移后的屏幕坐标时，我们把偏移量和屏幕坐标的z分量相乘，这是为了模拟深度越大、折射程度越大的效果。如果读者不希望产生这样的效果，可以直接把偏移值叠加到屏幕坐标上。随后，我们对scrPos进行了透视除法，再使用该坐标对抓取的屏幕图像 RefractionTex 进行采样，得到模拟的折射颜色。

之后，我们把法线方向从切线空间变换到了世界空间下(使用变换矩阵的每一行，即TtoW0、TtoW1和TtoW2，分别和法线方向点乘，构成新的法线方向)，并据此得到视角方向相对于法线方向的反射方向。随后，使用反射方向对Cubemap进行采样，并把结果和主纹理颜色相乘后得到反射颜色。我们也对主纹理进行了纹理动画，以模拟水波的效果。
为了混合折射和反射颜色，我们随后计算了菲涅耳系数。我们使用之前的公式来计算菲涅耳系数，并据此来混合折射和反射颜色，作为最终的输出颜色。
在本例中,我们使用的噪声纹理(对应本书资源的 Assets/Textures/Chapter15/Water Noise.png)

如图15.5左图所示。由于在本例中，我们需要的是一张法线纹理，因此我们可以从该噪声纹理的灰度值中生成需要的法线信息，这是通过在它的纹理面板中把纹理类型设置为Normalmap，并选中Createfrom grayscale 来完成的。最后生成的法线纹理如图15.5右图所示。我们把生成的法线纹理拖曳到材质的WaveMap属性上，再单击运行后，就可以看到水面波动的效果了。

![image-20251125005517642](image-20251125005517642.png)

## 3.再谈全局雾效

我们在13.3节讲到了如何使用深度纹理来实现一种基于屏幕后处理的全局雾效。我们由深度纹理重建每个像素在世界空间下的位置，再使用一个基于高度的公式来计算雾效的混合系数，最后使用该系数来混合雾的颜色和原屏幕颜色。

13.3节的实现效果是一个基于高度的均匀雾效，即在同一个高度上，雾的浓度是相同的，如图15.6左图所示。然而，一些时候我们希望可以模拟一种不均匀的雾效，同时让雾不断飘动，使雾看起来更加飘渺，如图15.6右图所示。而这就可以通过使用一张噪声纹理来实现。

![image-20251125005613505](image-20251125005613505.png)

本节的实现非常简单,绝大多数代码和13.3节中的完全一样,我们只是添加了噪声相关的参数和属性，并在Shader的片元着色器中对高度的计算添加了噪声的影响。为了完整性，我们会给出本节使用的脚本和Shader的实现，但其中使用的原理不再赘述，读者可参见13.3节。

我们首先需要进行如下准备工作。

(1)新建一个场景。在本书资源中，该场景名为 Scene 15.3。在 Unity 5.2中，默认情况下场景将包含一个摄像机和一个平行光，并且使用了内置的天空盒子。在 Window ->Lighting->Skybox 中去掉场景中的天空盒子。

(2)我们需要搭建一个测试雾效的场景。在本书资源的实现中，我们构建了一个包含3面墙的房间，并放置了两个立方体和两个球体，它们都使用了我们在9.5节中创建的标准材质。

(3)新建一个脚本。在本书资源中，该脚本名为FogWithNoise.cs。把该脚本拖曳到摄像机上。

(4)新建一个 Unity Shader。在本书资源中，该Shader 名为 Chapter15-FogWithNoise。

我们首先来编写FogWithNoise.cs脚本。打开该脚本，并进行如下修改。

### FogWithNoise.cs

(1)首先，继承12.1节中创建的基类:

```
public class FogwithNoise:PostEffectsBase
```

(2)声明该效果需要的Shader，并据此创建相应的材质:

```
public Shader fogShader;
	private Material fogMaterial = null;

	public Material material {  
		get {
			fogMaterial = CheckShaderAndCreateMaterial(fogShader, fogMaterial);
			return fogMaterial;
		}  
	}
```

3) 在本节中，我们需要获取摄像机的相关参数，如近裁剪平面的距离、FOV 等，同时还需要获取摄像机在世界空间下的前方、上方和右方等方向，因此我们用两个变量存储摄像机的Camera组件和Transform组件:

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

	private Transform myCameraTransform;
	public Transform cameraTransform {
		get {
			if (myCameraTransform == null) {
				myCameraTransform = camera.transform;
			}
			
			return myCameraTransform;
		}
	}
```

(4)定义模拟雾效时使用的各个参数:

```
[Range(0.1f, 3.0f)]
	public float fogDensity = 1.0f;

	public Color fogColor = Color.white;

	public float fogStart = 0.0f;
	public float fogEnd = 2.0f;

	public Texture noiseTexture;

	[Range(-0.5f, 0.5f)]
	public float fogXSpeed = 0.1f;

	[Range(-0.5f, 0.5f)]
	public float fogYSpeed = 0.1f;

	[Range(0.0f, 3.0f)]
	public float noiseAmount = 1.0f;
```

fogDensity 用于控制雾的浓度，fogColor 用于控制雾的颜色。我们使用的雾效模拟函数是基于高度的，因此参数fogStant 用于控制雾效的起始高度，fogEnd用于控制雾效的终止高度。

noiseTexture 是我们使用的噪声纹理，fogXSpeed和fogYSpeed 分别对应了噪声纹理在X和Y方向上的移动速度，以此来模拟雾的飘动效果。最后，noiseAmount用于控制噪声程度，当noiseAmount为0时，表示不应用任何噪声，即得到一个均匀的基于高度的全局雾效。

(5)由于本例需要获取摄像机的深度纹理,我们在脚本的 OnEnable 函数中设置摄像机的相应状态:

```
	void OnEnable() {
		GetComponent<Camera>().depthTextureMode |= DepthTextureMode.Depth;
	}
```

(6) 最后，我们实现了 OnRenderlmage函数:

```
void OnRenderImage (RenderTexture src, RenderTexture dest) {
		if (material != null) {
			Matrix4x4 frustumCorners = Matrix4x4.identity;
			
			float fov = camera.fieldOfView;
			float near = camera.nearClipPlane;
			float aspect = camera.aspect;
			
			float halfHeight = near * Mathf.Tan(fov * 0.5f * Mathf.Deg2Rad);
			Vector3 toRight = cameraTransform.right * halfHeight * aspect;
			Vector3 toTop = cameraTransform.up * halfHeight;
			
			Vector3 topLeft = cameraTransform.forward * near + toTop - toRight;
			float scale = topLeft.magnitude / near;
			
			topLeft.Normalize();
			topLeft *= scale;
			
			Vector3 topRight = cameraTransform.forward * near + toRight + toTop;
			topRight.Normalize();
			topRight *= scale;
			
			Vector3 bottomLeft = cameraTransform.forward * near - toTop - toRight;
			bottomLeft.Normalize();
			bottomLeft *= scale;
			
			Vector3 bottomRight = cameraTransform.forward * near + toRight - toTop;
			bottomRight.Normalize();
			bottomRight *= scale;
			
			frustumCorners.SetRow(0, bottomLeft);
			frustumCorners.SetRow(1, bottomRight);
			frustumCorners.SetRow(2, topRight);
			frustumCorners.SetRow(3, topLeft);
			
			material.SetMatrix("_FrustumCornersRay", frustumCorners);

			material.SetFloat("_FogDensity", fogDensity);
			material.SetColor("_FogColor", fogColor);
			material.SetFloat("_FogStart", fogStart);
			material.SetFloat("_FogEnd", fogEnd);

			material.SetTexture("_NoiseTex", noiseTexture);
			material.SetFloat("_FogXSpeed", fogXSpeed);
			material.SetFloat("_FogYSpeed", fogYSpeed);
			material.SetFloat("_NoiseAmount", noiseAmount);

			Graphics.Blit (src, dest, material);
		} else {
			Graphics.Blit(src, dest);
		}
	}
```

我们首先利用 13.3节学习的方法计算近裁剪平面的4个角对应的向量，并把它们存储在一个矩阵类型的变量(frustumCormers)中。计算过程和原理均可参见13.3节。随后，我们把结果和其他参数传递给材质，并调用Graphics.Blit(src,dest,material)把渲染结果显示在屏幕上。

Chapter15-FogWithNoise.shader

下面，我们来实现 Shader 的部分。打开Chapter15-FogWithNoise，进行如下修改。

(1)我们首先需要声明本例使用的各个属性:

```
Properties {
	_MainTex ("Base (RGB)", 2D) = "white" {}
	_FogDensity ("Fog Density", Float) = 1.0
	_FogColor ("Fog Color", Color) = (1, 1, 1, 1)
	_FogStart ("Fog Start", Float) = 0.0
	_FogEnd ("Fog End", Float) = 1.0
	_NoiseTex ("Noise Texture", 2D) = "white" {}
	_FogXSpeed ("Fog Horizontal Speed", Float) = 0.1
	_FogYSpeed ("Fog Vertical Speed", Float) = 0.1
	_NoiseAmount ("Noise Amount", Float) = 1
}
```

(2)在本节中,我们使用 CGINCLUDE来组织代码。我们在 SubShader 块中利用CGINCLUDE和ENDCG语义来定义一系列代码:

```
SubShader {
	CGINCLUDE
	
	……

	ENDCG
	
	……
	}
```

(3)声明代码中需要使用的各个变量:

```
float4x4 _FrustumCornersRay;

sampler2D _MainTex;
half4 _MainTex_TexelSize;
sampler2D _CameraDepthTexture;
half _FogDensity;
fixed4 _FogColor;
float _FogStart;
float _FogEnd;
sampler2D _NoiseTex;
half _FogXSpeed;
half _FogYSpeed;
half _NoiseAmount;
```

FrustumCornersRay虽然没有在Properties中声明，但仍可由脚本传递给 Shader。除了Properties 中声明的各个属性，我们还声明了深度纹理 CameraDepthTexture，Unity 会在背后把得到的深度纹理传递给该值。

(4)定义顶点着色器，这和13.3节中的实现完全一致。读者可以在 13.3 节找到它的实现和相关解释。

(5)定义片元着色器:

```
fixed4 frag(v2f i) : SV_Target {
	float linearDepth = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv_depth));
	float3 worldPos = _WorldSpaceCameraPos + linearDepth * i.interpolatedRay.xyz;
	
	float2 speed = _Time.y * float2(_FogXSpeed, _FogYSpeed);
	float noise = (tex2D(_NoiseTex, i.uv + speed).r - 0.5) * _NoiseAmount;
			
	float fogDensity = (_FogEnd - worldPos.y) / (_FogEnd - _FogStart); 
	fogDensity = saturate(fogDensity * _FogDensity * (1 + noise));
	
	fixed4 finalColor = tex2D(_MainTex, i.uv);
	finalColor.rgb = lerp(finalColor.rgb, _FogColor.rgb, fogDensity);
	
	return finalColor;
}
		
```

我们首先根据深度纹理来重建该像素在世界空间中的位置。然后，我们利用内置的_Time.y变量和 FogXSpeed、FogYSpeed 属性计算出当前噪声纹理的偏移量，并据此对噪声纹理进行采样，得到噪声值。我们把该值减去0.5，再乘以控制噪声程度的属性NoiseAmount，得到最终的噪声值。随后，我们把该噪声值添加到雾效浓度的计算中，得到应用噪声后的雾效混合系数fogDensity。最后，我们使用该系数将雾的颜色和原始颜色进行混合后返回。

(6)随后，我们定义了雾效渲染所需的Pass:

```
Pass {          	
	CGPROGRAM  
	
	#pragma vertex vert  
	#pragma fragment frag  
	  
	ENDCG
}
```

(7)最后，我们关闭了 Shader的 Fallback:

```
Fallback off
```

完成后返回编辑器，并把Chapter15-FogWithNoise 拖曳到摄像机的 FogWithNoise.cs脚本中的 fogShader 参数中。

![image-20251125010950682](image-20251125010950682.png)

当然，我们可以在 FogWithNoise.cs的脚本面板中将 fogShader 参数的默认值设置为Chapter15-FogWithNoise，这样就不需要以后使用时每次都手动拖曳了。本节使用的噪声纹理(对应本书资源的Assets/Textures/Chapter15/Fog_Noise.jpg)如图 15.7 所示。我们把该噪声纹理拖曳到FogWithNoise.cs脚本中的noiseTexture参数中，我们也可以参照之前的方法，直接在FogWithNoise.cs的脚本面板中将 noiseTexture 参数的默认值设置为FogNoise.jpg，这样就不需要以后使用时每次都手动拖曳了。

## 4.扩展阅读

读者在阅读本章时，可能会有一个疑问:这些噪声纹理都是如何构建出来的?这些噪声纹理可以被认为是一种程序纹理(Procedure Texture)，它们都是由计算机利用某些算法生成的。

**Perlin 噪声**(https:/en.wikipedia.org/wiki/Perlin noise)和 **Worley 噪声**(htps://en.wikipedia.org/wiki/Worley noise )是两种最常使用的噪声类型，例如我们在15.3节中使用的噪声纹理由 Perlin 噪声生成而来。

Perlin噪声可以用于生成更自然的噪声纹理，而 Worley 噪声则通常用于模拟诸如石头、水、纸张等多孔噪声。现代的图像编辑软件，如Photoshop等，往往提供了类似的功能或插件，以帮助美术人员生成需要的噪声纹理，但如果读者想要更加自由地控制噪声纹理的生成，可能就需要了解它们的生成原理。

读者可以在这个博客(http://lafla2.github.io/2014/08/09/perlinnoise.html)中找到一篇关于理解 Perlin 噪声的非常好的文章，在文章的最后，作者还给出了很多其他出色的参考链接。关于 Worley 噪声，读者可以在作者 Worley1998年发表的论文四中找到它的算法和实现细节。在另-个非常好的博客(htp://scrawkblog.com/category/procedural-noise/)中，博主给出了很多程序噪声在 Unity中的实现，并包含了实现源码。