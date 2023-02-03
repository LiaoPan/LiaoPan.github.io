---
title: "Qsiprep 扩散像预处理"
date: 2023-01-13T17:28:26+08:00
draft: true
weight: 6
pre: <b>5. </b>
---

#### 简介
QSIPrep是用来配置处理弥散加权MRI（dMRI）数据的Pipelines。

#### QSIPrep能做什么?
- 用BIDS格式的方法来预处理几乎所有种类的现代弥散MRI数据。

- 自动生成预处理管道，包括correctly group, distortion correct, motion correct, denoise, coregister and resample your scans，产生视觉报告和QC指标。

- 一个运行最先进的重建pipelines的系统，包括来自Dipy、MRTrix、DSI Studio和其他的算法。

- 一种新型的运动校正（motion correction）算法，适用于DSI和随机Q空间(q-space)采样方案。

![qsiprep](/qsiprep/images/workflow_full.png)

#### 预处理
预处理管道（pipelines）是根据现有的BIDS格式输入建立的，确保正确处理场图(fieldmaps)。预处理工作流程包括执行头部运动校正(head motion correction)、磁敏感失真校正(susceptibility distortion correction)、MP-PCA去噪、与T1w图像的配准、使用ANT的空间归一化以及组织分割(tissue segmentation)。

#### 重建
[预处理管道](https://qsiprep.readthedocs.io/en/latest/#preprocessing-def)的输出可以对接许多其他软件包中来进行重建。我们在`qsiprep`中提供了一套精心策划的[重建工作](https://qsiprep.readthedocs.io/en/latest/api/index.html#recon-workflows)流程，可以运行ODF/FOD重建、纤维束成像(tractography)、Fixel估计(Fixel estimation )和区域连接(regional connectivity)。


[QSIPrep官网文档](https://qsiprep.readthedocs.io/en/latest/)
