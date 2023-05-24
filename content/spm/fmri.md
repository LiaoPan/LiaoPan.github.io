---
title: "SPM教程 #3. fMRI的分析"
date: 2023-02-07T13:03:26+08:00
draft: true
---


- [预处理](#预处理)
  - [步骤1. Realignment](#步骤1-realignment)
  - [步骤2. Slice-Timing Correction](#步骤2-slice-timing-correction)
  - [步骤3. Coregistration \& Normalization](#步骤3-coregistration--normalization)
  - [步骤4. Segment](#步骤4-segment)
  - [步骤5. Normalization](#步骤5-normalization)
  - [步骤6. Smoothed](#步骤6-smoothed)


#### 预处理
- Realignment
- Slice-Timing Correction：时间层校正
- Coregistration & Normalization：将功能像和结构像统一到标准空间，方便对比分析
- Smoothed:强化信号，减少噪音

当我们对fMRI数据进行预处理时，我们需要清理我们获得的每个TR的三维图像。一个fMRI体积(volume)不仅包含我们感兴趣的信号--含氧血液的变化，还包含我们不感兴趣的波动(fluctuations)，如头部运动(head motion)、随机漂移(random drifts)、呼吸(breathing)和心跳(heartbeats)。我们把这些波动称为噪声，因为我们想把它们与我们感兴趣的信号分开。其中一些可以通过建模从数据中回归出来，另一些可以通过预处理减少或去除。

![preprocess](/spm/images/01_fmri_preprocess.png)

##### 步骤1. Realignment
Realignment会将所有volumes数据整理对齐。

##### 步骤2. Slice-Timing Correction
![slice-time correction](/spm/images/02_SliceTimingCorrection.gif)


##### 步骤3. Coregistration & Normalization


1.仿射变换，Affine Transformations
![Affine Transfomrations](/spm/images/03_AffineTransformations1.gif)

{{% notice note %}}
与刚体变换一样，缩放（zooms）和剪切(shears)都有三个自由度。你可以沿X、Y或Z轴缩放或剪切图像。那么，总的来说，仿射变换有12个自由度。
{{% /notice %}}


![registration normalization](/spm/images/04_Registration_Normalization_Demo1.gif)

1. [Coregistration & Normalization参考](https://andysbrainbook.readthedocs.io/en/latest/SPM/SPM_Short_Course/SPM_04_Preprocessing/03_SPM_Coregistration.html)




##### 步骤4. Segment


[更加详细参考](https://andysbrainbook.readthedocs.io/en/latest/SPM/SPM_Short_Course/SPM_04_Preprocessing/04_SPM_Segmentation.html)

##### 步骤5. Normalization


[Normalization](https://andysbrainbook.readthedocs.io/en/latest/SPM/SPM_Short_Course/SPM_04_Preprocessing/05_SPM_Normalize.html)

##### 步骤6. Smoothed


![Smoothing](/spm/images/06_Smoothing_Demo.gif)

[Smoothing](https://andysbrainbook.readthedocs.io/en/latest/SPM/SPM_Short_Course/SPM_04_Preprocessing/06_SPM_Smoothing.html)