---
layout:     post
title:      "在Unity地形（Terrain）中使用图集（Atlas）"
subtitle:   ""
date:       2018-08-02 09:00:00
author:     "AK"
header-img: "img/post-bg.jpg"
catalog: true
tags:
    - Unity
---

Unity地形集成了许多功能：高度图、树、草等。  
本文仅专注于其中一部分：地形贴图。  
主要介绍了Unity地形贴图的三种实现方式：
- Unity自带地形贴图
- Terrain转Mesh（T2M）解决方案
- 图集（Atlas）解决方案（重点内容）
    - 导出所需贴图
    - 使用贴图渲染
    - 后续改进
    - 和T2M对比
    - 更多功能

文末提供Demo下载。    

![](/img/in-post/terrain-atlas/1.png)
<small class="img-hint">三种实现方式</small>

# 一 Unity自带地形贴图
Unity自带的Terrain组件是支持刷多层贴图的：

![](/img/in-post/terrain-atlas/2.png)
<small class="img-hint">设置了五层贴图</small>

# 二 Terrain转Mesh（T2M）解决方案
自带Terrain因为集成了众多功能，扩展不方便，且顶点数目较多影响性能。一般会利用诸如TerrainToMesh等插件将Terrain转为Obj，由于本文的测试地形十分简单，所以用极少的顶点数即可：

![](/img/in-post/terrain-atlas/3.png)
<small class="img-hint">转为Obj之后顶点数少了许多</small>

![](/img/in-post/terrain-atlas/4.png)
<small class="img-hint">T2M材质</small>

查看Shader代码，5层贴图采样了7次（包括两张控制图），而随着贴图增加，采样次数也会相应提高

```
	half4 splat_control = tex2D (_V_T2M_Control, IN.uv_V_T2M_Control);

	fixed4 splatColor1 = fixed4(0, 0, 0, 0);

	splatColor1 = splat_control.r * tex2D(_V_T2M_Splat1, IN.uv_V_T2M_Control * _V_T2M_Splat1_uvScale);

	fixed4 mainTex = splatColor1;
	fixed4 splatColor2 = splat_control.g * tex2D(_V_T2M_Splat2, IN.uv_V_T2M_Control * _V_T2M_Splat2_uvScale);
	       mainTex += splatColor2;
		   testColor = splat_control.g;
	
	#ifdef V_T2M_3_TEX
		   fixed4 splatColor3 = splat_control.b * tex2D(_V_T2M_Splat3, IN.uv_V_T2M_Control * _V_T2M_Splat3_uvScale);
		mainTex += splatColor3;
	#endif
	#ifdef V_T2M_4_TEX
		fixed4 splatColor4 = splat_control.a * tex2D(_V_T2M_Splat4, IN.uv_V_T2M_Control * _V_T2M_Splat4_uvScale);
		mainTex += splatColor4;
	#endif


	#ifdef V_T2M_2_CONTROL_MAPS
		 half4 splat_control2 = tex2D (_V_T2M_Control2, IN.uv_V_T2M_Control2);

		 fixed4 splatColor5= tex2D(_V_T2M_Splat5, IN.uv_V_T2M_Control2 * _V_T2M_Splat5_uvScale) * splat_control2.r;
		 mainTex.rgb += splatColor5.rgb;

		 #ifdef V_T2M_6_TEX
		 fixed4 splatColor6 = tex2D(_V_T2M_Splat6, IN.uv_V_T2M_Control2 * _V_T2M_Splat6_uvScale) * splat_control2.g;
			mainTex.rgb += splatColor6.rgb;
		 #endif

		 #ifdef V_T2M_7_TEX
			fixed4 splatColor7 = tex2D(_V_T2M_Splat7, IN.uv_V_T2M_Control2 * _V_T2M_Splat7_uvScale) * splat_control2.b;
			mainTex.rgb += splatColor7.rgb;
		 #endif

		 #ifdef V_T2M_8_TEX
			fixed4 splatColor8 = tex2D(_V_T2M_Splat8, IN.uv_V_T2M_Control2 * _V_T2M_Splat8_uvScale) * splat_control2.a;
			mainTex.rgb += splatColor8.rgb;
		 #endif
	#endif
```

