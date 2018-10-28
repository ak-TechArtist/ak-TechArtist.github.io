---
layout:     post
title:      "在Unity地形中刷模型并导出"
subtitle:   ""
date:       2018-10-28 17:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

之前地形中的花草等细节模型是场景美术手动摆放的： 

![](/img/in-post/terrain-export-model/1.png)
<small class="img-hint">成片的花</small>

但这样摆放问题在于布置大片地形元素的时候略显麻烦。且每次编辑地形，如调整高低起伏后都需要重新摆放，不够灵活。  
不胜其烦的美术便提了个需求：能不能在UnityTerrain中刷地形细节，然后导出？批量操作的同时，地形细节也是贴合地形走势的，节省重新摆放的工作量。  

# 一 TerrainDetails？
因为之前做过一版导出地形贴图细节（BillboardDetails）的功能，首先想到的便是改进此功能，使其支持导出细节模型。  

但经过研究，发现存在以下问题：  
1 模型细节本身并非所见即所得  
TerrainDetailMesh使用的是Unity自带的Shader，而场景内模型草希望使用自身的Shader，它们在颜色、运动方式等方面是有差异的：

![](/img/in-post/terrain-export-model/2.png)
<small class="img-hint">表现差异</small>

2 大小和方位等信息无法获取  
要想做到所见即所得，则需要获取当前模型的大小、朝向等信息，从而导出属性一致的模型。但这些信息TerrainData并没有找到公开的接口，无法获取。

哎，此路暂时不通，看来只有另辟蹊径了、、  

# 二 TerrainTrees！
依稀记得T2M这样的插件有导出树的功能，那么将其以树的形式种下并导出是否可行？  
试着走了一遍流程，此路也不通？：  
刷出来的模型无法差异化，都是同样的大小和朝向：  
![](/img/in-post/terrain-export-model/3.png)
<small class="img-hint">大小和朝向都是一致的</small>

其次是用T2M插件导出来的模型大小、朝向和原来的不一致：  
![](/img/in-post/terrain-export-model/4.png)
<small class="img-hint">可以看到本来很小的花占满了整个屏幕，朝向也有变化</small>

用Unity自带的资源试试呢？  
不仅能体现差异性，通过T2M插件导出也是所见即所得的：  
![](/img/in-post/terrain-export-model/5.png)
<small class="img-hint">Standard资源中的树体现出了差异性</small>

![](/img/in-post/terrain-export-model/6.png)
<small class="img-hint">T2M插件导出所见即所得</small>

看来，是自己资源的问题，仔细和Standard资源对比，并尝试数次之后发现，需要新建一个空物体，为其添加LOD组件，并将需要刷的模型赋给LOD0：  

![](/img/in-post/terrain-export-model/7.png)
<small class="img-hint">空父物体及LOD组件</small>

再刷并导出即可：

![](/img/in-post/terrain-export-model/8.png)
<small class="img-hint">体现出了差异性</small>

![](/img/in-post/terrain-export-model/9.png)
<small class="img-hint">插件导出：所见即所得</small>

最终，一行代码没写，就解决了此问题，美滋滋~


































