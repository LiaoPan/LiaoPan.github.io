---
title: "MRtrix3教程 #2 预处理"
date: 2023-05-23T11:05:36+08:00
weight: 2
draft: false
pre: <b>2. </b>
---

- [简介](#简介)
- [dwi\_denoise命令使用](#dwi_denoise命令使用)
  - [mrcalc命令使用](#mrcalc命令使用)
- [mri\_degibbs命令使用](#mri_degibbs命令使用)
- [提取反相编码图像(dwiextract、mrcat、mrmath)](#提取反相编码图像dwiextractmrcatmrmath)
- [将它们放在一起：使用`dwipreproc`进行预处理(Eddy处理，去除涡流)](#将它们放在一起使用dwipreproc进行预处理eddy处理去除涡流)
- [检查损坏的切片](#检查损坏的切片)
- [生成Mask(dwibiascorrect、dwi2mask)](#生成maskdwibiascorrectdwi2mask)
- [参考资料](#参考资料)


#### 简介
就像其他神经影像学数据一样，扩散数据在分析前应进行预处理。预处理可以去除图像中的噪声源，如运动伪影和其他失真。扩散数据尤其容易受到相位编码方向的影响而产生扭曲的伪影： 一般来说，主要的编码方向--如前向后(Anterior to Posterior, AP)--会使大脑的前部看起来更 "压扁"，就像从前部方向吹来的强风。后至前(Posterior to Anterior,PA)，相位编码方向的情况则相反。有时这些扭曲是非常微妙的，但其他时候它们是明显的。
![AP_PA](/mrtrix/images/02_AP_PA_Comparisons.png)

以下是使用 MRtrix 完成的常见预处理步骤:
#### dwi_denoise命令使用
我们要做的第一个预处理步骤是通过使用MRtrix的`dwidenoise`命令对数据进行去噪。这需要一个输入和一个输出参数，你也可以选择用`-noise`选项来输出噪声图。比如说：
```
# 查看使用说明
$ dwidenoise -h
MRtrix 3.0.4                       dwidenoise                        Dec 14 2022

     dwidenoise: part of the MRtrix3 package

SYNOPSIS

     dMRI noise level estimation and denoising using Marchenko-Pastur PCA

USAGE

     dwidenoise [ options ] dwi out

        dwi          the input diffusion-weighted image.

        out          the output denoised DWI image.
......
# 运行此命令需要几分钟时间。
$ dwidenoise sub-CON02_ses-preop_acq-AP_dwi.mif sub-CON02_ses-preop_acq-AP_dwi_denoise.mif -noise noise.mif
dwidenoise: [100%] preloading data for "sub-CON02_ses-preop_acq-AP_dwi.mif"
dwidenoise: [100%] running MP-PCA denoising
```

##### mrcalc命令使用
一个质量检查是看残差(residuals)是否加载到解剖结构的任何部分。如果是这样，这可能表明大脑区域受到某种伪影或扭曲的影响不成比例。为了计算这个残差，我们将使用另一个MRtrix命令，叫做`mrcalc`

```
$ mrcalc -h
MRtrix 3.0.4                         mrcalc                          Dec 14 2022

     mrcalc: part of the MRtrix3 package

SYNOPSIS

     Apply generic voxel-wise mathematical operations to images

USAGE

     mrcalc [ options ] operand [ operand ... ]

        operand      an input image, intensity value, or the special keywords
                     'rand' (random number between 0 and 1) or 'randn' (random
                     number from unit std.dev. normal distribution) or the
                     mathematical constants 'e' and 'pi'.
...
EXAMPLE USAGES

     Double the value stored in every voxel:
       $ mrcalc a.mif 2 -mult r.mif
     This performs the operation: r = 2*a  for every voxel a,r in images a.mif
     and r.mif respectively.

     A more complex example:
       $ mrcalc a.mif -neg b.mif -div -exp 9.3 -mult r.mif
     This performs the operation: r = 9.3*exp(-a/b)

     Another complex example:
       $ mrcalc a.mif b.mif -add c.mif d.mif -mult 4.2 -add -div r.mif
     This performs: r = (a+b)/(c*d+4.2).

...

basic operations

  -abs  (multiple uses permitted)
     |%1| : return absolute value (magnitude) of real or complex number

  -neg  (multiple uses permitted)
     -%1 : negative value

  -add  (multiple uses permitted)
     (%1 + %2) : add values

  -subtract  (multiple uses permitted)
     (%1 - %2) : subtract nth operand from (n-1)th
...
# 使用原始扩散数据减去经过dwidenoise降噪后的数据，会得到相关的残差数据；
$ mrcalc sub-CON02_ses-preop_acq-AP_dwi.mif sub-CON02_ses-preop_acq-AP_dwi_denoise.mif -subtract residual.mif
mrcalc: [100%] computing: (sub-CON02_ses-preop_acq-AP_dwi.mif - sub-CON02_ses-preop_acq-AP_dwi_denoise.mif)

```
然后使用mriview软件来查看这个残差图(residual map)。
```
mrview residual.mif
```
![residuals](/mrtrix/images/02_residuals.png)
常见的是看到大脑的灰色轮廓，如上图所示。然而，灰质和白质内的一切都应该是相对均匀和模糊的；如果你看到任何清晰的解剖标志，如个别的脑回或脑沟，这可能表明大脑的这些部分已经被噪声所破坏(如下图所示)。
![residuals_2](/mrtrix/images/02_dwinoise_extend.png)

如果发生这种情况，你可以将去噪过滤器的范围从默认的5增加到一个更大的数字，如7；
```shell
$ dwidenoise your_data.mif your_data_denoised_7extent.mif -extent 7 -noise noise.mif
```


#### mri_degibbs命令使用
一个可选的预处理步骤是运行`mri_degibbs`，它可以从数据中去除[**吉布斯伪影,Gibbs' ringing artifacts**](https://mriquestions.com/gibbs-artifact.html)。这些伪影看起来像池塘里的涟漪，在b值为0的图像中最为明显,通常表现为紧邻高对比度界面的多条精细平行线。
![gibbs](/mrtrix/images/02_gibbs.gif)


首先用`mrview`查看你的扩散数据，确定是否有任何吉布斯伪影；如果有，那么你可以通过指定输入文件和输出文件来运行mrdegibbs，例如:
```
$ mrdegibbs -h
MRtrix 3.0.4                        mrdegibbs                        Dec 14 2022

     mrdegibbs: part of the MRtrix3 package

SYNOPSIS

     Remove Gibbs Ringing Artifacts

USAGE

     mrdegibbs [ options ] in out

        in           the input image.

        out          the output image.


DESCRIPTION

     This application attempts to remove Gibbs ringing artefacts from MRI
     images using the method of local subvoxel-shifts proposed by Kellner et
     al. (see reference below for details).
    ....

$ mrdegibbs sub-CON02_ses-preop_acq-AP_dwi_denoise.mif sub-CON02_ses-preop_acq-AP_dwi_den_unr.mif
mrdegibbs: [100%] performing Gibbs ringing removal
```
像往常一样，用`mrview`检查前后的数据，以确定预处理步骤是否使数据变得更好、更差，或者没有影响。
如果你在你的数据中没有看到任何吉布斯伪影，那么我建议省略这一步骤。

#### 提取反相编码图像(dwiextract、mrcat、mrmath)
大多数扩散数据是由两个独立的成像文件组成的：一个是以主相位编码方向（primary phase-encoding direction）获取的，一个是以反向相位编码方向获取的。主相位编码方向用于获取不同b值的大部分扩散图像。另一方面，反向相位编码文件用于消除主相位编码文件中存在的任何失真现象。

为了理解这一点，想象一下，你正在用吹风机吹头发。假设你把吹风机对准你的后脑勺，它把你的头发向前吹，吹到你的脸前；让我们把这称为后部到前部（PA）的相位编码方向。现在你的头发看起来很乱，你想消除空气从后脑勺吹到前脑勺的影响。所以你把吹风机对准你的脸的前面，它把你的头发吹向后面。如果你取其中两次吹风的平均值，你的头发应该回到正常位置。

同样，我们使用两种相位编码方向，在两者之间创造一种平均值。我们知道这两种相位编码方式都会给数据带来两种独立的、相反的扭曲，但我们可以用消除扭曲来抵消它们。

我们的第一步是将反向相位编码的NIFTI文件转换为.mif格式。我们还将其b值和b向量添加到mif头文件中。
```
$ mrconvert sub-CON02_ses-preop_acq-PA_dwi.nii.gz - -fslgrad sub-CON02_ses-preop_acq-PA_dwi.bvec sub-CON02_ses-preop_acq-PA_dwi.bval | mrmath - mean mean_b0_PA.mif -axis 3

mrconvert: [100%] uncompressing image "sub-CON02_ses-preop_acq-PA_dwi.nii.gz"
mrconvert: [100%] copying from "sub-CON02_ses-preop_acq-PA_dwi.nii.gz" to "/var/folde...0gn/T/mrtrix-tmp-XwJdp7.mif"
mrmath: [100%] preloading data for "/var/folders/39/93zn2fy95cd9b_b38zvmvgjc0000gn/T/mrtrix-tmp-XwJdp7.mif"
mrmath: [100%] computing mean along axis 3...

```
**注意这里的`-`为临时文件命名方式，我们就可以在`|`后的shell命令下直接继续使用它。**
原PA文件`sub-CON02_ses-preop_acq-PA_dwi.nii.gz`数据维度为96\*96\*60\*2 ,得到的文件`mean_b0_PA.mif`的数据维度变成了96\*96\*60，即指定了axis=3，将在时间维度（或者说volumes）层面上进行了平均（mean）。

接下来，我们从初级相位编码图像中提取b值，然后将两者与`mrcat`结合：

```
$ dwiextract -h
MRtrix 3.0.4                       dwiextract                        Dec 14 2022

     dwiextract: part of the MRtrix3 package

SYNOPSIS

     Extract diffusion-weighted volumes, b=0 volumes, or certain shells from a
     DWI dataset

USAGE

     dwiextract [ options ] input output

        input        the input DW image.

        output       the output image (diffusion-weighted volumes by default).


EXAMPLE USAGES

     Calculate the mean b=0 image from a 4D DWI series:
       $ dwiextract dwi.mif - -bzero | mrmath - mean mean_bzero.mif -axis 3
     The dwiextract command extracts all volumes for which the b-value is
     (approximately) zero; the resulting 4D image can then be provided to the
...
OPTIONS

  -bzero
     Output b=0 volumes (instead of the diffusion weighted volumes, if
     -singleshell is not specified).

  -no_bzero
     Output only non b=0 volumes (default, if -singleshell is not specified).
...


$ mrcat -h 
MRtrix 3.0.4                          mrcat                          Dec 14 2022

     mrcat: part of the MRtrix3 package

SYNOPSIS

     Concatenate several images into one

USAGE

     mrcat [ options ] image1 image2 [ image2 ... ] output

        image1       the first input image.

        image2       additional input image(s).

        output       the output image.


EXAMPLE USAGES

     Concatenate individual 3D volumes into a single 4D image series:
       $ mrcat volume*.mif series.mif
     The wildcard characters will find all images in the current working

# 使用dwiextract提取AP方向b值为0的所有volumes，并获取其均值mif文件；
$ dwiextract sub-CON02_ses-preop_acq-AP_dwi_denoise.mif - -bzero | mrmath - mean mean_b0_AP.mif -axis 3

# 使用mrcat命令合并两个图像文件,这将创建一个新图像“b0_pair.mif”，其中包含两个相位编码图像的平均b=0图像。
$ mrcat mean_b0_AP.mif mean_b0_PA.mif -axis 3 b0_pair.mif
mrcat: [100%] concatenating "mean_b0_AP.mif"
mrcat: [100%] concatenating "mean_b0_PA.mif"

```

#### 将它们放在一起：使用`dwipreproc`进行预处理(Eddy处理，去除涡流)
现在我们有了运行主要预处理步骤所需的一切，该步骤由`dwipreproc`调用。在大多数情况下，这个命令是一个包装器，它使用FSL的命令，如`topup`和`eddy`，来解除数据的扭曲并去除涡流。在本教程中，我们将使用下面这行代码：
```
$ dwifslpreproc -h
MRtrix 3.0.4                      dwifslpreproc

     dwifslpreproc: part of the MRtrix3 package

SYNOPSIS

     Perform diffusion image pre-processing using FSL's eddy tool; including
     inhomogeneity distortion correction using FSL's topup tool if possible

USAGE

     dwifslpreproc [ options ] input output

        input        The input DWI series to be corrected

        output       The output corrected image series

DESCRIPTION

     This script is intended to provide convenience of use of the FSL software
     tools topup and eddy for performing DWI pre-processing, by encapsulating
     some of the surrounding image data and metadata processing steps. It is
     intended to simply these processing steps for most commonly-used DWI
     acquisition strategies, whilst also providing support for some more exotic
     ...
OPTIONS
  -pe_dir PE
     Manually specify the phase encoding direction of the input series; can be a
     signed axis number (e.g. -0, 1, +2), an axis designator (e.g. RL, PA, IS),
     or NIfTI axis codes (e.g. i-, j, k)
  -eddy_options " EddyOptions"
     Manually provide additional command-line options to the eddy command
     (provide a string within quotation marks that contains at least one space,
     even if only passing a single command-line option to eddy)
  -se_epi image
     Provide an additional image series consisting of spin-echo EPI images,
     which is to be used exclusively by topup for estimating the inhomogeneity
     field (i.e. it will not form part of the output image series)
    ...
Options for specifying the acquisition phase-encoding design; note that one of the -rpe_* option
s MUST be provided

  -rpe_none
     Specify that no reversed phase-encoding image data is being provided; eddy
     will perform eddy current and motion correction only

  -rpe_pair
     Specify that a set of images (typically b=0 volumes) will be provided for
     use in inhomogeneity field estimation only (using the -se_epi option)

  -rpe_all
     Specify that ALL DWIs have been acquired with opposing phase-encoding



```
第一个参数是输入和输出；第二个选项，`-nocleanup`将保留临时处理文件夹，其中包含一些我们以后要检查的文件。
`-pe_dir AP`表示主相位编码方向是从前到后，而`-rpe_pair`与`-se_epi`选项相结合，表示下面的输入文件（即 "b0_pair.mif"）是一对自旋回波图像，是用反向相位编码方向获取的。最后，`-eddy_options`指定了FSL命令eddy的特定选项。[**你可以访问eddy用户指南**](https://fsl.fmrib.ox.ac.uk/fsl/fslwiki/eddy/UsersGuide)，了解更多的选项和它们的详细作用。现在，我们只使用选项`--slm=linear`（这对获取的数据少于60个方向的数据很有用）和`--data_is_shelled`（这表明扩散数据是用多个b值获取的）。

```
# dwifslpreproc <input.mif> <output.mif> -nocleanup -pe_dir AP -rpe_pair -se_epi b0_pair.mif -eddy_options " --slm=linear --data_is_shelled"
$ dwifslpreproc sub-CON02_ses-preop_acq-AP_dwi_denoise.mif sub-02_den_preproc.mif -nocleanup -pe_dir AP -rpe_pair -se_epi b0_pair.mif -eddy_options " --slm=linear --data_is_shelled"

dwifslpreproc:
dwifslpreproc: Note that this script makes use of commands / algorithms that have relevant articles for citation; INCLUDING FROM EXTERNAL SOFTWARE PACKAGES. Please consult the help page (-help option) for more information.
dwifslpreproc:
dwifslpreproc: Generated scratch directory: /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwifslpreproc-tmp-RUWVON/
Command:  mrconvert /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/sub-CON02_ses-preop_acq-AP_dwi_denoise.mif /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwifslpreproc-tmp-RUWVON/dwi.mif -json_export /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwifslpreproc-tmp-RUWVON/dwi.json
Command:  mrconvert /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/b0_pair.mif /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwifslpreproc-tmp-RUWVON/se_epi.mif
dwifslpreproc: Changing to scratch directory (/Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwifslpreproc-tmp-RUWVON/)
dwifslpreproc: Total readout time not provided at command-line; assuming sane default of 0.1
Command:  mrinfo dwi.mif -export_grad_mrtrix grad.b
Command:  mrconvert se_epi.mif topup_in.nii -import_pe_table se_epi_manual_pe_scheme.txt -strides -1,+2,+3,+4 -export_pe_table topup_datain.txt
Command:  topup --imain=topup_in.nii --datain=topup_datain.txt --out=field --fout=field_map.nii.gz --config=/Applications/fsl/etc/flirtsch/b02b0.cnf --verbose
...

```
这个命令可能需要几个小时来运行，这取决于你的计算机速度。对于一台拥有8个处理核心的iMac来说，大约需要2个小时。当它完成后，检查输出，看看涡流校正和取消扭曲如何改变了数据；理想情况下，你应该看到在诸如眶额皮层等区域恢复了更多的信号，这些区域特别容易受到信号丢失的影响。

然后，我们可以使用`mrview`命令加载原始数据以及经过处理后的数据，对比查看其预处理效果。
```
# mrview <经过预处理后数据> -overlay.load <原始数据>
$ mrview sub-02_den_preproc.mif -overlay.load sub-CON02_ses-preop_acq-AP_dwi_denoise.mif
```
![BeforeAfterEddy](/mrtrix/images/02_BeforeAfterEddy.png)


#### 检查损坏的切片
`dwifslpreproc`命令中的一个选项，"-nocleanup"，保留了一个标题为 "tmp "的目录。在这个目录中，有一个名为`dwi_post_eddy.eddy_outlier_map`的文件，它包含了0和1的字符串。每一个1代表一个切片是一个离群点，可能是因为运动量太大、涡流或其他原因。

下面的代码，从`dwi`目录下运行，将导航到 "tmp "文件夹，并计算出离群片的百分比：
{{< tabs >}}
{{% tab name="shell" %}}
```
#/bin/bash
set -e
# 进入dwifslpreproc-tmp-*临时目录
cd dwifslpreproc-tmp-*

# 统计slices数量：找到dwi.mif文件的维度(Dimensions: 96 x 96 x 60 x 102)，,并得到第6列值为60，第8列值为102，相乘计算整体的slices数量；
totalSlices=`mrinfo dwi.mif | grep Dimensions | awk '{print $6 * $8}'`

# 统计离群点的个数：从前到后累加dwi_post_eddy.eddy_outlier_map文件的数值，然后输出；
# 注意这里相对原教程新增了NR>1,原因为目前输出dwi_post_eddy.eddy_outlier_map文件的第一行为说明文字，需要去除第一行，避免统计。
totalOutliers=`awk 'NR>1{ for(i=1;i<=NF;i++)sum+=$i } END { print sum }' dwi_post_eddy.eddy_outlier_map`

# 如果离群点数量超过10，则建议不要使用该数据；
echo "If the following number is greater than 10, you may have to discard this subject because of too much motion or corrupted slices"

# 计算离群点的百分比比例： 离群点数量/Slices总数）* 100
echo -n "the percentage of outlier slices:" | tee percentageOutliers.txt
echo "scale=5; ($totalOutliers / $totalSlices * 100)/1" | bc | tee -a percentageOutliers.txt

cd ..

```
{{% /tab %}}

{{% tab name="dwi_post_eddy.eddy_outlier_map" %}}
```text
One row per scan, one column per slice. Outlier: 1, Non-outlier: 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
```

{{% /tab %}}


{{< /tabs >}}


#### 生成Mask(dwibiascorrect、dwi2mask)
与fMRI分析一样，创建一个掩膜（mask）来限制你的分析只限于大脑体素是很有用的，这将加速我们的分析速度。

要做到这一点，事先运行一个叫做`dwibiascorrect`的命令是很有用的。这可以去除数据中检测到的不均匀性(inhomogeneities)，从而获得更好的掩膜估计。然而，在某些情况下，它可能导致更坏的估计；**与所有的预处理步骤一样，你应该在每个步骤前后检查它**：

```
$ dwibiascorrect -h
MRtrix 3.0.4                     dwibiascorrect

     dwibiascorrect: part of the MRtrix3 package

SYNOPSIS

     Perform B1 field inhomogeneity correction for a DWI volume series

USAGE

     dwibiascorrect algorithm [ options ] ...

        algorithm    Select the algorithm to be used to complete the script operation;
                     additional details and options become available once an
                     algorithm is nominated. Options are: ants, fsl

Options for importing the diffusion gradient table

  -grad GRAD
     Provide the diffusion gradient table in MRtrix format

  -fslgrad bvecs bvals
     Provide the diffusion gradient table in FSL bvecs/bvals format
...

# 使用dwibiascorrect来消除数据中检测到的不均匀性
$ dwibiascorrect ants sub-02_den_preproc.mif sub-02_den_preproc_unbiased.mif -bias bias.mif

dwibiascorrect:
dwibiascorrect: Note that this script makes use of commands / algorithms that have relevant articles for citation; INCLUDING FROM EXTERNAL SOFTWARE PACKAGES. Please consult the help page (-help option) for more information.
dwibiascorrect:
dwibiascorrect: Generated scratch directory: /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwibiascorrect-tmp-PRRP4X/
Command:  mrconvert /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/sub-02_den_preproc.mif /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwibiascorrect-tmp-PRRP4X/in.mif
dwibiascorrect: Changing to scratch directory (/Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwibiascorrect-tmp-PRRP4X/)
Command:  dwi2mask in.mif mask.mif
Command:  dwiextract in.mif - -bzero | mrmath - mean mean_bzero.mif -axis 3
Command:  mrconvert mean_bzero.mif mean_bzero.nii -strides +1,+2,+3
Command:  mrconvert mask.mif mask.nii -strides +1,+2,+3
Command:  N4BiasFieldCorrection -d 3 -i mean_bzero.nii -w mask.nii -o [corrected.nii,init_bias.nii] -s 4 -b [100,3] -c [1000,0.0]
Command:  mrcalc mean_bzero.mif mask.mif -mult - | mrmath - sum - -axis 0 | mrmath - sum - -axis 1 | mrmath - sum - -axis 2 | mrdump -
Command:  mrcalc corrected.nii mask.mif -mult - | mrmath - sum - -axis 0 | mrmath - sum - -axis 1 | mrmath - sum - -axis 2 | mrdump -
Command:  mrcalc init_bias.nii 0.7908883751651254 -mult bias.mif
Command:  mrcalc in.mif bias.mif -div result.mif
Command:  mrconvert result.mif /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/sub-02_den_preproc_unbiased.mif
Command:  mrconvert bias.mif /Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/bias.mif
dwibiascorrect: Changing back to original directory (/Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi)
dwibiascorrect: Deleting scratch directory (/Volumes/Touch/Datasets/DTI/ds001226-download/sub-CON02/ses-preop/dwi/dwibiascorrect-tmp-PRRP4X/)

```


{{% notice note %}}
上面的命令使用了`-ants`选项，这需要在你的系统上安装ANTs。我推荐这个程序，但万一你无法安装它，你可以用-fsl选项代替它。
```
# 如下展示了未安装ants，造成的dwibiascorrect运行报错信息。
...
dwibiascorrect: [ERROR] Could not find ANTS program N4BiasFieldCorrection; please check installation
...
```
{{% /notice %}}

然后，我们就可以使用`dwi2mask`创建掩膜，这会将把我们的分析限制在我们想位于大脑内的体素。
```
$ MRtrix 3.0.4                        dwi2mask                         Dec 14 2022

     dwi2mask: part of the MRtrix3 package

SYNOPSIS

     Generates a whole brain mask from a DWI image

USAGE

     dwi2mask [ options ] input output

        input        the input DWI image containing volumes that are both
                     diffusion weighted and b=0

        output       the output whole-brain mask image


DESCRIPTION

     All diffusion weighted and b=0 volumes are used to obtain a mask that
     includes both brain tissue and CSF.

     In a second step peninsula-like extensions, where the peninsula itself is
...

$ dwi2mask sub-02_den_preproc_unbiased.mif mask.mif
dwi2mask: [100%] preloading data for "sub-02_den_preproc_unbiased.mif"
dwi2mask: [done] computing dwi brain mask
dwi2mask: [done] applying mask cleaning filter
```

使用mrview命令查看mask数据
```
$ mrview mask.mif
```
![mask](/mrtrix/images/02_mask.png?width=50pc)


MRtrix的`dwi2mask`命令在大多数情况下工作得很好。然而，你可以从上面的图片中看到，在脑干和小脑内的掩膜上有几个洞。你可能对这些区域不感兴趣，但建议确保掩膜在任何地方都没有洞。

为此，你可以使用FSL的`bet2`这样的命令。例如，你可以使用以下代码将无偏的扩散加权图像转换成NIFTI格式，用`bet2`创建一个掩膜，然后将掩膜转换成.mif格式：
```
$ mrconvert sub-02_den_preproc_unbiased.mif sub-02_unbiased.nii

# 注意：在实践中，-f指定为0.2达到了最好的掩膜效果，与原教程0.7有所不同。
$ bet2 sub-02_unbiased.nii sub-02_masked -m -f 0.2

# 注意：在nifti转换为mif文件时，可能存在转换数据类型的报警信息
$ mrconvert sub-02_masked_mask.nii.gz mask.mif
```
你可能必须对分数强度阈值(fractional intensity threshold)（由`-f`指定）进行试验，以便产生一个你满意的掩膜。根据我的经验，对大多数大脑来说，这个阈值可以在0.2和0.7之间变化，以便生成一个足够的掩膜。
![bet2_mask](/mrtrix/images/02_mask_bet2.png?width=35pc)


另外，我们可以使用`MRIcron`软件的`Draw > Open VOI`功能，去打开nifti格式的掩膜文件，去逐帧修改，确保掩膜文件不存在漏洞。
![mricron](/mrtrix/images/02_mricron.png?width=45pc)

#### 参考资料
- https://andysbrainbook.readthedocs.io/en/latest/MRtrix/MRtrix_Course/MRtrix_04_Preprocessing.html