---
layout:     post
title:      "渲染教程一 矩阵（译）"
subtitle:   ""
date:       2020-3-2 08:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

（阿坤注：此为CatlikeCoding[渲染教程](https://catlikecoding.com/unity/tutorials/rendering/)的第一部分内容，本人英语水平有限，如有疑问可参考[原文](https://catlikecoding.com/unity/tutorials/rendering/part-1/)
，或留言交流。）

- 创建一个正方形网格
- 支持缩放，位移，旋转
- 使用变换矩阵
- 创建简单的摄像机投影

这是渲染基础系列教程的第一课，将覆盖矩阵变换的内容。如果你还不了解网格的原理，可以先看一下网格基础系列，其开篇为[程序网格](https://catlikecoding.com/unity/tutorials/procedural-grid/)。本系列课程将探索网格转换为像素显示的过程。

本课使用的Unity版本为5.3.1
![](/img/in-post/rendering-matrices/1.png)
<small class="img-hint">调整空间中点的位置</small>
# 1 可视化空间
你现在已经知道网格是什么及它们如何在场景中定位了。但是这个定位到底是如何实现的？shader怎么知道将物体绘制在什么地方？当然，我们可以交给Unity的transform组件及shader实现，但若想对此过程有所掌控，则明白其原理是至关重要的。为理解得透彻，最好是由我们自己来实现。  
网格的移动、旋转和缩放是通过改变顶点位置实现的。这涉及到空间变换，为明白其中原理，我们先要让其变得可见。可通过创建一个3D点的网格来实现，网格的点可由任意Prefab充当。 

```
public class TransformationGrid : MonoBehaviour {

	public Transform prefab;

	public int gridResolution = 10;

	Transform[] grid;

	void Awake () {
		grid = new Transform[gridResolution * gridResolution * gridResolution];
		for (int i = 0, z = 0; z < gridResolution; z++) {
			for (int y = 0; y < gridResolution; y++) {
				for (int x = 0; x < gridResolution; x++, i++) {
					grid[i] = CreateGridPoint(x, y, z);
				}
			}
		}
	}
}
```
>问：为什么不使用粒子来可视化这些点？ 
答：当然可以，只是我觉得粒子系统更适合放在其他单独的主题里。

创建一个点的步骤可分为实例化预制件、设置坐标、赋值一个特殊的颜色。 

```
	Transform CreateGridPoint (int x, int y, int z) {
		Transform point = Instantiate<Transform>(prefab);
		point.localPosition = GetCoordinates(x, y, z);
		point.GetComponent<MeshRenderer>().material.color = new Color(
			(float)x / gridResolution,
			(float)y / gridResolution,
			(float)z / gridResolution
		);
		return point;
	}
```

正方体看起来很显眼，就组件一个正方体网格吧。设置其设置在原点。这样，变换（尤其是旋转和缩放）就和网格的中点相关了。
```
	Vector3 GetCoordinates (int x, int y, int z) {
		return new Vector3(
			x - (gridResolution - 1) * 0.5f,
			y - (gridResolution - 1) * 0.5f,
			z - (gridResolution - 1) * 0.5f
		);
	}
```
我将使用一个默认的cube预制体作为网格中的点，缩小一半大小，这样Cube间能有间隔。  
![cube 预制体](https://upload-images.jianshu.io/upload_images/9476896-4cd6f25e0badf80f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

创建一个空物体，添加脚本，并赋值cube预制体。运行游戏，一个中心在原点的正方体网格就出现啦。

| ![空物体](https://upload-images.jianshu.io/upload_images/9476896-2ef54144bca99185.png) | ![正方形网格](https://upload-images.jianshu.io/upload_images/9476896-25a32ab6371ac8b3.png) |
|:-|:-|

[unitypackage下载](https://catlikecoding.com/unity/tutorials/rendering/part-1/visualizing-space/visualizing-space.unitypackage)


# 2 变换
理想情况下，我们可以为这个网格添加任意数量的多种形式变换。但目前让我们仅局限于位移、旋转和缩放上。 
如果我们能为每种变换创建一个脚本，那么就可以按任意顺序和数量变换网格了。不过每种变化的细节不一样，所以需要一个具体的方法去变换每个网格中的点。 
创建一个所有变换的父类。这是一个无法直接使用的抽象类。同时添加Apply方法，使得子类可以自定义具体的实现。
```
using UnityEngine;

public abstract class Transformation : MonoBehaviour {

	public abstract Vector3 Apply (Vector3 point);
}
```
在添加了这些组件给我们的网格物体后，还需要能获取到它们，以实现对每个点的具体变换。在这里我们使用一个链表去存储这些变换的引用。
```
using UnityEngine;
using System.Collections.Generic;

public class TransformationGrid : MonoBehaviour {
    ...
    List<Transformation> transformations;
	void Awake () {
        ...
        transformations = new List<Transformation>();
	}
}
```
然后在update中循环整个网格的点进行变换。
```
	void Update () {
		GetComponents<Transformation>(transformations);
		for (int i = 0, z = 0; z < gridResolution; z++) {
			for (int y = 0; y < gridResolution; y++) {
				for (int x = 0; x < gridResolution; x++, i++) {
					grid[i].localPosition = TransformPoint(x, y, z);
				}
			}
		}
	}
```
>问：为何每帧都获取组件？ 
答：这可以使我们在运行时也能修改变换组件，并立即看到结果。

>问：为何使用链表而非数组？ 
答：最直接的GetComponents方法会直接返回一个该类型的数组。这意味着每次调用都会创建一个新的数组，而我们又在每帧调用。更合适的办法是使用一个链表参数，会将获取的组件放入链表中，而非创建一个新的数组。
在本例中这并非一个至关重要的问题，但如果经常获取组件的话，用链表作为参数是一个不错的习惯。

我们需要依据每个点的原始坐标进行变换，而不是基于点的当前坐标。这是因为这些点的位置已经被改变了，而我们又不想叠加这些变换。

```
	Vector3 TransformPoint (int x, int y, int z) {
		Vector3 coordinates = GetCoordinates(x, y, z);
		for (int i = 0; i < transformations.Count; i++) {
			coordinates = transformations[i].Apply(coordinates);
		}
		return coordinates;
	}
```

### 2.1 移动
首先要实现的是位移变换，它似乎是最简单的了。新建一个继承自Transformation的子类，添加一个用于位移的变量。
```
public class PositionTransformation : Transformation {

	public Vector3 position;
}
```
实现Apply方法：移动原始点的位置。
```
	public override Vector3 Apply (Vector3 point) {
		return point + position;
	}
```
给我们的网格物体添加移动组件，可以移动网格中每个点的位置。这样的变换发生在网格的物体坐标系中。
![](https://upload-images.jianshu.io/upload_images/9476896-b22b666fc70f8d31.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![位置变换](https://upload-images.jianshu.io/upload_images/9476896-78ffd57a6258943b.gif?imageMogr2/auto-orient/strip)


### 2.2 缩放
缩放和位移类似，不过是相乘，而非相加。
```
using UnityEngine;

public class ScaleTransformation : Transformation {

	public Vector3 scale;

	public override Vector3 Apply (Vector3 point) {
		point.x *= scale.x;
		point.y *= scale.y;
		point.z *= scale.z;
		return point;
	}
}
```
添加此组件，就可以缩放网格了。注意，我们仅是更改了网格点的位置，所以单个点的大小并不会变化。

![](https://upload-images.jianshu.io/upload_images/9476896-07020b2efacc0dea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![调整缩放](https://upload-images.jianshu.io/upload_images/9476896-36db82c61cc55cf3.gif?imageMogr2/auto-orient/strip)

尝试同时位移和缩放，你会发现缩放也会影响点的位置。这是因为我们是先位移再缩放的。而Unity则相反，调整两个组件的位置，让缩放位于位移上方。

![调整组件位置](https://upload-images.jianshu.io/upload_images/9476896-e508fa97990f4990.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.3 旋转
第三部分是旋转，这要比前两部分难一些。新建一个旋转组件，暂时仅返回点的位置。
```
using UnityEngine;

public class RotationTransformation : Transformation {

	public Vector3 rotation;

	public override Vector3 Apply (Vector3 point) {
		return point;
	}
}
```
旋转的原理是什么？让我们先从只绕Z轴旋转开始。绕一个轴旋转就像是滚动一个车轮。因为Unity采用的是左手坐标系，在朝向Z轴正方向看时，一个正数的旋转会让车轮逆时针运动。

![绕Z轴的二维旋转](https://upload-images.jianshu.io/upload_images/9476896-2d43a564f20bcca5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来想一下旋转时点的坐标发生了什么变化？最简单的方法是想象点都在一个半径为1的圆（单位圆）上。而X轴和Y轴与圆有两个交点，如果我们按90°旋转，则这两个交点的坐标将由0,1,-1这三个数组成。

![90°和180°旋转(1,0)与(0,1) ](https://upload-images.jianshu.io/upload_images/9476896-4fdc806e3d4c8d3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第一次旋转90°，(1,0)变成了(0,1)，接着是(-1,0)，然后是(0,-1)，最后变回(1,0)。
如果从(0,1)开始，则循环更早一步。从(0,1)到(-1,0)到(0,-1)到(1,0)，然后回到起始点。
所以，坐标在0,1,0,-1这几个数字上循环，他们只是拥有不同的起始数字罢了。

如果按45°旋转呢？那会增加XY平面对角线上的点。因为距离原点的距离没有改变，所以点坐标将取于（$\pm\sqrt{1/2} $,$\pm\sqrt{1/2} $）。这就扩展我们的循环为0，$\sqrt{1/2} $，1，$\sqrt{1/2} $，0，$-\sqrt{1/2} $，-1，$-\sqrt{1/2} $了。继续缩小旋转的度数幅度，就会得到一个正弦曲线。
![正弦和余弦](https://upload-images.jianshu.io/upload_images/9476896-0221fa9dbe1612cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在我们的例子中，正弦曲线对应了从(1,0)开始的y值变化，余弦曲线则对应其x值变化。这意味着我们可以重定义(1,0)为(cos z, sin z)。相应的，(0,1)可被(-sin z, cos z)替代。
让我们先算出绕Z轴旋转的正弦和余弦值。因为正弦和余弦函数需要提供弧度参数，所以需要先转换角度为弧度。

```
	public override Vector3 Apply (Vector3 point) {
		float radZ = rotation.z * Mathf.Deg2Rad;
		float sinZ = Mathf.Sin(radZ);
		float cosZ = Mathf.Cos(radZ);

		return point;
	}
```

>问：为什么是弧度？ 
答：和角度一样，弧度也可以度量旋转。在单位圆上，弧度和点在周长上移动的距离相符。一个圆的周长为2π乘以半径，所以1个弧度等于π/180角度。
注意，这里π的定义为圆周与直径的比率。

我们找到了旋转(1,0)和(0,1)的方法，很好。但如何旋转任意位置的点呢？既然这些点的定义与X和Y轴相关。就可以分解任意的二维点(x,y)为 xX + yY。这等于 x(1,0) + y(0,1)，也就是 (x,y)。在旋转时，我们可以用 x(cosZ, sinZ) + y(-sinZ, cosZ)来表示一个旋转之后的点。可以想象将点对应到单位圆上，旋转之后再对应回来即可，旋转后的坐标为(xcosZ - ysinZ, xsinZ + ycosZ)。
```
		return new Vector3(
			cosZ * point.x - sinZ * point.y,
			sinZ * point.x + cosZ * point.y,
			point.z
		);
```
添加一个旋转组件给网格物体，并放在变换中间。这意味着会先缩放，再旋转，最后位移，这和Unity的变换顺序是一样的。目前我们只能绕Z轴旋转，另外两个轴的旋转将在后面加入。
![](https://upload-images.jianshu.io/upload_images/9476896-fa71b123d5ac1700.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![三种变换的组合](https://upload-images.jianshu.io/upload_images/9476896-4309a13fd5239f40.gif?imageMogr2/auto-orient/strip)
[unitypackage下载](https://catlikecoding.com/unity/tutorials/rendering/part-1/transformations/transformations.unitypackage)

# 3 全方位旋转
目前我们仅能绕Z轴旋转，为和Unity的transform组件一样，我们还需要实现X和Y轴的旋转。单独来看，这和绕Z轴旋转类似，但要组合到一起就变得麻烦了。为解决此问题，我们需要用一个更好的数学方法来表示。
### 3.1 矩阵
从现在开始，我们将点的坐标由横向变为纵向。将用$$\begin{bmatrix}x\\y \end{bmatrix}$$来表示(x,y)，用$
\begin{bmatrix}
xcosZ - ysiznZ\\xsinZ + ycosZ 
\end{bmatrix}$来表示(xcosZ - ysiznZ,xsinZ + ycosZ )，这样读起来更容易些。
x和y在竖排的列中显示，如果用某个值乘以$\begin{bmatrix}x\\y \end{bmatrix}$，这将是一个二维乘法。实际上，我们要进行的乘法为$
\begin{bmatrix}
cosZ - sinZ\\sinZ + cosZ 
\end{bmatrix}\begin{bmatrix}x\\y \end{bmatrix}$。这是一个矩阵乘法，2x2矩阵的第一列表示X轴，第二列表示Y轴。
![用二维矩阵表示X和Y轴](https://upload-images.jianshu.io/upload_images/9476896-a59094e78fde7787.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通常来讲，矩阵相乘时，第一个矩阵是按行的顺序乘以第二个矩阵的列。最终结果为元素相乘之和，这意味着第一个矩阵的列数需要与第二个矩阵的行数一致。
![2x2矩阵相乘](https://upload-images.jianshu.io/upload_images/9476896-3a71e98c5b955d23.png)
结果矩阵的第一行包含 行1 x 列1，行1 x 列2等。第二行包含 行2 x 列2，行2 x 列2等。因此，结果矩阵和第一个矩阵有相同的行数，与第二个矩阵有相同的列数。
### 3.2 三维旋转矩阵
可以用一个2x2的矩阵来表示绕Z轴的二维旋转，但处理的是一个三维的点。而$
\begin{bmatrix}
cosZ - sinZ\\ sinZ + cosZ 
\end{bmatrix}\begin{bmatrix}x\\y\\ z \end{bmatrix}$是没有意义的，因为两个矩阵的行列数不相匹配。所以需要扩展旋转矩阵到3x3。如果将新加的元素都填为0，会怎样？
$
\begin{bmatrix}
cosZ & -sinZ & 0 \\sinZ & cosZ & 0\\0 & 0 & 0
\end{bmatrix}
\begin{bmatrix}
x\\y\\z 
\end{bmatrix}
=
\begin{bmatrix}
xcosZ - ysinZ + 0z\\xsinZ + ycosZ + 0z \\ 0x + 0y + 0z
\end{bmatrix}
=
\begin{bmatrix}
xcosZ - ysinZ \\xsinZ + ycosZ\\0
\end{bmatrix}
$
X和Y值看起来不错，但Z值却始终为0。为保证Z值不变，需要将旋转矩阵右下角的元素替换为1。因为第三列代表的是Z轴，也就是$
\begin{bmatrix}
0\\0\\1
\end{bmatrix}
$。
$
\begin{bmatrix}
cosZ &-sinZ&0 \\sinZ&cosZ&0 \\ 0&0&1
\end{bmatrix}
\begin{bmatrix}
x\\y\\z 
\end{bmatrix}
=
\begin{bmatrix}
xcosZ-ysinZ \\xsinZ + ycosZ\\z
\end{bmatrix}
$
如果我们修改矩阵为单位矩阵：对角线上的元素为1，其他部分为0。它表现得像一个可以让任何值通过的过滤器一样，不会更改被乘值。
$
\begin{bmatrix}
1&0&0\\0&1&0 \\0&0&1
\end{bmatrix}
\begin{bmatrix}
x\\y\\z 
\end{bmatrix}
=
\begin{bmatrix}
x\\y\\z 
\end{bmatrix}
$
### 3.3 X和Y轴的旋转矩阵
和绕Z轴旋转类似，我们可以找到一个绕Y轴旋转的矩阵。首先，X轴自$
\begin{bmatrix}
1\\0\\0 
\end{bmatrix}
$逆时针旋转90°之后变成了$
\begin{bmatrix}
0\\0\\-1 
\end{bmatrix}$。这意味着旋转之后的X轴可以表示为$
\begin{bmatrix}
cosY\\0\\-sinY 
\end{bmatrix}$。Z轴延迟90°，所以是$
\begin{bmatrix}
sinY\\0\\cosY 
\end{bmatrix}$。Y轴没有变，所以得出的旋转矩阵为$
\begin{bmatrix}
cosY&0&sinY\\0&1&0\\-sinY&0&cosY 
\end{bmatrix}$。
类似的，可得出绕X轴旋转的旋转矩阵$
\begin{bmatrix}
1&0&0\\0&cosX&-sinX\\ 0&sinX&cosX 
\end{bmatrix}$。
### 3.4 整合三维旋转矩阵
我们有了三个绕单轴旋转的矩阵，可以通过一个个轴旋转来整合它们。例如ZYX的顺序：先绕Z轴旋转点，然后是Y轴，最后是X轴。
或者，也可以将这三个矩阵相乘来生成一个新的矩阵，同时绕三个轴旋转。让我们先整合Y x Z。
第一个元素的值为 cosYcosZ - 0sinZ - 0sinY = cosYcosZ。要得出结果，会有大量的乘法运算，不过许多部分都是0，可以舍弃掉。
$
\begin{bmatrix}
cosYcosZ&-cosYsinZ&sinY\\sinZ&cosZ&0\\-sinYcosZ&sinYsinZ&cosY 
\end{bmatrix}$
接着执行 X × (Y × Z)以得到最终的矩阵。
$\begin{bmatrix}
cosYcosZ&-cosYsinZ&sinY\\
cosXsinZ+sinXsinYcosZ&cosXcosZ-sinXsinYsinZ&-sinXcosY\\
-sinXsinZ-cosXsinYcosZ&sinXcosZ+cosXsinYsinZ&cosXcosY 
\end{bmatrix}$
>问：矩阵相乘的顺序重要吗？
答：组合矩阵相乘的顺序是不重要的，如X × (Y × Z) = (X × Y) × Z。虽然中间步骤不一样，但结果是一样的。
但是，改变矩阵的位置即改变旋转的顺序，会产生一个不同的值。即X × Y × Z $\neq $ Z × Y × X。
而Unity采用的旋转顺序是ZXY。

根据这个矩阵，我们就知道如何构建绕X，Y，Z轴旋转的矩阵了。
```
	public override Vector3 Apply (Vector3 point) {
		float radX = rotation.x * Mathf.Deg2Rad;
		float radY = rotation.y * Mathf.Deg2Rad;
		float radZ = rotation.z * Mathf.Deg2Rad;
		float sinX = Mathf.Sin(radX);
		float cosX = Mathf.Cos(radX);
		float sinY = Mathf.Sin(radY);
		float cosY = Mathf.Cos(radY);
		float sinZ = Mathf.Sin(radZ);
		float cosZ = Mathf.Cos(radZ);


		Vector3 xAxis = new Vector3(
			cosY * cosZ,
			cosX * sinZ + sinX * sinY * cosZ,
			sinX * sinZ - cosX * sinY * cosZ
		);
		Vector3 yAxis = new Vector3(
			-cosY * sinZ,
			cosX * cosZ - sinX * sinY * sinZ,
			sinX * cosZ + cosX * sinY * sinZ
		);
		Vector3 zAxis = new Vector3(
			sinY,
			-sinX * cosY,
			cosX * cosY
		);

		return xAxis * point.x + yAxis * point.y + zAxis * point.z;
	}
```
![绕三个轴旋转](https://upload-images.jianshu.io/upload_images/9476896-ca4315566909bcfd.gif?imageMogr2/auto-orient/strip)

[unitypackage下载](https://catlikecoding.com/unity/tutorials/rendering/part-1/full-rotations/full-rotations.unitypackage)

# 4 矩阵变换
如果我们能组合三维旋转到一个矩阵，是否能将缩放、旋转、位移也组合起来呢？若能通过矩阵的形式表示缩放和位移的话，这就是可行的。
缩放矩阵可以直接构建，修改单位矩阵的元素即可。
$\begin{bmatrix}
2&0&0\\0&3&0\\0&0&4
\end{bmatrix}
\begin{bmatrix}
x\\y\\z 
\end{bmatrix}
= 
\begin{bmatrix}
2x\\3y\\ 4z 
\end{bmatrix}$
但位移呢？这不能通过重定义三个轴来实现，所以我们需要额外添加一列。
$\begin{bmatrix}
1&0&0&2\\0&1&0&3\\0&0&1&4
\end{bmatrix}
\begin{bmatrix}
x\\y\\z 
\end{bmatrix}
= 
\begin{bmatrix}
x+2\\y+3\\z+4 
\end{bmatrix}$
但矩阵每行的长度变为了4，这是无效的。需要为点也添加一个元素，因为它会和位移相乘，所以其值应该是1。同时我们也希望这个1能够保留下来，以用到后续的矩阵乘法中。这导致了一个4x4的矩阵和一个四维的点相乘。
$\begin{bmatrix}
1&0&0&2\\0&1&0&3\\0&0&1&4\\ 0&0&0&1
\end{bmatrix}
\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
= 
\begin{bmatrix}
x+2\\y+3\\z+4\\1
\end{bmatrix}$
所以，我们必须使用一个4x4的矩阵。这意味着表示缩放和旋转的矩阵增加了一行(0,0,0,1)。点也获得了第四个值为1的坐标。
### 4.1 齐次坐标
第四个坐标到底有什么意义？它代表了什么？如果其值为1则表示位移，若值为0则不可位移，不过缩放和旋转仍然有效。
而能被缩放和旋转却不能移动，这不是点，而是向量，它代表了方向。
所以$\begin{bmatrix}
x\\y\\z\\ 1
\end{bmatrix}$代表一个点，而$\begin{bmatrix}
x\\y\\z\\0
\end{bmatrix}$则代表向量。这意味着我们应用相同的矩阵到点、法线、切线上。
但如果第四个坐标不是0或者1会怎样？其实这并没啥区别，这和现在使用的齐次坐标系有关。在齐次空间中，一个点可以有无数种表示方式。最直接的是将第四个坐标设置为1。其他表达方式如同乘以一个任意数。
$\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
=
\begin{bmatrix}
2x\\2y\\2z\\2
\end{bmatrix}
=
\begin{bmatrix}
3x\\3y\\3z\\3
\end{bmatrix}
=
\begin{bmatrix}
wx\\wy\\wz\\w
\end{bmatrix}= 
w\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
$
而为了获取实际的三维点，我们可以用每个坐标除以第四个值即可。
$\begin{bmatrix}
x\\y\\z\\ w
\end{bmatrix}
=\frac{1}{w}
\begin{bmatrix}
x\\y\\z\\w
\end{bmatrix}
=
\begin{bmatrix}
\frac{x}{w}\\\frac{y}{w} \\ \frac{z}{w}\\1
\end{bmatrix}
=
\begin{bmatrix}
\frac{x}{w}\\\frac{y}{w}\\ \frac{z}{w}
\end{bmatrix}$
显然，当第四个坐标为0时这是不生效的。因为除以0意为无穷大，这也是为什么它们表现得像向量的原因了。
### 4.2 使用矩阵
从现在开始，我们将使用Unity的Matrix4x4结构体来执行矩阵乘法。
添加一个只读的虚属性到Transformation类中以获取变换矩阵。
```
	public abstract Matrix4x4 Matrix { get; }
```
Apply也不再是虚方法，它将执行矩阵乘法。
```
	public Vector3 Apply (Vector3 point) {
		return Matrix.MultiplyPoint(point);
	}
```
注意Matrix.MultiplyPoint的参数是一个三维向量，其默认第四个坐标的值为1。这保留了齐次坐标到欧拉坐标的转变。如果你想乘以一个向量，可以用 Matrix.MultiplyVector方法。
具体的transform类需要将Apply的实现移到Matrix属性中来。
首先是PositionTransformation，matrix.SetRow方法提供了一个便捷方式去填充矩阵。
```
	public override Matrix4x4 Matrix {
		get {
			Matrix4x4 matrix = new Matrix4x4();
			matrix.SetRow(0, new Vector4(1f, 0f, 0f, position.x));
			matrix.SetRow(1, new Vector4(0f, 1f, 0f, position.y));
			matrix.SetRow(2, new Vector4(0f, 0f, 1f, position.z));
			matrix.SetRow(3, new Vector4(0f, 0f, 0f, 1f));
			return matrix;
		}
	}
```
接下来是ScaleTransformation。
```
	public override Matrix4x4 Matrix {
		get {
			Matrix4x4 matrix = new Matrix4x4();
			matrix.SetRow(0, new Vector4(scale.x, 0f, 0f, 0f));
			matrix.SetRow(1, new Vector4(0f, scale.y, 0f, 0f));
			matrix.SetRow(2, new Vector4(0f, 0f, scale.z, 0f));
			matrix.SetRow(3, new Vector4(0f, 0f, 0f, 1f));
			return matrix;
		}
	}
```
至于RotationTransformation，按照我们现有的代码，按列设置更为方便。
```
	public override Matrix4x4 Matrix {
		get {
			float radX = rotation.x * Mathf.Deg2Rad;
			float radY = rotation.y * Mathf.Deg2Rad;
			float radZ = rotation.z * Mathf.Deg2Rad;
			float sinX = Mathf.Sin(radX);
			float cosX = Mathf.Cos(radX);
			float sinY = Mathf.Sin(radY);
			float cosY = Mathf.Cos(radY);
			float sinZ = Mathf.Sin(radZ);
			float cosZ = Mathf.Cos(radZ);

			Matrix4x4 matrix = new Matrix4x4();
			matrix.SetColumn(0, new Vector4(
				cosY * cosZ,
				cosX * sinZ + sinX * sinY * cosZ,
				sinX * sinZ - cosX * sinY * cosZ,
				0f
			));
			matrix.SetColumn(1, new Vector4(
				-cosY * sinZ,
				cosX * cosZ - sinX * sinY * sinZ,
				sinX * cosZ + cosX * sinY * sinZ,
				0f
			));
			matrix.SetColumn(2, new Vector4(
				sinY,
				-sinX * cosY,
				cosX * cosY,
				0f
			));
			matrix.SetColumn(3, new Vector4(0f, 0f, 0f, 1f));
			return matrix;
		}
	}
```
### 4.3 组合矩阵
现在可以将变换矩阵组合为一个了，在TransformationGrid中添加一个名为transformation的矩阵变量。
```
	Matrix4x4 transformation;
```
我们将在每帧更新这个值，先获取第一个变换矩阵，然后按正确顺序乘以其他变换矩阵。
```
	void Update () {
		UpdateTransformation();
		for (int i = 0, z = 0; z < gridResolution; z++) {
			...
		}
	}

	void UpdateTransformation () {
		GetComponents<Transformation>(transformations);
		if (transformations.Count > 0) {
			transformation = transformations[0].Matrix;
			for (int i = 1; i < transformations.Count; i++) {
				transformation = transformations[i].Matrix * transformation;
			}
		}
	}
```
网格也不再调用Apply方法，而是乘以矩阵。
```
	Vector3 TransformPoint (int x, int y, int z) {
		Vector3 coordinates = GetCoordinates(x, y, z);
		return transformation.MultiplyPoint(coordinates);
	}
```
这更有效率，因为之前对于每个点都需要创建旋转矩阵，而现在只需要创建一个组合的变换矩阵然后在每个点上复用即可。Unity正是对每个物体使用同一个旋转矩阵。
甚至可以更有效率一些，因为所有变换矩阵最下边行都是(0,0,0,1)，我们可以跳过这些计算，直接保留最后的除法即可。这正是Matrix.MultiplyPoint4x3方法。但是，我们并不打算使用它，因为接下来会用到这一行。

[unitypackage下载](https://catlikecoding.com/unity/tutorials/rendering/part-1/matrix-transformations/matrix-transformations.unitypackage)

# 5 投影矩阵
目前为止，我们已经实现将一个三维坐标变换到另一个三维坐标。但它们是怎么显示成二维的？这需要一个三维空间到二维空间的变换。
为投影创建一个具体的变换组件，先设置矩阵为单位矩阵。
```
public class CameraTransformation : Transformation {

	public float focalLength = 1f;

	public override Matrix4x4 Matrix {
		get {
			Matrix4x4 matrix = new Matrix4x4();
			matrix.SetRow(0, new Vector4(1f, 0f, 0f, 0f));
			matrix.SetRow(1, new Vector4(0f, 1f, 0f, 0f));
			matrix.SetRow(2, new Vector4(0f, 0f, 1f, 0f));
			matrix.SetRow(3, new Vector4(0f, 0f, 0f, 1f));
			return matrix;
		}
	}
}
```
添加到变换组件的末尾。
![相机投影放在最后](https://upload-images.jianshu.io/upload_images/9476896-ee8bff8a2ba61ab5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 5.1 正交相机
最直接的三维转二维方式是舍弃一个维度，三维空间将塌陷为一个平面。这个平面看起来就像一个绘制场景的画布。让我们试试舍弃Z维。
$\begin{bmatrix}
1&0&0&0\\0&1&0&0\\0&0&0&0\\0&0&0&1
\end{bmatrix}$
```
			matrix.SetRow(0, new Vector4(1f, 0f, 0f, 0f));
			matrix.SetRow(1, new Vector4(0f, 1f, 0f, 0f));
			matrix.SetRow(2, new Vector4(0f, 0f, 0f, 0f));
			matrix.SetRow(3, new Vector4(0f, 0f, 0, 1f));
```

| ![正交投影](https://upload-images.jianshu.io/upload_images/9476896-ff46145c84f2a1a2.png)| ![正交投影](https://upload-images.jianshu.io/upload_images/9476896-cf12901e8f46698a.png) |
|:-|:-|
这下，我们的网格就变成二维的了。虽然还是可以缩放、旋转和位移，但它被投影到了XY平面上。以上就是一个最基本的正交相机投影。
>问：为什么颜色会闪烁？
答：我们所有的点都被压缩在XY平面，这意味着它们在Z轴上堆叠。而哪个正方体显示在像素顶端则是随机的。

初始相机在原点，朝向Z轴的正方向。能否移动或旋转它？答案是可以的。在视觉上，移动相机和反向移动整个世界是一样的。旋转和缩放也是如此。虽然这有些不便，但我们是可以用现有的矩阵去移动相机的。在Unity中这是用逆矩阵来实现的。

### 5.2 透视相机
但正交相机并不能如实地反映我们看到的世界，透视相机却可以，它会让距离较远的物体显得更小。我们能通过缩放点到摄像机的距离来实现此效果。 
让我们用Z坐标值来除以XY坐标。这能通过将矩阵最后一行改为(0,0,1,0)来实现。使坐标的第四个值等于原始的Z值。在由齐次坐标转向欧拉坐标时，就可以相除了。
$\begin{bmatrix}
1&0&0&0\\0&1&0&0\\0&0&0&0\\0&0&1&0
\end{bmatrix}
\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
= 
\begin{bmatrix}
x\\y\\0\\z
\end{bmatrix}
\rightarrow 
\begin{bmatrix}
\frac{x}{z}\\\frac{y}{z}\\0
\end{bmatrix}
$
```
			matrix.SetRow(0, new Vector4(1f, 0f, 0f, 0f));
			matrix.SetRow(1, new Vector4(0f, 1f, 0f, 0f));
			matrix.SetRow(2, new Vector4(0f, 0f, 0f, 0f));
			matrix.SetRow(3, new Vector4(0f, 0f, 1f, 0f));
```
和正交投影中点直接向投影平面移动不同，透视投影会朝摄像机的位置（也就是原点）移动，直到它们碰到投影平面为止。当然，这仅对在摄像机之前的点有用。因为我们并没有舍弃掉摄像机后面的点，所以它们将发生错误的投影，所以在位移时要确保点都在摄像机前面。如果没有缩放或旋转网格，距离5就够了。否则，你可能需要离得更远。

| ![透视投影](https://upload-images.jianshu.io/upload_images/9476896-af43df99ea99efc6.png)| ![透视投影](https://upload-images.jianshu.io/upload_images/9476896-89fe14fcb39aad6a.png) |
|:-|:-|

正如照相机的焦距一样，相机到投影平面的距离也对投影有所影响。此值越大，则视场越小。我们现在的焦距为1，是一个90°的视场。让其变得可配置。
```
	public float focalLength = 1f;
```
![焦距](https://upload-images.jianshu.io/upload_images/9476896-c41998548d0f65fa.png)
焦距值意味着放大。
$\begin{bmatrix}
fl&0&0&0\\0&fl&0&0\\0&0&0&0\\0&0&1&0
\end{bmatrix}
\begin{bmatrix}
x\\y\\z\\1
\end{bmatrix}
= 
\begin{bmatrix}
xfl\\yfl\\0\\z
\end{bmatrix}
\rightarrow 
\begin{bmatrix}
\frac{xfl}{z}\\\frac{yfl}{z}\\0
\end{bmatrix}
$
```
			matrix.SetRow(0, new Vector4(focalLength, 0f, 0f, 0f));
			matrix.SetRow(1, new Vector4(0f, focalLength, 0f, 0f));
			matrix.SetRow(2, new Vector4(0f, 0f, 0f, 0f));
			matrix.SetRow(3, new Vector4(0f, 0f, 1f, 0f));
```
![调整焦距](https://upload-images.jianshu.io/upload_images/9476896-4376aa9a024bfe9d.gif?imageMogr2/auto-orient/strip)

现在我们已经实现了一个很简单的投影摄像机。如果需要全面模拟Unity的摄像机投影，则还需要定义远近平面。这要求投影到一个立方体中，而非一个平面上，深度信息得以保留。接下来还要考虑视口的横纵比。同时，Unity的摄像机是朝向Z轴负方向的，这需要一些负值。如果你想的话，可以尝试将它们都统一到投影矩阵中。
其实我们很少需要去定义矩阵以及投影矩阵，那以上内容到底有啥意义？其作用在于明白背后的过程：矩阵不过是将点和向量从一个空间转到另一个。而在后续的课程，我们在写自己的shader时，还会遇到它们。这将在第二部分，shader基础中涉及。

[unitypackage下载](https://catlikecoding.com/unity/tutorials/rendering/part-1/projection-matrices/projection-matrices.unitypackage)
































