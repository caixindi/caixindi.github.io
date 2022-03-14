---
title: 硬件加速器背景介绍（Lecture1）
date: 2021-12-03
categories:
- CS217
tags:
- CS217 Lecture
language: zh-CN
toc: true
---

##### 硬件加速器背景

​	如今计算能力限制了训练机器学习模型的工作，如果我们有更快的处理器我们可以运行更大的模型。

##### 现有的机器学习加速器

- CPU：线程、SIMD（单指令多数据流）
- GPU：大量线程、SIMD、HBM（高带宽存储器）
- FPGA：LUTs（查找表）、DSP、BRAM
- TPU：MM Unit、BRAM

<!--more-->

##### 关键问题

- 如何提高机器学习速度？

（1）摩尔定律减缓和能量墙

（2）提高性能/瓦数

（3）应用新的机器学习程序和能力

（4）使得机器学习易于使用

- 如何平衡性能和可编程性？

比如ASIC(专用集成电路)的能效比和处理器一样的灵活性

- 需要全栈协同设计（机器学习算法、编译器、硬件）

##### Scaling 技术

登纳德系数（Dennard's Factor）：
$$
\alpha=a/b
$$
 为两代特征尺寸之比。

耗散功率：
$$
Power Dissipation\approx CV^2f=\alpha^2(C/\alpha(V/\alpha)^2\alpha f)
$$
其中$V$为供电电压，$f$为时钟频率。提升α可得到更多晶体管（ $\alpha^2 $ ），更高性能（ $\alpha f$ ）。

随着晶体管尺度缩小到达极限，供电电压V和时钟频率F无法继续等比缩小，$CV^2f=\alpha^2(C/\alpha V^2\alpha f)=\alpha CV^2f$， 造成耗散功率增加。

