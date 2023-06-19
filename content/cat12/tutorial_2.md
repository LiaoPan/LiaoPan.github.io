---
title: "CAT12 Tutorial #2:"
date: 2023-05-08T18:57:35+08:00
draft: true
weight: 2
---

- [介绍](#介绍)
  - [提取整个颅内容量，Extracting Total Intracranial Volume(TIV)](#提取整个颅内容量extracting-total-intracranial-volumetiv)
- [组水平分析(Group-Level Analysis)](#组水平分析group-level-analysis)
- [参考](#参考)



#### 介绍
做组水平分析与对功能像fMRI数据类似，但有一个重要区别：由于解剖学数据没有时间序列，所以不需要指定condition和onset时间，因此也不需要做第一级分析（first-level analysis）。因此，我们可以直接进行组水平分析，在这一过程中，我们可以比较各组，检查个体差异测量和皮质体积测量之间的相关性，或者进行纵向分析。在本教程中，我们将简单地比较两组，看看它们之间是否有任何明显的差异。

##### 提取整个颅内容量，Extracting Total Intracranial Volume(TIV)
在分割和预处理过程中，我们为每个受试者计算了一个称为颅内总体积（TIV）的测量值，代表颅内空间的总体积，单位为立方厘米($cm^3$)。由于在研究组间皮质体积的差异时，这可能是一个可信的混杂因素，我们将在分析中把它作为一个协变量。

![TIV](/cat12/images/2.get_tiv.png)
【操作步骤】
点击打开CAT12的GUI界面，然后点击`Get TIV`,双击右侧Batch Editor中的`XML files`,然后选择包含cat12分析结果的主目录，点击`rec`按钮递归遍历搜索相关的xml文件。（一个受试者对应一个xml文件）

在Batch Editor中设置`Save values`和`Add filenames`可以保存更多的分割结果数值，包括GM/WM/CSF/WMH，同时也可以保存每个数值对应的文件名称，方便一一对照。
如下图所示，展示了文件名称和TIV数值。
![TIV_CSV](/cat12/images/3.tiv_csv.png)


#### 组水平分析(Group-Level Analysis)


#### 参考
1. https://andysbrainbook.readthedocs.io/en/latest/CAT12/CAT12_04_Analysis.html
2. https://andysbrainbook.readthedocs.io/en/latest/CAT12/CAT12_04_Analysis.html#doing-the-group-level-analysis