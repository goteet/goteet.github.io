---
layout: post
title: 射线追踪(4) 重要性采样
tags: rendering ray-tracing monte-carlo importance-sampling 
mathjax: true
---

# 射线追踪(4) 重要性采样

我们接着蒙特卡罗积分往下聊聊重要性采样。不过因为涉及到概率论的一些基础知识，我不确定能不能在一篇文内讲完。

$$
Q_{IS} = \cfrac{1}{N} \sum_i^N {\cfrac{f(x_i)}{pdf(x_i)}}
$$

$$
Q_{WIS} = \cfrac{\sum_i^N {pdf(x_i) \cdot f(x_i)}} {\sum_i^N {pdf(x_i)}}
$$

重要性采样是实施蒙特卡罗积分的一种方法，它通过一个有倾向性的随机分布进行采样，来评估原函数的值。既然讨论到分布的概念，我们还是先来回顾一下随机数相关的属性。

## Probability Distribution

随机变量有离散型和连续型的，我们主要讨论的是连续性的随机变量。不过为了理解方便，我同时把离散型随机变量的对应概念罗列一下。

对于某一个随机变量 X，我们使用累积分布函数 `CDF(Cumulative Distribution Function)` 来描述其发生特定事件的概率。CDF 作为随机现象发生的数学描述，是随机变量的重要特征。我们将其定义为如下形式：

$$
F(x) = Prob\{X \le x\}, -\infty < x < +\infty
$$

cdf 式子可以用来表示在定义域上取任意 x，随机变量 X <=x 这一事件的发生概率。为了简单，有时候也会称其为 X 的分布函数。

对于离散型的随机变量，我们可以对任意出现的可数结果（有些是无穷结果不可列举）都计算其概率。一般我们使用概率质量函数 PMF (Mass) 来描述每一个结果的单独概率。

$$
p(x) = Prob\{X=x\}
$$

那么我们提到的 CDF 就可以表示为

$$
F(x) =\sum_{x_i \le x}{p(x_i)}
$$

上文提到的概率分布，可以看成一张表格。分别列出可能的结果和与之对应的概率 $p(x_i)$ 。以骰子为例。概率分布如下：

| $X_i$  | 1 | 2 | 3 | 4 | 5 | 6 |
|--------|---|---|---|---|---|---|
|$p(X_i)$|1/6|1/6|1/6|1/6|1/6|1/6|

我列这个表格就是说明概率分布是一张表格，概率分布函数是一个累加函数。如果非要说概率分布是一个分段函数也不是不行。

而对于连续型随机变量呢，CDF 被定义成一个积分。

$$
F(x) = P(X \le x) = \int_{-\infty}^{x} {pdf(t)dt}
$$

其中的被积函数 pdf(x) 被称之为连续性随便两的概率密函数。这个是 CDF 的定义，前提是pdf可测。也就是说 pdf 被定义为概率分布函数在定义域上的某处的微分，可以认为随机变量在某点 X 上的概率是无穷小。

对于某个精确值的发生概率，我们一般以一个极小的区间来近似表示： $[x, x+Δx]$ 的概率为 $Prob(x)=pdf(x)*Δx$

同时，也可以说明， pdf 只是描述随机变量在某一点附近（区间）发生的可能性，不能直接表示某点上的概率。因为 pdf 是可以超过 100% 的，只要 Δx 足够小，能保证乘积结果小于 1 即可。

如果我们考虑一个区间 $[a,b]，则概率分布的定义可以写成

$$
F(a,b) = F(b)-F(a) = P\{a \le X \le b\} = \int_{a}^{b}{pdf(t)dt}
$$

从式子可以知道，某一个具体数值 x 的发生概率都永远为 0。所以我们考虑发生概率一般都是选取某一个区间。你问我求极限行不行？hmm我也不知道。

