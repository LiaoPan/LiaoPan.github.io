---
title: "FSL系列教程 #1.FSL安装"
date: 2023-02-01T19:00:48+08:00
weight: 1
pre: "<b>1. </b>"
draft: true
---

- [安装方法简述](#安装方法简述)
- [安装校验](#安装校验)
- [在MATLAB上使用FSL](#在matlab上使用fsl)
- [Flanker公开数据下载](#flanker公开数据下载)
  - [通过fsleyes查看结构像数据](#通过fsleyes查看结构像数据)
  - [通过fsleyes查看功能像数据](#通过fsleyes查看功能像数据)
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


#### Flanker公开数据下载
访问Openeuro官网，下载[Flanker task](https://openneuro.org/datasets/ds000102/versions/00001)的fMRI数据集。
下载的`ds000102`数据集使用的是Flanker任务，其目的是为了挖掘一种被称为认知控制的心理过程。在本教程中，我们将把认知控制定义为为了正确完成任务而忽略不相关的刺激的能力。

在Flanker任务中，箭头要么指向左边，要么指向右边，受试者被要求按下两个按钮中的一个，指示中间箭头的方向。如果它指向左边，受试者就按下 "左 "字按钮；如果它指向右边，受试者就按下 "右 "字按钮。中间的箭头两侧有其他的箭头，这些箭头要么与中间的箭头指向同一方向，要么与中间的箭头指向相反的方向。
![Flanker Example](/fsl/images/01_Flanker_Example.png)
<center>Flanker任务的两个条件的例子。在不协调(Incongruent condition)条件下，中央的箭头（被试所关注的）与侧面的箭头指向相反的方向；在协调条件(Congruent condition)下，中央的箭头与侧面的箭头指向相同的方向。在这个例子中，不协调条件下的正确反应是按 "左 "键，而协调条件下的正确反应是按 "右 "键。</center>


我们可以想象，如果中央的箭头与侧面的箭头指向同一方向，任务就比较容易，如果指向相反的方向，就比较困难。我们将前者称为 "一致 "条件，后者称为 "不一致 "条件。受试者在 "不一致 "条件下通常反应较慢，准确性较低，而在 "一致 "条件下则反应较快，准确性较高。由于反应时间的差异是稳健和可靠的，因此在我们的fMRI数据中，我们应该看到BOLD信号也有明显的差异。
![Flanker Design](/fsl/images/01_Flanker_Design.png)
<center>本研究中Flanker任务的图示，改编自Kelly等人（2008）。受试者会看到一个固定的十字架（fixation cross），以便将注意力集中在屏幕中心，然后呈现一个 "一致 "或 "不一致 "的Flanker试验，持续2000ms。在试验过程中，被试按下左边或右边的按钮。(注意，抖动间隔通常以秒为单位递增；在这种情况下，给定试验的抖动将是以下的随机选择之一： 8,000ms、9,000ms、10,000ms、11,000ms、12,000ms、13,000ms和14,000ms）。另一个固定交叉点被呈现，开始下一个试验。</center>

我们的目标是估计每个条件的BOLD信号的大小，然后对比（即取其差异）两个条件，看它们是否有明显的差异。

{{% notice note %}}
这个任务的描述带来了一个关于设计fMRI研究的良好做法的重要观点： 如果你能设计一个能**产生强烈和可靠效应的行为任务**，**你将增加在成像数据中发现效应的几率。**fMRI数据是出了名的嘈杂--如果你在研究中没有看到行为效应，你很可能也不会在成像数据中发现效应。
{{% /notice %}}

{{< tabs >}}
{{% tab name="Flanker Task Directory Structure" %}}
```
...
├── participants.tsv
├── sub-01
│   ├── anat
│   │   └── sub-01_T1w.nii.gz
│   └── func
│       ├── sub-01_task-flanker_run-1_bold.nii.gz
│       ├── sub-01_task-flanker_run-1_events.tsv
│       ├── sub-01_task-flanker_run-2_bold.nii.gz
│       └── sub-01_task-flanker_run-2_events.tsv
├── sub-02
│   ├── anat
│   │   └── sub-02_T1w.nii.gz
│   └── func
│       ├── sub-02_task-flanker_run-1_bold.nii.gz
│       ├── sub-02_task-flanker_run-1_events.tsv
│       ├── sub-02_task-flanker_run-2_bold.nii.gz
│       └── sub-02_task-flanker_run-2_events.tsv
...
```
{{% /tab %}}
{{< /tabs >}}


##### 通过fsleyes查看结构像数据
```
 fsleyes anat/sub-01_T1w.nii.gz
```
![anat_first_look](/fsl/images/01_anat_firstLook.png)
<center>fsleyes中显示的解剖学图像。灰质和白质之间的对比度似乎很低，但这是因为颈部的血管（用橙色箭头表示）比大脑的其他部分要亮很多。</center>

![anat_change_contrast](/fsl/images/01_anat_changeContrast.png)
<center>这可以通过改变对比度框中的数字来解决。在这里，最大值被降低到800，将最亮的信号限定在这个值上。这使得它更容易看到组织之间的对比。</center>

![anat_change_contrast](/fsl/images/01_anat_contrast_after.png)

通过点击和拖动鼠标来检查图像。你可以通过点击相应的窗口来切换查看窗格。请注意，当你移动鼠标时，其他窗口会实时更新。这是因为MRI数据是作为三维图像收集的，沿着其中一个维度移动也会改变其他窗口。

{{% notice note %}}
你可能已经注意到，这个被试缺少了他的脸。这是因为来自OpenNeuro.org的数据已经被隐藏了身份：不仅姓名和扫描日期等信息已从头文件中删除，而且人脸也已被抹去。这样做是为了确保受试者的匿名性。
{{% /notice %}}

当我们继续检查图像时，这里有两件事需要注意：
1. **Gibbs Ringing Artigacts**:看起来像池塘里的波纹的线条。这些被称为吉布斯伪影，它们可能表明来自扫描仪的MR信号的重建出现了错误。这些波纹也可能是由于受试者在扫描过程中移动太多造成的。在任何一种情况下，如果波纹足够大，它们可能会导致预处理步骤，如脑提取或归一化失败。
2. **Abnormal intensity difference**:灰质或白质内的异常强度差异。这可能表明有病变，如动脉瘤（aneurysms）或海绵瘤(cavernomas)，应立即向放射科医生报告；确保你熟悉实验室报告伪影的协议。关于你在MRI图像中可能看到的病变，[请点击这里](http://www.mrishark.com/brain2.html)。


##### 通过fsleyes查看功能像数据
当你看完解剖图像后，从屏幕上方的菜单中点击`Overlay -> Remove All`。然后，点击`File -> Add from File`，导航到sub-01的func目录，并选择以`run-1_bold.nii.gz`结尾的图像。这张图片看起来也像一个大脑，但它不像解剖学图片那样清晰。这是因为其分辨率较低。典型的研究是收集高分辨率的T1加权（即解剖学）图像和低分辨率的功能图像，部分原因是我们收集功能图像的速度更快。

![functional_firstlook](/fsl/images/01_functional_firstLook.png)

**质量检测**
- 对功能图像的许多质量检查与解剖图像相同：注意灰质或白质中是否有极亮或极暗的斑点，以及图像的扭曲，如不正常的伸展或扭曲。常见的一个地方是在大脑的眶额部(orbitofrontal)，就在眼球上方，会看到一点点的失真。有一些方法可以减少这种失真，但现在我们将忽略它。

- 另一个质量检查是确保没有过度的运动。功能性图像通常是以时间序列的形式收集的；也就是说，多个volume被串联成一个数据集。你可以通过点击`fsleyes`中的电影卷轴图标，像翻书一样快速翻阅所有的volumes（当然，也可以设置图像正下方的`Location-->Voxel location`中的volume数值，手动翻阅不同的volume）。请注意任何观察窗格中的任何突然的、生硬的动作。在预处理过程中，我们将对运动的程度进行量化，以决定是否保留或丢弃该被试的数据。



#### 参考
1. [官网安装教程](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FslInstallation)
2. [Using FSL from MATLAB, MAC OS](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/FslInstallation/MacOsX)
3. [fMRI_DataDownload](https://andysbrainbook.readthedocs.io/en/latest/fMRI_Short_Course/fMRI_01_DataDownload.html)