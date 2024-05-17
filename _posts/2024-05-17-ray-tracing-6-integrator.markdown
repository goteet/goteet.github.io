---
layout: post
title: 射线追踪(6) 积分器：组织不同 BxDF Terms
tags: rendering ray-tracing integrator
mathjax: true
---

# 射线追踪(6) 积分器：组织不同 BxDF Terms

在上一篇文章里我们尝试了使用重要性采样对 Lambertian 漫反射进行了计算。自然而然的，我们也希望能够对高光进行采样。我考虑到高光采样的复杂性，我决定分两部分来讨论这个问题。首先假设我已经能对高光进行采样了，这部分我们留在后面慢慢聊。

## 如何把 Diffuse 和 Specular 在路径追踪中结合起来？
我们熟知的光照分成 Diffuse 和 Specular 两部分，但是我们之前讨论的路径追踪，为了减少指数增长的问题，每次只会往一个方向进行采样。应该如何把 Diffuse 和 Specular 的结果合并在一起呢？（不看论文的我曾经被这个问题困扰了很久，成为我学习障碍的最大绊脚石。）

在 Kajiya1986 有明确回答过这个问题：

>  保持反射、折射、阴影等不同类型的射线的正确比例是非常重要的。我们一般有两个办法来解决：
> * 方法一：记录每一种类型的射线的数量，采样的时候让他们尽量保持希望的分布。这也是Kajiya用的方法.
> * 方法二：随机的生成不同类型的射线，但是每次都乘上他们的分布权重。这应该是 PBRT 用的方法。

论文里并没有提到 unbiased 的相关论证。只有一个大概方法描述，我猜这也是为了保持无偏的努力。在没有查阅其他相关资料的时候，这个解答有点模糊。好在我们有伟大的 github 来帮我们解决这个问题。

### pbrt-v3 vs. mitsuba-v1

爬一下代码，PBR-v3 使用的的方案是第种：均匀的选取不同分量，并根据其类型和与之对应的 pdf 确定随机方向。

```
Spectrum BSDF::Sample_f( ... )
{
    //... line 726, pbrt-v3/src/core/reflection.cpp
    int comp = std::min(
        (int)std::floor(u[0] * matchingComps),
        matchingComps - 1
    );
```

不过别忘了，最后的 pdf 实际上还需要经过 $\cfrac{1}{components}$ 的调整。这个道理很简单，就是 Mixture Distribution。$pdf=\cfrac{P_{diffuse} + P_{specualr}}{2}$

对于渲染器来说， 我们把 Diffuse，Specular Term 视为某一种BxDF。BSDF对象内部维护了一个关于 BxDF 的数组。我的小伙伴 ycc 告诉我，可以将这些 BxDF Term 视为一“层”(Layer?)，每次采样的时候只采样其中的一层。

在采样的时候，材质都会调用 ComputeScatteringFunctions() 动态生成当前 shading point 拥有的不同 BxDF。每一层（BxDF Term）的权重就由他们实现各自保存。并且在采样 f() 的时候一并返回。可以在 line 105, pbrt-v3/src/materials/disney.cpp 上下找到相应的佐证。

> 说实话我不是很理解为什么需要动态创建，采样过程中会大量发起 BxDF 的查询。完全可以通过设计规避这个问题啊？

和 pbrt 类似的，mitsuba 也是这么抽象材质的。不过 mitsuba 将 BSDF 直接特化成不同类型的材质表面了。如：Phong, Plastic, Conductor 等等。我们看一眼 PhongBSDF


```
Float dAvg = m_diffuseReflectance->getAverage().getLuminance();
Float sAvg = m_specularReflectance->getAverage().getLuminance();
m_specularSamplingWeight = sAvg / (dAvg + sAvg);

//...line 197, mitsuba/src/bsdfs/phong.cpp
choseSpecular = true;
if (sample.x <= m_specularSamplingWeight)
{
    sample.x /= m_specularSamplingWeight;
}
else
{
    sample.x = (sample.x - m_specularSamplingWeight) / (1-m_specularSamplingWeight);
    choseSpecular = false;
}
```

可以从代码中明确 Mitsuba 采取了另一种策略。而且从对应的 `Spectrum eval()` 函数里，没有发现需要额外的权重。毕竟方向已经按照分布来生成了。



## Integrator 积分器

在渲染器上下文中，一般把计算 Lo 的过程封装到积分器里。很显然我们积分器都是数值积分，主要是封装采样流程。

```
Integrator::Li(Scene, Ray)
{
    Intersection = Scene.Query(Ray);

    BxDFs = Intersection.BSDF.Components;
    BxDF = RandomPick(BxDFs);
    d = BxDF.SampleDirection();
    NoL = saturate(d.z); //dot({0,0,1}, d)

    //Method-1: Weighted Resulting.
    f = 0;  pdf = 0;
    foreach(bxdf in BxDFs)
    {
        f    += bxdf.f(d) * bxdf.weight;
        pdf  += bxdf.pdf(d);
    }
    pdf /= BxDFs.Num();

    // Recurssive occurs here and it wont quit
    NextRay = Ray{ Intersection.P, Intersection.Transform(d) };
    Li = Integrator::Li(Scene, NextRay);
```
还记得重要性采样吗？
$$
Q = \cfrac{1}{N}\sum^N{\cfrac{f(W_i) Li(W_i) (N\cdot D(W_i))^+}{pdf(W_i)}}
$$
最后别忘了除以样本数量。对于路径追踪来说，我们的样本数量是 1
```
    Lo = f * Li * NoL / pdf;
    Le = Intersection.Object.IsLightsource 
        ? Intersection.Object.Emitt()
        : Spectrum(0);
    return  Le + Lo;
}
```

代码本身不复杂，不过后期会因为其他的采样策略而产生变化。这些变化包括直接光源采样，镜面管采样等会对流程产生影响。所以单独封装了这部分。

虽然后期多了 biddirectional path tracing 等更新的积分策略，但是从效果来说，和现在最常用的还是 Path Tracing  区别特别小。所以稳定还是使用 path tracing。

## Combine

至于标题的答案，我想现在已经从积分器的伪代码中回答了。单次采样的时候，我们根据某一个 BxDF::pdf 随机生成采样方向。方向生成之后，我们对 BSDF 进行采样时，对每一个 BxDF 项都进行了采样，并考虑的 mixture pdf 而非单一项。