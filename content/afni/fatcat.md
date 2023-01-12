---
title: "Fatcat"
date: 2023-01-09T18:45:18+08:00
draft: true
---

### 介绍
FATCAT(The Functional And Tractographic Connectivity Toolbox)包含了处理和分析MRI数据多个程序，包括FMRI和Diffusion相关的数据。该程序被设计用来与AFNI和SUMA其余部分的分析工具一起使用。
![fatcat](/content/afni/images/FAT_overview.jpeg)

### FATCAT能做什么？
- 计算静息态功能连接(RSFC,resting state functional connectivity)参数，比如ReHo, ALFF, fALFF, RSFA等. 
  - ```3dReHo, 3dRSFC; `-regress_RSFC’ switch in afni_proc.py;```

- 计算 ROI 网络和/或全脑连接图之间的相关矩阵
  - ```3dNetCorr```
  
- 将FMRI和其他数据转换为目标 ROI 网络以进行纤维束成像
  - ```3dROIMaker```
- 估计不同数据间的体积差异（volume matching）
  - ```3dMatch ```
- 简单的扩散像梯度操作
  - ```1dDW_Grad_o_Mat, 3dTORTOISEtoHere```
- 确定性、小概率和全概率纤维束追踪
  - ```3dTrackID, 3dDWUncert```
- 生成模拟的莱斯噪声（Rician-noised）数据
  - ```3dDTtoNoisyDWI```
- 组水平的统计分析
  - ```3dMVM (fat_*.py)```
- 输出矩阵的保存、查看以及行的选择
  - ```fat_roi_row.py, fat_mat_sel.py```


参考:
https://afni.nimh.nih.gov/pub/dist/doc/htmldoc/FATCAT/FATCAT_All.html