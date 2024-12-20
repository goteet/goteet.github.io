---
layout: post
title: 射线追踪(1) 光的建模
tags: rendering ray-tracing
mathjax: true
---

2022-01-17 第一次整理成文<br/>
2024-04-16 根据学习线路重新整理文章。<br/>

# 射线追踪(1) 光的建模

看了眼旧的文章，发现顺序和主题是混乱的，所以决定重新更新时间戳。

如果抛开感性认识，我觉得我们第一件该做的事情就是对光和整个模拟系统的建模。感性认识网上很多资料，我自己比较推荐 Ray Tracing in One Week Series [[1](#1)<a name="ref-1"></a>]。

但是随着近几年基于物理的渲染逐步兴起，以物理的角度来理解光、对光进行模拟，反而排在其他数学和渲染知识之前。他能让我们理解什么是物理正确的计算，以保证渲染结果的相对准确。

## 光的物理建模

光是一种复杂的物理现象，一般我们会分两个视角去讨论光在渲染中的影响。

为了理解光会对最终色彩产生的影响，我们会基于光波的物理光学特性去讨论光在渲染时可能发生的现象。

但是在程序设计时，我们往往采用基于射线的几何光学视角进行建模，以减小对光线的模拟的复杂度。

在 Siggraph 系列课程 Physically Based Shading in Theory and Practice [[2](#2)<a href="ref-2"></a>] 里，Naty Hoffman 在其前导课程 Physically-Based Shading Models in Film and Game Production [[3](#3)<a href="ref-3"></a>] [[4](#3)] 中详细讨论了光的两种特性。

### 光与介质

光时一种电磁横波，和机械波不一样，光是可以脱离介质在真空中传播的。我们知道真空中传播是有恒定速度 $c$ 进行传播的。

![light wave]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/light_wave.png)


#### 光与物质的交互

和风一样，我们感知光的主要方式，主要是观察光和介质进行交互后产生的变化来实现的。所以我们在渲染范畴，主要理解的也是光与介质的交互。

我们知道，物质是由许多原子紧密排列组成的。物质中单个原子、或者许多原子组成的原子簇，被我们统称为粒子。当光与物质表面接触到时候，光会被这些粒子改变前进方向。

![light medium interaction]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/light_medium_interaction.png)

当光照射到这些粒子的时候，粒子吸收其能量，极化之后重新向其他方向射出新的光线。当然这些光线的速度又可能会和原来的光线速度不一样了，也就是他们的波长变化了。

#### 复合波

说到波长，我们人眼识别光线，只能识别有限范围波长的可见光，从400nm的紫光，一直到700nm的红光。自然界的彩色世界中，光是以多种不同的波长组合而成的，如下图所示：

![light medium interaction]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/complex_light_wave.png)

当然这个波看起来有点点复杂，波峰波谷的变化有点吓人。不过幸运的是，人们发现，光线的复合波是可以通过单播叠加组合到一起形成。

![light wave comparison]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/light_wave_composition.png)

左上角是这个光的光谱分布，`Spectrum Power Distrubtion`(SPD)，右侧则是他的波形。于是自然而然的，工程师们想到了用单波组合来模拟整个光谱的方法。我们常用红绿蓝三种单波来组成光线的记录。

### 物理光学特性-电磁波

再回到光和物质粒子交互的话题上。由于粒子超微观的实际情况，我们根本无法使用计算机进行模拟。于是我们将其中的物理现象抽象成宏观概念。

#### 介质

对于那些光可以在其内部沿着直线传播物质，我们称其为均匀介质。

显然粒子在物质中不可能完全均匀，但是对于那些密度均匀，材质单一的物质，宏观上不影响光前进的方向。那么均匀介质这个概念显然对我们进行模拟有好处。

当然，也有不那么均匀的介质，比如说浓雾密度不均匀的情况。那我们抽象出粒子这个概念，用来指代大分子或者许多分子组成的分子团。 这些粒子会使光线产生不一致的方向变化。形成另一种渲染效果。

#### 吸收与散射

