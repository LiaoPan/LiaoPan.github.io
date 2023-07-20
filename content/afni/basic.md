---
title: "AFNI基础"
date: 2023-01-06T12:36:33+08:00
pre: "<b>2.1 </b>"
draft: true
---


### 基本概念

volume matching 体积匹配
- 是将一个体积（volume）叠加到另一个体积的过程，这个能方便我们寻找到两个体积之间的异同。主要目的是比较来自不同样本或在不同条件下收集的体积数据的差异。

![volume matching](/afni/images/volume_match.png)

resting state 静息态
- 不做什么任务的状态，通常有三种方式，闭上眼睛，睁开眼睛，盯着屏幕上的十字。

TR 重复时间
- 控制扫描连续的功能像volume之间的时间（秒），该值用于时间层校正和滤波。 
- 功能像每隔一个TR（一般为2秒）会记录一次全脑的BOLD信号，这样持续数分钟后会形成一系列具有先后顺序的功能像图像序列

![slicetimingcorrection](/fsl/images/02_SliceTimingCorrection_Demo.gif)
- https://mriquestions.com/tr-and-te.html


slice number 层数
- 扫描大脑是一层一层扫描的，通常是隔层扫描，因为信号可能会受到相邻层的影响，然后再把这些层重建成一个完整的3D大脑

volume 体积
- 一个重建好的3D大脑就是一个volume，一个TR时间内扫了slice number 层，然后就重建出了一个volume

time point 时间点
- 通常一个volume就是一个time point，静息态通常要扫好多time point或者说volume，因为通常要就算的是这段时间的时间序列变化。如果扫描了120个时间点，如果TR=2s, 则需要120X2/60=4 分钟时间

voxel 体素
- 这跟图像的单位pixel 像素相对应，这是体积的基本单位 volume+elment,一个体素就是一个单位，大脑被分成了许多小方块就是许多体素，通常功能像的分辨率为3mmx3mmx3mm也就是这个小方块就是一个单位，一个体素。

mask 掩码
- 加上mask后，只要mask里面的内容，mask之外的内容就会被忽略。

ROI 感兴趣区
- 感兴趣的脑区，比如研究者发现可能某个脑区很重要，我们就可以针对该脑区进行单独的分析。

Remove time points
- 由于机器的启动以及被试在适应所处环境时会造成噪音，故建议去除前N个时间点。（比如10个）
- Slice Timing
  - 时间层校正，即从扫描开始的第一个切片到最后一个切片之间存在一定的时间差，导致采集到的数据并不是同一个时间点。故需要在同一个时间尺度校正数据，保证Volume所有体素获取的时间在理论上都是一致的。
- Normalize
  - 空间标准化，将不同被试的空间数据对应到一个标准空间上，以解决不同被试之间的脑形态差异以及扫描时空位置不一致等问题。
- Smooth
  - 平滑处理，提高数据的信噪比
- Detrend
  - 去除线性漂移，由于机器工作时间较长而升温和被试长时间扫描而产生的疲劳会随着时间的积累存在一个线性趋势。
- Filter
  - 滤波，中低频的BOLD信号具有生理意义(0.01-0.08或者通常指小于0.1Hz)
- Nuisance Regress
  - 去除协变量，CSF（脑脊液）、WM（白质）、GM（灰质）的信号影响

AFNI示例数据下载：
1. https://afni.nimh.nih.gov/pub/dist/edu/data/


AFNI的学习资源：
1. [AFNI Start to Finish Tutorail](https://www.youtube.com/playlist?list=PLIQIswOrUH6-v5EWwFdMsTZttt4407KW9)(需要翻墙访问)


参考：
1. https://zhuanlan.zhihu.com/p/104795624
2. https://zhuanlan.zhihu.com/p/148999417
3. https://www.ebi.ac.uk/training/online/courses/structural-volume-data/what-is-volume-matching/
