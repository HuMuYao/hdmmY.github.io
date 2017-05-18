# 高亮效果 --- Unity 特效

---
[1.什么是相交高亮(Intersection Highlight)](#1)

[2. 相交高亮着色器的工作原理](#2)

[3. 首先，明白我们要做什么](#3)

[4. Depth Buffer 的相关预备知识](#4)

[5. 如何获取一个片元所在屏幕位置的DepthBuffer](#5)

[6. 千呼万唤始出来的 Vertex Shader](#6)

[7. 需要调整数值的 Fragment Shader](#7)

[8. 最终效果](#7)

<h3 id='1'> 什么是相交高亮(Intersection Highlight) </h3>

相交高亮, 是一种附加在Mesh上的着色器特效, 其功能是将所有其他穿过该Mesh表面的截面轮廓绘制出来, 产生一种类似于扫描一样的效果。 多用于科幻类游戏中。

![image_1bgcjimpgiqi3o629pbvn9vm.png-257.1kB][1]

<h3 id='2' > 相交高亮着色器的工作原理 </h3>

获取当前摄像机渲染的场景的DepthBuffer, 在渲染当前模型的时候判断每一个经过坐标变换的片元(fragment shander 中的点)的世界坐标Z是否和DepthBuffer的对应点深度足够接近. 如果足够接近, 则将其渲染成另一种颜色.

这样，就将两个物体的接触表面高亮了，如上图中的黄绿色的高亮线。

<h3 id='3' > 首先，明白我们要做什么 </h3>

仔细观察上面那张图，我们会发现，除了那个**黄色的幕布**，其他的都很正常。我们要实现的，就是那个黄色的幕布的Material。

从图中我们可以清楚的看出，那个幕布后面的物体在幕布外是非常正常的，在幕布内，它们被蒙上了一层黄色，但是**和幕布相交截面的外轮廓被绘制成了黄绿色**。

我们怎么做到这个效果呢？

1.[**Blend Mode**](https://docs.unity3d.com/Manual/SL-Blend.html)为: Blend SrcAlpha OneMinusSrcAlpha 
2. [**RenderQueue**](https://docs.unity3d.com/Manual/SL-SubShaderTags.html)为Transparent 很明显, 我们当然是希望这个正方体在Geometry后渲染出来, 这样才能透过它看到优先渲染的Opaque Materials. 
3. 正因为我们的正方体是被后渲染出来的, 所以我们可以通过当前的[**DepthBuffer**](https://docs.unity3d.com/Manual/SL-DepthTextures.html)等资源来以某种方式处理相交截面。
4. 但是不管截面到底是怎么被处理出来的, 我们必须得知道**屏幕上某点的世界坐标相对于正方体某个片元的世界坐标的相对关系。** 

<h3 id='4' > Depth Buffer 的相关预备知识 </h3>

**Depth Buffer(深度缓冲区)**：Depth Buffer 是一个有着和你的目标有相同长宽的 Buffer，它记录了每一个需要渲染的像素的深度。
每一个像素的深度是由你的Camera的 view 和 projection 矩阵决定的(这个也不会的童鞋，可以看[这里](https://learnopengl-cn.readthedocs.io/zh/latest/01%20Getting%20started/08%20Coordinate%20Systems/))，当一个像素点在 projection plane 的 near 端时，它的深度值是0；当一个像素点在 projection plane 的 far 端时，它的深度值是1。

在Unity中，你想要得到 Depth Buffer，你必须去使用 [Render Texture](https://docs.unity3d.com/Manual/SL-CameraDepthTexture.html)。 Render Texture 是一种特殊的材质，它从Camera产生，每帧都更新一次，你可以用它来实现一些后处理效果。

在我们这次的例子中，我们需要 Camera 产生的 **Depth Texture(深度图) 中的深度信息**。

要获得 Depth Texture，我们首先要设置 [DepthTextureMode = DepthTextureMode.Depth](https://docs.unity3d.com/Manual/SL-CameraDepthTexture.html)，也就是告诉 Camera，我们需要你产生 Depth Texture，而不是其他的 Texture。注意，默认情况下，Camera 是不会产生任何 Texture的。

我们可以写一个脚本挂载到 Camera 上，来设置 Camera.DepthTextureMode：

```c#
using UnityEngine;

namespace TestShader
{
    [ExecuteInEditMode]
    public class SetCameraTextureMode:MonoBehaviour
    {
        public DepthTextureMode m_depthMode = DepthTextureMode.None;

        private DepthTextureMode _prevDepthMode = DepthTextureMode.None;
        private Camera _camera;

        private void Start()
        {
            _camera = GetComponent<Camera>();

            _camera.depthTextureMode = DepthTextureMode.None;
        }

        private void Update()
        {
            if(_prevDepthMode != m_depthMode)
            {
                _camera.depthTextureMode = m_depthMode;

                _prevDepthMode = m_depthMode;
            }
        }
    }
}
```
![image_1bgcmr8velkr1vnv1aei1s6g1jk513.png-28.6kB][2]

根据 [Unity官方文档](https://docs.unity3d.com/Manual/SL-CameraDepthTexture.html)：

    Depth textures are available for sampling in shaders as global shader properties. By declaring a sampler called _CameraDepthTexture you will be able to sample the main depth texture for the camera.

    _CameraDepthTexture always refers to the camera’s primary depth texture. By contrast, you can use _LastCameraDepthTexture to refer to the last depth texture rendered by any camera. This could be useful for example if you render a half-resolution depth texture in script using a secondary camera and want to make it available to a post-process shader.

我们可以知道，通过在 Shader 中声明一个新变量：*_CameraDepthTexture*，就可以用它来引用 Camera 产生的 Depth Texture了。

<h3 id='5' > 如何获取一个片元所在屏幕位置的DepthBuffer </h3>

首先，我们怎么获得一个片元的投影坐标呢？下面这段代码你应该是看烂了：

    o.vertex = mul(UNITY_MATRIX_MVP, input.vertex);

这样我们取得的是 *output.vertex* 的世界坐标，那么我们怎么把它转换到[0,1]的区间呢，毕竟这样我们才能使用 Depth Buffer。不用担心，Unity已经帮我们[做好了](https://docs.unity3d.com/Manual/SL-BuiltinFunctions.html)：
| 函数| 功能 | 
|:--:|:--:|
| float4 ComputeScreenPos (float4 clipPos) | Computes texture coordinate for doing a screenspace-mapped texture sample. Input is clip space position. | 

在获得 output.vertex 映射到[0,1]上的坐标后，我们需要得到当前点对于相机的距离。正好，Unity的 ["UnityCG.cginc include file"](https://docs.unity3d.com/Manual/SL-BuiltinIncludes.html)中有一个宏定义可以满足我们的要求：(也可以看[这里](https://docs.unity3d.com/Manual/SL-DepthTextures.html))

| 宏| 格式 | 说明 |
|:--:|:--:|:--:|
| fUNITY_OUTPUT_DEPTH | UNITY_OUTPUT_DEPTH(o) | 计算 vertex 的 eye space depth 并输出到 o|

<h3 id='6' > 千呼万唤始出来的 Vertex Shader </h3>

```  UnityShader
vertexOutput vert(vertexInput v)
{
	vertexOutput o;

	o.vertex = mul(UNITY_MATRIX_MVP, v.vertex); //得到片元的投影坐标

	o.projPos = ComputeScreenPos(o.vertex);   // 得到片元投影坐标映射到(0, 1)上后的坐标
	COMPUTE_EYEDEPTH(o.projPos.z);            // 计算这个点相对Camera的距离

	o.texcoord = TRANSFORM_TEX(v.texcoord, _MainTex); // 更加精准的匹配texture

	return o;
}
```

<h3 id='7' > 需要调整数值的 Fragment Shader </h3>

我们的思路：

1. 先计算当前点对于屏幕的距离 partZ --- 其实这个在 Vertex Shader 中已经做完了。
2. 再计算深度缓冲区 Depth Buffer 中的在屏幕上位置一样的那个点的深度 sceneZ。
3. 如果两者相近，则绘制高亮图案（说明这个是位于轮廓线上的点）；否则，绘制正常的图案。

实际的 Fragment Shader 代码：
```  UnityShader
sampler2D _CameraDepthTexture;   // Depth Texture -- 深度信息Texture
float _InvFade;      // 一个控制参数

fixed4 frag(vertexOutput i) : COLOR
{
	float partZ = i.projPos.z;
	float sceneZ = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE_PROJ(_CameraDepthTexture, UNITY_PROJ_COORD(i.projPos)));

	float fade = saturate(_InvFade * (abs(sceneZ - partZ))); // 片元坐标的深度和Depth Buffer中的深度越接近，fade(消散程度)越小

	_HighLightColor.a = 1 - fade; // fade(消散程度)越小，描绘轮廓的线越清楚

	fixed4 highlightCol = 2.0 *  _HighLightColor;                       // 轮廓线颜色
	fixed4 tintCol = 2.0 *  _TintColor * tex2D(_MainTex, i.texcoord);   // 正常的颜色

	fixed4 col = lerp(highlightCol, tintCol, fade);  // fade(消散程度)越小，颜色越偏向轮廓线的颜色

	return col;
}
```

<h3 id='8' > 最终效果 </h3>

1. 我的 Material 面板：
    ![image_1bgcrosm93fmv4j1f8117liln816.png-25kB][3]

2. 将这个材质添加到一个Quad上：
    ![image_1bgcrqj8p16lettv7oej7mnhu1j.png-49.4kB][4]

3. 设置 Tint Color 和 Highlight Color，调整 Soft Factor
    1. 不加 Intersection Texture时：
    ![image_1bgcs4roj1vs512i7o9rlej1bb920.png-744.6kB][5]
    2. 加 Intersection Texture时：
    ![image_1bgcs9gv3npi19t9189s1nf71tga2d.png-876.6kB][6]

<h3 id='8' > Shader 源代码 </h3>

```   UnityShader
Shader "Test/IntersectionHightlight" {
	Properties
	{
		_TintColor("Tint Color", Color) = (0.5,0.5,0.5,0.5)
		_HighLightColor("Highlight Color", Color) = (0.5, 0.5, 0.5, 0)
		_MainTex("Intersection Texture", 2D) = "white" {}
		_InvFade("Soft Factor", Range(0.1,10.0)) = 1.0
	}

		

	SubShader
	{
		Tags
		{ 
			"RenderType" = "Transparent"
			"Queue" = "Transparent" 
			"IgnoreProjector" = "True" 
		}
		
		Blend SrcAlpha OneMinusSrcAlpha
		ColorMask RGB
		Cull Off 
		Lighting Off 
		ZWrite Off 

		Pass{

			CGPROGRAM

			#pragma vertex vert
			#pragma fragment frag

			#include "UnityCG.cginc"

			sampler2D _MainTex;
			fixed4 _TintColor;
			fixed4 _HighLightColor;

			struct vertexInput 
			{
				float4 vertex : POSITION;
				float2 texcoord : TEXCOORD0;
			};

			struct vertexOutput {
				float4 vertex : SV_POSITION;
				fixed4 color : COLOR;
				float2 texcoord : TEXCOORD0;
				float4 projPos : TEXCOORD1;
			};

			float4 _MainTex_ST;

			vertexOutput vert(vertexInput v)
			{
				vertexOutput o;
		
				o.vertex = mul(UNITY_MATRIX_MVP, v.vertex); //得到片元的投影坐标
				 
				o.projPos = ComputeScreenPos(o.vertex);   // 得到片元投影坐标映射到(0, 1)上后的坐标
				COMPUTE_EYEDEPTH(o.projPos.z);            // 计算这个点相对Camera的距离
		
				o.texcoord = TRANSFORM_TEX(v.texcoord,_MainTex); // 更加精准的匹配 texture

				return o;
	        }

			sampler2D _CameraDepthTexture;   // Depth Texture -- 深度信息Texture
			float _InvFade;      // 一个控制参数

			fixed4 frag(vertexOutput i) : COLOR
			{
				float partZ = i.projPos.z;
				float sceneZ = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE_PROJ(_CameraDepthTexture, UNITY_PROJ_COORD(i.projPos)));

				float fade = saturate(_InvFade * (abs(sceneZ - partZ))); // 片元坐标的深度和Depth Buffer中的深度越接近，fade(消散程度)越小

				_HighLightColor.a = 1 - fade; // fade(消散程度)越小，描绘轮廓的线越清楚

				fixed4 highlightCol = 2.0 *  _HighLightColor;                       // 轮廓线颜色
				fixed4 tintCol = 2.0 *  _TintColor * tex2D(_MainTex, i.texcoord);   // 正常的颜色
				
				fixed4 col = lerp(highlightCol, tintCol, fade);  // fade(消散程度)越小，颜色越偏向轮廓线的颜色
				
				return col;
			}
		
			ENDCG
		}
	}
}

```

  [1]: http://static.zybuluo.com/HandY/tvm76oujt6o35d7fl2d8bko0/image_1bgcjimpgiqi3o629pbvn9vm.png
  [2]: http://static.zybuluo.com/HandY/vf4wcb96lda5dexgwkwymtsk/image_1bgcmr8velkr1vnv1aei1s6g1jk513.png
  [3]: http://static.zybuluo.com/HandY/uarqrekmfnzmr56l2cpr0rf7/image_1bgcrosm93fmv4j1f8117liln816.png
  [4]: http://static.zybuluo.com/HandY/xefkcv8rz4sc8d0d4arjomsf/image_1bgcrqj8p16lettv7oej7mnhu1j.png
  [5]: http://static.zybuluo.com/HandY/ka6kfaxa7n7mk84c2pfoj977/image_1bgcs4roj1vs512i7o9rlej1bb920.png
  [6]: http://static.zybuluo.com/HandY/juoak3dw89e5zwq9u0ny2olo/image_1bgcs9gv3npi19t9189s1nf71tga2d.png
