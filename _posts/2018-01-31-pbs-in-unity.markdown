---
layout:     post
title:      "Unity对PBS的应用"
subtitle:   ""
date:       2018-01-31 12:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

本文旨在梳理Unity中PBS的应用流程：首先介绍了何为PBS，它相对传统渲染的优势，以及Unity对PBS的实现，最后以一个Demo为例简要说明了Unity中PBS的具体使用。

# 一 什么是PBS
---
从字面意思来看，是基于物理的渲染技术（Physically Based Shading）。其实可以将其理解为一种光照模型，类似Lambert 或 Phong，只是更贴合现实，因为其高消耗，在最近的实时渲染中才成为可能。  
PBS能实现更好的法线高光效果：

![](/img/in-post/pbs-in-unity/1.png)
<small class="img-hint"></small>

另一主要优势在于对用户更友好：使用一个Shader就能实现各种材质效果，且变换环境也无需频繁修改Shader参数，只需修改光照环境即可：

![](/img/in-post/pbs-in-unity/1.gif)
<small class="img-hint"></small>

# 二 Unity对PBS的实现
---
自Unity5开始，官方的Standard Shader就吸收了PBS光照模型，其是一个众多属性的集合，通过设置贴图和参数，可以启用或者禁用其中的一些属性。  
具体使用可参见 [Unity Maunal](https://docs.unity3d.com/Manual/shader-StandardShader.html)

![](/img/in-post/pbs-in-unity/2.png)
<small class="img-hint"></small>

# 三 Unity中PBS的具体使用
---
![](/img/in-post/pbs-in-unity/3.png)
<small class="img-hint"></small>

接下来，以此简单场景为例，介绍Unity中PBS的使用，内容分为以下两部分：

### 1 具体物体
利用Unity提供的表格调节StandardShader的具体参数，决定了物体是什么样的材质（木头 还是 黄金？）：

![](/img/in-post/pbs-in-unity/4.jpg)
<small class="img-hint"></small>

![](/img/in-post/pbs-in-unity/5.png)
<small class="img-hint"></small>

### 2 光照环境
调整好物体的材质之后，还需要和整个场景交互。这相当于物体所处的上下文：环境不同，物体所呈现出来的方式也不尽相同，这也是贴合现实的。
而环境主要由以下几部分组成：

##### 1）天空盒子 & 平行光
设置场景的反射源为Skybox，同时调整平行光的颜色和方向与当前天空盒子相契合

![](/img/in-post/pbs-in-unity/6.png)
<small class="img-hint"></small>

![](/img/in-post/pbs-in-unity/7.png)
<small class="img-hint"></small>

##### 2）反射探针
在某些室内或者暗处，需要用反射探针来作为反射源，

![](/img/in-post/pbs-in-unity/8.png)
<small class="img-hint"></small>


##### 3）光探针
采集光照信息用于动态物体（如角色）

![](/img/in-post/pbs-in-unity/9.png)
<small class="img-hint"></small>

# 四 参考资料
---
> Demo源于[Viking Viilage](https://assetstore.unity.com/packages/essentials/tutorial-projects/viking-village-29140)  
> Viking Village相关[Blog](https://blogs.unity3d.com/cn/2015/02/18/working-with-physically-based-shading-a-practical-approach/)  
>Standard Shader的表格可以在[这里](https://assetstore.unity.com/packages/essentials/tutorial-projects/shader-calibration-scene-25422)找到














