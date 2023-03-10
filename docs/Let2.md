# 让我们建立一个 JPEG 解码器:概念

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Let>

本文是探索 JPEG 解码概念和实现的系列文章的第一部分；目前有四个部分可用，其他部分预计将随后推出。

*   [第一部分:概念](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-Concepts)
*   [第二部分:文件结构](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-File-Structure)
*   第三部分:霍夫曼表
*   [第四部分:帧和比特流](http://web.archive.org/web/20220810161352/http://imrannazar.com/Let%27s-Build-a-JPEG-Decoder%3A-Frames-and-Bitstreams)

建一个 JPEG 解码器？为什么，我们已经有这么多了？

JPEG 是我们都认为理所当然的东西:大部分网络包括以 JPEG 文件传输的图片，以及基于 JPEG 技术的视频文件。事实证明，这些图像背后的概念跨越了近 200 年的数学和计算理论，从原始文件到图像需要大量有趣的工作。

### 核心:频率分析

在 *[压缩简介](http://web.archive.org/web/20220810161352/http://imrannazar.com/An-Introduction-to-Compression)* 中，我研究了“无损”和“有损”压缩的区别:这两种方法的区别在于，无损压缩保留了输入的所有固有信息，而有损压缩丢弃了大部分信息。只有当信息被认为对正确处理文件没有必要时，才扔掉信息；例如，对于计算机程序来说，这是不可能的，因为其中的每个字节都是必须保留的语句。

图像或视频，以及它们在音频世界中的同类，依靠*感知*来确定什么需要被丢弃:就像人耳只能分辨特定频率范围内的声音一样，人眼具有特定的分辨率，在很短的距离内发生的任何颜色变化基本上都是看不见的。分辨率也可以被认为是“视觉频率”，可以像声波频率或其他类型的波一样进行操作。

由此可见，一段声音或一幅图像可以通过移除人类体验范围之外的频率范围来压缩:那些我们不关心的频率范围，没有这些频率范围，本质就不会丢失。要做到这一点，有三个步骤:

*   **变换样本**:声波是声级随时间的变化，图像是颜色经过二维空间的变化。我们需要通过某种转换，将它们转换成它们自身的频率表示。
*   **带通**:一旦我们进入频域，我们就需要截掉我们感兴趣的频率部分或频带。这方面的技术术语是“带通滤波器”。
*   **逆变换**:我们新减少的信号可以通过变换反向发送，产生一个去掉了多余信息的声音或图像。

变换用途源自[傅立叶变换](http://web.archive.org/web/20220810161352/http://en.wikipedia.org/wiki/Fourier_transform)，它采用可积分的数学函数 *f(x)* 并生成频谱的方程 *f(s)* 。傅立叶变换只对连续的数字范围起作用，并且从负无穷大扩展到正无穷大；这使得它无法用于我们这里需要的数字转换。相反，使用傅立叶变换的离散版本:MP3 和 JPEG 技术使用[离散余弦变换](http://web.archive.org/web/20220810161352/http://en.wikipedia.org/wiki/Discrete_cosine_transform) (DCT)将一组数据值转换成一组等价的频率。

下图给出了压缩过程的两个例子:首先是一个简短的声音样本。

![Sound bandpass filtering](img/0cf4dc9357d98f4b9e1312445a99ec55.png) *Figure 1: Chorus riff, "More Than a Feeling" (Boston, 1976)
Encoded to 40kbps MPEG audio Layer 3
Frequency analysis courtesy of [ARTA](http://web.archive.org/web/20220810161352/http://artalabs.hr/) for Windows, by Ivo Mateljan*

下图显示了与上面相同的过程，但是是在二维空间中应用，而不是在一维时间中。通常认为一次将整个图像转换到视觉频域是低效的，因此 JPEG 算法一次转换八个像素正方形的块。

![Image bandpass filtering](img/97102e1c219db1822a4ff8efec9874b4.png) *Figure 2: Icon from the [DryIcons Shine set](http://web.archive.org/web/20220810161352/http://dryicons.com/free-icons/preview/shine-icon-set/)
Encoded to JPEG, q=25%*

在图 2 中，右边的一组图像显示了 DCT 及其过滤后的版本。每个 8x8 区块中最重要的数字是左上角的“DC 分量”,它决定了整个区块的基本级别。右边的值给出了水平方向上方差发生频率的信息，相反，DC 分量以下的值提供了垂直方向上的频率信息。由此可见，DCT 块右下角的值描述了图像中的最高保真度变化，并且滤波包括在每个块上画一条对角线并保留顶部，丢弃关于高保真度变化的信息。

### 色彩空间和缩减取样

JPEG 利用了人眼在视觉变化时具有最高分辨率的事实。眼睛的另一个特征是，它对颜色变化的敏感度低于亮度变化:视网膜中对频率敏感的“锥体”细胞的密度低于更简单的频率不可知的“杆状细胞”，这意味着颜色的视觉分辨率更低。

通过利用该信息，减少用于编码与亮度相关的颜色值的信息量，可以进一步压缩图像。不幸的是，计算机和电视显示器使用的红/绿/蓝传统加色空间没有保留关于相对亮度和色彩饱和度的信息；为了检索该信息，RGB 值必须被转换成另一种颜色模型。

JPEG 格式最常用 Y'CbCr 颜色，其中 **Y'** 表示特定像素的亮度，而 **Cb** 和 **Cr** 分量描述两个轴上的色度量，对应于蓝色和红色的百分比。从 RGB 像素值到 Y'CbCr 的转换充当所有可能的 RGB 值的立方和可能的亮度/色度值的立方之间的旋转，如图 3 所示。

![YCbCr and RGB cubes](img/a9a3e26dfa753d83a1e2f83e6a2cb6f0.png) *Figure 3: The RGB cube, rotated within the Y'CbCr cube
From the [Intel Integrated Performance Primitives manual](http://web.archive.org/web/20220810161352/http://software.intel.com/sites/products/documentation/hpc/ipp/ippi/)*

一旦转换为 Y'CbCr，色度通道就被分离出来，并可以进行操作。在这种情况下，采用“下采样”:颜色通道的分辨率在两个维度上减半，使得一个颜色信息“块”覆盖四个亮度块的等效区域。在图 4 中，颜色通道已经按照这个比率进行了下采样:可以看出，**Y’**通道对于图像的完整性是最重要的，因此它的分辨率仍然很高。

![YCbCr channels](img/b751dd8a872985586173c9f302334e01.png) *Figure 4: A JPEG image broken into its Y'CbCr channels, with chrominance downsampling*

对于本系列文章的其余部分，将假设此处所示类型的双向下采样:网络上的大多数 JPEG 图像都采用这种颜色压缩，因此探索一下是有用的。由于分辨率的降低，前面提到的 JPEG 算法使用的八像素正方形变成了 16 像素正方形的最小单元大小，四个 8×8 亮度块伴随着每个颜色信息轴的一个块。

### 下一次:文件格式

JPEG 文件存储编码图像的附加信息:解码过程使用的查找表、注释和分辨率信息。在第二部分中，我将看一看组成 JPEG 文件的片段，以及如何保存为后续部分的解码实现提供的一些信息。

2013 年 1 月，伊姆兰·纳扎尔<>。

*文章日期:2013 年 1 月 5 日*