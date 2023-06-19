---
title: "MRtrix3教程 #6 Connectome"
date: 2023-05-31T11:15:56+08:00
draft: false
weight: 6
pre: <b>6. </b>
---

- [简介](#简介)
- [创建脑连接组（Connectome）](#创建脑连接组connectome)
- [可视化脑连接组](#可视化脑连接组)
- [参考](#参考)


#### 简介
现在我们已经创建了一个流线图，我们就可以创建一个连接组，代表连接大脑不同部分的流线数量。为了做到这一点，我们必须首先将大脑分割成不同的区域，或节点。做到这一点的一个方法是是使用脑图谱（**atlas**），将大脑中的每个体素分配给特定的ROI。

你可以使用你想要的任何脑图谱，但在本教程中，我们将使用FreeSurfer附带的脑图谱。因此，我们的第一步将是通过`recon-all`来处理被试的解剖图像。

```
# 如果想更换数据生成的目录，可以设置SUBJECTS_DIR环境变量到自己想要的目录路径
$ export SUBJECTS_DIR=<your_custom_path>
# 耗时很长，需要慢慢等。
$ recon-all -i sub-CON02_ses-preop_T1w.nii.gz -s sub-CON02_recon -all
```

#### 创建脑连接组（Connectome）
当`recon-al`完成后，我们将需要把FreeSurfer解析的标签转换为MRtrix能理解的格式。`labelconvert`命令将使用FreeSurfer的注解（parcellation）和分割输出来创建一个新的.mif格式的注解（parcellation）文件：
```
$ labelconvert -help
MRtrix 3.0.4                      labelconvert                       Dec 14 2022

     labelconvert: part of the MRtrix3 package

SYNOPSIS

     Convert a connectome node image from one lookup table to another

USAGE

     labelconvert [ options ] path_in lut_in lut_out image_out

        path_in      the input image

        lut_in       the connectome lookup table corresponding to the input
                     image

        lut_out      the target connectome lookup table for the output image

        image_out    the output image
$ labelconvert sub-CON02_recon/mri/aparc+aseg.mgz $FREESURFER_HOME/FreeSurferColorLUT.txt /usr/local/mrtrix3/share/mrtrix3/labelconvert/fs_default.txt sub-CON02_parcels.mif
```

然后，我们需要创建一个全脑连接组，代表图谱中每个注解对之间的流线（本例中为84x84）。"symmetric" 选项将使下对角线与上对角线相同，而 "scale_invnodevol"选项将以节点大小的倒数来缩放连接组：

```
$ tck2connectome -h
MRtrix 3.0.4                     tck2connectome                      Dec 14 2022

     tck2connectome: part of the MRtrix3 package

SYNOPSIS

     Generate a connectome matrix from a streamlines file and a node
     parcellation image

USAGE

     tck2connectome [ options ] tracks_in nodes_in connectome_out

        tracks_in    the input track file

        nodes_in     the input node parcellation image

        connectome_out  the output .csv file containing edge weights
...
Options for outputting connectome matrices

  -symmetric
     Make matrices symmetric on output

  -zero_diagonal
     Set matrix diagonal to zero on output
     ...
  -out_assignments path
     output the node assignments of each streamline to a file; this can be used
     subsequently e.g. by the command connectome2tck
     ...


$ tck2connectome -symmetric -zero_diagonal -scale_invnodevol -tck_weights_in sift_1M.txt tracks_10M.tck sub-CON02_parcels.mif sub-CON02_parcels.csv -out_assignment assignments_sub-CON02_parcels.csv
```

#### 可视化脑连接组
创建parcels.csv文件后，我们可以在Matlab中将其视为矩阵。首先，需要导入它：
```matlab
connectome = importdata('sub-CON02_parcels.csv')
% 更高的结构连接对更亮
imagesc(connectome)
% 为了使这些关联更加明显，您可以更改颜色图的缩放比例
imagesc(connectome,[0,1])
```
![viewconnectome](/mrtrix/images/06_ViewingConnectome.webp)
<center>最明显的特征是将图分成两个不同的"盒子"，代表每个半球内的结构连接性增加。你还会观察到一条沿对角线描画的相对较亮的线，代表附近节点之间更高的结构连接。在对立的左下角和右上角更亮的方框代表了同源区域(homologous regions)之间结构连接的增加。</center>

![viewconnectome](/mrtrix/images/06_ViewingConnectome_Scaled.webp)
<center>调整缩放比例后的脑连接图</center>

#### 参考
- https://andysbrainbook.readthedocs.io/en/latest/MRtrix/MRtrix_Course/MRtrix_08_Connectome.html
