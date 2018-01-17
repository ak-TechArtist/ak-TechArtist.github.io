---
layout:     post
title:      "手游效果逆向分析"
subtitle:   ""
date:       2017-12-23 12:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

# 一 概览
---
本文以实际工作中的一个美术需求为例完整地演示了手游效果的逆向分析过程：从需求到准备工作，最终在Unity中实现，着重讲了具体的逆向分析步骤。文章的最后还汇总了一些常见问题。

# 二 美术需求
---
王者荣耀 英雄甄姬 游园惊梦皮肤 的水面背景效果（别总是盯着美女看。**看水，看水，看水！**）

![](/img/in-post/mobile-game-revert/gif1.gif)
<small class="img-hint"></small>

# 三 实现效果
---
基本还原了此效果（Gif图被压缩得有些狠，和实际差异较大、、、）

![](/img/in-post/mobile-game-revert/gif2.gif)
<small class="img-hint"></small>

# 四 准备工作
---
## 1 硬件 
一台高通骁龙处理器的安卓手机，本次使用锤子坚果Pro（骁龙626处理器）

## 2 软件
Adreno Profiler 4.0，通过截图可以看出此工具在3年前就停止维护更新了，但个人感觉在逆向分析上还是蛮好用的

![](/img/in-post/mobile-game-revert/pic1.png)
<small class="img-hint"></small>

# 五 具体步骤
---
## 1 连接手机
需要选中具体的游戏App

![](/img/in-post/mobile-game-revert/pic2.png)
<small class="img-hint"></small>

## 2 抓取帧

![](/img/in-post/mobile-game-revert/pic3.png)
<small class="img-hint"></small>

游戏会有略微的卡顿，抓取成功之后的界面：

![](/img/in-post/mobile-game-revert/pic4.png)
<small class="img-hint"></small>

## 3 定位具体的Drawcall（绘制调用）
可以看到，这一帧有许多Drawcall。接下来就需要从中找出绘制背景图水面效果的Drawcall。找到第59个为目标Drawcall。

![](/img/in-post/mobile-game-revert/pic5.png)
<small class="img-hint"></small>

该Drawcall用到了两张贴图，猜测一张为噪音扰动图，一张为背景图：

![](/img/in-post/mobile-game-revert/pic6.png)
<small class="img-hint"></small>

![](/img/in-post/mobile-game-revert/pic7.png)
<small class="img-hint"></small>

## 4 分析Shader
### 1）打开Shader分析器
![](/img/in-post/mobile-game-revert/pic8.png)
<small class="img-hint"></small>

![](/img/in-post/mobile-game-revert/pic9.png)
<small class="img-hint"></small>

可以查看到到编译后的Shader源码为：

```
Vertex：
attribute vec4 _glesVertex;
attribute vec4 _glesColor;
attribute vec4 _glesMultiTexCoord0;
uniform highp vec4 _Time;
uniform highp mat4 glstate_matrix_mvp;
uniform highp vec4 _MainTex1_ST;
uniform highp vec4 _MainTex2_ST;
uniform highp float _ScrollX;
uniform highp float _ScrollY;
varying highp vec4 xlv_TEXCOORD0;
varying lowp vec4 xlv_TEXCOORD1;
varying highp vec4 xlv_TEXCOORD2;
void main ()
{
  highp vec4 tmpvar_1;
  highp vec4 tmpvar_2;
  highp vec2 tmpvar_3;
  tmpvar_3.x = _ScrollX;
  tmpvar_3.y = _ScrollY;
  tmpvar_1.xy = (((_glesMultiTexCoord0.xy * _MainTex1_ST.xy) + _MainTex1_ST.zw) + fract((tmpvar_3 * _Time.x)));
  tmpvar_1.zw = ((_glesMultiTexCoord0.xy * _MainTex2_ST.xy) + _MainTex2_ST.zw);
  gl_Position = (glstate_matrix_mvp * _glesVertex);
  xlv_TEXCOORD0 = tmpvar_1;
  xlv_TEXCOORD1 = _glesColor;
  xlv_TEXCOORD2 = tmpvar_2;
}

Fragment
precision highp float;
uniform sampler2D _MainTex1;
uniform sampler2D _MainTex2;
uniform highp vec4 _UVXX;
varying highp vec4 xlv_TEXCOORD0;
varying lowp vec4 xlv_TEXCOORD1;
void main ()
{
  mediump vec2 uv_1;
  lowp vec4 o_2;
  lowp vec4 tmpvar_3;
  tmpvar_3 = texture2D (_MainTex1, xlv_TEXCOORD0.xy);
  highp vec2 tmpvar_4;
  tmpvar_4 = vec2(((tmpvar_3.x * _UVXX.x) * xlv_TEXCOORD1.w));
  uv_1 = tmpvar_4;
  lowp vec4 tmpvar_5;
  highp vec2 P_6;
  P_6 = (xlv_TEXCOORD0.zw + uv_1);
  tmpvar_5 = texture2D (_MainTex2, P_6);
  o_2.xyz = tmpvar_5.xyz;
  o_2.w = tmpvar_5.w;
  gl_FragData[0] = o_2;
}
```

