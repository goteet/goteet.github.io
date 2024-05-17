---
layout: post
title: 射线追踪(3) 蒙特卡罗与路径追踪
tags: rendering ray-tracing monte-carlo
mathjax: true
---

# 射线追踪(3) 蒙特卡罗与路径追踪

蒙特卡罗射线追踪实际上是稍微有些模棱两可的术语。

大多数情况下，它指代的是路径追踪，一个由 Kajiya 在 1986 年提出的改进的射线追踪方法。该方法使用了蒙特卡罗积分来解算渲染方程。实际上对于大多数的现代渲染技术，比如路径追踪，双向路径追踪，渐进式光子映射，VCM(Vertex Connection and Merging)等等，都可以归类为蒙特卡罗积分光线追踪技术。所以应该刻意的避开蒙特卡罗射线追踪这样有歧义的词语。

蒙特卡罗积分通过在函数定义域上随机采样，并计算函数样本的平均值，来评估积分的近似解。通过这种方法，我们可以用少量的计算去求解一个复杂的，不易计算的积分式。


## 射线追踪的发展历程

从时间线来开， Turner Whitted 在其论文 An improved illumination model for shaded display 中开创性的引入了射线追踪渲染的方法。我们将最初的射线追踪成为 Whitted-Style ray tracing。

这种方式的线追踪在遇到物体表面时，会向光源方向投射 Shadow Ray，并向反射方向、折射方向投射新的射线。这让画面效果显得更真实，但是缺少了间接光照产生的漫反射效果，或者常说的 GI。

![whitted style ray tracing]({{ site.url }}/assets/2024-04-18-ray-tracing-3-monte-carlo-and-path-tracing/whitted_style_ray_tracing.png)

随后在 1984年， Robert L. Cook 根据蒙特卡罗的思路，在时间和空间上随机投射射线，以实现动态模糊、半高光折射、面积光和景深的效果。

当第一次投射的观察射线相交于物体表面之后，蒙特卡罗的方式要求其投射大量的随机射线。于是我们将其称为 Distributed ray tracing。

而每一次随机射线再次命中物体时，就会指数地产生新的射线。假设每次射线相交后，生成 N 个射线，则第 X 次相交时，射线的数量将会变为  $N^X$ 。假设N=10，X=5，则我们一个像素中射出的观察射线就已经产生了 10 万根射线。而其中有可能只有 20~30 个有效射线，会对最终画面提供有效颜色信息，大部分时间我们都浪费在对画面影响很小的计算上。

![distributed ray tracing]({{ site.url }}/assets/2024-04-18-ray-tracing-3-monte-carlo-and-path-tracing/distributed_ray_tracing.png)

光源射出的射线，只有极少数会射入相机的镜头中。这张图是从光源角度出发的，和我们讨论的情况正好想法。不过作为示意，从光路是可逆的这一角度出发，也可以知道反之亦然。

在2年以后， Kajiya 在1986年发表了渲染方程，并提出了路径追踪的方法。路径追踪解决了之前射线会随着反射次数指数型增长的问题。

在每次反射时，只会产生一个采样方向，更少的射线极大的减少了计算量，大量减少了因为无效方向带来的噪点。同时还提出来的 unbias 的评估可以获得更准确的结果。

![path tracing]({{ site.url }}/assets/2024-04-18-ray-tracing-3-monte-carlo-and-path-tracing/path_tracing.png)

现在，分布式射线追踪已经不是指代原本的算法了，更多的是动态模糊、景深、软阴影等不能用一个采样的射线追踪器实现的效果联系在一起。而路径追踪也更多的是指那些使用渲染方程来实现GI的方法，以便和光子映射方案进行区分。实际上大部分的路径追踪器在每次遇到物体表面时，已经不再是只产生一个采样方向了。

## 渲染方程与蒙特卡罗积分

我们之前提到过渲染的关键问题是计算 $L_o$，沿着视线反方向的，来自于物体表面的能量辐射。这个辐射在 Kajiya 1986 中定义的渲染方程来描述：

