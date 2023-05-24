---
title: "命令行工具总览"
date: 2023-05-23T15:38:05+08:00
draft: true
weight: 10
pre: <b>#. </b>
---


```mermaid
graph LR
    a(MRtrix CLI)-->1(mrconvert)
    a-->2(mrinfo)
    a-->3(mrview)
    a-->4(dwidenoise)
    a-->5(mrcalc)
    a-->6(mridegibbs)
    a-->7(mrmath)
    a-->8(mrcat)
    a-->9(dwiextract)
    a-->10(dwibiascorrect)
    a-->11(dwi2mask)
  
```

##### 1. 如何从4D的扩散像中抽取任意的3D数据？
```
# 使用mrconvert命令
# -force 表示可以强制覆写文件
# -coord 3 0 表示在第3个维度（coord从0开始计数，即3表示时间维度），提取volume索引为0的数据;（-coord 3 <#> selects volumes (the fourth dimension) from the series; -axes 0,1,2 includes only the three spatial axes in the output image.）
# -axes 0,1,2 表示在输出图像中仅包含三个空间轴。
$ mrconvert -force -coord 3 0 -axes 0,1,2 <input data> <output data>
$ mrconvert -force -coord 3 0  -axes 0,1,2 sub-02_den_preproc_unbiased.mif sub02_1.nii
```

https://mrtrix.readthedocs.io/en/dev/reference/commands_list.html