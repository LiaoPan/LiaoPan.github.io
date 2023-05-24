---
title: "Cat12系列教程"
date: 2023-04-24T11:22:08+08:00
draft: true
---

#### 安装方法简述
在[CAT12官网](https://neuro-jena.github.io/cat//index.html#DOWNLOAD)下载CAT12工具箱，解压压缩包到spm12文件夹下的toolbox子文件夹下。
在Matlab命令行下，输入`spm fmri`,会打开GUI页面，然后在toolbox下选择`cat12`,即可打开CAT12 GUI页面。
或者在Matlab命令行下，输入`cat12`,也会直接打开CAT12 GUI界面。


#### 安装时常见问题汇总
1. 在Mac环境运行时，碰到MEX文件权限不够或者安全问题(如下图所示)。
![permissions](/cat12/images/mac_permission.png?width=40pc)

[【解决方法】](https://en.wikibooks.org/wiki/SPM/Installation_on_64bit_Mac_OS_(Intel)#Troubleshooting)
在spm目录或者cat12目录下，打开终端，输入以下命令，即找到所有的mex文件，然后让其相关文件免于验证即可，即通过xattr -d 来去除com.apple.quarantine的隔离属性。
```shell
find . -name "*.mexmaci64" -exec xattr -d com.apple.quarantine {} \;
```


2. 在Mac环境运行时，点击`SPM fmri`的界面下的toolbox，无法打开`cat12`,报以下错误：
   ```
   ....
   Error in spm (line 954) evalin('base',xTB(i).prog);
   Error while evaluating UIControl Callback.
    ...
   ```
【解决方法】
- 第一种：彻底删除matlab已设置的spm相关软件路径，然后再重新导入加入了cat12的spm工具箱路径。
- 第二种：在matlab命令行输入cat12，绕过通过spm启动cat12的方式，也可直接打开cat12软件。

#### 参考资料
- [CAT12官网](https://neuro-jena.github.io/cat//index.html#DOWNLOAD)
- [CAT12官网教程](https://neuro-jena.github.io/cat12-help/)
- [CAT12教程参考](https://andysbrainbook.readthedocs.io/en/latest/CAT12/CAT12_01_DownloadInstall.html)

