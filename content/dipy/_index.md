---
title: "DIPY系列教程"
date: 2023-01-05T23:05:15+08:00
draft: false
weight: 8
# chapter: true
pre: "<b>8. </b>"
---

### [DIPY简介](https://dipy.org/)
DIPY是Python中标准的3D/4D成像库，包含空间归一化、信号处理、机器学习、统计分析和医学图像可视化的通用方法。此外，它还包含计算解剖学的专门方法，包括扩散、灌注和结构成像。

### DIPY功能
- 命令行接口
  - 所有算法都可以使用命令行的方式调用
  - 创建自己的命令行
- 统计分析
  - BUAN
  - AFQ
  - K折交叉验证
- 重构
  - Single Shell:DTI, CSA, SFM, SDT, Q-Ball, CSD, ...
  - Multi-Shell:GQI, DTI, DKI, SHORE, MAPMRI, MSMT-CSD, ...
- 配准
  - 仿射变换
  - 2D/3D微分同胚配准（Diffeomorphic 2D/3D Registration）
- 纤维束成像
  - 概率性纤维束追踪
  - 确定性纤维束追踪
  - PFT纤维束追踪
- 去噪
  - Patch2self
  - Gibbs Unringing
  - LPCA - MPPCA
  - Non Local Means
- 可视化
  - ODFs可视化
  - 交互式的纤维束可视化
- 预处理
  - Brain extraction
  - SNR estimation
  - Reslice Datasets

### 安装方式
```shell
$ pip install nibabel # 用于读写神经影像数据
$ pip install dipy
$ pip install fury # 某些可视化依赖库
```
当然，也可以使用conda方式进行安装。

[更多安装教程，请参考官网](https://dipy.org/documentation/1.5.0/installation/)

{{% notice tip %}}
如果看到这不知道什么是pip或者conda，可以去学习一下python相关基础。
{{% /notice %}}


### 教程数据下载
1. 使用python代码下载，**数据集默认会下载在主目录下的.dipy目录内。**
{{<jupyter dipy dipy_quickstart 724>}}

{{% notice warning %}}
尽量不要使用[jupyter](https://jupyter.org/)来执行上述命令下载数据，因为目前jupyter的下载进度支持不好，无法查看下载是否已完成且容易僵住。
{{% /notice %}}

2. 使用命令行工具下载,选择特定目录，执行下面命令即可(推荐)
  ```shell
  $ dipy_fetch list # 查看所有可使用的数据集

  INFO:Please, select between the following data names: bundle_atlas_hcp842, bundle_fa_hcp, bundles_2_subjects, cenir_multib, cfin_multib, file_formats, fury_surface, gold_standard_io, isbi2013_2shell, ivim, mni_template, qtdMRI_test_retest_2subjects, qte_lte_pte, resdnn_weights, scil_b0, sherbrooke_3shell, stanford_hardi, stanford_labels, stanford_pve_maps, stanford_t1, syn_data, taiwan_ntu_dsi, target_tractogram_hcp, tissue_data

  $ dipy_fetch {specific_dataset} --out_dir {specific_data_out_folder} # 选择特定数据集，下载到特定目录，注意{}内内容需要替换。

  $ dipy_fetch sherbrooke_3shell --out_dir . # 举例，将sherbrooke_3shell数据集下载到当前目录。

  ```

  {{%notice tip %}}
  使用dipy_fetch时，--out_dir不写，即下载到默认目录下(~/.dipy)
  {{% /notice%}}

  {{%notice note %}}
  
  注意 1: 不需要马上下载上述所有数据集，在后续教程中，会陆续使用命令下载教程相关数据。
  注意 2: 在下载过程中，经常会碰到报500错误(HTTP Error 500: Internal Server Error)的情况，重新开始即可。
  {{% /notice%}}

  <!-- 
  参考教程：
  http://www.diffusion-imaging.com/ -->