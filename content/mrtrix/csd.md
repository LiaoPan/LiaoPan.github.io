---
title: "MRtrix3教程 #3 约束球面反卷积(CSD) "
date: 2023-05-25T09:55:01+08:00
draft: true
weight: 3
pre: <b>3. </b>
---

- [简介](#简介)
- [dwi2response命令使用](#dwi2response命令使用)
- [纤维取向密度(Fiber Orientation Density, FOD)](#纤维取向密度fiber-orientation-density-fod)
- [归一化(Normalization)](#归一化normalization)
- [参考文献](#参考文献)


**约束球面反卷积(Constrained Spherical Deconvolution,CSD)**

#### 简介
为了确定每个体素内的扩散方向，我们将根据受试者数据创建一个基础函数（basis function）。通过从有代表性的灰质、白质和脑脊液体素中提取扩散信号，我们将建立一个模型来估计信号在不同方向和应用不同b值时应该是什么样子。这个概念类似于使用血液动力学响应函数（Hemodynamic response function, HRF）作为fMRI数据的基础函数：我们有一个典型的形状，即我们认为fMRI信号在应对单一事件时应该是什么样子，然后我们对其进行调制以适应观察到的数据。

响应函数类似于我们在fMRI研究中使用的典型的HRF。然而，在这种情况下，我们要估计每个组织类型的响应函数。如果你碰巧收集了具有多个b值的扩散数据，那么MRtrix的这种方法被称为多壳多组织（**multi-shell multi-tissue,MSMT**）。


#### [dwi2response命令使用](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwi2response.html)
与大多数fMRI研究使用事先创建的基础函数不同，**MRtrix将从扩散数据中推导出一个基础函数**；使用单个受试者的数据对该受试者来说更加精确和具体。`dwi2response`命令有几种不同的算法供你选择，但在本教程中我们将使用"dhollander"算法：
```
$ dwi2response -h
MRtrix 3.0.4                      dwi2response

     dwi2response: part of the MRtrix3 package

SYNOPSIS

     Estimate response function(s) for spherical deconvolution

USAGE

     dwi2response algorithm [ options ] ...

        algorithm    Select the algorithm to be used to complete the script operation;
                     additional details and options become available once an
                     algorithm is nominated. Options are: dhollander, fa,
                     manual, msmt_5tt, tax, tournier

DESCRIPTION

     dwi2response offers different algorithms for performing various types of
     response function estimation. The name of the algorithm must appear as the
     first argument on the command-line after 'dwi2response'. The subsequent
     arguments and options depend on the particular algorithm being invoked.
...

# 注意：如果想查看子命令的帮助文档，可以使用如下使用方法。
$ dwi2response dhollander -h
MRtrix 3.0.4                 dwi2response dhollander

     dwi2response dhollander: part of the MRtrix3 package

SYNOPSIS

     Unsupervised estimation of WM, GM and CSF response functions that does not
     require a T1 image (or segmentation thereof)

USAGE

     dwi2response dhollander [ options ] input out_sfwm out_gm out_csf

        input        Input DWI dataset

        out_sfwm     Output single-fibre WM response function text file

        out_gm       Output GM response function text file

        out_csf      Output CSF response function text file
...

# 运行下述命令，若出现error信息，则可以使用-debug和-nocleanup来测试和保留分析环境。
$ dwi2response dhollander sub-02_den_preproc_unbiased.mif wm.txt gm.txt csf.txt -voxels voxels.mif


# 命令运行错误的调试：
# 比如，我运行时出现过以下报错信息;(出现以下情况，一般都是输入数据经过`dwifslpreproc`命令处理后存在问题，需要返回重新检查。)
dwi2response: [ERROR] Error trying to calculate statistics from image 'safe_mask.mif'

# 通过debug模式，我发现为mrstats运行时出错,发现safe_mask.mif无法统计，经查看该数据为空，即存在异常。
dwi2response: Command: '/usr/local/mrtrix3/bin/mrstats safe_mask.mif -output mean -output median -output std -output std_rv -output min -output max -output count -mask safe_mask.mif' (piping data to local storage)

```
让我们来解读一下这个命令的作用。首先，它使用一种算法来分解纤维取向分布（FODs）--换句话说，它试图将扩散信号分解成一组较小的单个纤维方向。你有几种算法可供选择，但最常见的是**Tournier**和**Dhollander**。**Tournier**算法用于单壳数据和单一组织类型（如白质）。**dhollander**算法可用于单壳或多壳数据，也可用于多种组织类型。估算每种组织类型的FOD将有助于我们以后做解剖学上的约束性纤维束成像（anatomically constrained tractography）。

下一个参数指定你的输入数据，以及不同组织类型的结果反应函数（顺序很重要）。你可以随心所欲地对输出文件命名，但把它们标记为 "白质"、"灰质 "和 "脑脊液 "的某种变化是最有意义的（这里标记为 "wm.txt"、"gm.txt "和 "csf.txt"）。最后一个选项，"-voxels"，指定了一个输出数据集，显示图像中的哪些体素被用来构建每种组织类型的基函数。这个数据集可以通过输入以下内容来查看：
```
$ mrview sub-02_den_preproc_unbiased.mif -overlay.load voxels.mif
```
![voxels_view](/mrtrix/images/03_voxels.webp)
<center>`dwi2response`命令的输出voxels.mif，显示哪些体素被用来构建每个组织类型的基础函数。红色：脑脊液体素；绿色：灰质体素；蓝色：白质体素。确保这些颜色位于它们应该在的地方；例如，红色体素应该在脑室内。</center>

然后，您可以通过键入以下内容来检查每种组织类型的响应函数：
```
# 查看球面谐波表面图
$ shview -h
MRtrix 3.0.4                         shview                          Dec 14 2022

     shview: part of the MRtrix3 package

SYNOPSIS

     View spherical harmonics surface plots

USAGE

     shview [ options ] [ coefs ]

        coefs        a text file containing the even order spherical harmonics
                     coefficients to display.

$ shview wm.txt
$ shview gm.txt
$ shview csf.txt

```

逐一查看这些文件。弹出的第一张图片看起来像一个球体；这代表了当b值为0时，该组织类型内的扩散情况，换句话说，就是没有扩散梯度时的情况。通过按键盘左右方向键，你可以看到应用不同b值时基础函数的样子。

下图显示了每个组织类型和b值组合的基函数是如何变化的。请注意，当应用较高的b值时，每种组织类型的球体的整体幅度（或大小）如何变小；尽管较高的b值对扩散的变化更敏感，但整体信号更小，更容易受噪声影响。在白质内，当应用扩散梯度时，球体趋于平坦成**饼状**，反映了这些体素中沿白质束的优先扩散方向。另一方面，对于灰质和脑脊液，基函数在所有的b值中都**保持球形**。

![shview_png](/mrtrix/images/03_bvals_tissues.webp)


#### 纤维取向密度(Fiber Orientation Density, FOD)
现在我们将使用上面生成的基础函数来创建纤维取向密度，或称FODs。这些是对三个正交方向中每个方向的扩散量的估计。正如介绍性章节中所描述的，这些类似于传统扩散研究中使用的张量。然而，MRtrix允许对单个体素内的多个交叉纤维进行估计，并能将扩散信号解析为多个方向。

为此，我们将使用命令`dwi2fod`将基函数应用于扩散数据。选项"-mask"指定了我们要使用的体素；这只是为了将我们的分析限制在大脑体素上，以减少计算时间。每个基函数后面指定的".mif "文件将输出该组织类型的FOD图像：

```
$ dwi2fod -h
MRtrix 3.0.4                         dwi2fod                         Dec 14 2022

     dwi2fod: part of the MRtrix3 package

SYNOPSIS

     Estimate fibre orientation distributions from diffusion data using
     spherical deconvolution

USAGE

     dwi2fod [ options ] algorithm dwi response odf [ response odf ... ]

        algorithm    the algorithm to use for FOD estimation. (options are:
                     csd,msmt_csd)

        dwi          the input diffusion-weighted image

        response odf  pairs of input tissue response and output ODF images


DESCRIPTION

     The spherical harmonic coefficients are stored according the conventions
     ...
$ dwi2fod msmt_csd sub-02_den_preproc_unbiased.mif -mask mask_bet2.mif wm.txt wmfod.mif gm.txt gmfod.mif csf.txt csffod.mif
dwi2fod: [100%] preloading data for "sub-02_den_preproc_unbiased.mif"
dwi2fod: [100%] performing MSMT CSD (4 shells, 3 tissues)
```

为了查看这些FOD，我们将把它们合并成一张图像。命令`mrconvert`将从wmfod.mif文件中提取第一张图像，即b值为0的图像。然后，该命令的输出被用作mrcat命令的输入，该命令将所有三种组织类型的FOD图像合并为一张图像，我们将称之为 "vf.mif"：
```
$ mrconvert -coord 3 0 wmfod.mif - | mrcat csffod.mif gmfod.mif - vf.mif
mrconvert: [100%] copying from "wmfod.mif" to "/var/folde...0gn/T/mrtrix-tmp-zRjRvl.mif"
mrcat: [100%] concatenating "csffod.mif"
mrcat: [100%] concatenating "gmfod.mif"
mrcat: [100%] concatenating "/var/folders/39/93zn2fy95cd9b_b38zvmvgjc0000gn/T/mrtrix-tmp-zRjRvl.mif"
```

然后可以将白质FODs叠加在vif.mif图像上，这样我们就可以观察到白质FODs是否确实落在白质内，以及它们是否沿着我们预期的方向：
```
$ mrview vf.mif -odf.load_sh wmfod.mif
```
![FOD](/mrtrix/images/03_FODs.webp)
<center>白质FOD覆盖在针对每种组织类型进行颜色编码的图像上。绿色代表灰质，脑脊液用红色表示，白质用蓝色表示。</center>


![FOD_CC](/mrtrix/images/03_FODs_CC.webp)
我们可以通过按住命令并滚动鼠标滚轮来放大图像。关注一个区域，如胼胝体；如果FODs被正确估计，胼胝体的主要颜色应该是红色，因为红色表示主要方向是由左至右。

记住，绿色代表从后往前，蓝色代表从下往上的方向。通过使用所有三个正交视图，看看你是否能找到诸如上纵束和放射冠之类的神经束。这些是否与你所期望的颜色相吻合？

#### 归一化(Normalization)
后续，我们将学习如何用为每个受试者生成的数据做一个组分析。为了使跨学科的比较有效，我们将需要对FODs进行标准化。这可以确保我们看到的任何差异不是由于图像中的强度差异造成的，类似于我们在比较不同受试者的体积差异时对大脑的大小进行校正。

为了归一化数据，我们将使用`mtnormalise`命令。这需要每个组织类型的输入和输出，以及一个掩码来限制对大脑体素的分析。
```
$ mtnormalise -h
MRtrix 3.0.4                       mtnormalise                       Dec 14 2022

     mtnormalise: part of the MRtrix3 package

SYNOPSIS

     Multi-tissue informed log-domain intensity normalisation

USAGE

     mtnormalise [ options ] input output [ input output ... ]

        input output  list of all input and output tissue compartment files
                     (see example usage).


DESCRIPTION

     This command takes as input any number of tissue components (e.g. from
     multi-tissue CSD) and outputs corresponding normalised tissue components
     corrected for the effects of (residual) intensity inhomogeneities.
     Intensity normalisation is performed by optimising the voxel-wise sum of
     all tissue compartments towards a constant value, under constraints of
     spatial smoothness (polynomial basis of a given order). Different to the
...

$ mtnormalise wmfod.mif wmfod_norm.mif gmfod.mif gmfod_norm.mif csffod.mif csffod_norm.mif -mask mask.mif
```
在这个命令中，"-act"选项指定我们将使用解剖学上的分割图像来约束我们的分析，使其限于白质。"-backtrack"表示如果当前的流线在一个奇怪的地方（如脑脊液）时终止并返回并再次运行同一流线；"-maxlength"设置允许的最大流线长度（体素）；"-cutoff"指定终止流线的FOD振幅（例如，0.06的值不允许流线沿着低于这个数字的FOD走）。"-seed_gmwmi"把使用5tt2gmwmi命令生成的灰质/白质边界作为输入。

"-nthreads"指定你希望使用的处理核心的数量，以加快分析速度。最后，"-select"表示要生成多少条总的流线。注意，如果你愿意，可以使用速记法；比如说，10000000，你可以改写成10000k（意思是 "一万个"，等于 "一千万"）。最后两个参数指定了输入（`wmfod_norm.mif`）和输出的标签（`tracks_10M.tck`）。

如果您想可视化输出，我建议使用`tckedit`提取输出的一个子集：
```
$ tckedit tracks_10M.tck -number 200k smallerTracks_200k.tck
```

#### 参考文献
- https://andysbrainbook.readthedocs.io/en/latest/MRtrix/MRtrix_Course/MRtrix_05_BasisFunctions.html

