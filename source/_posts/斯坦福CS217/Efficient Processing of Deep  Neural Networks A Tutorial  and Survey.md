---
title: Efficient Processing of Deep  Neural Networks: A Tutorial and Survey
date: 2021-12-03
categories:
- CS217
tags:
- 深度神经网络的高效处理：教程和综述
language: zh-CN
toc: true
---

### [Efficient Processing of Deep  Neural Networks: A Tutorial  and Survey](https://www.rle.mit.edu/eems/wp-content/uploads/2017/11/2017_pieee_dnn.pdf)

​		这是一篇有关于深度神经网络加速的综速。本文介绍了DNN的历史背景和作用，介绍的DNN的结构以及当前流行的DNN模型。后面不仅仅介绍了如何从硬件层面加速神经网络，也介绍了从算法，从软件，从软硬件结合的层面实现DNN加速。

<!--more-->

#### 1. DNN的背景

- Artificial Intelligence and DNNs

  讲述了人工智能、机器学习、神经网络之间的关系，这在智能计算系统的课程中已经学过。

- Neural Networks and DNNs 

  从神经网络到深度神经网络，深度神经网络比浅层神经网络拥有更强的表征学习能力，这使得DNN在很多任务中都能有很高的性能。

- Inference Versus Training 

  推理和训练，训练就是要确定神经网络中的权重和偏差的值，通常使用梯度下降法来优化损失函数。推理就是使用训练得出的神经网络来计算任务。同时也介绍了一些训练权重的方法，比如监督学习、无监督学习和半监督学习。

- Development History

  这里讲了神经网络的发展历程，如图所示：

  <img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/DNN%20Timeline.png" />

  主要就是神经网络是什么时候提出的（1940年代），但直到1980年代后期出现了手写数字识别的LeNet网络后才被广泛使用。随着数据量增大、计算能力的提升和算法的发展，2011年微软的语音识别系统出现，基于DNN的应用才蓬勃发展起来。本文主要关注的是DNN推理的高效处理，具体的如何让DNN推理在资源有限的嵌入式设备（而不是云）上执行。

- Applications of DNNs 

  - Image and video
  - Speech and language
  - Medical DNNs
  - Game play
  - Robotics DNNs

  现在DNN的应用已经非常广泛，如何实现DNN的高效处理成为了新的挑战。

- Embedded Versus Cloud 

  这边引出了本文的优化目标，具体来说，我们都知道训练通常需要大量的数据集以及大量的计算资源，所以训练过程通常都在在云上执行。然而在许多的应用中，出于节约通信成本、安全性（比如无人驾驶）和隐私安全的原因，都更希望推理在设备本地执行，但是执行DNN推理的嵌入式平台都有严格的能耗、计算和内存限制，在这些限制之下，如何使得DNN高效处理变得至关重要。

#### 2. DNN概述

​		这里首先介绍了处理神经网络的两种主要形式：前馈和循环。在前馈网络中，所有计算都是作为对前一层输出的一系列操作来执行的，最后一组操作生成网络的输出，网络是没有记忆的。而循环神经网络是有记忆的，它允许一些中间操作生成的值存储在网络内部，并用作其他操作的输入。

- Convolutional Neural Networks (CNN)

  卷积神经网络，由CONV（卷积层、高维）、FC（全连接层）、Pooling（池化层）、softmax层构成。常用的激活函数有Sigmoid，Tanh，ReLu

- Popular DNN Models

  文中给出了一些热门的DNN模型，如表所示：

  <img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/Summary%20of%20Popular%20DNNS-16342872863077.png" />

  文中通过上表给出了DNN的几个趋势，一个是网络越神，准确性越高；另一个是大部分计算都在卷积层，并且卷积层在权重层面也是主要方面，所以硬件加速的重点应该放在解决卷积层效率上。

#### 3.  DNN开发资源

- Frameworks

  介绍了一些深度学习框架：Caffe、TensorFlow、PyTorch。

- Models

  预训练模型。

- Popular Data Sets for Classification 

  数据集，需要注意的是，在评估DNN准确性时，需要考虑数据集的准确性。

- Data Sets for Other Tasks

  其他任务的数据集，现今最先进的DNN在图像分类任务上的表现已经优于人类的准确性，由此人们也开始挑战一些更困难的任务，比如单目标定位和目标检测。

#### 4. 支持DNN处理的硬件

​		由于 DNN 的流行，许多硬件平台都针对 DNN 处理增加了新的功能。 因此，重要的是要很好地了解如何在这些平台上执行处理，以及如何为 DNN 设计特定于应用程序的加速器，以进一步提高吞吐量和能源效率。

