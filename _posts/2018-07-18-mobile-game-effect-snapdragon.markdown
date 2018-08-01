---
layout:     post
title:      "手游效果分析（Snapdragon篇）"
subtitle:   ""
date:       2018-07-18 21:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

之前写过一篇[基于Aderno Profiler的手游效果分析](https://ak-techartist.github.io/2017/12/23/mobile-game-revert/)，但这个工具高通早就没有维护了，在使用其分析游戏效果的时候也是错误百出。  
还好，失之东隅，收之桑榆：Snapdragon profiler在更新之后要好用许多，足够我们完成对游戏效果的分析了。  

本篇主要着眼于介绍工具本身，而不是具体的渲染技术，将以下面这个风动树为例来介绍：

![](/img/in-post/mobile-game-effect-snapdragon/1.gif)
<small class="img-hint">带碰撞的风动树</small>

# 一 连接手机

![](/img/in-post/mobile-game-effect-snapdragon/1.png)
<small class="img-hint">调出连接手机的窗口</small>

![](/img/in-post/mobile-game-effect-snapdragon/2.png)
<small class="img-hint">点击连接</small>

![](/img/in-post/mobile-game-effect-snapdragon/3.png)
<small class="img-hint">连接成功</small>

# 二 截取一帧

![](/img/in-post/mobile-game-effect-snapdragon/4.png)
<small class="img-hint">调出Snapshot窗口</small>

![](/img/in-post/mobile-game-effect-snapdragon/5.png)
<small class="img-hint">选中游戏，然后点击Take Snapshot</small>

![](/img/in-post/mobile-game-effect-snapdragon/6.png)
<small class="img-hint">截取完毕之后的界面</small>

# 三 定位Drawcall（以下简称DC）
定位到绘制树的DC：
![](/img/in-post/mobile-game-effect-snapdragon/7.png)
<small class="img-hint">风动树所绘制帧</small>

# 四 分析DC
通过分析贴图、Shader等就基本能还原其效果了，这里分享几个小技巧：

### 1 定位具体资源
勾选Used，将仅当前DC所用到的资源（贴图、Shader等）显示出来

![](/img/in-post/mobile-game-effect-snapdragon/8.png)
<small class="img-hint">该帧所用主要资源</small>

### 2 TEXCOORD0对应哪个贴图？
通过展开DC能看到TEXCOORD0、TEXCOORD1等分别对应哪个贴图

![](/img/in-post/mobile-game-effect-snapdragon/9.png)
<small class="img-hint">TEXCOORD0对应的是编号为96的贴图</small>

### 3 修改Shader代码
修改Shader代码并应用，能看到具体的效果

![](/img/in-post/mobile-game-effect-snapdragon/10.png)
<small class="img-hint">修改树的输出颜色为白色，并应用</small>

### 4 Program Inspector
通过ProgramInspector窗口，能看到该Shader所采用的全局变量具体的值，这在后续的效果还原中是相当重要的

![](/img/in-post/mobile-game-effect-snapdragon/11.png)
<small class="img-hint">可以看到具体的全局风向值</small>

# 五 获取资源
接下来需要获取游戏相关的资源：
### 1 贴图
贴图很好获取，直接在Profiler中保存即可：
![](/img/in-post/mobile-game-effect-snapdragon/12.png)
<small class="img-hint">保存贴图</small>

### 2 模型
通过NinjiaRipper可获取，具体操作可参见其[官网](http://cgig.ru/ninjaripper/)

# 六 在Unity中还原
最后利用获取到的资源，还原出来的效果（带碰撞）：

![](/img/in-post/mobile-game-effect-snapdragon/2.gif)
<small class="img-hint">最终效果</small>

































