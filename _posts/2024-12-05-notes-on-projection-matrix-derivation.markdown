---
layout: post
title: 透视投影矩阵推导速记
tags: render perspective-projection-matrix
mathjax: true
---

# 投影矩阵推导速记

投影矩阵的推导是一个入门而且古老的话题。网上虽然有许多中文介绍，但大部分笔记都是有奇奇怪怪的错误的。现在老了记不住事情，每次用都要自己手推一下，不如留点速记。


## 投影矩阵推导

如果从几何角度来考虑，推导会变得相对简单，我们可以认为投影矩阵实际上是将一个 Frustumn 内的坐标，映射到一个标准立方体内。

![alt text]({{ site.url }}/assets/2024-12-05-notes-on-projection-matrix-derivation/x-y-projection.png)

后续我将以 DirectX 的左手坐标系和范围要求来推导的。由于 DirectX 要求最终坐标 $x,y \in [-1, 1],  z \in [0, 1]$。由此我们能得到关于 x 和 y 的基本变换。
设空间的点为 $v=(x,y,z)$ ， 变换后的点为 $v'=(x',y',z')$，那么有：

$$
\begin{align}
x' &= \cfrac{x}{W} \\
y' &= \cfrac{y}{H}
\end{align}
$$

由于锥体的缘故， W 和 H 是会随着 z 的变化而变化的。我们可以使用相似三角形的知识，推导出 W H。

![x-y-derivation]({{ site.url }}/assets/2024-12-05-notes-on-projection-matrix-derivation/x-y-derivation.png)

假设进近裁剪平面的的W = width， H=height，我们可以得到如下比例关系：

$$
\begin{align}
\cfrac{W}{width}  &= \cfrac{z}{near} \ \rightarrow \ W = z\cfrac{width}{near} = z \cdot tan(\cfrac{fov}{2})\\
\cfrac{H}{height} &= \cfrac{z}{near} \ \rightarrow \ H = z\cfrac{height}{near} = z \cdot r \cdot tan (\cfrac{fov}{2})
\end{align}
$$

把 (3)(4) 带回到 (1)(2) 里可以得到：

$$
\begin{align}
x' &= \cfrac{x}{W} = \cfrac{1}{z}\cfrac{x}{tan(\cfrac{fov}{2})}\\
y' &= \cfrac{y}{H} = \cfrac{1}{z}\cfrac{y}{r \cdot tan(\cfrac{fov}{2})}
\end{align}
$$

在推导 z' 之前，需要插入一下透视除法的讨论。因为GPU光栅化时时需要 z 进行对一些 attributes 比如纹理坐标进行透视矫正的。所以GPU是希望在输入的数据里保留z值的；而且看(5)(6) 公式，用矩阵式没办法表示 $\cfrac{1}{z}$ 的，所以 DirectX 想了个办法，对 $v'=(x',y',z',1.0)$ 同时乘上 z，得到 $v_c=(zx',zy',zz',z)$，这相当于是在 w 分量里存储 z 值，并且消除了xy分量中的 $\cfrac{1}{z}$。

API直接接受在做完矩阵乘法变换的 $v_c$ 作为输入，也就是我们常说的变换到 clipspace 坐标。在光栅化时，通过硬件计算计算除法： $v'= v_c \cdot \cfrac{1}{v_c.w}$ 变换到最后的 NDC 坐标。这里的 $\cfrac{1}{z}$ 就是上文推导出的分母：W, H 根据 z 越大而变大，产生近大远小的效果。我们称这一步除法为透视除法。由此，我们可以直接根据方程组构造出矩阵形式：
$$
\begin{align*}
x_c &= z * x' = \cfrac{1}{tan(\cfrac{fov}{2})}*x + 0*y + 0*z + 0 \\
y_c &= z * y' = 0*x +\cfrac{1}{r\cdot tan(\cfrac{fov}{2})} * y +  0*z + 0 \\
z_c &= z * z' = 0*x + 0*y + A*z + B\\ 
w_c &= z = 0*x + 0*y + 1*z + 0
\end{align*}
$$

