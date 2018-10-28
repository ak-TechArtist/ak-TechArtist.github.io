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

之前做过一个[地形效果](https://ak-techartist.github.io/2018/08/02/terrain-atlas/)，最近打包到真机上评估了一下性能，发现了不少有意思的东西，特发文总结、分享下。  

- 测试概览
- 测试结果
- 有趣发现

# 一 测试概览
### 1 测试目的
对比地形的两种渲染方式（Splash & Atlas）及它们的子模式所耗性能，并结合美术表现需求选择最合适的模式。

### 2 测试环境
简单地调整摄像机位置，让地形铺满整个屏幕，排除其他干扰。   
同时，提供按钮切换不同状态，且实时显示当前帧耗时和相应的帧率。

![](/img/in-post/mobile-performance-test-summary/1.png)
<small class="img-hint">真机截图</small>

其中第一部分为当前帧耗时，括号里为帧率(1000ms / 耗时) 。  
第二部分为切换分辨率：1920 x 1080， 3840 x 2160， 5760 x 3240。本测试主要以5760x3240为主：此时，被测手机跑不满帧。   
第三部分为两个模式（Splash & Atlas），两者的主要区别在于前者采样次数较多，而后者Shader计算更复杂（需从图集中找出具体的贴图）。  
第四部分对应子模式，Splash下有四层、五层、六层贴图混合，Atlas下有两层、三层混合（具体的混合算法差异很大）。

### 3 “涉案”手机
利用了以下三个手机进行测试：

![](/img/in-post/mobile-performance-test-summary/2.png)
<small class="img-hint">“涉案”三兄弟</small>

这三个手机还是很有代表性的，它们分别使用了华为、高通、苹果的上一代芯片：麒麟970、骁龙835、A10。

# 二 测试结果
发热降频之后录得的稳定数值：

![](/img/in-post/mobile-performance-test-summary/3.png)
<small class="img-hint">发热降频后稳定数值</small>

可以看到在Splash模式，也就是贴图采样更多的情况下，在Mi6和7P上，随着贴图的增加，性能损耗是线性增加的。而在P20Pro上则不然：5层时损耗大增，6层相比5层则损耗增加较少。  

在Atlas模式下，Mi6和7P上结果也是类似的：3层比2层消耗多约20%，2层Atlas比4层Splash损耗低，3层Atlas比5层Splash损耗略低。但是华为上即使是3层Atlas也比Splash4层的损耗低。

总结起来，从性能上来讲2层Atlas是最合适的，但考虑到效果可以使用3层Atlas，这也是比5层Splash损耗低的。再考虑到5层Splash并不能满足美术的创作需求，而6层Splash又面临Sampler最多16个的限制，最终选择了3层Atlas模式。

# 三 有趣发现
整个测试过程有许多意料之外的有趣发现：  
### 1 机型间性能对比
无论是Splash还是Atlas模式下，帧率都是7P 略> Mi6 远> P20Pro，验证了一般而言在性能上： 苹果 > 高通 > 华为。

### 2 不走寻常路的P20Pro
P20Pro对于采样次数的敏感程度要大于Shader计算，尤其体现在3层Atlas的性能损耗都比4层Spalsh低。同时，5层Splash相较4层Splash有一个剧变，但感觉过了某个阈值，采样次数对性能的损耗就那么不明显了，比如6层相较5层也就多了3.5%的损耗。    
这给我的测试带来了相当的麻烦，后面直接舍弃了P20Pro，而仅考虑Mi6和7P的数据了。

### 3 天下没有免费的午餐
以4层Splash为例，刚运行测试的时候，7P渲染一帧仅耗时32ms(Mi6为50ms，P20Pro为45ms)，乍看P20Pro性能相比Mi6还占优啊，但它很快就掉到了98ms，主动放弃得相当早，虽然帧率降低了，却很好地控制了发热。  
7P则一开始就发热较大，每帧耗时也是慢慢上升：32ms -> 35ms -> 36ms ->38ms，到了某个临界点后突变到57ms左右，之后就一直稳定。   
相比7P，Mi6则是发热略慢，虽然一开始每帧耗时较高，但也保持得较好，缓缓上升到61ms之后稳定了下来。  
总结起来，在面对高渲染压力时P20Pro早早地就“放弃治疗”了，7P挣扎了一段儿也“屈服”下来，Mi6虽然一开始损耗就不高，却是退步得最少的。    
这也体现在运行一段时间帧率稳定之后的手机温度上：7P 略> Mi6 远> P20Pro。  
看来世界上真没有免费的午餐：高性能总是伴随着高发热~



之前地形中的花草等细节模型是场景美术手动摆放的： 

![](/img/in-post/terrain-export-model/1.png)
<small class="img-hint">成片的花</small>

但这样摆放问题在于布置大片地形元素的时候略显麻烦。且每次编辑地形，如调整高低起伏后都需要重新摆放，不够灵活。  
不胜其烦的美术便提了个需求：能不能在UnityTerrain中刷地形细节，然后导出？批量操作的同时，地形细节也是贴合地形走势的，节省重新摆放的工作量。  

#一 TerrainDetails？
因为之前做过一版导出地形贴图细节（BillboardDetails）的功能，首先想到的便是改进此功能，使其支持导出细节模型。  

但经过研究，发现存在以下问题：  
1 模型细节本身并非所见即所得  
TerrainDetailMesh使用的是Unity自带的Shader，而场景内模型草希望使用自身的Shader，它们在颜色、运动方式等方面是有差异的：

![](/img/in-post/terrain-export-model/2.png)
<small class="img-hint">表现差异</small>

2 大小和方位等信息无法获取  
要想做到所见即所得，则需要获取当前模型的大小、朝向等信息，从而导出属性一致的模型。但这些信息TerrainData并没有找到公开的接口，无法获取。

哎，此路暂时不通，看来只有另辟蹊径了、、  

#二 TerrainTrees！
依稀记得T2M这样的插件有导出树的功能，那么将其以树的形式种下并导出是否可行？  
试着走了一遍流程，此路也不通？：  
刷出来的模型无法差异化，都是同样的大小和朝向：  
![](/img/in-post/terrain-export-model/3.png)
<small class="img-hint">大小和朝向都是一致的</small>

其次是用T2M插件导出来的模型大小、朝向和原来的不一致：  
![可以看到本来很小的花占满了整个屏幕，朝向也有变化](/img/in-post/terrain-export-model/4.png)
<small class="img-hint"></small>

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


