$$
L_o(x, \omega_o) = L_e(\omega_o) + \int_{\Omega}{f_r(x, \omega_i \to \omega_o) L_i(x, \omega_i) (N \cdot R(\omega_i))^+d\omega_i}
$$

这个式子说明了三个有趣的问题。

1. 光的能量是线性叠加的，这个我们在之前的文中聊过。
2. 我们观测的是沿着某一方向的能量辐射，分为两部分，其中一部分来自于物体自己发光。
3. 最后一部分半球积分表示的物体的反射能量。

我们简单来说说积分部分， 其中的 $\Omega$ 是物体表面的 shading point x 以 N 为法线的半球。主要是因为背面照射物体表面是没办法反射的原因。所以我们只积分半球。在半球上寻找有能量的立体角进行积分。

![hemisphere integration]({{ site.url }}/assets/2024-04-18-ray-tracing-3-monte-carlo-and-path-tracing/hemisphere_integration.png)

物体在半球的接收到入射能量后，根据其表面特性，以 f() 确认有多少可以沿着 $R(\omega_o)$ 方向反射能量。这个 f 就是我们熟悉的 BRDF，不过它的定义实在是有点别扭。考虑最终量纲 L 是 $J \cdot m^{-2} \cdot sr^{-1}$，这里的 $f$ 量纲应该为 $1/sr$

> 毕竟 $E =\int{Ld\omega}$ 的量纲是 $J/m^2$


同时我们在之前的文里也有提到过，由于我们有意识的忽略介质的影响。所以有

$$
L_i(x, \omega_i) = L_o( p(x, \omega_i), -\omega_i)
$$

这个等式的，x 的入射能量 radiance，等于另一个物体的出射能量。而出射的点就是射线从 x 点出发，沿着方向 $R(\omega_i)$ 相交到的另一个物体 x' 上。使得 Rendering Equation 形成了一个递归式。

### 蒙特卡罗积分

很明显，这个半球积分是通过场景物体之间的关系来定义的。我们没办法知道其解析解。在这种情况下，使用蒙特卡罗积分方法计算其数值积分成为很方便的一种选择。

蒙特卡罗是一种非确定性算法。简单来说我们通过随机数来对积分式进行采样。通过将随机值带入到积分式中获得采样结果，并更具多次采样结果对原积分进行一个近似值，作为其评估结果。

带入到这个积分式求解的问题中，我们要做的就是在半球上均匀分布的生成随机方向 $W$。然后将随机采样 $W_i$ 带入到积分式的被积分式中

$$
L'(\omega) = f_r(x, \omega \to \omega_o) L_i(x, \omega) (N \cdot R(\omega))^+
$$

并得到一系列的样本结果：

$$
\begin{align*}
L'(W_0) = f_r(x, W_0 \to \omega_o)L_i(x, W_0)(N \cdot W_0)^+
\\
L'(W_1) = f_r(x, W_1 \to \omega_o)L_i(x, W_1)(N \cdot W_1)^+
\\
L'(W_2) = f_r(x, W_2 \to \omega_o)L_i(x, W_2)(N \cdot W_2)^+
\\
...
\end{align*}
$$

然后我们将其加权平均，就可以得到半球积分的评估值了。

