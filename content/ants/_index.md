---
title: "ANTs系列教程"
date: 2023-02-01T17:45:06+08:00
draft: false
weight: 11
pre: "<b>11. </b>"
---

- [下载ANTs工具箱](#下载ants工具箱)
- [安装ANTs工具箱](#安装ants工具箱)
  - [设置环境变量PATH、ANTSPATH](#设置环境变量pathantspath)


#### 下载ANTs工具箱
点击[下载链接](https://github.com/ANTsX/ANTs/releases)，选择想要的版本进行下载。
Mac环境下，等待下载完成后，解压到特定文件下`/opt/ants-<version>/[bin,lib]`，

#### [安装ANTs工具箱](https://github.com/ANTsX/ANTs/wiki/Installing-ANTs-release-binaries)

##### 设置环境变量PATH、ANTSPATH
在Mac环境下，打开`~/.bash_profile`文件,写入下述内容：
```
# ANTs
export ANTSPATH=/opt/ants-2.4.4/bin/
export PATH=${ANTSPATH}:$PATH

```
然后，```source ~/.bash_profile```来激活环境变量。
最后，我们来检查安装是否成功。
```
# 在终端输入以下命令，会输出antsRegistration的对应路径
$ which antsRegistration
/opt/ants-2.4.4/bin//antsRegistration

# 在终端输入以下命令，会输出antsRegistrationSyN.sh的使用方法
$ antsRegistrationSyN.sh

Usage:

antsRegistrationSyN.sh -d ImageDimension -f FixedImage -m MovingImage -o OutputPrefix

Example Case:

antsRegistrationSyN.sh -d 3 -f fixedImage.nii.gz -m movingImage.nii.gz -o output

Compulsory arguments:

     -d:  ImageDimension: 2 or 3 (for 2 or 3 dimensional registration of single volume)

     -f:  Fixed image(s) or source image(s) or reference image(s)
...
```

若在mac环境下，使用ANTs命令时，出现以下问题，我们就需要给所有相关二进制文件授权。
![verified](/ants/images/00_developer_verified.png?width=20pc)

授权方式如下：
```
$ xattr -r -d com.apple.quarantine <指定授权文件夹>
$ xattr -r -d com.apple.quarantine /opt/ants-2.4.4/
```



https://github.com/ANTsX/ANTsPy
https://github.com/ANTsX/ANTs


https://github.com/ANTsX/ANTs/wiki/Installing-ANTs-release-binaries
https://andysbrainbook.readthedocs.io/en/latest/ANTs/ANTs_Overview.html?highlight=ants