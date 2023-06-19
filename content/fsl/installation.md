---
title: "FSL系列教程 #1.安装"
date: 2023-02-01T19:00:48+08:00
weight: 1
draft: true
---

- [安装方法简述](#安装方法简述)
- [安装校验](#安装校验)
- [在MATLAB上使用FSL](#在matlab上使用fsl)
- [公开数据下载](#公开数据下载)
- [参考](#参考)


#### 安装方法简述

1. 访问[网址](http://fsl.fmrib.ox.ac.uk/fsldownloads),填写信息，下载对应版本的软件；
2. 跟着教程指导，安装FSL软件。
   1. 打开终端命令行，使用管理员权限运行下述脚本
```
cd ~/Your_Downloads_DIR
sudo python fslinstaller.py
```

{{% notice warning %}}
注意: 1. 你的电脑上得有**python的运行环境**，不然无法执行下述python脚本。
2. **有网络**，可以方便脚本自动从网络上下载相关软件包。
{{% /notice %}}

#### 安装校验
1. 打开终端命令行，执行下述脚本:
   ```
   $ echo $FSLDIR # 校验环境变量
   $ flirt -version # 校验软件是否正常使用，成功会输出:FLIRT version 6.0,版本号会有差异。
   $ which imcp # 校验miniconda environment installation
   /usr/local/fsl/share/fsl/bin/imcp
   ```

2. 通过在终端命令行输入`fsl`,可打开UI界面；
3. 通过类似`<fsl命令>_gui`方式，可以打开UI界面。比如`Bet_gui`,注意fsl命令的首字母需要大写。



#### 在MATLAB上使用FSL
{{% notice note %}}
在 macOS 上，fslinstaller 脚本通常会为您进行设置，因此您不需要这样做。但是，如果安装程序由于某种原因无法配置 MATLAB，您可能需要手动执行此操作。
{{% /notice %}}

{{< tabs >}}
{{% tab name="MATLAB" %}}
```octave
% FSL Setup
setenv( 'FSLDIR', '/usr/local/fsl' );
setenv('FSLOUTPUTTYPE', 'NIFTI_GZ');
fsldir = getenv('FSLDIR');
fsldirmpath = sprintf('%s/etc/matlab',fsldir);
path(path, fsldirmpath);
clear fsldir fsldirmpath;
```
{{% /tab %}}
{{< /tabs >}}


#### 公开数据下载
访问Openeuro官网，下载[Flanker task](https://openneuro.org/datasets/ds000102/versions/00001)的fMRI数据集。


#### 参考
1. [官网安装教程](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FslInstallation)
2. [Using FSL from MATLAB, MAC OS](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FslInstallation/MacOsX)
3. [fMRI_DataDownload](https://andysbrainbook.readthedocs.io/en/latest/fMRI_Short_Course/fMRI_01_DataDownload.html)