光在穿过介质进行传输过程中，有一部分能量会被吸收点，想象一下光向海底传播，白光逐渐变蓝消失的情况；而另一部分则会则发生散射现象，其中我们比较熟悉的是折射、反射。当然也有其他散射现象。

介质的表现效果，大致就由这种吸收和散射来决定的。针对均匀的介质，我们往往采用折射率 `Index of Refraction` IOR 来描述。

这是一个复杂的参数，实际上由一个复数来分别表示介质内光速，和介质吸收光的能力。

有些介质会呈现出特定的颜色，这一般是介质吸收了某些特定波长的光。而另一些介质不吸收光，所以这部分就是 0，折射率变成一个有理数。

#### 表面衍射

渲染时，我们往往关注于介质表面。在高中物理中由学过，介质表面就是将两种均匀介质之间的交接处抽象成的一个平面。

但是对于光学角度来说，没有绝对平整的表面。这又回到了一开始的原子级别的考虑。对于那些大小远小于光波长的粒子，称这种微小的不规则性为 `Nanogeometry`，这些粒子在影响光的方向时，由于方向不一致，会发生衍射和互相干涉的现象。

![surface diffraction]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/surface_diffraction.png)

从惠更斯-菲涅尔定律，我们可以认为光再碰到粒子的时候会发出 360° 的偏折，从图中来看就是类似一个圆形、或者半圆形。

当光在这些 `Nanogeometry` 接触时，处于介质边缘的粒子会使其发生弯折，形成衍射现象，这就是为什么我们看到物体的阴影边缘是模糊的的原因。这一点是在我们后面提及的几何光学特性里无法模拟的情况，也是几何光学模拟缺失的部分。

而另一方面，由于微小的不规则性，重新反射的光方向不同。互相干涉之后不能保持一个固定的方向，则会发生散射的现象。反之，越多的光线方向相同，则他们干涉后的方向也是基本一致的，这是我们常常讨论的反射现象。

### 几何光学特性

聊到反射，就进入到我们的几何光学特性了。

#### Reflection and Refraction

几何光学是我们模拟渲染中主要使用的视角，这里忽略了光的干涉和衍射现象，只讨论折射反射的情况。

![reflection and refraction]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/reflection_and_refraction.png)

在几何光学中的讨论中，光被建模为射线，而假设介质表面是平坦表面。光线射线在接触到介质表面时发生的散射现象，主要是反射和折射两种类型。所以在这个范畴里，我们主要考量反射和折射这两种散射。他们各自的方向偏转，和介质表面的法线相关。

光击中介质表面时，反射和折射会同时发生。

不过对于金属材质的无体，折射光会被瞬间吸收并重新进行反射。这个过程和反射一样，都发生在相同的入射点。而那些非金属无体折射光线会进入到介质内部。

考虑到真实世界的测量方法，我们一般将介质分为三种，导体`Conductor`、半导体`Semi-conductor`和绝缘体`Deletric`，包含了大气在内许多介质。但考量半导体在游戏中出现概率比较低，我们大多数情况只考虑金属、非金属两种类型的介质。

#### Diffuse and Specular

`Deletric` 折射后，经过多次粒子的转向之后，最终会被反射出平面。这些最终反射出介质的光线，其反射点（出射）、和入射点是有可能不在相同位置的。当我们在远处进行宏观观测时候，一个像素可能能同时涵盖入射出射点，此时我们认为这种反射和金属吸收重新激发的类似。

为了模拟方便，我们常会将直接反射的光线称为 Specular 高光反射，和这些特殊反射的光线称为 Diffuse 漫反射。这个划分主要还是基于测量的方便性，易于单独测量而进行了区分。其实他给我后来的程序设计时造成了很大的困扰，不过为了简单我们过后再聊。

![diffuse vs subsurface scattering]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/diffuse_or_subsurface_scattering.png)

然而，当我们近距离观测这些 `Deletric` 的时候，可能 折射-出射 的光线偏移原入射点是可感知的情况。这些位移横跨了多个像素，那么我们就无法再用 Diffuse 来模拟这种现象。此时我们称这个现象为此表面散射 `Subsurface Scattering`。