$$
\begin{align*}
\int_{W \in \Omega}{L'(\omega)d\omega} 
&\approx \cfrac{1}{N}\sum_{i=0}^{N}{L'(W_i)}
\\
&\approx \cfrac{1}{N}\left(L'_0 + L'_1 + L'_2 + ... + L'_{N-1}\right)
\end{align*}
$$

通过这个快速累加，我们就可以算出最终的 $L_o$ 以确认图像的颜色。在接下来的研究中，我们的目标就是如何使用更少的采样，用更快的速度，以尽可能小的误差获得这个近似解。

## 路径追踪

路径追踪可能需要一些更深入的知识才能表述的清楚。但是在深入探究之前，我们在这里从解递归的角度入手，聊聊其思路和概念。

路程路径的前提是我们每次遇到表面时，只发射一根随机射线。并且从摄像机角度开始，渐进式的构建这条路径。


![path construction]({{ site.url }}/assets/2024-04-18-ray-tracing-3-monte-carlo-and-path-tracing/path_construction.png)

由于只会有考虑一个采样方向，积分符号被消除。这意味着，积分式 
$$
\int f(\omega)L_i(\omega)cos\theta_{\omega} d\omega 
= f(W_i)L_i(W_i)cos\theta_i
$$

我们在上文提到 $L_i(x, \omega) = L_o(p(x,\omega), -\omega)$，其中的 $p(x,\omega)$ 是从 $x$ 点向 $\omega$ 方向发射射线，并且碰到下一个物体表面 $x'$。在下文中，我们就用下标 $x_0, x_1, ...$ 来指代以此碰撞的物体表面。其中 $x_0$ 是摄像机位置。把这个式子带入到渲染方程中，那我们就会得到一个新的等式

$$
\begin{align*}
L_o
&= L_e + \int fL_icos\theta d\omega = L_e + fL_icos\theta
\\
&= L_{e_1} + f_1L_{i_1}cos\theta_1 
 = L_{e_1} + f_1\left(L_{e_2} + f_2L_{i_2}cos\theta_2\right)cos\theta_1
\\
&= L_{e_1} + f_1L_{e_2}cos\theta_1 + f_1f_2L_{i_2}cos\theta_1cos\theta_2
\\
...
\\
&= L_{e_1} + f_1L_{e_2}cos\theta_1 + f_1f_2L_{e_2}cos\theta_1cos\theta_2 + f_1f_2f_3L_{e_3}cos\theta_1cos\theta_2cos\theta_3 + ...
\end{align*}
$$

那什么时候是个头呢？两种情况这个递归等式会终止。

第一种，我们遇到了光源，为了计算的简单，我们一般认为光源就不在计算其反射项，只计算其 $L_e$， 自然而然就终止了。

另一种情况，则为系数 $f_1*f_2*....*f_n*cos\theta_1 * cos\theta_2 * ...cos\theta_n \le \epsilon$ 的时候，我们认为 $L_i$ 可以忽略不记。

正常情况下，如果这条路径没有遇到光源，一般在第4、5级的时候，材质的反射系数 BRDF 累乘起来就几乎认为是终止了。

### 渐进式构建

为了表达方便，我们需要定义一些额外函数和额外的操作。

先定义 $G(i) = f_i * cos\theta_i$， 这样我们的系数可以写成：

$$
f_1f_2f_3L_{e_3}cos\theta_1cos\theta_2cos\theta_3
= L_{e_3} \cdot \prod_{k=1}^{3} G(k)
$$

自然而然的，我们可以把加法多项式中每一项都用 W 来表示：
$$
W(0) = 1, \ \ \ W(i) = \prod_{i=1}^{i}G(i)
$$

这么写不仅最终的多项式可以改写成：

$$
\begin{align*}
L_o &= L_{e_0}W(0) + L_{e_1}W(1) + L_{e_2}W(2) + ...
\\
&= \sum_{i=0}^{\infty} {L_{e_i} \cdot W(i) }
\end{align*}
$$

而且由于累乘的特性，我们可以得到如下等式

$$
W(i+1) = W(i) * G(i+1)
$$

这样，我们就可以把递归的形式改写成 for 循环的形式了。

```
let W = 1;
let Lo = 0;
let x_i = eye;
for(i = 1 -> n)
{
    let omega_i = random_direction();
    x_i = ray(x_i, omega_i).test(scene);
    Lo +=  Le(x_i, omega_i) * W;
    W *= brdf(x_i, omega_i) * dot(omega_i, Normal(x_i));
}
```

实际上在这个概念里我们忽略了很多有趣的系数，我这纯粹是照猫画虎做了个简化版的推导。篇幅有限，今天就先聊到这，我们下次再详细展开。

[1] Kajiya86, The Rendering Equation, <https://www.cs.cmu.edu/afs/cs/academic/class/15462-s13/www/lec_slides/86kajiyaRenderingEquation.pdf>