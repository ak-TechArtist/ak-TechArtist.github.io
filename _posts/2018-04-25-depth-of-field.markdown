---
layout:     post
title:      "景深效果（译）"
subtitle:   ""
date:       2018-04-25 12:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

[原文](http://catlikecoding.com/unity/tutorials/advanced-rendering/depth-of-field/)是Catlike写的一篇教程。这也是我第一次尝试翻译技术文章，比想象中的要累不少- -！

# 景深效果（Depth of Field）
> 实现此效果的步骤为：  
1 决定模糊圈（CoC）   
2 创建焦外效果（bokeh）  
3 对焦图片以及取消对焦  
4 分离及融合前景、后景  

此课程演示了如何创建一个景深的屏幕后处理效果，其是继[Bloom](http://catlikecoding.com/unity/tutorials/advanced-rendering/bloom/)之后的又一课程。
此课程使用unity 2017.3.0p3制作。

![对焦效果](/img/in-post/depth-of-field/1.jpg)
<small class="img-hint">对焦效果</small>

### 一 设置场景
我们能感知到光是因为光子碰撞到视网膜。同样的，摄像机能记录光也是因为光子碰撞到胶片或者图像传感器。在所有的例子中，光都被聚焦以产生锐利的图片，但同一时间不是所有物体都能被聚焦。只有特定距离的物体能被聚焦，而其他物体则表现得在焦点外。此视觉效果被称为景深。而焦点外效果则被称作散景（bokeh），此词源于日语中的模糊（日本的摄影还是强的呵---译者注）  
通常来讲，在用我们的双眼看东西的时候，我们察觉不到景深效果，这是因为我们将注意力放在那些我们聚焦的物体上了。而在看图片或者视频的时候，却明显得多，因为我们能聚焦那些在摄像机焦点外的物体。而由此产生的散景能用于引导观众的注意力，所以，这是一个艺术工具。  
GPUs不需要对焦，它们表现得就像完美的拥有无数焦点的摄像机。它会产生清晰的图片，但这样的话就没有景深效果了。但有几种方法可以模拟它，在此课程中我们仿造Unity的post effect stack第二版来创建景深效果，同时保持尽可能地简单。  

##### 1 设置场景
为了测试景深效果，我们创建一个包含不同距离物体的场景。一个10x10颜色丰富的平面打底，放置了许多物体在其之上，且设置了4个悬空物体靠近摄像机，你可以通过文后的链接下载此示例场景，或者制作你自己的场景。

![](/img/in-post/depth-of-field/2.png)
<small class="img-hint">示例场景</small>

首先，我们创建一个名为DepthOfField的shader
```
Shader "Hidden/DepthOfField" {
	Properties {
		_MainTex ("Texture", 2D) = "white" {}
	}

	CGINCLUDE
		#include "UnityCG.cginc"

		sampler2D _MainTex;
		float4 _MainTex_TexelSize;

		struct VertexData {
			float4 vertex : POSITION;
			float2 uv : TEXCOORD0;
		};

		struct Interpolators {
			float4 pos : SV_POSITION;
			float2 uv : TEXCOORD0;
		};

		Interpolators VertexProgram (VertexData v) {
			Interpolators i;
			i.pos = UnityObjectToClipPos(v.vertex);
			i.uv = v.uv;
			return i;
		}

	ENDCG

	SubShader {
		Cull Off
		ZTest Always
		ZWrite Off

		Pass {
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					return tex2D(_MainTex, i.uv);
				}
			ENDCG
		}
	}
}
```
其次，创建名为DepthOfFieldEffect的脚本

```
using UnityEngine;
using System;

[ExecuteInEditMode, ImageEffectAllowedInSceneView]
public class DepthOfFieldEffect : MonoBehaviour {

	[HideInInspector]
	public Shader dofShader;

	[NonSerialized]
	Material dofMaterial;

	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		if (dofMaterial == null) {
			dofMaterial = new Material(dofShader);
			dofMaterial.hideFlags = HideFlags.HideAndDontSave;
		}

		Graphics.Blit(source, destination, dofMaterial);
	}
}
```
设置脚本的默认引用为之前创建的Shader，正如Unity提醒的那样，默认引用只会在editor下有效。

![](/img/in-post/depth-of-field/3.png)
<small class="img-hint">默认Shader引用</small>

将此脚本赋值给摄像机，我们假设是在线性空间、HDR下绘制场景的。且因为我们需要读取深度缓冲，所以此效果不能和MSAA同时使用。同时因为此效果是基于深度缓冲的，所以半透明物体是不会被纳入其中的。

![](/img/in-post/depth-of-field/4.png)
<small class="img-hint">Camera设置</small>

### 二 决定模糊圈
最简单的摄像机是一个理想化的单孔摄像机，和所有的摄像机一样，它有一个用于记录光的图像平面。在此平面之前是一个被称为光圈（aperture）的小孔：仅允许一个光束通过。在摄像机前的物体会发射或者反射许多不同方向的光线。但对于没一点而言，仅单一的设想能通过此孔且被记录。

![](/img/in-post/depth-of-field/5.png)
<small class="img-hint">记录三个点</small>

因为每个点只会产生一个光束，所以此图片总是锐利的，但是单一的光束并不足够亮，所以很难看见结果图。你需要等待一阵直到足够的光去累积出一张清晰的图片，这意味着摄像机需要一个长曝光时间。对于静态场景来说，这都不是事儿，但如果场景是动态的，则会产生许多运动模糊，所以这不是一个可行的用于记录锐利图像的摄像机。  
为减少曝光时间，光需要积累得足够快。唯一的方法是同时记录多个光线。这能通过增加光圈的半径来实现。假设光圈为圆形，这意味着每一个点都会在图片平面上投影光束，而不是光线。所以我们会接收更多的光线，但不再是一个点的，而是一个圆盘的。至于该被覆盖的区域有多大则取决于点、孔、图片平面之间的距离。最后的结果是图片变得更亮了，但也更模糊了。

![](/img/in-post/depth-of-field/6.png)
<small class="img-hint">使用更大的光圈</small>

为了重新聚焦光线，我们需要将光束还原为一个点，这可以通过在光圈前加一个镜头来实现。镜头可以弯曲光线，使其重新聚焦。这能形成一个亮且锐利的成像，但是只在一个范围内。对于远处的点，聚焦不够，而对于近处的点，聚焦又过了。两者都让成像变为了光束，也就变模糊了，而此模糊投影正是模糊圈（circle of confusion），简称为CoC

![](/img/in-post/depth-of-field/7.png)
<small class="img-hint">只有一个点被聚焦</small>

##### 1 可视化模糊圈
模糊圈的半径可用于测量一个点在焦点外是如何成像的。让我们首先可视化此值。添加一个常量到脚本中去声明第一个pass：模糊圈pass。在bltting的时候显式调用之。
```
	const int circleOfConfusionPass = 0;

	…

	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		…

		Graphics.Blit(source, destination, dofMaterial, circleOfConfusionPass);
	}
```

因为CoC是建立在距离摄像机的距离的，所以，我们需要读取深度buffer。实际上，深度图正是我们需要的，因为摄像机的焦点区域正是一个平行摄像机的平面，假设镜头和图片平面是排列好的。所以，采样深度图，并转换到线性depth中显示。
```
	CGINCLUDE
		#include "UnityCG.cginc"

		sampler2D _MainTex, _CameraDepthTexture;
		…

	ENDCG

	SubShader {
		…

		Pass { // 0 circleOfConfusionPass
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
//					return tex2D(_MainTex, i.uv);
					half depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
					depth = LinearEyeDepth(depth);
					return depth;
				}
			ENDCG
		}
	}
```

##### 2 选择一个简单的模糊圈
我们最终要获取的是CoC值，为了计算它，我们需要在脚本中定一个焦点距离：摄像机和焦点平面的距离。
```
[Range(0.1f, 100f)]
public float focusDistance = 10f;
```

CoC的值会随着点距离焦点平面的距离的增加而增加。具体的关系取决于摄像机及其配置，通常来讲是挺复杂的。在这里我们不模拟真实摄像机，而只是使用一个简单的焦点区域来方便我们理解和控制。我们的CoC在这个范围中，会在0和最大值之间变动，且和焦点距离是相关的。

```
	[Range(0.1f, 10f)]
	public float focusRange = 3f;
```

![](/img/in-post/depth-of-field/8.png)
<small class="img-hint"></small>

这些参数需要设置给Shader

```
		dofMaterial.SetFloat("_FocusDistance", focusDistance);
		dofMaterial.SetFloat("_FocusRange", focusRange);

		Graphics.Blit(source, destination, dofMaterial, circleOfConfusionPass);
```
为Shader添加参数。我们需要使用float值类型去计算CoC。如果我们用的是基于half的HDR缓冲，则需要使用half值类型，虽然这在桌面硬件上并没什么关系。

```
		sampler2D _MainTex, _CameraDepthTexture;
		float4 _MainTex_TexelSize;

		float _FocusDistance, _FocusRange;
```
使用深度值d，焦点距离f，以及焦点范围r，我们能通过(d - f) / r 将CoC找出来。
```
				half4 FragmentProgram (Interpolators i) : SV_Target {
					float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
					depth = LinearEyeDepth(depth);
//					return depth;

					float coc = (depth - _FocusDistance) / _FocusRange;
					return coc;
				}
```

![](/img/in-post/depth-of-field/9.png)
<small class="img-hint">原始CoC值</small>

正值表明那些在焦点距离之后的点，负值表明那些在焦点距离之前的点。-1和1是CoC的极限，所以我们应该将其限制在这个范围内。
```
					float coc = (depth - _FocusDistance) / _FocusRange;
					coc = clamp(coc, -1, 1);
					return coc;
```

更改负值的颜色为红，使得能更好地区分前景和后景。
```
					coc = clamp(coc, -1, 1);
					if (coc < 0) {
						return coc * -half4(1, 0, 0, 1);
					}
					return coc;
```

![](/img/in-post/depth-of-field/10.png)
<small class="img-hint">负值为红的CoC值</small>

##### 3 缓冲模糊圈
我们会在另一个pass中通过CoC的值去调整点的投影。所以需要存储CoC的值到一个临时缓冲中。因为我们只需要存储一个单一的值，所以用单通道的贴图即可完成，使用RenderTextureFormat.RHalf。同时，此缓冲包含的是CoC值，而不是颜色值。所以需要被当成线性数据来对待，让我们显式指定之，虽然我们假设我们是是在线性空间中进行渲染的。  
在第一个Blit中将值渲染到CoC缓冲中，接着新建一个blit将此缓冲显示出来。最后释放该缓冲。
```
		RenderTexture coc = RenderTexture.GetTemporary(
			source.width, source.height, 0,
			RenderTextureFormat.RHalf, RenderTextureReadWrite.Linear
		);

		Graphics.Blit(source, coc, dofMaterial, circleOfConfusionPass);
		Graphics.Blit(coc, destination);

		RenderTexture.ReleaseTemporary(coc);
```

因为我们使用的缓冲贴图只有R通道，所以整个CoC看起来是红色的。且我们需要存储真实的CoC值，所以去除对其颜色的改变。同时，我们可以将返回值由half4更改为half。
```
//				half4
				half FragmentProgram (Interpolators i) : SV_Target {
					float depth = SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, i.uv);
					depth = LinearEyeDepth(depth);

					float coc = (depth - _FocusDistance) / _FocusRange;
					coc = clamp(coc, -1, 1);
//					if (coc < 0) {
//						return coc * -half4(1, 0, 0, 1);
//					}
					return coc;
				}
```

### 三 创建焦外效果
CoC决定了每个点的模糊程度，而光圈则决定了其表现。基本来讲，一个图片会由许多光圈形状的投影组成。所以其中一中焦外效果的创建就是对一个像素多次投影，但这会造成非常大的性能损耗：Overdraw。  
另一种实现方式则认为该像素会受到周边像素的影响，此方法不会产生额外的几何体，但是会对纹理多次采样。我们采用的正是后面这种方式。

##### 1 累积焦外效果
创建一个新的pass用于生成焦外效果。先将主贴图的颜色传入，我们并不关系其alpha通道的值。

```
		Pass { // 0 CircleOfConfusionPass
			…
		}

		Pass { // 1 bokehPass
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					half3 color = tex2D(_MainTex, i.uv).rgb;
					return half4(color, 1);
				}
			ENDCG
		}
```

使用source texture为输入，将此pass用于第二个blit。先忽略CoC数据，假设整个图都在焦点外。
```
	const int circleOfConfusionPass = 0;
	const int bokehPass = 1;

	…

	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		…

		Graphics.Blit(source, coc, dofMaterial, circleOfConfusionPass);
		Graphics.Blit(source, destination, dofMaterial, bokehPass);
//		Graphics.Blit(coc, destination);

		RenderTexture.ReleaseTemporary(coc);
	}
```

为创造焦点外效果，我们需要将周围的像素值进行一个平均。此例中我们围绕当前像素使用一个9x9的方块区域。总共采样81次。
```
				half4 FragmentProgram (Interpolators i) : SV_Target {
//					half3 color = tex2D(_MainTex, i.uv).rgb;
					half3 color = 0;
					for (int u = -4; u <= 4; u++) {
						for (int v = -4; v <= 4; v++) {
							float2 o = float2(u, v) * _MainTex_TexelSize.xy;
							color += tex2D(_MainTex, i.uv + o).rgb;
						}
					}
					color *= 1.0 / 81;
					return half4(color, 1);
```

![](/img/in-post/depth-of-field/11.png)
<small class="img-hint">块状焦外效果</small>

结果是一个马赛克状的图片。实际上，我们使用了一个块状光圈。图片变得更明亮了，因为较亮的像素扩散到了一个较大的区域。这个效果更像是bloom，而不是景深，但一个夸张的效果更容易看出发生了什么。所以我们先保持其明亮，在后面再将其调暗。  
因为我们的简单焦外效果是基于纹素的尺寸的，它的大小是建立在目标分辨率上的。降低分辨率将增大纹素尺寸，也会导致光圈和焦外效果增大。在此教程剩下部分，我将使用半分辨率以让纹素信息更容易被看见。这会导致焦外效果的图形被放大了一倍。

![](/img/in-post/depth-of-field/12.png)
<small class="img-hint">半分辨率下的同一场景</small>

对9x9的一个纹素块进行采样需要进行81次，已经够多了。如果我想再放大一倍焦外效果的尺寸，我需要采样17x17=289次，这就是不能接受的。我们可以通过扩大采样间距而不是增加采样次数，来达成此目的。
```
					float2 o = float2(u, v) * _MainTex_TexelSize.xy * 2;
```

![](/img/in-post/depth-of-field/13.png)
<small class="img-hint">稀疏的采样</small>

我们现在是增大了光圈的半径，但是却没有足够的采样去覆盖整个形状。降采样的结果破坏了光圈投影，将其转换为了点阵。这就像我们使用了一个9x9个孔的光圈。这看起来不咋地，但却清晰地展示出来了每一个采样。

##### 2 圆形焦外模糊
理想的光圈是圆形而不是方形的，这会产生一个多次叠加的饼状焦外模糊。我们可以简单地通过距离中心点的距离来判断像素是否在圆形圆形模糊内。有多少次采样决定了累积颜色的权重，我们可以用此对其进行归一化操作。
```
					half3 color = 0;
					float weight = 0;
					for (int u = -4; u <= 4; u++) {
						for (int v = -4; v <= 4; v++) {
//							float2 o = float2(u, v) * _MainTex_TexelSize.xy * 2;
							float2 o = float2(u, v);
							if (length(o) <= 4) {
								o *= _MainTex_TexelSize.xy * 2;
								color += tex2D(_MainTex, i.uv + o).rgb;
								weight += 1;
							}
						}
					}
					color *= 1.0 / weight;
					return half4(color, 1);
```

![](/img/in-post/depth-of-field/14.png)
<small class="img-hint">圆形焦外效果</small>

我们还可以通过一个数组来定义各偏移然后对其进行循环采样。这也意味着我们不用拘泥于规则的形状。既然我们是对圆形采样，可以用螺旋状或同心圆。让我们使用Unity中post effect stack第二版定义在DiskKernels.hlsl中的采样卷积核。那些卷积核包含一个单位圆中的偏移，最小的卷积核包含16次采样：1个中心点，5个采样点围绕其周围，以及更外围的10个采样点圆周。
```
				// From https://github.com/Unity-Technologies/PostProcessing/
				// blob/v2/PostProcessing/Shaders/Builtins/DiskKernels.hlsl
				static const int kernelSampleCount = 16;
				static const float2 kernel[kernelSampleCount] = {
					float2(0, 0),
					float2(0.54545456, 0),
					float2(0.16855472, 0.5187581),
					float2(-0.44128203, 0.3206101),
					float2(-0.44128197, -0.3206102),
					float2(0.1685548, -0.5187581),
					float2(1, 0),
					float2(0.809017, 0.58778524),
					float2(0.30901697, 0.95105654),
					float2(-0.30901703, 0.9510565),
					float2(-0.80901706, 0.5877852),
					float2(-1, 0),
					float2(-0.80901694, -0.58778536),
					float2(-0.30901664, -0.9510566),
					float2(0.30901712, -0.9510565),
					float2(0.80901694, -0.5877853),
				};

				half4 FragmentProgram (Interpolators i) : SV_Target {
					…
				}
```

循环偏移累积采样，为了保持和之前一样的采样半径，这里需要对偏移乘以8。
```
				half4 FragmentProgram (Interpolators i) : SV_Target {
					half3 color = 0;
//					float weight = 0;
//					for (int u = -4; u <= 4; u++) {
//						for (int v = -4; v <= 4; v++) {
//							…
//						}
//					}
					for (int k = 0; k < kernelSampleCount; k++) {
						float2 o = kernel[k];
						o *= _MainTex_TexelSize.xy * 8;
						color += tex2D(_MainTex, i.uv + o).rgb;
					}
					color *= 1.0 / kernelSampleCount;
					return half4(color, 1);
				}
```

![](/img/in-post/depth-of-field/15.png)
<small class="img-hint">使用小卷积核</small>

卷积核越大，则质量越高。DiskKernels.hlsl中定义了几个，你可以将其复制过来并进行比较。在此课程中，我会使用中度卷积核：也是两个环，但会进行22次采样。

```
#define BOKEH_KERNEL_MEDIUM

				// From https://github.com/Unity-Technologies/PostProcessing/
				// blob/v2/PostProcessing/Shaders/Builtins/DiskKernels.hlsl
				#if defined(BOKEH_KERNEL_SMALL)
					static const int kernelSampleCount = 16;
					static const float2 kernel[kernelSampleCount] = {
						…
					};
				#elif defined (BOKEH_KERNEL_MEDIUM)
					static const int kernelSampleCount = 22;
					static const float2 kernel[kernelSampleCount] = {
						float2(0, 0),
						float2(0.53333336, 0),
						float2(0.3325279, 0.4169768),
						float2(-0.11867785, 0.5199616),
						float2(-0.48051673, 0.2314047),
						float2(-0.48051673, -0.23140468),
						float2(-0.11867763, -0.51996166),
						float2(0.33252785, -0.4169769),
						float2(1, 0),
						float2(0.90096885, 0.43388376),
						float2(0.6234898, 0.7818315),
						float2(0.22252098, 0.9749279),
						float2(-0.22252095, 0.9749279),
						float2(-0.62349, 0.7818314),
						float2(-0.90096885, 0.43388382),
						float2(-1, 0),
						float2(-0.90096885, -0.43388376),
						float2(-0.6234896, -0.7818316),
						float2(-0.22252055, -0.974928),
						float2(0.2225215, -0.9749278),
						float2(0.6234897, -0.7818316),
						float2(0.90096885, -0.43388376),
					};
				#endif
```
![](/img/in-post/depth-of-field/16.png)
<small class="img-hint">使用中卷积核</small>

这给了我们更高质量的卷积核，依然较明显地区分两个采样环，所以我们能看到我们的效果是如何起作用的。

##### 模糊焦外效果
虽然使用特定的采样卷积核要比使用规则的网格好，但这仍然需要许多次采样才能得到一个不错的效果。为了让采样次数覆盖更多的区域，我们可以降低分辨率，就像bloom效果一样。这虽然会使焦外效果变得模糊，但这个代价是值得的。   
为使用半分辨率，我们需要将其blit至一个半大小的纹理，在此分辨上创建焦外效果，接着blit至全分辨上。这需要两个额外的临时纹理。  

```
		int width = source.width / 2;
		int height = source.height / 2;
		RenderTextureFormat format = source.format;
		RenderTexture dof0 = RenderTexture.GetTemporary(width, height, 0, format);
		RenderTexture dof1 = RenderTexture.GetTemporary(width, height, 0, format);

		Graphics.Blit(source, coc, dofMaterial, circleOfConfusionPass);
		Graphics.Blit(source, dof0);
		Graphics.Blit(dof0, dof1, dofMaterial, bokehPass);
		Graphics.Blit(dof1, destination);
//		Graphics.Blit(source, destination, dofMaterial, bokehPass);

		RenderTexture.ReleaseTemporary(coc);
		RenderTexture.ReleaseTemporary(dof0);
		RenderTexture.ReleaseTemporary(dof1);
```
为了保持同样的焦外效果大小，我们需要将采样偏移减半。

```
						o *= _MainTex_TexelSize.xy * 4;
```

![](/img/in-post/depth-of-field/17.png)
<small class="img-hint">半分辨率的焦外效果</small>

此时焦外效果更实了，但是采样间依然没有完全连接起来。我们需要缩小其大小或者让其更模糊一些。在这里我们不再降采样，而是在创建焦外效果之后添加一个额外的模糊pass，所以其是一个后处理pass。
```
	const int circleOfConfusionPass = 0;
	const int bokehPass = 1;
	const int postFilterPass = 2;
```

我们用半分辨率执行此后处理pass，这样我们就可以复用第一个半大小的纹理了。
```
		Graphics.Blit(source, coc, dofMaterial, circleOfConfusionPass);
		Graphics.Blit(source, dof0);
		Graphics.Blit(dof0, dof1, dofMaterial, bokehPass);
		Graphics.Blit(dof1, dof0, dofMaterial, postFilterPass);
		Graphics.Blit(dof0, destination);
```
此后处理pass会在同分辨率下执行一个小高斯模糊，这是通过一个半纹素偏移的方形滤波器来实现的。这会导致重叠采样，创建一个3x3被称为帐篷滤波（tent filter）的卷积核。

![](/img/in-post/depth-of-field/18-1.png)
![](/img/in-post/depth-of-field/18-2.png)
<small class="img-hint">3x3 帐篷滤波</small>

```
		Pass { // 2 postFilterPass
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					float4 o = _MainTex_TexelSize.xyxy * float2(-0.5, 0.5).xxyy;
					half4 s =
						tex2D(_MainTex, i.uv + o.xy) +
						tex2D(_MainTex, i.uv + o.zy) +
						tex2D(_MainTex, i.uv + o.xw) +
						tex2D(_MainTex, i.uv + o.zw);
					return s * 0.25;
				}
			ENDCG
		}
```

![](/img/in-post/depth-of-field/19.png)
<small class="img-hint">帐篷滤波后的效果</small>

##### 4 焦外效果大小
有了模糊后处理pass，我们的焦外效果在半径为4半分辨率纹素下终于看得过去了。此效果不是完全光滑的，有些像镜头被弄脏了。这是因为非常亮的投影被叠加在暗背景之上，且我们目前极大地夸张了此效果。但你也许想要通过降低尺寸来获取一个更高质的焦外效果，或者提高尺寸来得到一个低至的效果。接下来让我们将焦外效果的半径设置为可配置：1-10的一个范围，默认值为4，代表半分辨率纹素。我们不应该将半径设置为小于1，因为这将是一个降采样的模糊效果，而不是一个焦外效果。
```
	[Range(1f, 10f)]
	public float bokehRadius = 4f;
```

![](/img/in-post/depth-of-field/20.png)
<small class="img-hint">焦外效果半径的滚动条</small>

将焦外效果半径传递给shader。
```
		dofMaterial.SetFloat("_BokehRadius", bokehRadius);
		dofMaterial.SetFloat("_FocusDistance", focusDistance);
		dofMaterial.SetFloat("_FocusRange", focusRange);
```

为shader添加一个float参数，用于纹理采样。
```
		float _BokehRadius, _FocusDistance, _FocusRange;
```
最后，使用可配置的半径取代之前的固定值4。
```
						float2 o = kernel[k];
						o *= _MainTex_TexelSize.xy * _BokehRadius;
```
![](/img/in-post/depth-of-field/21.gif)
<small class="img-hint">可配置的焦外效果半径</small>

### 四 对焦
目前为止我们能决定模糊圈的大小，同时我们能创建最大尺寸的焦外效果了。下一步就是将两者结合起来实现一个可变的焦外效果，去模拟摄像机的对焦。

##### 1 降采样模糊圈
因为我们使用了半分辨率创建焦外效果，所以我们的模糊圈效果也需要在半分辨率。默认的blit或者纹理采样会简单地对邻近纹素取均值，这对于深度信息来说没啥意义。所以，我们需要自己通过一个前处理pass来降采样。

```
	const int circleOfConfusionPass = 0;
	const int preFilterPass = 1;
	const int bokehPass = 2;
	const int postFilterPass = 3;
```

除了需要读取source纹理之外，此前处理pass还需要读取CoC纹理。所以在降采样之前，将CoC纹理传递给它。
```
		dofMaterial.SetTexture("_CoCTex", coc);

		Graphics.Blit(source, coc, dofMaterial, circleOfConfusionPass);
		Graphics.Blit(source, dof0, dofMaterial, preFilterPass);
		Graphics.Blit(dof0, dof1, dofMaterial, bokehPass);
		Graphics.Blit(dof1, dof0, dofMaterial, postFilterPass);
		Graphics.Blit(dof0, destination);
```

同时，在shader中添加相应的变量。
```
		sampler2D _MainTex, _CameraDepthTexture, _CoCTex;
```

接下来，创建一个新的pass来执行降采样。此颜色值可以从一个单通道纹理采样。从高分辨率纹素中采样然后平均到低分配率中，接着将此值存储在alpha通道中。

```
		Pass { // 1 preFilterPass
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					float4 o = _MainTex_TexelSize.xyxy * float2(-0.5, 0.5).xxyy;
					half coc0 = tex2D(_CoCTex, i.uv + o.xy).r;
					half coc1 = tex2D(_CoCTex, i.uv + o.zy).r;
					half coc2 = tex2D(_CoCTex, i.uv + o.xw).r;
					half coc3 = tex2D(_CoCTex, i.uv + o.zw).r;
					
					half coc = (coc0 + coc1 + coc2 + coc3) * 0.25;

					return half4(tex2D(_MainTex, i.uv).rgb, coc);
				}
			ENDCG
		}

		Pass { // 2 bokehPass
			…
		}

		Pass { // 3 postFilterPass
			…
		}
```

以上为一个常规的降采样，但这并非我们需要的。相反，我们仅取邻近四纹素中最显著的CoC值，正值或者负值。
```
//					half coc = (coc0 + coc1 + coc2 + coc3) * 0.25;
					half cocMin = min(min(min(coc0, coc1), coc2), coc3);
					half cocMax = max(max(max(coc0, coc1), coc2), coc3);
					half coc = cocMax >= -cocMin ? cocMax : cocMin;
```

##### 2 使用正确的模糊圈
为正确使用CoC半径，我们需要将CoC值乘以焦外效果半径。
```
					float coc = (depth - _FocusDistance) / _FocusRange;
					coc = clamp(coc, -1, 1) * _BokehRadius;
```
要知道一个卷积核采样是否对该像素的焦外效果有所贡献，我们需要检查此采样的CoC是否和该像素交叠。我们需要知道此采样的卷积核半径：就是其偏移值的长度。我们测量的单位是纹素，所以必须在乘以纹素值前进行操作。
```
					for (int k = 0; k < kernelSampleCount; k++) {
						float2 o = kernel[k] * _BokehRadius;
//						o *= _MainTex_TexelSize.xy * _BokehRadius;
						half radius = length(o);
						o *= _MainTex_TexelSize.xy;
						color += tex2D(_MainTex, i.uv + o).rgb;
					}
```
如果此采样的CoC值至少和卷积核的半径一样大，则认为此点是和该像素交叠的。否则，此点不应该影响此像素，而被忽略。这意味着我们又得记录累积颜色的权重以便后面归一化它。
```
					half3 color = 0;
					half weight = 0;
					for (int k = 0; k < kernelSampleCount; k++) {
						float2 o = kernel[k] * _BokehRadius;
						half radius = length(o);
						o *= _MainTex_TexelSize.xy;
//						color += tex2D(_MainTex, i.uv + o).rgb;
						half4 s = tex2D(_MainTex, i.uv + o);

						if (abs(s.a) >= radius) {
							color += s.rgb;
							weight += 1;
						}
					}
					color *= 1.0 / weight;
```

![](/img/in-post/depth-of-field/22.png)
<small class="img-hint">基于CoC的焦外效果</small>

##### 3 使采样更平滑
现在，我们的图片包含各个半径的焦外圆环效果，但两种半径效果的过渡是突变的。为了更好地查看此问题，你需要增大焦外效果半径，在能方便地看到单个采样之后调整焦点区域范围或者距离。

![](/img/in-post/depth-of-field/23.gif)
<small class="img-hint">某些采样按照CoC值排除在外了</small>

卷积核同一个环上的采样所得到的CoC值大体上是一样的，这意味着要么舍弃、要么同时包含。这样会导致一般只有三种显示效果：没有环，一个环或者两个环。且对两个卷积核进行采样的像素也会被忽略，因为其卷积核半径已经略微大于1了。  
有个办法能同时解决这两个问题：灵活修改是否包含一次采样的规则。不再只是简单的取舍，而是为其赋一个0到1的权重。此权重基于CoC值和offset半径，可以将其提炼为一个单独的函数。
```
				half Weigh (half coc, half radius) {
					return coc >= radius;
				}

				half4 FragmentProgram (Interpolators i) : SV_Target {
					half3 color = 0;
					half weight = 0;
					for (int k = 0; k < kernelSampleCount; k++) {
						…

//						if (abs(s.a) >= radius) {
//							color += s;
//							weight += 1;
//						}
						half sw = Weigh(abs(s.a), radius);
						color += s.rgb * sw;
						weight += sw;
					}
					color *= 1.0 / weight;
					return half4(color, 1);
				}
```
而对于权重函数，我们可以使用CoC值减去半径，然后限制在0到1的范围内。加入一个小的值，然后除以它，我们引入了一个偏移且让其斜率变抖。结果是一个更平滑的过渡且所有的卷积核都被包含在期间了。
```
				half Weigh (half coc, half radius) {
//					return coc >= radius;
					return saturate((coc - radius + 2) / 2);
				}
```

![](/img/in-post/depth-of-field/24.gif)
<small class="img-hint">更平滑的采样</small>

##### 4 保持对焦
半分辨率的一个缺点就是整个图都被降采样了，这会导致一个低程度的模糊。但是在焦点被的像素不应该被模糊化。为确保这些像素锐利，我们需要将半分辨率的效果呵全分辨率的原始图结合起来：通过CoC值来混合它们。  
新建一个pass，以source为输入，同时也需要读取深度图，所以将此也传给shader。
```
	const int circleOfConfusionPass = 0;
	const int preFilterPass = 1;
	const int bokehPass = 2;
	const int postFilterPass = 3;
	const int combinePass = 4;

	…

	void OnRenderImage (RenderTexture source, RenderTexture destination) {
		…

		dofMaterial.SetTexture("_CoCTex", coc);
		dofMaterial.SetTexture("_DoFTex", dof0);

		Graphics.Blit(source, coc, dofMaterial, circleOfConfusionPass);
		Graphics.Blit(source, dof0, dofMaterial, preFilterPass);
		Graphics.Blit(dof0, dof1, dofMaterial, bokehPass);
		Graphics.Blit(dof1, dof0, dofMaterial, postFilterPass);
		Graphics.Blit(source, destination, dofMaterial, combinePass);
//		Graphics.Blit(dof0, destination);

		…
	}
```
在shader中添加相应的变量。
```
		sampler2D _MainTex, _CameraDepthTexture, _CoCTex, _DoFTex;
```

同时创建新的pass，一开始仅执行一个简单的复制source功能。
```
		Pass { // 4 combinePass
			CGPROGRAM
				#pragma vertex VertexProgram
				#pragma fragment FragmentProgram

				half4 FragmentProgram (Interpolators i) : SV_Target {
					half4 source = tex2D(_MainTex, i.uv);

					half3 color = source.rgb;
					return half4(color, source.a);
				}
			ENDCG
		}
```
source纹理的颜色都是在焦点区域内的。DoF纹理包含了焦点内和焦点外的像素，但即使是焦点内的像素也是模糊的。所以，我们必须用到source纹理。我们不能突然地切换纹理，所以需要混合。假设CoC值小于0.1的为完全清晰而应该使用source纹理。而CoC值大于1的为完全模糊，需要使用DoF纹理，其他中间值则需要使用smoothstep函数来混合。
```
					half4 source = tex2D(_MainTex, i.uv);
					half coc = tex2D(_CoCTex, i.uv).r;
					half4 dof = tex2D(_DoFTex, i.uv);

					half dofStrength = smoothstep(0.1, 1, abs(coc));
					half3 color = lerp(source.rgb, dof.rgb, dofStrength);
					return half4(color, source.a );
```

![](/img/in-post/depth-of-field/25.png)
<small class="img-hint">焦点区域锐利</small>

##### 5 分开前景 & 后景
不幸的是，当一个焦外前景在一个焦点背景的时候，和source纹理进行混合会产生错误的结果。这之所以会发生是因为模糊前景应该在背景之上，但因为我们依照背景的CoC值来选择了source纹理，所以消除了此效果。为了解决此问题，我们得区分前景以及后景。  
让我们从一个仅包含后景的卷积核采样开始，后景的CoC是正的。能通过在采样的时候使用0和CoC的最大值，而非CoC的绝对值来分配权重。因为这可能会导致没有采样，所以需要确保除数不为0，如当权重为0的时候，加1。
```
				half4 FragmentProgram (Interpolators i) : SV_Target {
					half3 bgColor = 0;
					half bgWeight = 0;
					for (int k = 0; k < kernelSampleCount; k++) {
						float2 o = kernel[k] * _BokehRadius;
						half radius = length(o);
						o *= _MainTex_TexelSize.xy;
						half4 s = tex2D(_MainTex, i.uv + o);

						half bgw = Weigh(max(0, s.a), radius);
						bgColor += s.rgb * bgw;
						bgWeight += bgw;
					}
					bgColor *= 1 / (bgWeight + (bgWeight == 0));
					
					half3 color = bgColor;
					return half4(color, 1);
				}
```
![](/img/in-post/depth-of-field/26.png)
<small class="img-hint">仅后景焦外模糊，依然和source纹理混合</small>

但现在我们能看见部分背景覆盖了前景，这是不应该发生的，因为前景挡着后面呢。这能通过取采样的CoC值和片段的CoC值来消除此不良效果。
```
					half coc = tex2D(_MainTex, i.uv).a;
					
					half3 bgColor = 0;
					half bgWeight = 0;
					for (int k = 0; k < kernelSampleCount; k++) {
						…

						half bgw = Weigh(max(0, min(s.a, coc)), radius);
						bgColor += s.rgb * bgw;
						bgWeight += bgw;
					}
```

![](/img/in-post/depth-of-field/27.png)
<small class="img-hint">不再覆盖前景</small>

接下来，用同样的方法记录前景的颜色。因为前景的权重是基于为负值的CoC值，所以需要对其取反。

```
					half3 bgColor = 0, fgColor = 0;
					half bgWeight = 0, fgWeight = 0;
					for (int k = 0; k < kernelSampleCount; k++) {
						float2 o = kernel[k] * _BokehRadius;
						half radius = length(o);
						o *= _MainTex_TexelSize.xy;
						half4 s = tex2D(_MainTex, i.uv + o);

						half bgw = Weigh(max(0, s.a), radius);
						bgColor += s.rgb * bgw;
						bgWeight += bgw;

						half fgw = Weigh(-s.a, radius);
						fgColor += s.rgb * fgw;
						fgWeight += fgw;
					}
					bgColor *= 1 / (bgWeight + (bgWeight == 0));
					fgColor *= 1 / (fgWeight + (fgWeight == 0));
					half3 color = fgColor;
```

![](/img/in-post/depth-of-field/28.png)
<small class="img-hint">仅前景焦外效果</small>

##### 6 重组前景 & 后景
我们将把前景和后景放在一个缓冲中。因为前景在背景的前面，所以当至少有一个前景采样的时候我们将使用前景。我们能通过前景的权重来插值后景和前景颜色，插值最大为1。
```
					fgColor *= 1 / (fgWeight + (fgWeight == 0));
					half bgfg = min(1, fgWeight);
					half3 color = lerp(bgColor, fgColor, bgfg);
```

![](/img/in-post/depth-of-field/29.png)
<small class="img-hint">重组</small>

为修正之前混合source纹理的问题，我们需要修改混合前景时候的方式。combine pass需要知道前景和后景是如何混合的，使用 bgfg来进行插值，将其放在DoF纹理的alpha通道中。

```
					half bgfg = min(1, fgWeight);
					half3 color = lerp(bgColor, fgColor, bgfg);
					return half4(color, bgfg);
```

在combine pass中，我们首先需要对CoC正值进行插值，用于获取后景。接着利用前景权重对DoF进行插值。结果是一个source和DoF间的一个非线性插值，使得DoF值略微变大。
```
//					half dofStrength = smoothstep(0.1, 1, abs(coc));
					half dofStrength = smoothstep(0.1, 1, coc);
					half3 color = lerp(
						source.rgb, dof.rgb,
						dofStrength + dof.a - dofStrength * dof.a
					);
```

![](/img/in-post/depth-of-field/30.png)
<small class="img-hint">保留了前景，但是失焦了</small>

此时前景重回图片中，但是焦点区域却被擦掉了。这是因为就算是只有一个卷积核采样属于前景，其强度也是最大的。为了让其按比例缩放，需要让bgfg除以采样的总数
```
					half bgfg = min(1, fgWeight / kernelSampleCount);
```

![](/img/in-post/depth-of-field/31.png)
<small class="img-hint">按比例缩放前景</small>

这使得前景又太弱了，且在物体的边缘出现了错误的表现。我们需要用一个因子来重新增强前景。既然我们使用的是圆形焦外效果，就用π吧。这会让前景比分离前更亮，但看起来还不错。如果你觉得此效果太强或者太弱了，可以尝试其他值。
```
					half bgfg =
						min(1, fgWeight * 3.14159265359 / kernelSampleCount);
```

![](/img/in-post/depth-of-field/32.png)
<small class="img-hint">前景得到了增强</small>

然而在前景的边缘，依然有一些错误。尤其是在平面（plane）的左下角很明显。这是因为在前景和很远的后景之间产生了一个突变的渐变。以致很大部分的卷积核采样处于后景，而削弱了前景的影响。为了解决此问题，我们需要在combine pass中重新启用CoC的绝对值。

```
					half dofStrength = smoothstep(0.1, 1, abs(coc));
```

![](/img/in-post/depth-of-field/33.png)
<small class="img-hint">前景边缘不再有错误效果</small>

##### 7 调暗焦外效果
再将焦外效果调暗，我们就能大功告成啦！此过程不应该过多地改变图片的总亮度。我们能通过在降采样的前处理pass中对颜色进行一个权重平均，而不仅是取四纹素的颜色均值来达成效果。我们将对每一个颜色按此公式进行平均：1 / (1 + max(max(c.r, c.g), c.b))。

```
				half Weigh (half3 c) {
					return 1 / (1 + max(max(c.r, c.g), c.b));
				}

				half4 FragmentProgram (Interpolators i) : SV_Target {
					float4 o = _MainTex_TexelSize.xyxy * float2(-0.5, 0.5).xxyy;

					half3 s0 = tex2D(_MainTex, i.uv + o.xy).rgb;
					half3 s1 = tex2D(_MainTex, i.uv + o.zy).rgb;
					half3 s2 = tex2D(_MainTex, i.uv + o.xw).rgb;
					half3 s3 = tex2D(_MainTex, i.uv + o.zw).rgb;

					half w0 = Weigh(s0);
					half w1 = Weigh(s1);
					half w2 = Weigh(s2);
					half w3 = Weigh(s3);

					half3 color = s0 * w0 + s1 * w1 + s2 * w2 + s3 * w3;
					color /= max(w0 + w1 + w2 + s3, 0.00001);

					half coc0 = tex2D(_CoCTex, i.uv + o.xy).r;
					…
					half coc = cocMax >= -cocMin ? cocMax : cocMin;

//					return half4(tex2D(_MainTex, i.uv).rgb, coc);
					return half4(color, coc);
				}
```

![](/img/in-post/depth-of-field/34.png)
<small class="img-hint">焦外效果变暗了</small>

现在你就拥有了一个和Unity的post effect stack 第二版中粗略差不多的景深效果啦。此效果还有许多微调的空间，或者你可以用知其所以然的知识去调整Unity中的景深效果。

































