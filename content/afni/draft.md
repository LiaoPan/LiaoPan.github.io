---
title: "Draft"
date: 2023-01-09T18:45:18+08:00
draft: true
---

问题：：为什么要把T1配准到fMRI？


![fmri_T1_registration](/content/afni/images/fmri_T1_registration.png)

由图可见，T1图像配准到标准空间需要两步，而fMRI配准到标准空间只需一步。T1在第一步配准时，是直接配准到该样本的平均fMRI上，因此第一步只需配准一次。那为什么要把T1配准到fMRI上，再配准到MNI空间呢？

假设我们反过来，先把fMRI配到T1，再配到MNI空间，此时会出现两个问题：

① T1图像的高分辨率特性没有发挥出来

在把fMRI配到T1，再配到MNI空间这个过程中，T1实际上并没有发挥作用。因为fMRI缺少解剖细节，即使已经先配准到精细的解剖图像T1，但到后面再配准到MNI空间时都只能在大体轮廓上对准，而里面的解剖结果会非常不准确，所以后面生成的“fMRI空间→MNI空间”转换矩阵是很不精准的。为了解决这个问题，就需要先把T1配准到fMRI上，再把已经在fMRI空间的T1图像配准到MNI空间，这样得到的“fMRI空间→MNI空间”转换矩阵就更为准确，包括更多解剖结构的配准细节。

② 先把fMRI配到T1，再配到MNI空间，那么fMRI配准到MNI空间共需要两步

配准过程中需要对图像进行插值等处理，因此配准次数越多，图像就会积累更多的误差。fMRI本身已经比较模糊，越多的配准次数只会使之更加不准确，这将对后面的网络分析带来严重影响。

因此在一般的fMRI配准流程中，都是把T1配准到fMRI上，以借助T1精细的解剖结构得到更准确的转换矩阵

AFNI常用命令汇总
https://learning-archive.org/wp-content/uploads/2017/09/FSL%E5%92%8CAFNI%E5%B8%B8%E7%94%A8%E5%91%BD%E4%BB%A4%E6%B1%87%E6%80%BB.pdf


原文链接：https://blog.csdn.net/qq_33924470/article/details/108106069