有趣的是，在一些论文里，会将 probability distribution 和 probability density 混用。一般在统计学上的讨论， distribution function 都是指代 CDF，而 density 本身是不能指代概率的。甚至有一些文章会将 probability density 等同于 probability mass，造成了更深的误解，这里需要区分的是 density 只用在连续随机变量上。

## 重要性采样 

不过没关系，pdf 还是能衡量一个随机变量在某处发生的可能性大小。回到重要性采样的话题上，我们的使用 pdf 来构造蒙特卡罗的评估式：

$$
\int{f(x)dx} \approx \cfrac{1}{N} \sum^{N}{\cfrac{f(X_i)}{pdf(X_i)}}
$$

这个式子说的是分布 $pdf()$ 的随机数 $X$ 进行随机生成样本 $X_i$，然后带入到评估式上里采样计算。将最终多个采样累加值就能得到评估解。

so why this？

### 数学期望

为了能证明这个评估结果是合理而且有效的，我们需要借助另一个数学工具来协助我们。就是数学期望。数学期望对一个随机变量 $X$ 的定义如下：

$$
E[X]=\int_{-\infty}^{+\infty}{x \cdot pdf(x) dx}
$$

数学期望可以用来表示一个随机变量的平均值。有时我们将其简称为期望。或者均值。从概念上很好理解，就是将随机变量在定义域上的任何一个取值，根据其发生的可能性加权求平均。

对于随机变量 $X$，关于它的函数 $R(X)$ 也是一个随机变量。那么我们设 $R(X)=\cfrac{f(x)}{pdf(x)}$，$R(X)$作为一个随机变量，他的期望可以写为如下形式：

$$
\begin{align*}
E[R(X)] 
&= \int_{-\infty}^{+\infty}{R(x) \cdot pdf(x) dx} \\
&= \int_{-\infty}^{+\infty}{\cfrac{f(x)}{pdf(x)} \cdot pdf(x) dx} \\
&= \int_{-\infty}^{+\infty}{f(x)dx}
\end{align*}
$$

发生了什么？随机变量 $R(X)$ 的期望等于原积式，这可这是太美妙了。然后我们再来看看我们的评估 $Q_N$

$$
Q_N = \cfrac{1}{N} \sum^{N}{\cfrac{f(X_i)}{pdf(X_i)}}
$$

由于数学期望的特性 $E(X+Y) = E(X) + E(Y)$，我们可以推出 $E\left[\sum_{i}^{N}{X_i}\right]=\sum_{i}^{N}{E\left[X_i\right]}$，带入到评估式 $E[Q_N]$ 稍微展开得到：

$$
\begin{align*}
E[Q_N]
&= \cfrac{1}{N} \cdot E\left[\sum^{N}{\cfrac{f(X_i)}{pdf(X_i)}}\right] \\
&= \cfrac{1}{N} \cdot \sum_{i}^{N}{E\left[X_i\right]} \\
&= \cfrac{1}{N} \cdot N \cdot E\left[\cfrac{f(X)}{pdf(X)}\right] \\
&= \int{\cfrac{f(x)}{pdf(x)} \cdot pdf(x) dx} \\
&= \int{f(x)dx}
\end{align*}
$$

结果是一样的。

### Unbiased and Consistent

我们将原始积分的期望，和评估式的期望做一个 diff。得到的结果 $E \left[\int{ f(x)dx }\right] - E[Q_N]$，称为 bias，如果 $bias=0$，我们认为两者期望相等的情况下，将其称为 unbias。对应的评估被称为 `unbias estimator`

另一方面，我们需要考量采样的方差 $V[Q_N]$，如果采样数量的增加，方差逐渐收敛。我们认为这个评估是一致的。是可以得到稳定的结果的。由大数定理可知，多次试验的平均值，会随着试验次数的不断增多，而逐步趋近于期望。