### 2）分析并优化
主要是去掉无关参数，以及理清其代码逻辑
```
Vertex：
attribute vec4 _glesVertex; //模型顶点坐标
attribute vec4 _glesColor; //用于控制扰动强度
attribute vec4 _glesMultiTexCoord0; //用于UV Tiling & Offset
uniform highp vec4 _Time; //时间参数
uniform highp mat4 glstate_matrix_mvp; //模型坐标转换矩阵
uniform highp vec4 _NoisyTex_ST; //扰动UV
uniform highp vec4 _MainTex_ST; //主贴图UV
uniform highp float _ScrollX; //X方向速度
uniform highp float _ScrollY; //Y方向速度
varying highp vec4 UVs;//存储NoisyTex & MainTex的UV                                
void main ()
{
  //UV坐标随时间转变
  UVs.xy = (((_glesMultiTexCoord0.xy * _NoisyTex_ST.xy) + _NoisyTex_ST.zw) + 
  fract((vec2(_ScrollX, _ScrollY) * _Time.x)));
  UVs.zw = ((_glesMultiTexCoord0.xy * _MainTex_ST.xy) + _MainTex_ST.zw);
  gl_Position = (glstate_matrix_mvp * _glesVertex);
}

Fragment：
precision highp float;
uniform sampler2D _NoisyTex; //噪声贴图
uniform sampler2D _MainTex; //主贴图
uniform highp vec4 _UVXX; //噪声倍数
varying highp vec4 UVs; //由Vertex Shader传过来的UV坐标
void main ()
{
  lowp vec4 noisyValue = texture2D (_NoisyTex, UVs.xy);
  //由两个因子来控制噪声强度
  noisyValue.xy = vec2(((noisyValue.x * _UVXX.x) * _glesColor.w));
  //将噪声值加到主贴图的UV上，对其进行偏移
  UVs.zw += noisyValue.xy;
  lowp vec4 mainTexValue = texture2D (_MainTex, UVs.zw);
  gl_FragData[0] = mainTexValue;
}
```

## 5 在Unity中实现Shader
和上面Shader的主要差别是只利用一个_NoisyScale来控制噪声强度，然后添加了对雾效的支持
```
Shader "UIEffect/Noisy_UVScroll"
{
	Properties
	{
		_MainTex ("BackGround Texture", 2D) = "white" {}
		_XSpeed("X Speed", Range(-10, 10)) = 1
		_YSpeed("Y Speed", Range(-10, 10)) = 1
		_NoisyTex("Noisy Texture", 2D) = "white"{}
		_NoisyScale("Noisy Scale", Range(0, 1)) = 0.1
	}
	SubShader
	{
		Tags { "RenderType"="Opaque" }
		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			// make fog work
			#pragma multi_compile_fog
			#include "UnityCG.cginc"

			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float4 uv : TEXCOORD0;
				UNITY_FOG_COORDS(1)
				float4 vertex : SV_POSITION;
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;
			sampler2D _NoisyTex;
			float4 _NoisyTex_ST;
			float _NoisyScale;
			half _XSpeed;
			half _YSpeed;

			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
				o.uv.zw = TRANSFORM_TEX(v.uv, _NoisyTex) + frac(_Time.x * half2(_XSpeed, _YSpeed));
				UNITY_TRANSFER_FOG(o,o.vertex);
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target
			{
				half noisyValue = (tex2D (_NoisyTex, i.uv.zw)).x;
				noisyValue *= _NoisyScale;
				// sample the texture
				fixed4 col = tex2D(_MainTex, i.uv.xy + noisyValue);
				// apply fog
				UNITY_APPLY_FOG(i.fogCoord, col);
				return col;
			}
			ENDCG
		}
	}
	FallBack "Mobile/VertexLit"
}
```
## 6 运用到具体的UI上

![](/img/in-post/mobile-game-revert/pic10.png)
<small class="img-hint"></small>

![](/img/in-post/mobile-game-revert/pic11.png)
<small class="img-hint"></small>

# 六 其他问题
---
## 1 Snapdragon Profiler 还是 Adreno Profiler？
目前Snapdragon Profiler的最新版本是2.1，除了定位具体的Drawcall很慢之外，还存在定位不准的情况。
这其实也是本篇博客写了较长时间的原因：一直定位不到具体的Drawcall，不得已换成了Adreno Profiler。

## 2 连不上手机？
可以参考官方帮助手册，实践法线拔插手机，利用豌豆荚等手机助手安装驱动，乃至重启电脑都可能是有效的。

![](/img/in-post/mobile-game-revert/pic12.png)
<small class="img-hint"></small>

## 3 找不到具体的帧？
按照 不透明物体 -> 透明物体 -> 屏幕后期 -> UI 的一个大概顺序，逐帧进行查看。

## 4 更复杂的效果？
需要更丰富的经验对Shader代码和所用贴图的功能进行解读，但原理步骤都是相似的。

