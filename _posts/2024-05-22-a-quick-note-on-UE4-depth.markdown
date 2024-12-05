---
layout: post
title: 关于 UE4 深度的速记
tags: render UE4 depth-buffer projection
mathjax: true
---

# 关于 UE4 深度的速记

这几天看 UE4 代码，关于使用缓存深度重建世界坐标的代码。发现有些诡异，所以重新推导了一下。

## 问题

可疑代码是这样的：

```
float SampleDepth = DepthBuffer.Sample(...);
float SceneDepth = ConvertFromDeviceZ(SampleDepth);
float2 SceenPosition = Input.ScreenPosition.xy / Input.ScreenPosition.w;
float3 WorldPosition = mul(float4(ScreenPosition * SceneDepth, SceneDepth, 1), View.ScreenToWorld).xyz;
```

我的问题出在这一行代码：
```
ClipPosition = float4(ScreenPosition * SceneDepth, SceneDepth, 1);
```

我认为空间变换时，z component 不应该等于 SceneDepth。从投影矩阵的定义来看，应该是形如 $Az+B$，其中AB分别为：

```
float A = ProjectMatrix_33;
float B = ProjectMatrix_34;
float Z = A * SceneDepth + B;
```

如果我们回顾一个顶点通过投影矩阵转换到裁剪空间的过程。先考虑投影矩阵 ProjectionMatrix 的构成，大致形式如下：

$$
M_{porj} =
\begin{pmatrix}
P & 0 & 0 & 0 \\
0 & Q & 0 & 0 \\
0 & 0 & A & B \\
0 & 0 & 1 & 0
\end{pmatrix}
$$

那么我们就可以通过这个矩阵得出顶点变换的式子

$$
V_{clip}(x', y', z', w')  = M_{project} * V_{view}(x, y, z, 1) = (Px, Qy, Az+B, z)
$$

然后再通过透视除法转换到 NDC，
$$
V_{NDC}=(\cfrac{x'}{z}, \cfrac{y'}{z}, \cfrac{z'}{z}, 1)\\
$$

我们从 depth buffer中采样获得的 `SampleDepth=z'`，并将 `z'` 通过`ConvertFromDeviceZ()`转换得到 `z` 。如果我们需要拼凑 clip space 的齐次坐标，应该为：$V_{clip}=(xy*z, z'*z, z)$; 但是 UE 在 Z Component 上写的是 1.0 * z，可以认为 z'=1.0，那应该是远平面呀？而且w=1.0 是冲突的？

## 调查

这里犯了两个错误：其一是我混淆了几个不同的 z 数据；其二是 UE4 用的是 Reversed-Z， 所以 1.0 应该是近平面。接下来就记录一下调查结果。

为了保持和上文一致，我们
* 用 z 来表示 Viewspace linear Z，被投影矩阵变换之前的 z 值
* 用 z' 来表示 Clipspace linear Z，被投影矩阵变换之后的 z 值
* 用 SampleDepth 表示从深度缓冲区里读取出来的 z 值

那么深度缓冲里保存的值应该等于：

$$
SampleDepth = \cfrac{z'}{z}
$$

这是一个复合值，我们可以反算出 z 吗？可以的。从投影矩阵中我们可以得到变换公式

$$
z' = Az + B \rightarrow SampleDepth=\cfrac{z'}{z} = A + \cfrac{B}{z}
$$

据此，我们可以直接通过透视投影矩阵的 A，B 算出 z (viewspace linear z)：

$$
z = \cfrac{B}{SampleDepth - A} = \cfrac{1}{ \cfrac{1}{B}SampleDepth - \cfrac{A}{B} }
$$

这个也是 `ConvertFromDeviceZ(x)` 的实现，感兴趣的小伙伴可以查阅相关函数 `CreateInvDeviceZToWorldZTransform` [[1](#1)<a name="ref-1"></a>] 确认变换逻辑。

所以，我们需要的Z Component 应该是 $z'*z = Az+B$ 和实际上的写着的 z 有出入。是不是哪里错了？如果 scene depth z 表示的是 $Az+B$ ，那么我们不能用他对着 component xy 相乘？

## 猫腻

经过以上推导我们明确了 XYZ 分量应该的值。经过探查发现实际上是 `ScreenToWorld` 矩阵有猫腻。 这个矩阵的实现如下：

```
ViewUniformShaderParameters.ScreenToWorld = 
    FMatrix(
	    FPlane(1, 0, 0, 0),
	    FPlane(0, 1, 0, 0),
	    FPlane(0, 0, ProjectionMatrixUnadjustedForRHI.M[2][2], 1),
	    FPlane(0, 0, ProjectionMatrixUnadjustedForRHI.M[3][2], 0)
    )
    * InViewMatrices.GetInvViewProjectionMatrix();
```
可以看到除了 `Inverse(MatrixViewProject)` 之外，我们还有一个特殊的矩阵。由于UE 的向量是行向量，矩阵放在乘号右侧，所以我们看的时候需要脑内转置一下。

这个矩阵实际上就是 ProjectMatrix 抠出来的：
$$
\begin{pmatrix}
    1 & 0 & 0 & 0 \\
    0 & 1 & 0 & 0 \\
    0 & 0 & A & B \\
    0 & 0 & 1 & 0
\end{pmatrix}
$$

经过这个矩阵的变换，我们的构造的特殊矩阵就变成了：
```
  //Matrix * (x,y,z,1) = (x,y,Az+B,z) 
  //==>
  Matrix * (ScreenPos.xy * SceneDepth, SceneDepth, 1)
= (ScreenPos.xy * SceneDepth, A*SceneDepth+B, SceneDepth)
```

**Bingo!**

实际上我一开始是清醒的，但是我对投影矩阵关于 1/z 的线性有一点模糊，所以在没仔细推断的情况下一下蒙蔽了。手算一遍就好了。

## Reversed-Z

对于屏幕空间是关于 1/z 线性的这个话题，我还有一点模糊。我只知道 NDC 确实都需要做透视除法，所以他们就关于 1/z 线性了？在线性的基础上，可以直接做 Mipmaps，Screenspace Ray Tracing 之类的，所以还是分方便的。

关于 Reversed-Z，其充分利用了浮点数的效率。这个可以在 Nathan 的笔记[[2](#2)<a name="ref-2"></a>] 里确认到。

除此之外要使用上 Reversed-Z 还需要把材质里关于深度测试的 LessEqual 改成 GreaterEqual。

查阅了 RTR4 发现已经是标准操作了，看样子这几年确实拉下很多。这两个话题我弄清楚后下次讨论。

<a name="1"></a> [[1](#ref-1)] line 535, Unreal/Engine/Source/Runtime/Engine/Private/SceneView.cpp, <https://developer.nvidia.com/content/depth-precision-visualized><br/>
<a name="2"></a> [[2](#ref-2)] Depth Precision Visualized, Nathan, NVidia, <https://developer.nvidia.com/content/depth-precision-visualized>