因为能力有限我在此不做详细展开。可以参考一下这篇 [[1](#1)<a name="ref-1"></a>]

### 降低方差的原理

到这里我们基本上论证完评估为什么会是这个形式了。那问题来了，这个形式的评估是如何降低方差的呢？

对于图像生成来说，方差带来更多的噪点，为此，如何减小方差是射线追踪中的一个重要课题。

因此重要性采样蒙特卡罗积分，通过选择合适的随机分布能快速获得更高质量的图像。只需要简单的选择合适的随机分布，就可以获得降低方差的效果。隐藏在这背后的思路是：尽可能多的在函数贡献比较大的区域上生成更多的采样。这意味着用更少的采样数，获得更接近期望的近似值。

![sampling distribution]({{ site.url }}assets/2024-04-18-ray-tracing-4-importance-sampling/sampling_distribution.png)

虽然不同的 pdf ，经过大量的样本之后最终都能收敛，但是选用与原函数图像更接近的分布则收敛速度更快。假设我们选用的 $pdf \propto f(x)$ 和原函数图像完全正相关，用数学式子表达，可以认为 $pdf(x) = cf(x)$，经过消元，最终我们可以获得一个常量评估 $Q_N = N \cdot k$。

但很显然的，我们之前以场景采样举例。我们无法通过场景物体的空间结构计算出一个有效的分布函数。如果可以的话，我们是不是也可以轻松获得解析解？或者我们能不能通过空间关系构造出一个 pdf，使得随机变量尽可能向光源方向投射呢？不过也别忘了反射方程中的 $(n \cdot l)^+$，约接近法线方向的权重会更高。



### Weighted Importance sampling

说个题外话，这个我自己没在使用，不过上一期提到了，也查阅了一下文档。虽然有的时候我们是根据某个 概率分布 $pdf()$ 来生成的随机样本。但是我们并不能准确计算出 $\int{pdf(x)dx} = 1$。此时认为实际上准确的 $p'(x) = k \cdot pdf(x)$

然而我们没有能力去计算 $k=p'/pdf$，所以我们可能会选用权重重要性采样。他的评估式如下：

$$
Q = \cfrac{\sum_i^N {pdf(x_i) \cdot f(x_i)}} {\sum_i^N {pdf(x_i)}}
$$

但是要强调的是，这个评估是有偏的，需要谨慎选用。举个简单，加入我在半球上均匀随机，那么第一反应是 $pdf=1$，按照 WIS 的评估，我们带入 5 个样本来算一下：

$$
Q_{WIS} = \cfrac{ \sum{f(x_i)} }{5}
$$

然而从我们刚才讨论的定义出发，在半球上发生随机数的全概率 = 1，假设真实的 p'(x) = kpdf(x)

$$
F(x) = \int_{\Omega} p'(\omega)d\omega = 1 \\
A = \int_{\Omega} {1 d\omega} = \int_{0}^{2\pi}{\int_{0}^{\cfrac{\pi}{2}}{sin\theta d\theta}d\phi}= 2\pi
$$

我们可知实际上的 $p'(x) = \cfrac {1}{2\pi}$，无偏的重要性采样评估为：

$$
Q_{IS} = \cfrac{2\pi}{5} \cdot \sum{f(x_i)}
$$

这和期望差了 $bias=2\pi-1$ 呢！其余的能力有限我们把推导过程放在 [2][3] 里，我们后面用到再谈。

### 均匀采样

补充一个有趣的点，如果我们采用均匀采样，令 $p=1$ 带入到式子里，就可以看到我们熟悉的积分评估

$$
Q = \cfrac{1}{N}\sum_i^N{f(x)}
$$


<a name="1"></a> [[1](#ref-1)] Monte Carlo Integral with Multiple Importance Sampling, Consistent and Unbiased,  <https://agraphicsguynotes.com/posts/monte_carlo_integral_with_multiple_importance_sampling/><br/>
[2] Importance Sampling: A Review, <https://www2.stat.duke.edu/~st118/Publication/impsamp.pdf><br/>
[3] Weighted Importance Sampling, Section 1, <https://www.uni-ulm.de/fileadmin/website_uni_ulm/mawi.inst.110/lehre/ss14/MonteCarloII/reading2.pdf>