次表面散射和漫反射是同源一体两种表达。使用Diffuse 进行建模的时候，我们称之为 Local Reflectance；而描述此表面散射，就需要 Global Reflectance 来进行模拟。

#### 小结

我们通过抽象出均匀介质、介质表面，并采用几何建模的方式理解了光传播的方式。并通过光的波动特性，理解了折射、反射两种散射效果。这基本就时描述了光沿着路径行进的定性建模，之后就要开始我们的定量建模了。

## 光的数学建模

从这一步开始，我们就要开始讨论如何通过数学的办法，将光的物理特性描述为计算和程序能够理解并显示的颜色计算了。

### Colors

先从我们的人眼开始，考虑我们人眼时一种非常受限的受体的情况，基本只能吸收红、绿、蓝三种光波。而且人眼时一种难以置信的有损受体。

以下图片展示的是一个有色光的光谱分布，`Spectrum Power Distribution` 简称 SPD。经过试验，人们发现人类无法分辨出通过红绿蓝三种激光组合而成的简化波，和自然光波。

![spd comparison]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/spd_comparison.png)

所以这两个光谱在我们人类眼中呈现的是相同的色彩。

结合了之前单向波可以组成复合波的这一条件。我们常常在 400-700 这个光谱分布 SPD 进行了三个波长的能量采样，并用它来表示光的能量。这也就是我们 ` #define Spectrum Vector3(r,g,b) ` 的由来。

当然随着渲染的进步，开始有很多人考虑到采用更多的单波来模拟 SPD 进行光谱渲染。随着运算能力的发展，允许我们在光谱上进行更多的采样点，也就是我们的 color 就不再是RGB三通道，而是 RGBCMY 六通道了，说不定以后能有全光谱的模拟呢。

#### 能量计算

介质之所以能发光，是因为戒指内的电荷震荡时，发射出光波。光能、热能、电能、化学能都会使电荷震荡。光照射到介质表面时，其中一部分能量会重新向外辐射。

渲染考量的便是光波携带能量的能量变化。常见的击中测量方式中，我们常会使用到到 `Radiometry`。Radiometry计量方式主要记录四种数值，我列在表格里：

| Name               | Symbol   | Unit                      |
|--------------------|----------|---------------------------|
| radiance flux      | $\Omega$ | watt( $W$)                |
| radiance intensity | $I$      | $\cfrac{W}{sr}$           |
| irradiance         | $E$      | $\cfrac{W}{m^2}$          |
| radiance           | $L$      | $\cfrac{W}{m^2 \cdot sr}$ |


辐射度量的基本单位是辐射通量 `Radiance Flux`：，它度量的是单位时间内辐射能量的流动，但是一般我们不直接使用它。作为唯一的输入源，我们更倾向于采用光源的颜色等人类易于理解的归一化后的数值。

辐射强度 `Radiance Intensity` 描述的是沿某个方向立体角的通量密度。说实话我不太有印象这个数值的物理含义了，特意查了下PBRT，想起他应该只有在讨论点光源的时候有意义。


在渲染中，我们比较关注的是 $E$ 和 $L$ 两个参数。


 
$E$ 应该翻译成辐照度？$E$ 是辐射通量相对于面积的密度，它是一个非常重要的计量单位。

$$
E = irradiance = \cfrac{d\Omega}{dA}
$$

他的单位表明了，这个值表示的是某一块微小的区域接收到的光的能量总和。

$L$ 是辐射，在环境中认为它是一个关于位置 $x$ 和方向 $d$ 的分布函数，以表示在某个 shading point 沿着某个固定方向 $d$ 上提供的能量强度。

$$
L=radiance= \cfrac{d\Omega}{dA d\omega}
$$

大多是情况下，$d$ 都是以 $x$ 为起点表示的射线进行表示。不过我们常用 $L_o$ 来表示从 $x$ 向外射出的能量，使用$L_i$表示从环境指向 $x$ 方向的光线能量。

`radiance` 不会受到距离的影响，无论在多远观察，一个表面的出射 $L$ 总是常量。

