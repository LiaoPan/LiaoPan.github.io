---
title: "FreeSurfer教程 #1. 安装"
date: 2023-01-08T22:09:43+08:00
draft: false
weight: 1
---
#### 安装方法简述
1. 根据自己系统下载对应的安装包（FreeSurfer 7.3.2 ~5G）
2. 通过网址( https://surfer.nmr.mgh.harvard.edu/registration.html ),获取*license.txt*,将其放置在FreeSufer的安装目录下。
3. 依次执行下述脚本，设置环境变量,注意freesurfer的版本7.3.2需要替换成自己安装的版本。
{{< tabs >}}
{{% tab name="shell" %}}
```Bash
$ export FREESURFER_HOME=/Applications/freesurfer/7.3.2
$ export SUBJECTS_DIR=$FREESURFER_HOME/subjects
$ source $FREESURFER_HOME/SetUpFreeSurfer.sh
-------- freesurfer-darwin-macOS-7.3.2-20200429-3a03ebd --------
Setting up environment for FreeSurfer/FS-FAST (and FSL)
WARNING: /Users/synpro/freesurfer/fsfast does not exist
FREESURFER_HOME   /Applications/freesurfer/7.3.2
FSFAST_HOME       /Users/synpro/freesurfer/fsfast
FSF_OUTPUT_FORMAT nii.gz
SUBJECTS_DIR      /Applications/freesurfer/7.3.2/subjects
MNI_DIR           /Users/synpro/freesurfer/mni

$ which freeview
/Applications/freesurfer/7.3.2/bin/freeview

```
{{% /tab %}}
{{< /tabs >}}

{{% notice tip %}}
注意上述方式时临时有效，即仅对当前打开的终端有效，关闭该终端后，上述设置都会失效。
{{% /notice %}}

4. 永久设置环境变量方法：
   将上述脚本写入~/.bash_profile文件中,操作命令如下：
   {{< tabs >}}
   {{% tab name="shell" %}}
   ```
   $ vi ~/.bash_profile  #打开该文本，写入内容(若不熟悉vim，请使用其他工具打开即可);

# FREESURFER
export FREESURFER_HOME=/Applications/freesurfer/7.3.2
export SUBJECTS_DIR=$FREESURFER_HOME/subjects
source $FREESURFER_HOME/SetUpFreeSurfer.sh

   $ source .bash_profile # 让设置立即生效
   
   ```
   {{% /tab %}}
   {{< /tabs >}}
   

   {{% notice tip %}}
   需要注意的是，上面的设置默认使用bash，如果你在Mac上使用的是zsh等shell工具，需要再做如下配置，来达到环境变量设置的永久生效。
   {{% /notice %}}

1.`$ vim ~/.zshrc`  
2.在开头添加以下内容：
```zsh
if [ -f ~/.bash_profile ]; then
   source ~/.bash_profile
fi
```
3.使用下面的命令使之立即生效
`$ source ~/.zshrc`

之后，每次打开新终端都可以正常使用啦。但每次都输出该信息太丑了，如图:
```
Last login: Sat Feb  4 17:38:25 on ttys012
-------- freesurfer-darwin-macOS-7.3.2-20220804-6354275 --------
Setting up environment for FreeSurfer/FS-FAST (and FSL)
FREESURFER_HOME   /Applications/freesurfer/7.3.2
FSFAST_HOME       /Applications/freesurfer/7.3.2/fsfast
FSF_OUTPUT_FORMAT nii.gz
SUBJECTS_DIR      /Applications/freesurfer/7.3.2/subjects
MNI_DIR           /Applications/freesurfer/7.3.2/mni
```

那怎么可以隐藏掉，方法如下：打开`~/.bash_profile`文件，删除或者注释掉
`source $FREESURFER_HOME/SetUpFreeSurfer.sh`,然后将上述终端输出消息和`echo $PATH`后的FreeSurfer相关信息，改为如下格式，并写入`.bash_profile`文件。
{{< tabs >}}
{{% tab name="~/.bash_profile" %}}
```bash

# FREESURFER
export FREESURFER_HOME=/Applications/freesurfer/7.3.2
export SUBJECTS_DIR=$FREESURFER_HOME/subjects
# source $FREESURFER_HOME/SetUpFreeSurfer.sh

export PATH=/Applications/freesurfer/7.3.2/bin:/Applications/freesurfer/7.3.2/fsfast/bin:/Applications/freesurfer/7.3.2/mni/bin:$PATH

export FSFAST_HOME=/Applications/freesurfer/7.3.2/fsfast
export FSF_OUTPUT_FORMAT=nii.gz
export SUBJECTS_DIR=/Applications/freesurfer/7.3.2/subjects
export MNI_DIR=/Applications/freesurfer/7.3.2/mni

```
{{% /tab %}}
{{< /tabs >}}



#### 参考
1. [官网安装指导](https://surfer.nmr.mgh.harvard.edu/fswiki/DownloadAndInstall)
2. https://surfer.nmr.mgh.harvard.edu/fswiki//FS7_mac