​		CONV 和 FC 层的基本组成部分是乘法累加 (MAC) 操作，它很容易实现并行化。 从架构层面来分，主要分为时间架构（CPU/GPU）和空间架构（FPGA）。如图所示：

<img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/image-20211015200717712.png"  />

​		CPU/GPU相对于通用架构，可以利用向量（SIMD）或者并行线程（SIMT）来提高并行性，这种架构对大量的ALU进行集中控制，但是这些ALU只能从存储器中获取数据，不能直接通信。对于任意数据流图中的一个计算，该架构往往需要多个周期才能完成。

​		而加速器可以实现数据流处理，就是按照数据流图将ALU组成一个数据链，每个 ALU 可以有自己的控制逻辑和本地存储器，这样的ALU被称为PE，这样的结构非常高效，我们需要解决的问题就是如何提高数据的重用性以降低能耗。

- Accelerate Kernel Computation on CPU and  GPU Platforms

  加速CPU和GPU平台上的内核计算，在这些平台上，FC和CONV层通常都映射为矩阵乘法，完成加速的核心就是实现高性能矩阵计算（硬件和算法层面）。

- Energy-Efficient Dataflow for Accelerators

  加速器的能效数据流，首先以此MAC操作至少需要4次访存操作，包括三次内存读取（用于过滤器权重、fmap激活和部分和）和一次内存写入（用于更新的部分和）。而内存访问的能耗很高，不同层级的访存能耗相差几个数量级。因此加速器第一步要做的就是减少访存，换言之就是减少数据移动的开销。

  - 多级内存层次结构（如图）

    <img src="https://cxd-note-img.oss-cn-hangzhou.aliyuncs.com/typora-note-img/Memory%20hierarchy%20and%20data%20movement%20energy.png"/>

    例如，与从 DRAM 中获取数据相比，从 RF 或相邻 PE 获取数据所需的能量要低一到两个数量级。因此，每次将一条数据从高能耗层次移动到低能耗层次时，我们希望尽可能多得重用该数据，从而最大限度地减少对高能耗层次的访存。

  - 数据重用

    对于DNN，有卷积重用、特征图重用、过滤器重用。

    文中还介绍了几种数据流的分类方案：weight stationary，output stationary，no local reuse，row stationary。

    这几种方法就是每次重用的数据不一样，流动的方式也不一样。

#### 5. 近数据处理

​		上一节讲了数据移动在能源消耗中占主导低位，虽然空间架构分布的片上存储器使得数据更靠近计算，但是否能够更加靠近，甚至使片外高密度存储器更靠近计算或者直接集成计算。这称为**存内计算**。

- DRAM

  嵌入式 DRAM (eDRAM) ，就是把DRAM嵌入到芯片里，即片上高密度存储器，可以减少切换片外电容的高能耗成本，eDRAM的密度比 SRAM高2.85 倍，比DRAM (DDR3) 的能效高 321倍，eDRAM还拥有更高的带宽和更低的延迟。此外还可以使用硅通孔（TSV）将DRAM堆叠到芯片顶部，这种技术被称为3D内存。

- SRAM

  将计算引入内存，例如，乘法累加操作可以直接集成到 SRAM 阵列 的位单元中。

- Nonvolatile Resistive Memories

  乘法和累加操作也可以直接集成到高级非易失性高密度存储器中，将它们用作可编程电阻元件，通常称为[忆阻器](https://baike.baidu.com/item/%E5%BF%86%E9%98%BB%E5%99%A8/2196463?fr=aladdin) 。具体来说，就是以电阻的电导为权重，以电压为输入，以电流为输出，进行乘法运算。

- Sensors

  在某些应用中，例如图像处理，来自传感器本身的数据移动会占系统能耗的很大一部分。 因此，也有关于在尽可能靠近传感器的地方执行计算的研究。

#### 6. DNN模型和硬件的协同设计

​		早期DNN模型更加注重准确性，却没有考虑过程实现的复杂性。因此很可能导致设计难以实施和部署。因此提出了DNN模型和硬件协同设计，从而最大限度地减少能耗和成本。现在主要有两种应用方案：

​		（1）降低运算和操作数的精度（从浮点到定点、减少位宽、非均匀量化和权重共享）

​		（2）减小操作数量和模型大小（模型压缩、神经网络裁剪和稀疏化）

#### 7. 用于DNN评估和比较的基准测试指标

- DNN模型指标

  - 模型在ImageNet等数据集上的top-5的准确性。

  - 模型的网络架构

  - 模型权重的数量（应包括非零权重数量，反应理论上的最低存储要求）

  - 模型需要执行的MAC数量

- DNN硬件指标
  - 功耗和能耗
  - 延迟和吞吐量
  - 芯片成本（比如每乘法器的平方毫米核心面积以及所采用的工艺技术）