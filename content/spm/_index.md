---
title: "SPM系列教程"
date: 2023-01-05T23:05:15+08:00
draft: false
weight: 3
# chapter: true
pre: "<b>3. </b>"
---

###  SPM介绍
[SPM(Statistical Parametric Mapping)](https://www.fil.ion.ucl.ac.uk/spm/)分析功能成像数据主流的一个开源软件包，可以分析多种脑成像数据，包括fMRI，PET，SPECT，EEG和MEG。
SPM软件包专为分析脑成像数据序列而设计，这些序列包括不同队列（cohorts）的一系列图像或者同一个被试的时间序列。

### 下载与安装
**1. 软件下载**

点击[链接](https://www.fil.ion.ucl.ac.uk/spm/software/download/)，填写想要下载的版本、系统、MATLAB等版本信息后，即可下载SPM对应版本软件。
![SPM下载页面](/spm/images/spm_download_2.png)
点击链接，下载即可。


**2. 软件安装**

Mac系统的SPM8软件安装方法:
1. 解压已下载的spm软件包
   1. 方法一: 双击压缩包完成解压
   2. 方法二: 使用命令行解压
   {{< tabs >}}
   {{% tab name="Shell" %}}
   ```
   unzip spm8.zip
   ```
   {{% /tab %}}
   {{< /tabs >}}

2. 将SPM软件包地址加入到MATLAB的PATH中
   1. 方法一：在MATLAB 的界面上点击**Set Path**,选择**Add with Subfolders**来添加spm软件所在文件夹的路径
   
   2. 方法二：在MATLAB命令行(Command Window)输入如下命令
   {{< tabs >}}
   {{% tab name="Shell" %}}
   ```
   addpath /Volumes/Touch/Softwares/spm8 # 这里需要修改为你所在软件的路径
   savepath
   ```
    {{% /tab %}}
   {{< /tabs >}}

在设置完路径路径，在MATLAB命令行下输入```spm```命令，即可打开spm软件的GUI界面。


{{% notice note %}}
 如果我们想直接打开fMRI相关模块，我们只需要输入```spm fmri```命令即可打开fMRI的分析GUI界面。相关操作，以此类推。
{{% /notice %}}



其他系统的安装教程，详见
[官网安装教程链接](https://en.wikibooks.org/wiki/SPM#Installation)




