---
title: "FSL教程 #2.十分钟入门"
date: 2023-01-06T12:37:44+08:00
# weight: 2
draft: true
---

- [练习数据](#练习数据)
- [大脑提取工具（Brain Extraction,BET）](#大脑提取工具brain-extractionbet)
- [线性/非线性图像配准工具（FLIRT\&FNIRT）](#线性非线性图像配准工具flirtfnirt)
  - [**1.FLIRT**:表示FMRIB's的Linear Image Registration Tool](#1flirt表示fmribs的linear-image-registration-tool)
  - [**2. FNIRT**:表示FMRIB's的Non-Linear Image Registration Tool](#2-fnirt表示fmribs的non-linear-image-registration-tool)
- [案例分析](#案例分析)
  - [单步配准](#单步配准)
  - [多步配准](#多步配准)
  - [EPI变形校正](#epi变形校正)
- [病理异常图像配准](#病理异常图像配准)
- [配准故障如何排除？](#配准故障如何排除)
- [参考文献](#参考文献)


#### 练习数据
安装FSL后，打开终端，输入/粘贴以下命令即可下载课程数据(粘贴后不要忘记按回车键才能运行该命令): 
 ```shell
fsl_add_module
```



#### 大脑提取工具（Brain Extraction,BET）


#### 线性/非线性图像配准工具（FLIRT&FNIRT）
##### **1.FLIRT**:表示FMRIB's的Linear Image Registration Tool

![FLIRT](/fsl/images/03_flirt.png)

- 重定位(Reorientation)
  - 使用`fslreorient2std`实现重定位
- 大脑提取(Brain Extraction)
  - 使用`bet`实现大脑提取
- 偏置场校正(Bias-field correction)
  - 使用`FAST`实现偏置场校正 

##### **2. FNIRT**:表示FMRIB's的Non-Linear Image Registration Tool



#### 案例分析
##### 单步配准
- 场景1: 有两种或以上的来自同一个被试的不同类型图像，比如一个被试的T1加权和T2加权图像。
  - 目的：将图像对齐后，便于后续分析。比如将图像用于多模态分割。
  - 方法：使用FLIRT进行6自由度（Degrees of Freedom,DOF）的处理

##### 多步配准
- 场景2:进行功能性或者弥散研究时，有着对应每个被试的EPI或者T1加权图像。
  - 目的：需要将图像配准到一个标准空间从而进行组研究
  - 方法：使用（在FEAT中）FLIRT & FNIRT进行两步配准
  
对不通图像进行配准的难点在于：
- 个体的解剖结构不同
- 不同模态的对比不同
- 不同图像间的变形不同

如果要将EPI图像配准到标准空间模板（MNI152），请使用结构图像作为中介。

![FEAT](/fsl/images/04_feat.png)
![FEAT](/fsl/images/05_feat_01.png)
![FEAT](/fsl/images/06_feat_02.png)


##### EPI变形校正
- 场景3:进行功能性或者弥散研究
  - 目的：修正EPI图像中的变形，否则无法精确配准
  - 方法：使用FUGUE/FEAT进行基于场图的修正


EPI配准的难点：
- EPI图像出现变形和信息损失
  - 导致：无法很好地进行标准配准
  - 方法：
    - 通过`去变形`来消除变形
    - 忽略高信号损失区域
    - 需要一个场图（fieldmap）,需要专门采集

![EPI](/fsl/images/07_EPI.png)
![EPI](/fsl/images/08_EPI.png)

- EPI对于任何偏离均匀B<sub>0</sub>场的情况都很敏感
- 单独的场图可以测量B<sub>0</sub>的偏离

- 从场图中，我们可以获得：
  - 空间变形的幅度（仅限相位编码方向）
  - 估计信号损失
- 只需要花几分钟采集一个场图，就能大大地改善配准质量
- 每一次扫描会话都需要采集一个新的场图，因为场图会随着扫描的更新而变化（如它取决于头的朝向）
- 可使用FUGUE来`去变形`

![B0](/fsl/images/09_B0.png)
![B0](/fsl/images/10_B0.png)



#### 病理异常图像配准
- 场景4:图像中包含一些不同于其他图像的未知病理学异常或者伪影。比如说某些序列（如FLAIR）会突出其他序列很难发现的损伤。
  - 目的：基于健康组织，“忽视”异常或者伪影区域来对齐图像
  - 方法：代价函数加权（FLIRT 或 FNIRT）
    - 注意：所有的FLIRT&FNIRT代价函数都可以进行加权
    - 可对参考图像，输入图像分别或者同时进行加权
    - 体素的权重是相对的，反映了它们在整体匹配中的重要性。
      - 比如将异常区域的权重设为0或者小值，例如病理学异常区域或者伪影。
      - 将重要区域的权重设为大值，例如脑室匹配。
    - 不要将背景设定为0，因为这样会导致大脑/背景对比丢失。

#### 配准故障如何排除？
- 检查图像：
  - 体素大小
  - 伪影
  - 大的偏置场
- 检查大脑提取：检查大范围的/一致的错误
- 对于EPI：采集并使用场图来校正变形
- 对于fMRI或者弥散：使用结构像来进行两步配准以获取最佳结果
- 如果存在病理学异常/伪影：使用代价函数去权重
- 如果图像本身几乎已对齐：尝试限制搜索
- 对于FLIRT：可以尝试不同的代价函数
- 对于FNIRT：检查初始仿射对齐是否正常
- 对于小的视角(small FOV):采集全脑EPI来进行两步配准






![B0](/fsl/images/11_cost_function.png)


#### 参考文献
- https://fsl.fmrib.ox.ac.uk/fslcourse/2019_Beijing/lectures/reg.pdf
- 


FSL配准等工具的原理介绍

https://gate.nmr.mgh.harvard.edu/wiki/whynhow/images/8/8f/WhyNhow_FSL_final-WEB.pdf

transform matrix介绍
https://brainimaging.waisman.wisc.edu/~oakes/teaching/Coregistration_lecture.pdf




fslroi - extract region of interest (ROI) from an image 
https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/Fslutils#fslroi


白质高信号的自动分割
https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/BIANCA
