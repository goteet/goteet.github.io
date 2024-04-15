---
layout: post
title: Why Gamma Correction
tags: rendering
mathjax: true
---

2021-09-04 整理资料成文<br/>
2022-03-14 重新编排结构<br/>
2024-04-12 复活技术博客时重新梳理<br/>

# Why Gamma Correction

Gamma Correction 是针对特定环境，对颜色数据进行的编码和解码的操作。它几乎涵盖了整个游戏生产开发过程中的每一个步骤，影响着最终颜色呈现的准确性。

可以认为他是渲染知识的一个重要前置基础，但是很多新人在学习过程中却没有涉及相关的内容。所以今天来聊聊伽马矫正的问题。

## Understanding Gamma Correction

我们人眼能够感知到颜色，主要依靠光线提供的能量。光线的能量计算是线性的，即使考虑到光的波动性，光也可以视为多个单波线性累加成的，具体可以参考 [[1](#1)<a name="ref-1"></a>]。

人眼长期在自然界中的线性光照环境下生存，自然是渴望线性变化的光照能量。为了能让玩家看到准确的色彩，我们希望游戏画面的最终颜色输出，也是线性的。

### Gamma in Display Devices

不过遗憾的是，我们最常见的显示设备，LCD显示器并非是线性的。这里的“非线性”，指的是我们提供给显示器的信号，和显示器最终发射的光线能量并不是线性关系。

举个简单的例子，我们把亮度归一化到 [0, 1] 的数据范围，我们提供给显示器 0.5 的亮度，最终检测到显示器的输出亮度可能是 0.218。

$$
L(voltage)=voltage^{2.2}
$$

这是因为显示器内部，存在一个转换函数，把我们输入信号进行了调整。从普遍情况来看，这个转换函数的曲线大概等效于以上函数。

我们称之为 `Display Transfer Function`，这个转换函数是硬件的一部分，不同的显示器设备会有不同的函数曲线。但是我们常见的显示设备，转换函数都可以写成形式如

$$
V_{out} = \alpha V_{input}^{\gamma}
$$

对于我们的 LCD 液晶显示器来说，将 gamma 设定为 $\gamma=2.2$ 曲线如下图所示。

从图中可以很明显的看到转换函数如何影响显示设备，其输入缓冲的数据，和亮度等级之间的关系。（Radiance Level）

![gamma=2.2]({{ site.url }}/assets/2021-09-06-why-gamma-correction/gamma_2.2_curve.png)



当输入信号电压为 0% 时，显示黑色，电压为 100% 时，显示白色。黑、白两个端点是重合的，所以黑色和白色是准确的。

然而在两端中间，输入信号从 100% 降低到 50% 时，最终的输出亮度等级，只有 22% 左右的亮度，而非我们预料的 50%。实际观看起来，暗色部分会显得特别重，有一种死黑死黑的感觉。



#### 为什么会有这个转换函数？

经过资料的探查，一个比较有可能的原因可能是：兼容早期的 CRT 显示器。

CRT 显示器中使用到的阴极管，为了让发射的能量照亮整个输出区域。射线会一直扫过元器件覆盖的圆形范围。

![crt]({{ site.url }}/assets/2021-09-06-why-gamma-correction/crt.png)

我猜电压不变的情况下面积变成大，意味着等同能量需要覆盖更大的面积。Radiance 单位为 $W \over { sr \cdot m^2}$，最终亮度等级就会形成上文提到的非线性关系。

### Gamma Correction

为了能让最终的显示呈现出线性的光线变化，我们不得不想办法消除显示器 gamma 的影响。这也就是我们需要的 **Gamma Correction** 。

对于形式如 $V_{out} = \alpha V_{input}^{\gamma}$ 的函数而言，考量 $\gamma$ 的取值，我们根据 gamma 是否大于 1 来描述两种不同的操作：

**Gamma Encoding**： 如果取 γ < 1 时，我们将这个函数的操作称为 Encoding，或者伽马压缩 Compression，与之对应的 gamma 值被称为为 Encoding Gamma；

**Gamma Decoding**：反正当 γ > 1 时，我们称其为 Deocding Gamma，对应函数的操作为 Decoding，或者伽马展开 Expansion。

![gamma correction]({{ site.url }}/assets/2021-09-06-why-gamma-correction/gamma_correction.png)

自然而然的，我们在显示器上输出信号的时候，需要对 $V_{input}$ 施加一个 $\gamma={1 \over 2.2}=0.45$ 的变换，就能消除显示器造成的影响。

### Gamma in Image

实际上，我们大部分早期的互联网图像，都是保存的经过伽马压缩的数据，也就是线性色彩数据，经过 $\gamma=0.45$ 编码之后的数据。

那么在显示图像时，直接将图片数据当作显示信号提供给显示器即可。编码过的颜色数据再经过显示器的 Display Transfer Function 的转换，两个操作正好抵消，形成了线性的色彩数据。

但是如果你在电视上，他们的转换函数并非 $\gamma=2.2$，那么你就会看到发白如水洗一般的图片了。这就是为什么我们坐在家中的客厅，第一次启动游戏时，开发者需要玩家根据隐约可见的图标调整 Gamma 的原因。

#### 为什么要对图片数据进行编码？

是因为为了能在显示器上显示正确吗？答案并非如此简单。这么做的原因和我们人眼、数据范围息息相关。

先说我们人的眼睛。我们人眼感知亮度，是一种心理学范畴的内容。跳过复杂的研究，结论是我们人类的眼睛识别亮度并非线性的，而且观察环境会影响人类对亮度的判断。暗光环境、黑色背景的条件下会更为明显。

![linear ramp]({{ site.url }}/assets/2021-09-06-why-gamma-correction/linear_physical_brightness.png)

这是一张亮度物理量均匀变化的图，可以看到似乎变化不是很均匀。实际上，是因为人眼的感知，对暗部的变化更敏感，能识别出更多细节，所以黑色区域色阶变化很快。而亮部区域，我们感知到的变化不明显，所以感觉“全是白色”。

但我们人类的眼睛更喜欢这样的渐变过渡：

![perceptual ramp]({{ site.url }}/assets/2021-09-06-why-gamma-correction/linear_perceptual_brightness.png)

这表明人眼感知的“均匀”，和物理量的均匀是两回事。显然我们肯定是要以人眼为主要意志转移的。凑巧的是，物理线性经过如下函数的调整，我们人眼看起来就变得“视觉线性”了：

$$
eye = actual^2
$$

这从另一个侧面说明了，从 0% 纯黑提高一个色阶到 10% 亮度的暗色，我们可能只需要 10 的物理亮度，但是从 90% 亮度的白色提高一个 100% 纯白，我们可能需要 10000 的物理亮度。

#### 数据编码

我们仍在广泛使用的大部分图片，仍然是我们熟知的 RGB8 来存储颜色的，但显然8位能存储 256 个色阶，实在是不高。为了便于讨论，我们缩小到 16 个色阶来看看效果。

如果我们按照线性变化的物理量进行存储的话，我们会得到如下的一个 banding 效果：

![phyiscal band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/banded_physical_brightness.png)

为了正确显示白色色阶的变化，我们需要很大的步长。这就会让暗部区域的变化完全丢失了。还记得吗？我们对暗部的变化感知非常明显，很小的数值就能感知出变化。对于大步长，实际上我们丢掉了很多可感知的色阶。

这也是我们的美术同学常说的，暗部死黑没有细节。实际上指的就是暗部区域的色阶不够丰富。在美术和摄影上，这一点都是相同的，且符合人眼观察规律的，人们希望在较暗的区域看到更丰富的细节。

所以实际上我们需要的是这么样一个效果：

![perceptual band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/banded_perceptual_brightness.png)

我们需要的是一个可变步长的渐变。在暗部区域，我们需要小步长来记录更多的色阶，而亮部我们人眼不容易察觉变化的位置，则增大步长。这张图只是示意：

![perceptual band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/band_ppm_decoded.png)

为了能保存尽量多的色阶，很自然的我们想到直接保存人眼“视觉渐变”的数据，是不是就可以了？调整后的数据使用定长步长，相当于是原始数据使用了变长的步长。暗部存储获得更多的色阶。

这正是对图像的数据进行了 gamma encoding。我们熟知的 BMP、PNG、GIF、JPG 等图片格式，存储的是都是经过 gamma encoding 的数据。也就是把物理色彩经过 $\gamma=0.45$ 编码后才存进图片文件里，这就是标题所说的：图片伽马。

最终显示的时候，图片的 gamma 0.45 和显示器的 gamma 2.2 相抵消，呈现出了物理线性的亮度，形成了系统性的物理线性数据输出。

### Colorspace

色彩空间用来描述原始数据经过了何种变换。sRGB 是一种色彩空间的一种，我们常用 sRGB 来指代那些经过编码的图片数据，称这些图片的色彩空间是在 sRGB 空间中。而原始的物理数据被称为在线性空间中。

我们熟知的 BMP、PNG、JPG 等图片格式，由于年代久远，并没有明确保存他们的数据具体
保存在哪个色彩空间中。约定俗成的，我们统一认为他们保存在 sRGB space 里。

现在的相机、绘图软件等生产图片的工具，和图片查看器等显示软件都基于这个共识在工作的。除了 sRGB，还有许多很著名的色彩空间，我们熟知的马蹄图我这里就不贴了。不过值得于一提的是 rec.709，他和 sRGB 同源一体，都来源于 CIE 的观察试验，能力有限这里就按下不表了。有兴趣的可以去搜搜。

sRGB 和线性空间的变换不能简单等同于 $\gamma=0.45$。sRGB 变换是一个分段函数，他会在暗部进行特殊的线性变换，避免处理全黑的问题。转换函数可以在网上找到，我在这随便贴一个：

$$
f_{linear \to sRGB}(c)= 
    \begin{cases}
    12.92 c, &\ c\le0.0031308 \\
    1.055 c^{1/2.4} - 0.055, &\ c\gt0.0031308
    \end{cases}
$$

反过来也一样：

$$
f_{sRGB \to linear}(c)= 
    \begin{cases}
    {\cfrac{c}{12.92}}, &\ c\le0.0031308 \\
    \left(\cfrac{c + 0.055} {1.055}\right)^{2.4}, &\ c\gt0.0031308
    \end{cases}
$$

但是在 Shader 里，我们甚至都不会去算 power2.2，而是用平方这这种快速的方法。

#### Gamma in everywhere

印象里 rec709[[5](#5)<a name="ref-5"></a>] 和 rec.2100 分别定义了 SDR 和 HDR 的标准。对于 HDR 而言， 8bit 256色阶的数据是满足不了的。所以大部分 HDR related 的资源都改成了 32bit float 作为色彩通道的数据格式。

这就再也没有位不足的情况了，那么这些图片数据都是直接存储的物理量。他们是处于线性空间，而非 sRGB 空间的。你看，这里我一直在避免使用 “Gamma空间”这个有歧义的术语。

这是因为，Gamma 几乎已经融入到 SDR 图片资源的各个环节了。你的佳能相机，或者绘图软件Photoshop，设置的工作色彩空间是 sRGB 的话，你的数据也是会经过（类似） Gamma 编码；如果你在采用 Mac 机器，或者类似的 Linux 操作系统，在提交输出信号之后，也会进行相应的矫正。

所以为了让整个系统 $gamma=1$，显示链路的每一个环节都做了对应处理。其他的我们就不在此讨论了。


## Importance of Gamma Correction

这部分的内容，实际上是 GPU Gems 3[[2](#2)<a name="ref-2"></a>] 里有详细的讨论，我们就在这里列一列数据不准确产生的危害吧。

由于当时几年正好时渲染在朝着物理正确在发展的初期，为了能获取准确的渲染结果，正确的 Gamma Correction 就成为了第一个错误排查的步骤，避免因为显示器的影响导致观测结果的错误。

我们需要在 shader 或者 Gfx Device API 中抵消掉显示器的 DTF 曲线。否则，你就会获得这样一个屎结果

![perceptual band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/lighting_calculation_problem.png)

### Light Calculation

基于这个正确结果的同时，我们要求计算的过程中也是符合线性变换的。

之前讨论了这么多的 Gamma in Image 的情况，就是想明确，我们的图片资源也是需要 Gamma Correction 的。我们需要把 EncodedData 转到线性空间中来。

![perceptual band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/system_gamma_correction.png)

幸运的是，现代的 GPU 都是支持 sRGB 格式的，通过正确的设置，采样的时候，硬件会协助我们获得线性的结果。


### Mipmaps and Blending

在 shader 中手动转换并不是最好的选择，因为在贴图的采样过程中，Texture Filtering 也是需要在线性空间中计算才能获得正确的结果。

举个例子，我们生成 mip level 的时候，上一级的四个像素中分别有二个黑色像素和两个白色像素。我们预计的混合结果应该是 50% 的中度灰，得到 0.5 的数值结果。但是在 $\gamma=2.2$ 的设置下，实际上观测到的亮度只有 22%。

这个会让 mimaps 看起来更暗一些。在生成的过程中，可能不是很明显。但是当物体在渲染时，从近处移往远处时，mipmaps 切换的过程中，可以很明显的感知到亮度的跳变。

同理，输出到 Frame Buffer 之后，进行 Blending 的过程也需要线性空间。在 sRGB 空间中进行混合，得到的结果应该会变得过于明亮。

![perceptual band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/blending_calculation_problem.png)

如果使用了支持 sRGB 的格式，硬件会在每次混合之前完成转换，并在混合之后将结果重新编码到 sRGB 空间中。

### Anti-alasing

当我们渲染一个三角形的时候，三角形的边缘可能会覆盖部分的像素。如果没有正确进行 Gamma Correction，写入像素值的时候有可能会变得更暗。

![perceptual band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/anti_aliasing_problem_2.png)

这会使得抗锯齿算法将边缘误认为图12的右侧部分，最终影响输出图像。在做 Filtering 的时候，由于数值非线性，导致了还原出来的边界不是一条直线，最终造成了下图中左侧的 Roping 现象。

## Appendix 确认显示器 Gamma 的存在

为了验证显示器 Gamma 的存在，我们可以尝试做图像实验来。

用黑白两色方块均匀排列为棋盘状，将这个 Pattern 不断重复，直到看不清间隙。当色块密度足够时，看起来像是一个纯色块为止。

![perceptual band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/resize-large.png)

由于黑白格是均匀相间的，我们可以认为他们分别提供了整张图片的50%，据此推定此图为 50% 亮度的中度灰。以两纯色块作为参考对比，50%中度灰与 128 有很大差别，看起来与 187 的数值颜色能对应上。

![perceptual band]({{ site.url }}/assets/2021-09-06-why-gamma-correction/resize.png)

由此可知输入信号要达到 $\cfrac{187}{256}=73.3%$ 。根据之前对DTF，我们可以根据显示器 $\gamma=2.2$ 进行变换，最终结果正好是50%，也就是人眼睛观察的中度灰。

## References

<a name="1"></a> [[1](#ref-1)] 5.6. Display Encoding, Real-Time Rendering 4th edition<br/>
<a name="2"></a> [[2](#ref-2)] Chapter 24. The Importance of Being Linear, GPU Gems3, <https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-24-importance-being-linear><br/>
[3] WHAT EVERY CODER SHOULD KNOW ABOUT GAMMA, John Novak, <http://blog.johnnovak.net/2016/09/21/what-every-coder-should-know-about-gamma/><br/>
[4] UNDERSTANDING GAMMA CORRECTION, <https://www.cambridgeincolour.com/tutorials/gamma-correction.htm><br/>
<a name="5"></a> [[5](#ref-5)] Color spaces - REC.709 vs. sRGB, <https://www.image-engineering.de/library/technotes/714-color-spaces-rec-709-vs-srgb><br/>