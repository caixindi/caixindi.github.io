---
title: Anatomy of High-Performance Matrix Multiplication
date: 2021-12-03
categories:
- CS217
tags:
- 高性能矩阵乘法剖析
language: zh-CN
toc: true
---

##### [Anatomy of High-Performance Matrix Multiplication](https://www.cs.utexas.edu/users/pingali/CS378/2008sp/papers/gotoPaper.pdf)

​	现在我们进行机器学习训练，通常都会使用一些机器学习库，比如TensorFlow这样的库，并且在训练机器学习模型时，通常这些库对性能的提升是数量级的提升。以下以卷积计算为例，去剖析高性能矩阵计算。

```python
'''
Convolve `input` with `kernel` to generate `output`
    input.shape = [input_channels, input_height, input_width]
    kernel.shape = [num_filters, input_channels, kernel_height, kernel_width]
    output.shape = [num_filters, output_height, output_width]
'''
for filter in 0..num_filters
    for channel in 0..input_channels
        for out_h in 0..output_height
            for out_w in 0..output_width
                for k_h in 0..kernel_height
                    for k_w in 0..kernel_width
                        output[filter, out_h, out_w] += 
                            kernel[filter, channel, k_h, k_w] * 
                            input[channel, out_h + k_h, out_w + k_w]
```

​	<!--more-->

​	这是一个六层嵌套的for循环，也是通常我们能想到的卷积计算代码，但是它的执行效率非常低。主要原因就是嵌套for循环使得数据访问非常困难，这使得缓存利用率极低，缓存中的数据经常被换入换出。

​	在逻辑上我们将矩阵/图像/张量看出是多维数据，而事实上不管你逻辑上几维的数据，都将存储在线性的一维计算机内存中。为此，如何将这些高维数据按照一定的次序展开到内存中非常重要。

​	大部分现代dl库使用行主序存储，所以同一行的连续元素将被相邻存储，这也意味在访问内存时，第一维度（行）变化速度最慢。对于四维张量，我们知道有NCHW,NHWC等存储顺序，N代表数量， C代表channel，H代表高度，W代表宽度。比如下图所示，现有N块H×W图像的C通道，那么相同通道图像都是重叠的，同一通道C的所有像素也是重叠的。

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/storage-order.png" style="zoom:80%;" />

​	上面描述的只是一个很简单的卷积，但其执行速度已经很慢了，随着步长，填充等参数增加只会变得更加复杂。

​	通用矩阵乘（GEMM，General Matrix Multiplication）可以用于实现卷积。将图像块放到一个矩阵中的操作成为im2col，im2col 是计算机视觉领域中将图片的不同通道（channel）转换成矩阵的列（column）的计算过程。Caffe 在计算卷积时，首先用 im2col 将输入的三维数据转换成二维矩阵，使得卷积计算可表示成两个二维矩阵相乘，从而充分利用已经优化好的 GEMM 库来为各个平台加速卷积计算。如下图所示：

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/direct-conv-im2col.png" style="zoom:80%;" />

​	这是一个3×3卷积，下面通过im2col，随着卷积过滤器在输入上滑动，将被计算的那部分输入展开成一行大小的向量。在滑动结束后，则会得到特征矩阵(K²C×HW)，将过滤器展开成N×K²C的矩阵，最后卷积的结果就可以表示为两个矩阵相乘的结果。

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/matrix-im2col.png" style="zoom:80%;" />

​	im2col 计算卷积使用 GEMM 的代价是额外的内存开销，因为不同的图像块之间往往存在一定的重叠，因此im2col会产生一定的内存重复。当卷积核尺寸是 1×1 时，由于不需要重排输入，GEMM 可以直接在原始输入上运行，并且不需要使用额外的内存。

​	通过im2col已经将卷积运算转化为了矩阵乘法，现在需要做的就是如何实现矩阵乘运算加速（GEMM）。通常矩阵乘算法描述如下：

```python
for i in 0..M:
     for j in 0..N:
         for k in 0..K:
             C[i, j] += A[i, k] * B[k, j]
```

最内部执行两个浮点运算（乘法和加法），总共执行M×N×K次，因此这个GEMM的FLOPs为2MNK。随着矩阵尺寸的增大，我们可以发现运算性能不断下降。虽然我们研究的是如何使得矩阵计算更快，但首先需要解决当是如何快速获取数据，因为缓存的容量有限，如果矩阵特征尺寸很大，以至于缓存无法容纳矩阵，性能会急剧下降。我们都知道存取速度CPU>Cache>RAM。CPU每次从主存中获取数据时，都会将其相邻的数据加载到缓存中，期望利用局部性原理以提高效率。但是在矩阵运算中，例如A×B的矩阵运算，我们在A上按行遍历，在B上按列遍历，在A上我们很好地利用了局部性，可是在B上会发现（在B尺寸大的情况下）cache总是会miss，因为cache放入的是B中同一行的其他元素，而我们需要的是B的其他行的元素，所以每次放入缓存的都是我们下一次不需要的元素，这就极大地影响了效率。因此需要重新排列循环次序，从i,j,k到i,k,j，即：

```
for i in 0..M:
     for k in 0..K:
         for J in 0..N:
             C[i, j] += A[i, k] * B[k, j]
```

这是会发现矩阵计算效率得到极大的提升。这时我们会发现一个新的问题，当我们读取A的第一行，循环遍历B的每一列，用于计算结果矩阵的的第一行。当我们读取A的下一列，我们又会循环遍历B的每一列。这样同样数据就会在缓存中不断换入换出。通常我们并不能将整个矩阵都放入缓存，我们可以将原矩阵分解为若干个子矩阵，将矩阵乘法分解子矩阵上的矩阵乘法，就可以避免这样的现象。比如我们需要得到结果矩阵中一个小的m×n块，我们只需要A中的m行和B中的n列。

​	此外，我们可以通过矢量化，将标量数据矢量化，通过SIMD(单指令多数据流)实现在相同的时钟周期对多个值执行相同的操作，以此实现运算加速。另外我们可以设计专用硬件单元实现乘加运算，将两条指令合并为一条。并且我们可以通过线程优化实现加速，在多内核的CPU上，每个内核可以同时物理地执行多个指令。一个程序可以把自己分成多个线程，每个线程可以运行在一个单独的内核上。最后，我们可以通过循环展开的方法消除分支以及一些管理归纳变量的代码，以此摊销一些分支开销。

