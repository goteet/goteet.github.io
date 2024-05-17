---
layout: post
title: 射线追踪(2) 图片成像建模
tags: rendering ray-tracing physically-based
mathjax: true
---

# 射线追踪(2) 图片成像建模

在整理上一篇文章关于能量数值量化的过程中，我曾经在旧的文章里写下 $L$ 是计量图片最终颜色数值的物理量。

## 像素的能量计量

如果从量纲上考虑，$L$ 需要考虑立体角的微分，可以认为它只是某一个方向上的能量。然而无论是我们的图片像素，还是真实世界的摄像机传感器中的感光元件，它都是有大小的。

![sensor input]({{ site.url }}/assets/2024-04-17-ray-tracing-2-energy-to-image/sensor_energy_input.png)

用一张类似的图来演示一下，元器件透过微透镜之后，接收到的能量应该来自于一个立体角 $\theta$ 范围内的能量。（假设其他方向被屏蔽或者没有入射能量的情况）

$$
E = \int_{\theta}{Li(\omega) d\omega}
$$

经过简单的阅读, 发现 PBRT[[1](#1)<a name="ref-1"></a>] 是这么描述的。而且考虑到传感器和透镜之间没有其他光源输入。还把积分转换成了面积积分的形式。

## 能量与颜色的鸿沟

随着阅读的深入，发现从能量转成图像过程中，还存在很大的一个步骤需要努力。

这部分我先趴了 mitsuba 的代码，它的执行逻辑是：

1. Li Sampling
2. 把结果传入一个 Filter
3. Filter 根据权重计算后写入到 pixel accumulated buffer 里。

所以这个 Filter 是什么呢？经过后续的阅读PBRT[[2](#2)<a name="ref-2"></a>]，我发现了 sampling 和 image 的关系。我们略过中间 Pixel Sensor 的建模和 CIE XYZ 的变换，只讨论信号重建。

我们在讨论最终图像的时候，实际上是针对场景信号样本做 Reconstruction。文章里提到后期有根据物理进行重建的内容。现在只考量像素（传感器）的内容。

与传感器类似的，像素只会对其样本点落在周围的光线做出反应。这里有两种思考方法：

1. 我们考虑一个样本，然后根据滤波权重计算影响附近的像素，并写入数据。
2. 我们考虑像素面积，当样本不在像素面积内的时候，我们将其过滤掉。

于是我们构建一个滤波器，通过空间距离来调整样本的权重。

对于信号重建范畴的原理而言，我们可以把图像看作是连续信号。那么对于胶片上的任意一个点的能量 $E$，应该有如下等式：

$$
E_f(x,y) = \int{f(x-x',y-y')L(x', y')dx'dy'}
$$
其中过滤函数的积分应该为 $\int{f(x,y)dxdy} = 1$

但显然我们现在拿到的是采样点，而非连续函数。所以我们这里需要借助一个数值积分的办法来计算评估值：

$$
E_f(x,y) \approx \cfrac{A}{N}\sum_{i}^{N}{f(x-x_i,y-y_i)Sample(x_i,y_i)}
$$

这里的 A 是指胶片面积。

此时我们就能算出某个像素点的能量评估值。看到这里复杂的公式先别急，如果我们把p(x,y)设为像素的中心，再把滤波函数 $f(p)$ 影响的大小，改为一个像素的大小。那么这个累加操作是不是很像卷积。

![samples per pixel]({{ site.url }}/assets/2024-04-17-ray-tracing-2-energy-to-image/sample_per_pixel.png)

实际上，滤波函数能选的不多， mitsuba 里默认用的 gauss 滤波。除此之外还有我们熟知的box filtering，不过这玩意很容易产生 aliasing，就没有采用。

但是这个评估函数还需要计算 A/N factor，这个玩意有点麻烦。而且由于离散的关系，f() 的积分并不等于 1（只是期望等于1），所以容易产生噪点，书中推荐使用 weighted importance sampling 评估。

$$
E_f(x,y) \approx \cfrac{\sum_i{f(x-x_i,y-y_i)L(x_i,y_i)}}{\sum_i{f(x-x_i,y-y_i)}}
$$

这玩意印象在后面的 MIS 中会用到。不过今天就先到这里。我之前没有仔细考虑过这一部分内容。我只是算出多个 samples 之后算了一个算数平均。

现在看起来修改的方案为：假设像素大小为 A = 1，像素内的样本为 N，那么得到的结果就是
$$
E=\cfrac{1}{N}\sum{weight_i*L_i}
$$

然后我们再把 E 通过 gamma correction 转换成 sRGB，后续我在学到基于物理建模之后再进行改良。

<a name="1"></a> [[1](#ref-1)] The Camera Measurement Equation, Section 5.4.1,The Camera Measurement Equation <https://pbr-book.org/4ed/Cameras_and_Film/Film_and_Imaging#TheCameraMeasurementEquation>

<a name="2"></a> [[2](#ref-2)] Filtering Image Samples, Section 5.4.3,The Camera Measurement Equation, <https://pbr-book.org/4ed/Cameras_and_Film/Film_and_Imaging#FilteringImageSamples>