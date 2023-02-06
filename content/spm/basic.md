---
title: "SPM教程 #1. 安装与入门"
date: 2023-01-11T10:43:19+08:00
draft: true
weight: 1
---


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


{{% notice warning %}}
注意，如果在mac系统上因为mex文件编译版本不兼容或者Mac未授权该应用，会导致SPM软件打开失败，故需要重新编译mex文件(若发现make时报错，不能编译成功，建议**直接使用spm12可以正常在Mac上使用**。)或者打开Mac的设置/安全选项授权即可。
{{% /notice %}}

```octave
>> spm
**Error using spm_check_installation>check_basic (line 121)
SPM uses a number of MEX files, which are compiled functions.
These need to be compiled for the various platforms on which SPM
is run**. At the FIL, where SPM is developed, the number of
computer platforms is limited.  It is therefore not possible to
release a version of SPM that will run on all computers. See
   /Volumes/Touch/Softwares/spm8/src/Makefile and
   http://en.wikibooks.org/wiki/SPM#Installation
for information about how to compile mex files for MACI64
in MATLAB 9.8.0.1323502 (R2020a).

Error in spm_check_installation (line 25)
        check_basic;

Error in spm (line 303)
spm_check_installation('basic');
```
[编译官网参考链接](https://en.wikibooks.org/wiki/SPM/Installation_on_64bit_Mac_OS_(Intel)#Installation_2)



其他系统的安装教程，详见
[官网安装教程链接](https://en.wikibooks.org/wiki/SPM#Installation)








[SPM WikiBook](https://en.wikibooks.org/wiki/SPM)
