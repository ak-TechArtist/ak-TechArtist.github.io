---
layout:     post
title:      "《游戏引擎架构》概览"
subtitle:   ""
date:       2018-06-09 15:30:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

真希望能早些看到此书：在介绍各个知识点之外，更重要的是将相关的知识点给整合了起来，形成了一种框架感。  
比如Unity中的Lightmap，如果知道它是渲染引擎中资产调节阶段的一部分，就能和整个渲染流程产生联系了，而整个渲染引擎其实也是游戏引擎的一个子系统。  
我认为，这也是作者写这本书的目的所在吧！  

而此文的主要目的在于试图理解作者的框架、意图，作为总结、复习之用。    
接下来，按照作者的架构顺序，做一个简略的概览：

![](/img/in-post/game-engine-architecture/1.png)
<small class="img-hint">十五章内容</small>

# 一 基础

![](/img/in-post/game-engine-architecture/2.png)
<small class="img-hint">第一部分：基础</small>

作者首先介绍了游戏团队典型的结构：程序、美术、策划，接着阐释了游戏是什么，以及游戏引擎的历史。同时，游戏类型不同，则引擎不尽相同，如虚幻引擎一开始就是供FPS游戏使用的。

游戏引擎无疑是一套大型软件系统，下面两幅图便可见一斑：

![](/img/in-post/game-engine-architecture/3.png)
<small class="img-hint">运行时游戏架构</small>

![工具及资产管理通道](https://upload-images.jianshu.io/upload_images/9476896-cef170ce3b0d9774.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](/img/in-post/game-engine-architecture/4.png)
<small class="img-hint">工具及资产管理通道</small>

而全书接下来就是围绕这些内容展开的。  
而在步入正题之前，作者先介绍了必备的一些专业工具以及软件工程的基础，还有就是必不可少（让人头疼）的数学咯！  
数学这节除复习了许多点、矢量、矩阵等基础知识，终于弄明白了欧拉角的万向锁问题，可谓受益匪浅！

# 二 低阶引擎系统
接下来，作者从每个游戏都需要的一些底层支持系统入手：游戏支持系统、资源及文件系统、循环及实时模拟、人体学接口设备、调试及开发工具。

![](/img/in-post/game-engine-architecture/5.png)
<small class="img-hint">低阶引擎系统</small>

其中的方方面面，诸如内存管理或者资源管理，日志及追踪，都是游戏开发者会经常接触到的。

# 三 图形及动画
而第三部分就是个人最关心的了：渲染。  
本以为渲染引擎应该是引擎中最复杂的部分了吧，没想到动画系统、碰撞和刚体力学实现起来也是十分复杂的：这三个部分也分别占据了此书近100页的内容。

下面以自己看得最仔细的渲染引擎为例，来看作者的思路：

![](/img/in-post/game-engine-architecture/6.png)
<small class="img-hint">渲染引擎</small>

三维场景的渲染分为以下几个基本步骤：描述一个虚拟场景、定位一个虚拟摄像机、设置光源、描述场景中物体表面的视觉特性、着色方程。  
而在游戏引擎中，高级的渲染步骤是由名为管道的软件架构所实现的，其最大的优点在于非常适合并行化。  
下图描述了整个渲染管道中数据变换的过程：

![](/img/in-post/game-engine-architecture/7.png)
<small class="img-hint">渲染管道中数据的变换</small>

其中，制作模型、材质等在工具阶段进行，资产调节阶段则包含了导入资源、烘焙Lightmap等，而应用程序阶段则负责可见性判断、几何排序并提交图元，几何阶段处理各顶点，而光栅化阶段则是将三角形转换为片段并处理片段。

而光照算法对渲染逼真的场景是十分重要的，可以通过贴图信息来影响光照，也可以使用HDR保留更多的细节，阴影、AO则是全局光照的一部分内容，延时渲染则提供了另一种场景着色方法。

而在游戏中，除了渲染三维固体之外，还有涉及粒子效果、贴花、环境效果、后期处理等内容。

# 四 游戏性
在介绍完游戏引擎用到的技术之后，作者提醒我们：游戏的本质在于游戏性——玩儿游戏的整体体验，更具体一点就是游戏机制，在游戏性系统简介这章，作者讲了关卡、游戏编辑器、数据驱动等和游戏机制息息相关的部分。

![](/img/in-post/game-engine-architecture/8.png)
<small class="img-hint">第四部分内容</small>

而运行时游戏性基础系统，则唤起了之前作为Unity前端的种种回忆：C#及Lua脚本语言，事件驱动以及消息传递，MonoBehavior的Update()，动态内存池的管理，地图分块加载，任务系统等等等

# 五 总结
最后一章，作者补充了几个因为没有足够篇幅去讨论的引擎及游戏性系统：
![](/img/in-post/game-engine-architecture/9.png)
<small class="img-hint">其他内容</small>



