$$
\Rightarrow \\
\begin{bmatrix} x_c \\ y_c \\ z_c \\ w_c \end{bmatrix} = 
\left(\begin{matrix} 
\cfrac{1}{tan(\cfrac{fov}{2})} & 0 & 0 & 0 \\ 
0 & \cfrac{1}{r \cdot tan(\cfrac{fov}{2})} & 0 & 0 \\ 
0 & 0 & A & B \\ 
0 & 0 & 1 & 0
\end{matrix}\right) \cdot
\begin{bmatrix} x \\ y \\ z \\ 1.0 \end{bmatrix}
$$

现在我们来推关于 z' 的 A B 两个数据。从上文的讨论中，我们知道 $z_c = z' \cdot z$，(透视除法之前）于是可以得到如下线性变化。

$$
\begin{align}
z' = \cfrac{z_c}{z} = \cfrac{Az+B}{z} = A + \cfrac{B}{z}
\end{align}
$$

然后把范围边界(near,0.0), (far,1.0)代入求解： 


$$
\begin{align*}
z'_{min} &= A + \cfrac{B}{n} = 0.0 \\
z'_{max} &= A + \cfrac{B}{f} = 1.0 \\
&\Rightarrow A=\cfrac{f}{f-n}, \ B=-\cfrac{nf}{f-n}
\end{align*}
$$

到这里我们的透视投影矩阵就能得到结果了：

$$
\left(
\begin{matrix}
\cfrac{1}{tan(\cfrac{fov}{2})} & 0 & 0 & 0 \\ 
0 & \cfrac{1}{r \cdot tan(\cfrac{fov}{2})} & 0 & 0 \\ 
0 & 0 & \cfrac{f}{f-n} & -\cfrac{nf}{f-n} \\ 
0 & 0 & 1 & 0
\end{matrix}\right)
$$

不管是其他 NDC 范围或者 OpenGL，你都可以通过这个方法推出透视投影的矩阵结果。指的注意的是，如果你在用右手坐标系的情况下（OpenGL），在视空间中，z是向负方向变大的，所以需要绝对值 $w=-z$，这会让 m43 = -1，而不是上文中的 1.

*p.s. 现在很多游戏会将 fov 设为竖直方向夹角，所以 aspect 会在 m11 的分母上*

## Non-Linear-Zc and Reverse-Z
update later.

可以从等式(7)观察到， 最后的 z' 值是和视空间的 $\cfrac{1}{z}$ 线性相关的。所以随着视空间的物体 z 线性变化时，depth buffer z 并非线性增长。这会带来精度相关的问题，离屏幕越近的物体，精度相对较高，但是远处的物体深度精度较低。

![alt text]({{ site.url }}/assets/2024-12-05-notes-on-projection-matrix-derivation/z-precision.png)

如图所示，看左侧刻度，当深度在视空间线性增大时，depth buffer z 在近处获得更多的有效数值。（底部刻度）

所以现在游戏引擎常用的做法是使用 Reverse-Z 来存储深度：深度范围从[0,1] 变化为 [1, 0]，同时将远平面设为无限远。
无限远？那投影矩阵怎么设置？

$$
\begin{align*}
z'_{min} &= A + \cfrac{B}{n} = 1.0 \\
z'_{max} &= A + \cfrac{B}{\infty} = 0.0 \\
&\Rightarrow A= 0, \ B = n
\end{align*}
$$

That's it. 那今天就先聊到这吧，我们下次见！

## References

[1] OpenGL Projection Matrix, <https://www.songho.ca/opengl/gl_projectionmatrix.html><br/>
[2] Depth Precision Visualized, Nathan, NVidia, <https://developer.nvidia.com/content/depth-precision-visualized>