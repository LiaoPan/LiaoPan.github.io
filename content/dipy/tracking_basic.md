---
title: "纤维束追踪入门"
date: 2023-01-12T16:20:45+08:00
draft: false
weight: 2
pre: "<b>2. </b>"
---

### Basic Tracking(Local fiber tracking)
本教程知识点：
- 如何使用扩散像数据集来实现纤维束重建
  - 1)从扩散数据集获取方向(directions)的方法。
  - 2)识别追踪何时必须停止的方法。
  - 3)设置追踪种子(seed)。
- 如何保存trk文件
  - 使用`StatefulTractogram()`和`save_trk()`函数保存trk文件

{{% notice note %}}
局部纤维跟踪(Local fiber tracking)是一种通过从局部方向信息创建流线(streamlines)来模拟白质纤维的方法。其思想如下:如果一个区域/路径段的局部方向是已知的，可以沿着这些方向进行集成，以构建该结构的完整表示。局部纤维束追踪因其简单、鲁棒性好而被广泛应用于扩散MRI领域。

{{% /notice %}}

-----

**步骤一. 从扩散数据集获取方向(directions)的方法**  
基于 Constant Solid Angle ODF模型来拟合数据，该模型会评估每个voxel的取向分布函数(ODF,Orientation Distribution Function)。OODF 是作为方向函数的水扩散分布。ODF的峰值是图像中某一点上束段(tract segments)方向的良好估计。在这里，我们使用```peaks_from_model```来拟合数据，并计算白质所有体素中的纤维方向。


**步骤二. 识别追踪何时必须停止的方法**  
接下来，我们需要用某种方法将纤维追踪限制在具有良好方向性信息的区域。我们已经创建了白质掩码(white mask)，但我们可以更进一步，通过对广义分数各向异性（GFA,generalized fractional anisotropy）进行阈值处理，将纤维追踪限制在那些ODF显示出明显限制性扩散的区域。

**步骤三. 设置追踪种子(seed)**  
在我们开始追踪之前，我们需要指定在哪里 "seed"（开始）纤维追踪。一般来说，选择的种子将取决于人们感兴趣的建模路径。在这个例子中，我们将在胼胝体的矢状切面上使用一个每个体素的2x2x2的网格种子。从这个区域进行追踪将给我们一个胼胝体束的模型。这个切片在标签的图像中具有标签值2。

-----

#### 使用EuDX算法构建一个确定性的纤维束流线(streamlines)。
{{% notice tip %}}
所谓的确定性(deterministic)表示如果你重复纤维跟踪（保持所有输入相同），你将得到完全相同的一组纤维束流线。
{{% /notice %}}
{{<jupyter dipy dipy_quickstart_3 7224 >}}



{{% notice tip %}}
使用```csdeconv.mask_for_response_ssst()```函数来获取每个体素的各向异性配置（very anisotropic configurations）的信息，该函数会返回所选体素的mask。
通过这个mask，我们就可以通过```csdeconv.response_from_mask_ssst()```函数计算响应函数。
{{% /notice %}}

{{% notice tip %}}
**```load_nifti_data()```和```load_nifti()```函数的区别**:  
```load_nifti_data()```只加载nifti内的data array。  
```load_nifti()```除了加载data array,还要把其他信息也加载进来（data, img.affine, img, vox_size, nib.aff2axcodes(img.affine)）。

{{% /notice %}}


{{% notice tip %}}
**方向场（Direction Field）图如何看？**   
x - Red - 方向场图中为红色标识  
y - Green - 方向场图中为绿色标识  
z - Blue- 方向场图中为蓝色标识  
{{% /notice %}}