# 三 图集（Atlas）解决方案
T2M解决方案主要的问题在于贴图采样次数较多，对性能损耗较大。  
同时，贴图数受Sampler最大数（最多16个）影响，也导致表现的多样化受限。   

![](/img/in-post/terrain-atlas/5.png)
<small class="img-hint">最多注册16个Sampler</small>

是否可以如UI中那样，将所需的贴图制成为一张图集，从而降低采样次数呢？  
经过验证，答案是可行的。

### 1 导出所需贴图
首先，我们需要生成所需的贴图（Demo中调出窗口：Window -> TerrainAtlas -> ExportMaps）

![](/img/in-post/terrain-atlas/6.png)
<small class="img-hint">导出贴图的入口</small>

接下来，看一下如何实现这些接口：

##### 1）导出基础图集
需要将地形中使用的基础图集拼为图集的形式

![](/img/in-post/terrain-atlas/7.png)
<small class="img-hint">基础图集</small>

从TerrainData中读取基础贴图数组，利用UnlitShader将这些贴图信息渲染到RenderTexture上，然后新建一个预设分辨率大小的图集，并从之前的RenderTexture中读入图集之中，最后将图集保存到指定的文件夹。

```
    void ExportBasemap()
    {
        TerrainData _terrainData = _sourceTerrain.terrainData;
        SplatPrototype[] prototypeArray = _terrainData.splatPrototypes;

        //创建相应数目的RT
        RenderTexture[] rtArray = new RenderTexture[prototypeArray.Length];

        int texSize = BasemapRes / COLUMN_AND_ROW_COUNT;
        Texture2D[] texArray = new Texture2D[prototypeArray.Length];

        for (int i = 0; i < prototypeArray.Length; i++)
        {
            rtArray[i] = RenderTexture.GetTemporary(texSize, texSize, 24);
            texArray[i] = new Texture2D(texSize, texSize, TextureFormat.RGB24, false);
        }

        //使用一个UnlitShader来将贴图绘制到具体的RenderTexture上
        Shader shader = Shader.Find("Unlit/UnlitShader");
        Material material = new Material(shader);

        //将这些图读入相应数目的Tex2D中
        for (int i = 0; i < prototypeArray.Length; i++)
        {
            Graphics.Blit(prototypeArray[i].texture, rtArray[i], material, 0);
            RenderTexture.active = rtArray[i];
            texArray[i].ReadPixels(new Rect(0f, 0f, (float)texSize, (float)texSize), 0, 0);

            //如果有法线贴图，则将这些值读入
            if (prototypeArray[i].normalMap != null)
            {
                Graphics.Blit(prototypeArray[i].normalMap, rtArray[i], material, 0);
                RenderTexture.active = rtArray[i];
            }
        }


        //走一遍之前的流程（不过需要将写死的数值灵活对待咯！）
        Texture2D tex = new Texture2D(BasemapRes, BasemapRes, TextureFormat.RGB24, false);

        for (int i = 0; i < prototypeArray.Length; i++)
        {
            //需要根据图片的序号算出当前贴图在大贴图中的起始位置
            int columnNum = i % COLUMN_AND_ROW_COUNT;
            int rowNum = (i % (COLUMN_AND_ROW_COUNT * COLUMN_AND_ROW_COUNT)) / COLUMN_AND_ROW_COUNT;
            int startWidth = columnNum * texSize;
            int startHeight = rowNum * texSize;
            for (int j = 0; j < texSize; j++)
            {
                for (int k = 0; k < texSize; k++)
                {
                    Color color = texArray[i].GetPixel(j, k);
                    tex.SetPixel(startWidth + j, startHeight + k, color);
                }
            }
        }

        tex.Apply();
        // Encode texture into PNG
        byte[] bytes = tex.EncodeToPNG();
        string directoryPath = Application.dataPath + DIRECTORYNAME + "/" + _directoryName + "/";
        if (!Directory.Exists(directoryPath))
        {
            Directory.CreateDirectory(directoryPath);
        }
        File.WriteAllBytes(directoryPath + "MainTex.png", bytes);

        Debug.Log("Basemap Exported");
    }
```

