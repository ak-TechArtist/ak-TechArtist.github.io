---
layout:     post
title:      "次世代美术制作流程"
subtitle:   ""
date:       2018-05-22 21:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

一直挂着技术美术的羊头，卖图形程序的狗肉，终于得空将次世代美术的制作流程捋了一遍，也算正经了一回。

先看一下最终的效果：

![](/img/in-post/next-gen-art/gif1.png)
<small class="img-hint">挥手</small>

实现此效果的步骤为：

- 在3dsMax中创建模型
- 为模型建立骨骼并蒙皮、调整蒙皮权重
- 制作骨骼动画
- 展UV
- 利用ZBrush雕刻高模细节
- 导入SubstancePainter中绘制贴图
- 将模型、贴图等导入引擎中查看效果

# 一 在3dsMax中创建模型
由简单的球形 + 圆柱体组成：

![](/img/in-post/next-gen-art/pic1.gif)
<small class="img-hint">attach之后的模型</small>

# 二 为模型建立骨骼并蒙皮、调整蒙皮权重

![](/img/in-post/next-gen-art/pic2.png)
<small class="img-hint">调整右手臂的蒙皮权重</small>

# 三 制作骨骼动画
旋转上臂K了两帧，循环播放，即为重复挥手

![](/img/in-post/next-gen-art/gif2.png)
<small class="img-hint">K了两帧</small>

# 四 展UV
由于我们只在身体上绘制字母，所以将此部分的面积放大，以便操作

![](/img/in-post/next-gen-art/pic3.gif)
<small class="img-hint">平铺各部分UV</small>

# 五 利用ZBrush雕刻高模细节
这里将模型细分之后，利用Standard画笔雕刻出三个字母

![](/img/in-post/next-gen-art/pic4.png)
<small class="img-hint">雕刻三个字母</small>

# 六 导入SubstancePainter中绘制贴图
首先使用低模新建一个工程，然后倒入ZB中雕刻的高模烘焙贴图

![](/img/in-post/next-gen-art/pic5.png)
<small class="img-hint">高模烘焙出来的法线贴图</small>

然后为模型添加材质 以及 为字母上色

![](/img/in-post/next-gen-art/pic6.png)
<small class="img-hint">绘制完成图</small>

导出相关贴图

![](/img/in-post/next-gen-art/pic7.png)
<small class="img-hint">导出的三张贴图</small>

# 七 将模型、贴图等导入引擎中查看效果
此处以Unity为例，新建一个Animator，选择相应的动画，并赋给Prefab。材质用默认的Standard Shader即可，指定之前的贴图，点击运行，就可以看到文章开头的效果啦！

![](/img/in-post/next-gen-art/pic8.png)
<small class="img-hint">Unity中的设置</small>



































