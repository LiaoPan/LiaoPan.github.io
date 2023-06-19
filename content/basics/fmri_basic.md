---
title: "Fmri_basic"
date: 2023-01-12T11:38:30+08:00
draft: true
---
susceptibility distortion correction 磁敏感失真校正

EPI序列采集过程中会**因磁场的不均匀而产生空间形变**，fMRIPrep提供了SDC的功能来校正形变，并将其单独拿出作为一个独立的模块，以用于同为EPI序列的diffusion MRI。

通常有5种校正的方法：
- Phase Encoding POLARity：采集两个不同相位编码方向的图像，通过对两次采集的配准来估计像素位置的改变。
- Direct B0 mapping sequences：直接测量B0图像
- Phase-difference B0 mapping：利用相位差估计B0图像
- “Fieldmap-less” estimation：通过非线性的配准在缺少fieldmap情况下直接校正
- Point-spread function acquisition：目前fMRIPrep不支持

Slice time correction 时间层校正
fMRI中，slice-timing调用的是AFNI的3dTShift，所有的volume都被对齐到TR中间时间点的volume，这一步骤可以通过--ignore slicetiming来跳过，当BOLD时间序列可用volume数量少于5，slicetiming将无法进行。


Head motion estimation 头动估计
使用reference image来估计每个volume的头动参数，这里调用了FSL的mcflirt模块估计刚体变换的6个头动参数，并将每个时间点的头动参数写入了confounds文件当中，而且这个头动估计过程是先于任何时域的预处理（slice-timing）来获得更准确的头动参数估计。

BOLD reference image estimation
在BOLD预处理过程中，首先生成一个参考像，目的是用来生成一个brain mask。当你的数据中含有single-band reference的时候，fMRIPrep就自动把它作为reference image。如果没有的话，会优先把检测到的dummy scans以及non-steady volumes 的平均作为reference。再如果没有检测到上述的volumes，就会用头动校正后的中位volume作为reference。


参考博客网址：
1. https://zhuanlan.zhihu.com/p/222656040
2. https://fmrif.nimh.nih.gov/public/fmri-course/fmri-course-summer-2019