##### 2）导出索引贴图
索引贴图类似Terrain中的控制图：标记了基础贴图所占的区域。而不同的是，r通道标记了该位置占权重最大的贴图在图集中的索引，而g通道则记录了次要权重贴图在图集中的索引。

![](/img/in-post/terrain-atlas/8.png)
<small class="img-hint">rg通道分别记录权重最大和次之的贴图索引</small>

从TerrainData中读出混合贴图，然后新建一个同样大小的贴图作为索引贴图：

```
void ExportIndexMap()
    {
        TerrainData _terrainData = _sourceTerrain.terrainData;
        SplatPrototype[] prototypeArray = _terrainData.splatPrototypes;
        int _textureNum = prototypeArray.Length;

        //获取混合贴图
        Texture2D[] alphaMapArray = _terrainData.alphamapTextures;
        int witdh = alphaMapArray[0].width;
        int height = alphaMapArray[0].height;

        //新建和混合贴图一样大小的贴图
        Texture2D indexTex = new Texture2D(witdh, height, TextureFormat.RGB24, false, true);

        Color indexColor = new Color(0, 0, 0, 0);

        //对每一个像素进行计算
        for (int j = 0; j < witdh; j++)
        {
            for (int k = 0; k < height; k++)
            {
                //默认都是第一个贴图
                //这里支持将三层索引的信息导出，可供后续的Shader使用
                ResetNumAndWeight();

                //遍历所有Control的所有通道，识别出最大的通道所在的贴图序号
                for (int i = 0; i < _textureNum; i++)
                {
                    //根据贴图的序号算出当前应该计算的是哪个值
                    int controlMapNumber = (i % 16) / 4;
                    int controlChannelNum = i % 4;
                    Color color = alphaMapArray[controlMapNumber].GetPixel(j, k);
                    switch (controlChannelNum)
                    {
                        case 0:
                            CalculateIndex(i, color.r);
                            break;
                        case 1:
                            CalculateIndex(i, color.g);
                            break;
                        case 2:
                            CalculateIndex(i, color.b);
                            break;
                        case 3:
                            CalculateIndex(i, color.a);
                            break;
                        default:
                            break;
                    }
                }

                //将识别出来的序号写入IndexMap的r通道
                //需将此值转换到(0, 1)的范围内，因为最多支持16张贴图，而序号是0到15，则除以15即可
                indexColor.r = _maxTexNum / 15f;
                indexColor.g = _secondTexNum / 15f;
                indexTex.SetPixel(j, k, indexColor);

            }
        }

        string directoryPath = Application.dataPath + DIRECTORYNAME + "/" + _directoryName + "/";
        if (!Directory.Exists(directoryPath))
        {
            Directory.CreateDirectory(directoryPath);
        }

        indexTex.Apply();
        byte[] bytes = indexTex.EncodeToPNG();
        File.WriteAllBytes(directoryPath + "IndexTex.png", bytes);

        Debug.Log("Exported Index Map");
    }
```

对于每个像素，将控制图的各个通道所代表的贴图索引以及权重值传给以下函数：记录权重最大和次之的贴图索引，之后再将这两个值写入索引贴图的r和g通道中

```
    void CalculateIndex(int index, float curWeight)
    {
        //如果比最大的元素大，则取当前为最大，取之前第一为第二
        if (curWeight > _maxChannelWeight)
        {
            _secondChannelWeight = _maxChannelWeight;
            _secondTexNum = _maxTexNum;

            _maxChannelWeight = curWeight;
            _maxTexNum = index;
        }
        //如果仅是比第二的元素大，则取当前为第二
        else if (curWeight > _secondChannelWeight)
        {
            _secondChannelWeight = curWeight;
            _secondTexNum = index;
        }
    }
```


##### 3）导出混合权重贴图
导出混合权重贴图则相对简单，因为之前在生成索引贴图的时候已经有权重信息了。  
这里我们只需要新建一个权重贴图，然后将最大权重和第二权重填入相应的通道中即可

