---
layout:     post
title:      "各项异性头发效果"
subtitle:   ""
date:       2018-01-17 17:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

# 一 概览
---
本文讲解了各向异性头发效果的原理，Shader的具体实现，以及部分注意事项。  
社会你坤哥，人怂话又多，这次先上Demo： [Anisotropic Hair](https://github.com/ak-TechArtist/AnisotropicHair)

![](/img/in-post/Anisotropic-hair-effect/pic1.png)
<small class="img-hint"></small>

# 二 原理
---
### 1 面片模型
配合基本颜色贴图来模拟发丝的效果。

![](/img/in-post/Anisotropic-hair-effect/pic2.png)
<small class="img-hint"></small>

### 2 基本贴图
![](/img/in-post/Anisotropic-hair-effect/pic3.png)
<small class="img-hint"></small>

### 3 高光走向
Kajiya-Kay模型：利用切线（tangent）方向来计算高光强度。

![](/img/in-post/Anisotropic-hair-effect/pic4.png)
<small class="img-hint"></small>

###  4 两层高光
Marschner模型：两层高光，并为第二层高光添加噪点。

![](/img/in-post/Anisotropic-hair-effect/pic5.png)
<small class="img-hint"></small>

# 三 Shader实现
---
### 1 Diffuse Lighting
```
fixed4 albedo = tex2D(_MainTex, i.uv);
half3 diffuseColor = albedo.rgb * _MainColor.rgb;
```

### 2 两层高光
```
//获取头发高光
fixed StrandSpecular ( fixed3 T, fixed3 V, fixed3 L, fixed exponent)
{
	fixed3 H = normalize(L + V);
	fixed dotTH = dot(T, H);
	fixed sinTH = sqrt(1 - dotTH * dotTH);
	fixed dirAtten = smoothstep(-1, 0, dotTH);
	return dirAtten * pow(sinTH, exponent);
}
			
//沿着法线方向调整Tangent方向
fixed3 ShiftTangent ( fixed3 T, fixed3 N, fixed shift)
{
	return normalize(T + shift * N);
}

fixed3 spec = tex2D(_AnisoDir, i.uv).rgb;
//计算切线方向的偏移度
half shiftTex = spec.g;
half3 t1 = ShiftTangent(worldBinormal, worldNormal, _PrimaryShift + shiftTex);
half3 t2 = ShiftTangent(worldBinormal, worldNormal, _SecondaryShift + shiftTex);
//计算高光强度		
half3 spec1 = StrandSpecular(t1, worldViewDir, worldLightDir, _SpecularMultiplier)* _SpecularColor;
half3 spec2 = StrandSpecular(t2, worldViewDir, worldLightDir, _SpecularMultiplier2)* _SpecularColor2;

```
### 3 结合起来
```
fixed4 finalColor = 0;
finalColor.rgb = diffuseColor + spec1 * _Specular;//第一层高光
finalColor.rgb += spec2 * _SpecularColor2 * spec.b * _Specular;//第二层高光，spec.b用于添加噪点
finalColor.rgb *= _LightColor0.rgb;//受灯光影响
finalColor.a += albedo.a;
```
# 四 注意事项
---
### 1 半透明渲染次序错乱
头发挽起来的部位出现在了前面：

![](/img/in-post/Anisotropic-hair-effect/pic6.png)
<small class="img-hint"></small>

通过添加一个写入深度的Pass可解决此问题：
```
Pass
{
	ZWrite On //写入深度，被遮挡的像素将不能通过深度测试
	ColorMask 0 //不输出颜色
}
```

### 2 半透穿透
但随之而来却产生了另外一个问题：下层头发被剔除之后，上层头发直接和背景色进行了混合

![](/img/in-post/Anisotropic-hair-effect/pic7.png)
<small class="img-hint"></small>

除了调整贴图alpha值之外，还可在通过在新添加的Pass中添加一个基本的颜色来解决此问题：
```
half4 finalColor = half4(0, 0, 0, albedo.a);
finalColor.rgb += (albedo.rgb * _MainColor.rgb) * _LightColor0.rgb;
return finalColor;
```

### 3 高光方向
为确保头发的高光走向为沿着发根到发尖，需要设置模型的轴向为Y轴朝上（和Unity一致）
![](/img/in-post/Anisotropic-hair-effect/pic8.png)
<small class="img-hint"></small>

同时UV方向也要和模型方向一致
![](/img/in-post/Anisotropic-hair-effect/pic9.png)
<small class="img-hint"></small>














