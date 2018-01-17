---
layout:     post
title:      "深入理解线性空间与HDR"
subtitle:   ""
date:       2017-12-14 20:00:02
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

# 一 概览
---
随着基于物理的渲染技术（Physically Based Shading：PBS）的流行，线性空间也越来越常被提及。   
之前看过一些相关的介绍，但始终感觉有些懵懂，于是决定通过博客的形式将其梳理一下。  
本文主要解释了线性空间、伽马校正、HDR等基本概念及其对渲染的影响，并介绍了Unity中伽马空间与线性空间的差异。

# 二 线性空间
---
简单来讲，在线性空间内对颜色、光照等进行加减乘除等操作后能得到正确结果。  
而转换到非线性（伽马）空间，结果可能就是错误的：

![](/img/in-post/linear-space-and-hdr/pic1.png)
<small class="img-hint"></small>

既然非线性空间会导致错误的运算结果，有什么存在的理由呢？这就要说到图片编码中的伽马校正了

# 三 伽马校正
---

### 编码伽马
因为人眼对暗部的感知要远大于亮部，所以在有限的位数（一般为8位）限制下，用一半的像素来存储亮部区域未免有些浪费，而使用伽马编码则可以充分地利用存储空间：
![](/img/in-post/linear-space-and-hdr/pic2.png)
<small class="img-hint"></small>

![](/img/in-post/linear-space-and-hdr/pic3.png)
<small class="img-hint">γ = 0.45</small>

### 显示伽马
这样的图像直接显示肯定是会比原图要偏亮的，于是微软牵头提出了sRGB颜色空间标准，推荐显示器的显示伽马为2.2：因为0.45 x 2.2 ≈ 1，这就将之前的颜色还原了：
![](/img/in-post/linear-space-and-hdr/pic4.png)
<small class="img-hint"></small>

# 四 对渲染的影响
---
由于编码伽马和显示伽马的存在，导致游戏渲染中可能产生以下问题：
### 图像整体偏暗
因为显示伽马的存在，会导致图像整体偏暗，如0.5经过2.2的伽马校正之后就变成了约0.22
![](/img/in-post/linear-space-and-hdr/pic5.png)
<small class="img-hint"></small>

### 混合方面的影响
![](/img/in-post/linear-space-and-hdr/pic6.png)
<small class="img-hint"></small>

### 对纹理的错误操作
因为图像位数的关系，输入的图片很可能已经被编码，若直接对其进行操作，可能导致不正确的结果，具体过程请参看下节

# 五 Unity中的空间差异
---
Unity中同时支持两种空间（在Edit -> Project Settings -> Player -> Other Settings中设置）
### Unity中的伽马空间
采取放任的态度，不进行伽马校正，典型的过程如下：可以看到，一个x2的Shader操作却产生了x4.7的结果（你不过爆谁过爆 。。。）
![](/img/in-post/linear-space-and-hdr/pic7.png)
<small class="img-hint"></small>
### Unity中的线性空间
和伽马空间最大的不同是在输入和输出的时候都会进行伽马校正，保证操作都是在正确的空间（线性空间）内进行的：最终结果也是符合操作预期的
![](/img/in-post/linear-space-and-hdr/pic8.png)
<small class="img-hint"></small>
伽马如此麻烦，会多这么多步骤，啥时候能舍弃它呢？如果有一天我们的存储空间足够大：比如从8位上升到32位，也许就不需要它了。这就是接下来要说的HDR了

# 六 HDR
---
动态范围指最高和最低亮度的比值， 高动态范围（High Dynamic Range：HDR）则代表最亮处和最暗处有较高的比值，这是符合真实世界的：阳光 vs 星光
HDR的好处有：
- 存储位数更多，可以保留更高的颜色精度
- 可以存储>1的颜色值，后期再用Tonemapping映射，保留更多亮部细节
- 屏幕后期的控制力更强，如Bloom的阈值可以设置为>1，提取最亮部分

但事物皆有正反两面，HDR的缺陷有：
- 需要更大的存储空间，渲染速度也会下降
- 一些硬件并不支持HDR
- 一旦使用了HDR将无法利用硬件的抗锯齿功能（可通过屏幕后期来弥补）