```
        Texture2D blendTex = new Texture2D(witdh, height, TextureFormat.RGB24, false, true);
        Color blendColor = new Color(0, 0, 0, 0);

                //计算Blend因子，将其填入到贴图通道中
                blendColor.r = _maxChannelWeight;
                blendColor.g = _secondChannelWeight;
                blendTex.SetPixel(j, k, blendColor);

        blendTex.Apply();
        byte[] blendBytes = blendTex.EncodeToPNG();
        File.WriteAllBytes(directoryPath + "BlendTex.png", blendBytes);
```
点击导出按钮，便得到了混合权重贴图，由于r通道代表了权重最大贴图的权重，所以在仅有两层混合（后续会介绍更多层的混合）的情况下，此值是大于0.5的

![](/img/in-post/terrain-atlas/9.png)
<small class="img-hint">rg通道分别代表最大和第二权重数值</small>

### 2 使用贴图渲染

有了这些贴图之后，就可以开始渲染了，主要经过以下几步：

##### 1）新建材质及Shader
在顶点着色器（vertex shader）中，计算了两张贴图的tiling以及offset，世界坐标和世界法线向量则是为光照计算做准备

```
			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv.xy = TRANSFORM_TEX(v.uv, _MainTex);
				o.uv.zw = TRANSFORM_TEX(v.uv, _IndexTex);
				o.worldNormal = UnityObjectToWorldNormal(v.normal);
				o.worldPos = mul(unity_ObjectToWorld, v.vertex);

				UNITY_TRANSFER_FOG(o,o.vertex);
				return o;
			}


```

在片元着色器（fragment shader）中，取出索引及其权重。根据索引算出具体的贴图在图集中的位置，获取颜色，并对其进行混合，最后应用上Lambert光照模型

```
			fixed4 frag (v2f i) : SV_Target
			{
				//将贴图中被压缩到0,1之间的Index还原
				float indexLayer1 = floor((tex2D (_IndexTex, i.uv.zw).r * 15));
				float indexLayer2 = floor((tex2D (_IndexTex, i.uv.zw).g * 15));

				//利用Index取得具体的贴图位置
				float4 colorLayer1 = GetColorByIndex(indexLayer1, i.worldPos.xz);
				float4 colorLayer2 = GetColorByIndex(indexLayer2, i.worldPos.xz);

				//混合因子，其中r通道为第一层贴图所占权重，g通道为第二层贴图所占权重
				float2 blend = tex2D (_BlendTex, i.uv.xy).rg;
				half4 albedo = colorLayer1 * blend.r + colorLayer2 * blend.g;

				//Lambert 光照模型
				float3 lightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
				half NoL = saturate(dot(normalize(i.worldNormal), lightDir));
				half4 diffuseColor = _LightColor0 * NoL * albedo;
				return diffuseColor;
			}
```

根据索引获取颜色的具体代码：先算出行和列号，找到在图集中的起始位置，然后利用世界空间中xz平面的坐标来得出uv坐标

```
			half4 GetColorByIndex(float index, float2 worldPos)
			{
				float2 columnAndRow;
				//先取列再取行，范围都是0到3
				columnAndRow.x = (index % 4.0);
				columnAndRow.y = floor((float((index % 16.0))) / 4.0);

				float2 curUV;
				//由于是4x4的图集，所以具体的行列需要乘以0.25 
				//如1就是（0.25, 0），刚好对应第二张贴图的起始位置
				curUV = ((columnAndRow * 0.25) + frac(worldPos) * 0.25 );
				return tex2D(_MainTex, curUV);
			}
```

##### 2）Copy地形Obj并赋予材质
将之前从TerrainToMesh插件导出的地形复制一份，然后将新建的材质赋值给它，并将导出的贴图填入相应栏

![](/img/in-post/terrain-atlas/10.png)
<small class="img-hint">填入对应贴图</small>

