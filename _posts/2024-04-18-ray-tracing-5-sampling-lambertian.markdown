---
layout: post
title: 射线追踪(5) 采样 Lambertian BRDF
tags: rendering ray-tracing monte-carlo importance-sampling 
mathjax: true
---

# 射线追踪(5) 采样 Lambertian BRDF

看了这么多书，手痒痒了。今天试试实践一下重要性采样。

我们在光的建模里有提到，我们会把反射行为不同的光分为 Diffuse 和 Specular，那么反射方程我们可以拆开写成：

$$
L_o = \int_{\Omega}{ (f_d + f_s)L_icos\theta_i d\theta_i}
$$

今天我们不考虑 $f_s$，只聊聊 $f_d$。我们最常用的 $f_d$ Lambertian，虽然被证明是不太物理正确的，胜在他就是简单的常数。比起 Oran-Nayer 还是轻松不少。印象里 $f_d = \cfrac{\rho}{\pi}$, 分子的 rho 是 albedo，一般都是测量值。不过印象已经不深了。实际游戏里，都是靠美术提供的数值，我们更愿意直接用 $K_d $ 替代。

对于上式，我们列出评估式，还记得重要性采样的评估式吗？

$$
Q_d = \cfrac{1}{N}\sum{\cfrac{f_r(\omega_i)L_i(\omega_i)(cos\theta_i)^+}{pdf(\omega_i)}} 
= \cfrac{1}{N} \cdot\cfrac{\rho}{\pi} \sum {\cfrac{L_i(\omega_i)(cos\theta_i)^+}{pdf(\omega_i)} }
$$

常数项可以提出来，最终我们要考量的是积分式 $G(x)=L_i(x)cos(n\cdot x)^+$ 的分布情况。因为 $L_i$不可知，所以我们只能考虑 $cos\theta$，而且从实际情况来看，$\theta$越大的条件下，系数真的就很小呀。

所以我们选用余弦权重作为 $p$ 采样分布。考虑到全定义域的概率是 100%，向上次一样，我们仍然计算实际 $pdf$。先将其设为 $p=kcos\theta$，带入到半球积分里


$$
\int_{\Omega}kcos\theta d\omega = 1  \to p'=\cfrac{cos\theta}{\pi}
$$

这个还要演算么？ 

$$
k\iint{cos\theta sin\theta d\theta d\phi}
= k\iint{ cos\theta d(cos\theta)d\phi}
= k\int{\cfrac{cos^2\theta}{2}d\phi}
= k\pi \to k=\cfrac{1}{\pi}
$$

把 $p'$ 带回评估式里，我们可以直接消除掉 $cos\theta$，$\pi$ 得到如下结果

$$
Q_d = \cfrac{\rho}{N} \sum_i^N {L_i(\omega_i)}
$$

这下就简单多了对吧。至于平均采样，我们在(4) 的末尾里也有推过，这里就不重复展开了。但是，为了能说明如何按照分布来生成样本。我仍然要从均匀分布说起。

## 生成符合余弦分布的样本

考虑到半球的特殊性，我们如何在半球上生成均匀分布的方向？如果是两个 IID，我们可以将极坐标的参数 $\phi$，$\theta$ 分别采样。然而，你将会的得到这样的结果

![weighted circle samples]({{ site.url }}/assets/2024-04-18-ray-tracing-5-sampling-lambertian/weighted_circle_samples.png)