*一些题外话*

当然，波长除了影响颜色之外，他也会对反射、折射的方向有一定的影响。我们之前讨论过 SPD 的简化，实际上 `irradiance`、`radiance` 的量纲应该还包含一个 1/nm，不同波长能量是需要单独计算的。

正确的情况下，应该是先用光谱数值计算亮度，直到最后一步再转成 RGB 颜色值。因为 RGB 实际上是一种视觉上的观测描述，而不是物理量。用RGB渲染是从范畴上说出错了。

之前讨论的 SPD 替代，如果需要到达相同的颜色值，激光的 RGB 光谱亮度要比自然光的亮度高许多才能一致。但是材质反射曲线只能返回一个系数，这会导致激光的反射是不足的，导致激光的亮度低于自然光。因为自然光的光谱连续，也会有反射。但是激光的高亮度的RGB三个峰值波长上数值被降低，其他波长又没有数值。如果使用 RGB 计算的时候，直接用入射光的 RGB 乘上材质表面的 RGB 值。亮度是没有发生改变的。这种情况会对预测渲染程序产生影响导致颜色失真。但是对于可交互的程序影响不大。


## 射线追踪的程序建模

基于几何射线进行建模，我们可以构建起射线追踪的基本框架。

现代相机使用透镜替换了针孔。虽然透镜的引入带来了景深的限制，但是传感器却能获得更多的的光线摄入，不同方向的光线经过透镜折射，落在同一个传感器上。

![camera model]({{ site.url }}/assets/2024-04-16-ray-tracing-1-modeling-light/camera_model.png)

不过在渲染建模时，仍然采用的是类似古典针孔相机的模型。不过为了简化，我们将针孔后移到画面的后方，如图所示。以一个数学概念上大小为 0 的点，作为眼睛位置，渲染进行时，从眼睛位置 $c$ 处发出射线对场景进行采样。射线穿过感器表面形成采样点，最终与场景物体相交。

传感器横截面就相当于了我们最终渲染的画面，其上的每一个感光元件可以类比为一个像素。由于没有材料限制，采样可以不受限制的增加，在重建成图像信号时会更有帮助。

对于每个像素而言，它是有面积的，我们需要计算的是整个立体角的能量。这种基于几何的渲染模型在 shading 采样时只与一个方向射线交互，有时会带来错误，比如数值不稳定，或者一些锯齿效果。


### 计算 Radiance

假设我们的观察方向是 $v$，那么我们只要计算由 $-v$ 摄入相机的能量 L 的总和（像素对应的立体角）就可以得到画面色彩了。

对于刚才提到的，能量 radiance 一般在真空中不会变化，我们可以得到：

$L_i(c, -v) = L_o(p_{scene}, v)$

渲染过程一般会与场景中的多个物体，和场景中填充的介质共同完成。在真实世界里，场景中存在的介质（空气什么的）会对光线传输造成影响，比如浓雾，会吸收、散射光线。为了简单，假设光线在场景中传输不会受到介质（Participate Media）的影响，所以射入相机的光线，是等同于物体表面反射的光线。

等式中的 $p$ 指的是场景中物体表面与视线相交的交点。那么接下来，我们只需要研究光与介质如何交互，并算出对应数值即可。

这一部分，我们留给下一篇文。


<a name="1"></a> [[1](#ref-1)] Ray Tracing in One Week Series, <https://raytracing.github.io/><br/>

<a name="2"></a> [[2](#ref-2)] Physically Based Shading in Theory and Practice, <https://history.siggraph.org/person/stephen-mcauley/><br/>

<a name="3"></a> [[3](#ref-3)] Physics and Math of Shading, <https://www.youtube.com/watch?v=j-A0mwsJRmk>, <https://www.bilibili.com/video/BV1s741147Kt/><br/>

[[4](#ref-3)] Naty Hoffman, Physically-Based Shading Models in Film and Game Production, siggraph 2010, <http://renderwonk.com/publications/s2010-shading-course/hoffman/s2010_physically_based_shading_hoffman_a_notes.pdf><br/>