看一下效果，貌似和预期的不太一样 。。。 :(

![](/img/in-post/terrain-atlas/11.png)
<small class="img-hint">初步效果</small>

##### 3）贴图设置

忘了贴图导入设置（Import Settings）了！  
主贴图因为是图集，所以在Mipmap层级较高的地方会变为灰色，这也是产生灰色的边缘线的原因

![](/img/in-post/terrain-atlas/12.png)
<small class="img-hint">最高级Mipmap整个为灰色</small>

取消勾选Mipmap之后：

![](/img/in-post/terrain-atlas/13.png)
<small class="img-hint">朝预期效果迈进了一大步</small>

再设置好索引贴图和混合权重贴图的参数：

![](/img/in-post/terrain-atlas/14.png)
<small class="img-hint">索引贴图设置</small>

![](/img/in-post/terrain-atlas/15.png)
<small class="img-hint">混合权重贴图设置</small>

之后，再看看效果

![](/img/in-post/terrain-atlas/16.png)
<small class="img-hint">修改贴图设置之后的效果</small>

###3 后续改进
近距离观察发现远处会比较“碎”，这是因为一个像素对应了多个纹素所致

![](/img/in-post/terrain-atlas/17.png)
<small class="img-hint">远处“碎”是因为一个像素对应了多个纹素</small>

但如果开启Mipmap则会出现之前的白线。。。

![](/img/in-post/terrain-atlas/18.png)
<small class="img-hint">白线再现。。。</small>

这是因为如果用tex2D采样，则在边缘部分会因为uv变化太大，而取到级别很高的Mipmap上。  
要解决此问题，可以用tex2Dlod限制Mipmap的采样级别：  
首先，在顶点着色器中，记录观察空间的Z值（距离摄像机距离）

```
				//记录观察空间的Z值
				o.worldPos.w = mul(UNITY_MATRIX_MV, v.vertex).z;
```

其次，在片元着色器中，算出一个最低Mipmap采样level，然后再用tex2Dlod对纹理采样

```
				half lodLevel = min(-i.worldPos.w * 0.1, 3);

			half4 GetColorByIndex(float index, float lodLevel, float2 worldPos)
			{
				float2 columnAndRow;
				//先取列再取行，范围都是0到3
				columnAndRow.x = (index % 4.0);
				columnAndRow.y = floor((float((index % 16.0))) / 4.0);

				float4 curUV;
				//由于是4x4的图集，所以具体的行列需要乘以0.25 
				//如1就是（0.25, 0），刚好对应第二张贴图的起始位置
				curUV.xy = ((columnAndRow * 0.25) + frac(worldPos) * 0.25);
				curUV.w = lodLevel;
				return tex2Dlod(_MainTex, curUV);
			}

```

![](/img/in-post/terrain-atlas/19.png)
<small class="img-hint">限制了Mipmap最小值</small>

可以看到情况好了不少，但还是有依稀的白点。。。 继续改进~ 
将边缘的像素预留出来，这样应该就好一些了：

```
				curUV.xy = ((columnAndRow * 0.25) + ((frac((worldPos)) * 0.23) + 0.01));
```

可以添加一个参数_BlockParams来控制相关的参数，其中z值代表对基础纹理贴图的缩放，类似tiling的效果

```
				curUV.xy = ((columnAndRow * 0.25) + ((frac((worldPos * _BlockParams.z)) * _BlockParams.yy) + _BlockParams.xx));
```

最终效果：
![](/img/in-post/terrain-atlas/20.png)
<small class="img-hint">消除了白点</small>

### 4 和T2M对比
效果已经做完，来和T2M对比对比，总结一下利弊：

##### 1）优点
- 贴图数目少
仅需3张贴图即可支持最多16层贴图混合

- 采样次数少
也正是因为贴图数目较少的缘故，采样次数较少，手机上实测，较TerrainToMesh四层采样性能更高

- 表现多样性
当前最多支持16层贴图混合，不用受Sampler数量的限制，能很好地表现地形的多样性

##### 2）缺点
- 远处依然有些“碎”
由于图集的存在，当前是在不开mipmap和mipmap层级过高之间取了个平衡，在离得远的地方还是会因为多个纹素对应一个像素而看起来有些“碎”

- 边缘锯齿
将摄像机拉近之后查看，在混合边缘处能看到明显的锯齿

![](/img/in-post/terrain-atlas/21.png)
<small class="img-hint">混合边缘锯齿</small>

为了抗锯齿，纹理采样模式（FilterMode）一般设为Bilinear或Trilinear。  
而在当前的图集模式下，两种颜色权重是写到同一个通道的，这会在边缘采样时混合而造成突变，也就会产生较明显的锯齿

![](/img/in-post/terrain-atlas/22.png)
<small class="img-hint">不同颜色权重混合</small>

而T2M则因为是每层一个通道，所以能有一个较平滑的过渡

![](/img/in-post/terrain-atlas/23.png)
<small class="img-hint">T2M每层贴图权重单通道</small>

- 不支持分层缩放
使用多个贴图，可以分别设置Tiling，但是使用图集，则只可使用统一的参数控制，这样的话要想表现各层大小差异，则需要体现在贴图本身了。这会导致美术在实际使用中调节起来没那么方便。

### 5 更多功能
##### 1）添加法线贴图
导出法线图集和导出基础图集类似，只是如果当前层没有法线贴图的话，需要将其设置为(0.5,0.5,1)（(0,0,1) * 0.5 + 0.5）

```
                    Color normalColor = (prototypeArray[i].normalMap == null) ? new Color(0.5f, 0.5f, 1) : GetPixelColor(j, k, normalMapArray[i]);
                    normalTex.SetPixel(startWidth + j, startHeight + k, normalColor);
```

##### 2）三层混合
需要在计算权重的时候加入第三权重

```
    void CalculateIndex(int index, float curWeight)
    {
        //如果比最大的元素大，则取当前为最大，取之前第一为第二，取之前第二的为第三
        if (curWeight > _maxChannelWeight)
        {
            _thirdChannelWeight = _secondChannelWeight;
            _thirdTexNum = _secondTexNum;

            _secondChannelWeight = _maxChannelWeight;
            _secondTexNum = _maxTexNum;

            _maxChannelWeight = curWeight;
            _maxTexNum = index;
        }
        //如果仅是比第二的元素大，则取当前为第二，取之前的第二为第三
        else if (curWeight > _secondChannelWeight)
        {
            _thirdChannelWeight = _secondChannelWeight;
            _thirdTexNum = _secondTexNum;

            _secondChannelWeight = curWeight;
            _secondTexNum = index;
        }
        //如果仅是比第三的元素大，则取当前为第三
        else if (curWeight > _thirdChannelWeight)
        {
            _thirdChannelWeight = curWeight;
            _thirdTexNum = index;
        }
    }
```

然后将第三权重的贴图索引和混合权重分别写入到索引贴图以及混合权重贴图的b通道中

```
                //将识别出来的序号写入IndexMap的通道中
                //需将此值转换到(0, 1)的范围内，因为最多支持16张贴图，而序号是0到15，则除以15即可
                indexColor.r = _maxTexNum / 15f;
                indexColor.g = _secondTexNum / 15f;
                indexColor.b = _thirdTexNum / 15f;
                indexTex.SetPixel(j, k, indexColor);

                //计算Blend因子，将其填入到贴图通道中
                blendColor.r = _maxChannelWeight;
                blendColor.g = _secondChannelWeight;
                blendColor.b = _thirdChannelWeight;
                blendTex.SetPixel(j, k, blendColor);
```
接着在shader中计算：
```
				//将贴图中被压缩到0,1之间的Index还原
				float indexLayer1 = floor((tex2D (_IndexTex, i.uv.zw).r * 15));
				float indexLayer2 = floor((tex2D (_IndexTex, i.uv.zw).g * 15));
				float indexLayer3 = floor((tex2D (_IndexTex, i.uv.zw).b * 15));

				//利用Index取得具体的贴图位置
				float4 colorLayer1 = GetColorByIndex(indexLayer1, lodLevel, i.worldPos.xz);
				float4 colorLayer2 = GetColorByIndex(indexLayer2, lodLevel, i.worldPos.xz);
				float4 colorLayer3 = GetColorByIndex(indexLayer3, lodLevel, i.worldPos.xz);

				//混合因子，其中r通道为第一层贴图所占权重，g通道为第二层贴图所占权重，b通道为第三层贴图所占权重
				float3 blend = tex2D (_BlendTex, i.uv.xy).rgb;
				half4 albedo = colorLayer1 * blend.r + colorLayer2 * blend.g + colorLayer3 * blend.b;
```

最终效果：

![](/img/in-post/terrain-atlas/24.png)
<small class="img-hint">三层混合</small>

以上即为全部内容，Demo在[Github](https://github.com/ak-TechArtist/TerrainAtlas)上



