考虑 $d\omega=sin\theta d\theta d\phi$ ，这个 θ 并非是均匀变化的。点分布的环状在越靠近法线的方向就越小，所以随机点分布的越密集。[[1](#1)<a name="ref-1"></a>] 

### 反函数法

常见的方法是采用 Inverse Function Method。顾名思义，我们需要计算分布函数的反函数。然后把均匀随机变量 $\xi$ 带入其中，就可以得到符合 p 分布的随机变量 $\alpha$

$$
\alpha = P^-1(\xi)
$$

到这一步就很简单了，都是上面聊过的内容。不过对于立体角 $\omega$ 我们不太好直接积分，所以一般转化到球面坐标去计算：$\omega = R d\theta d\phi$。其中包含两个参数，对应的多参数的情况，二维联合概率分布如下：

$$
F(x,y)
= Prob\{X \le x \ and \ Y \le y \} 
= \int_{y_{min}}^{y}{\int_{x_{min}}^{x}{pdf(u,v)dudv}}
$$

这个积分看起来我们是没办法直接积分了。可以通过首先对某一个维度上进行全域积分，获得另一个维度上的边界分布函数 `Marginal cumulative distribution` 。

因为联合概率分布定义在 2D 空间 $[x_{min}, x_{max}] \times [y_{min}, y_{max}]$ 存在边界分布的定义

$$
F_X(x) = P(X \vert Y) = F(x,y_{max})
\\
F_Y(y) = P(Y \vert X) = F(x_{max},y)
$$

我们只需要把把 $y_{min}$ 和 $y_{max}$ 带入，就能得到 x 轴上的分布函数：

$$
F_X(x) = F(x, y_{max})=\int_{x_{min}}^{x}{\left( \int_{y_{min}}^{y_{max}}{pdf(u,v)du }\right)dv}
$$

$y$ 的上下界都是已知常量，括号内可以算出数值解的呀。我们用圆盘举例，圆盘极坐标是 $\omega=rd\theta dr$，均匀分布 $pdf=\cfrac{1}{\pi R^2}$，所以联合概率分布可以写成：

$$
F(r,\theta) 
= \int_{0}^{r}{\int_{0}^{\theta}{\cfrac{1}{\pi R^2}rd\theta dr}}
=\cfrac{\theta r^2}{2\pi R^2}
$$

同时边界概率分布还满足 Chain Rule： $F(x,y) = F(x \vert y) * F(y) = F(x) * F(y \vert x)$。根据定义，我们可以通过出发得到 y 轴上的边界概率分布函数：

$$
F_Y(y) = \cfrac{P(x,y)}{P(x \vert y)}
$$

求出这两个边界条件函数之后，我们只需要分别将 $\xi_1, \xi_2$ 带入其，就能得到符合 $pdf$ 的采样了

$$
\theta=F_X(\xi_1)
\\
\phi=F_Y(\xi_2)
$$

对应的圆盘边界概率分布为：

$$
F_R(r)
= F(r,\theta_{max})=\int_{0}^{r}{\int_{0}^{2\pi}{\cfrac{1}{\pi R^2}rd\theta dr}
= \cfrac{r^2}{R^2} \to r=F_R^{-1}(x)=\sqrt{\xi_1}R}
\\
F_\theta(\theta) = F(\theta \vert r) = \cfrac{F(r,\theta)}{F_R(r)}  = \cfrac{\theta}{2\pi} \to \theta = F_\theta^{-1}(x) = 2\pi \xi_2
$$

![weighted circle samples]({{ site.url }}/assets/2024-04-18-ray-tracing-5-sampling-lambertian/unified_circle_samples.png)

根据计算出来的两个式子，我们很明显看到采样 比原来均匀多了，可以证明此方法是有效的。

### 计算半球边界分布函数

根据上文提到的 $\omega=sin\theta d\theta d\phi$ 开始算联合概率分布：
$$
\begin{align*}
F(\theta, \phi)
&=\int_{\Omega} { pdf(\omega) d\omega } =\int_{\Omega} { \cfrac{ cos \theta }{ \pi } d\omega } \\
&= \cfrac{1}{\pi} \int_{0}^{\phi} {\int_{0}^{\theta} {cos\theta sin⁡\theta d\theta} d\phi}
 = \cfrac{1}{\pi} \int_{0}^{\phi} {\int_{0}^{\theta} {cos \theta d(cos\theta)} d\phi} \\
&= \cfrac{1}{\pi} \int_{0}^{\phi} {\left[ \cfrac{cos^2\theta}{2} \right]_{0}^{\theta} d\phi} 
 = \cfrac{1}{\pi} \int_{0}^{\phi} {\left(-\cfrac{cos^2\theta}{2}+{1\over 2} \right) d\phi} \\
&= \cfrac{1}{\pi} \int_{0}^{\phi} {(1- cos^2 \theta )d\phi } 
 = \cfrac{1}{2\pi} (1- cos^2 \theta )\left[ \phi \right]_{0}^{\phi} \\
&= \cfrac{sin^2 \theta \cdot \phi}{2\pi}
\end{align*}
$$

这里看起来 $\theta$ 比较好求，所以直接带入 $x_{max} = \theta_{max} = \frac{\pi}{2}$

$$
\begin{align*}
&F(\theta_{max}, \phi)
 = F({\pi \over 2},\phi)
 = \cfrac{sin^2\left(\frac{\pi}{2}\right) \phi}{2\pi}
 = \cfrac{\phi}{2\pi} \\
\to \ &F_{\phi} (\xi_1) =F^{−1} (\xi_1) = 2 \pi \xi_1
\end{align*}
$$

最后求 $F_{cos\theta}$，考虑到代码直接用的 cosine，就不需要纠结 arccos() 的问题了，直接做除

$$
\begin{align*}
&F(\theta, \phi_{max} )
 = \cfrac{F(\theta, \phi)}{F(\theta_{max}, \phi)} 
 = \cfrac{\cfrac{sin^2\theta \cdot \phi}{2\pi}}{\cfrac{\phi}{2\pi}}
= sin^2⁡\theta
= 1−cos^2⁡\theta  \\
\to \ &F_{cos⁡\theta}  (\xi_2 )=\sqrt{1−\xi_2 }=\sqrt{\xi_3}   \\
\to \ &F_\theta (\xi_4 )=arccos⁡(F_{cos⁡\theta} ) = arccos(\sqrt{\xi_4})
\end{align*}
$$

这时候我们拿到了最终的生成函数:

$$
\begin{cases}
\phi = 2 \pi \xi'_1 \\
cos\theta = \sqrt{\xi'_2}
\end{cases}
$$

![weighted circle samples]({{ site.url }}/assets/2024-04-18-ray-tracing-5-sampling-lambertian/cartisian_conversion.png)

再根据球面角到笛卡尔坐标系的的转换公式

$$
\begin{cases}
x &= rcos\phi sin\theta \\
y &= rsin\phi sin\theta \\
z &= rcos\theta
\end{cases}
$$

就可以轻松随机出符合余弦权重分布的方向了。我相信其他内容读者能够自行handle，到这里已经OK了。附上代码

```
float3 random_direction(u1, u2) {
    float cos_theta_sqr = u1; //xi'= 1-xi due to iid.
    float sin_theta = sqrt(1-cos_theta_sqr);
    float cos_theta = sqrt(cos_theta_sqr);
    float phi = 2 * PI * u2;
    float cos_phi = cos(phi);
    float sin_phi = sin(phi);
    return vector3
    (
        sin_theta * cos_phi,
        sin_theta * sin_phi,
        cos_theta
    );
}
```

<a name="1"></a> [[1](#ref-1)] Importance Sampling of a Hemisphere, https://www.mathematik.uni-marburg.de/~thormae/lectures/graphics1/code/ImportanceSampling/index.html