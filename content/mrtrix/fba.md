---
title: "MRtrix3教程 #8 FBA(Fixel-Based Analysis)"
date: 2023-06-05T12:07:01+08:00
draft: true
weight: 8
pre: <b>8. </b>
---

- [简介](#简介)
- [Fixel-Based Analysis的提前准备](#fixel-based-analysis的提前准备)
- [预处理步骤](#预处理步骤)
  - [1. Denoising and unringing](#1-denoising-and-unringing)
  - [2. Motion and Distortion correction](#2-motion-and-distortion-correction)
  - [3. Bias field correction](#3-bias-field-correction)
  - [4. Computing (average) tissue response functions](#4-computing-average-tissue-response-functions)
  - [5. Unsampling DW images](#5-unsampling-dw-images)
  - [6. Compute upsampled brain mask images](#6-compute-upsampled-brain-mask-images)
  - [7. Fibre Orientation Distribution estimation (multi-tissue spherical deconvolution)](#7-fibre-orientation-distribution-estimation-multi-tissue-spherical-deconvolution)
  - [8. Joint bias field correction and intensity normalisation](#8-joint-bias-field-correction-and-intensity-normalisation)
  - [9. Generate a study-specific unbiased FOD template](#9-generate-a-study-specific-unbiased-fod-template)
  - [10. Register all subject FOD images to the FOD template](#10-register-all-subject-fod-images-to-the-fod-template)
  - [11. Compute the template mask (intersection of all subject masks in template space)](#11-compute-the-template-mask-intersection-of-all-subject-masks-in-template-space)
  - [12. Compute a white matter template analysis fixel mask](#12-compute-a-white-matter-template-analysis-fixel-mask)
  - [13. Warp FOD images to template space](#13-warp-fod-images-to-template-space)
  - [14. Segment FOD images to estimate fixels and their apparent fibre density (FD)](#14-segment-fod-images-to-estimate-fixels-and-their-apparent-fibre-density-fd)
  - [15. Reorient fixels](#15-reorient-fixels)
  - [16. Assign subject fixels to template fixels](#16-assign-subject-fixels-to-template-fixels)
  - [17. Compute the fibre cross-section (FC) metric](#17-compute-the-fibre-cross-section-fc-metric)
  - [18. Compute a combined measure of fibre density and cross-section (FDC)](#18-compute-a-combined-measure-of-fibre-density-and-cross-section-fdc)
  - [19. Perform whole-brain fibre tractography on the FOD template](#19-perform-whole-brain-fibre-tractography-on-the-fod-template)
  - [20. Reduce biases in tractogram densities](#20-reduce-biases-in-tractogram-densities)
  - [21. Generate fixel-fixel connectivity matrix](#21-generate-fixel-fixel-connectivity-matrix)
  - [22. Smooth fixel data using fixel-fixel connectivity](#22-smooth-fixel-data-using-fixel-fixel-connectivity)
  - [23. Perform statistical analysis of FD, FC, and FDC](#23-perform-statistical-analysis-of-fd-fc-and-fdc)
  - [24. Visualise the results](#24-visualise-the-results)
- [脚本汇总](#脚本汇总)
- [创建GLM模型](#创建glm模型)
- [Fixel-Based Analysis on the Supercomputing Cluster](#fixel-based-analysis-on-the-supercomputing-cluster)
- [参考资料](#参考资料)


#### 简介
基于Fixel的分析（FBA）是一个新的框架，它解决了DTI的缺点，并在微观层面上估计了特定白质纤维束的个别群体的微观结构变化。术语 "fixel "被引入以描述特定体素内的纤维群。提出的作者认为FBA分析具有以下优点：
- （1）可适用于纤维交叉区域；
- （2）可在微观结构层次和宏观结构层次上进行分析。

到目前为止，我们一直专注于生成流线，即底层白质束的表示。这些是概率性纤维，考虑到附近的组织类型和角度的数量，以便在遥远的皮质区域之间建立生物学上合理的连接。MRtrix还允许生成固定点（**fixels**），或体素内的特定纤维群。

请记住，传统的扩散张量方法不能解决交叉纤维（**crossing-fibers**）的问题，即两束纤维以直角交叉会使该体素内出现均匀扩散的现象。通过使用以单一纤维取向的受限球形反卷积（CSD）作为基础函数，我们可以将体素内测得的信号解析为其组成纤维，或纤维取向分布（**FODs**，fiber orientation distributions）。然后，这可以用来量化底层白质的纤维密度（**FD**,fiber density），以及给定体素内横断面的整体大小变化（纤维横断面，或**FC**,fiber cross-section）。这两个指标可以进一步结合成一个单一的指标，称为纤维密度和横截面（**FDC**,fiber density and cross-section）。


[Raffelt et al., 2015](https://www.sciencedirect.com/science/article/pii/S1053811916304943#f0015)通过使用`7x7`体素矩阵示例说明了这些指标的差异。**给定体素内一定量的白质纤维，纤维数量的减少可以通过三种不同的方式发生**:
1. 白质纤维占据体素量不变，但总体密度降低（FD降低）；
2. 密度保持不变，但白质纤维占用的体素数减少（FC减少）；
3. 占用的体素数和这些体素内的白质密度都减少（FDC减少）。

![Fixel metrics diagrammatic illustrations](/mrtrix/images/08_Raffelt_2017_Fig1.jpeg)


#### Fixel-Based Analysis的提前准备

进行基于Fixel的分析的一些步骤与前几章的内容相似。然而，一旦纤维方向被估计出来，然后被扭曲成一个模板，就会有一些明显的区别。

如果要分析大量的受试者--例如，几十个或更多--我们将需要一台至少有几百G内存的计算机，最好有多核心的处理器。基于Fixel的分析可以在本地机器上完成，但它可能需要很长的时间；无论如何，你可以通过使用MRtrix的`for_each`命令来处理这些图像（[**关于该命令的细节可以在这里找到**](https://mrtrix.readthedocs.io/en/dev/reference/commands/for_each.html)）。为了说明如何使用这个批处理命令，我们将采取一个相对简单的案例，对原始扩散加权数据进行去噪。

首先，我们将组织我们的数据，使每个被试文件夹包含转换后的原始数据，称为`dwi.mif`。例如我们有三个来自BTC_Preop数据集的被试，我们可以创建一个新的子文件夹，称为`Fixel_Analysis`。在该文件夹中，创建另一个名为`subjects`的文件夹，其中包含三个子文件夹`sub-01`、`sub-02`和`sub-03`。假设这些文件夹中的每一个都包含每个受试者相应的原始扩散数据以及bvals和bvecs文件，你可以通过在`subjects`文件夹中输入以下代码来转换它们:


{{< tabs >}}

{{% tab name="自动创建文件与拷贝脚本" %}}
```shell
#!/bin/bash
# 使用shell脚本自动创建一个新的子文件夹，称为`Fixel_Analysis`。
# 在该文件夹中，创建另一个名为`subjects`的文件夹，其中包含三个子文件夹`sub-01`、`sub-02`和`sub-03`
set -e
workDir=Fixel_Analysis/subjects
mkdir -p ${workDir}

cnt=1
for i in sub-CON03 sub-CON04 sub-CON05; do
  echo "Copy From "$i
  mkdir -p ${workDir}/sub-0$cnt
  cp -r $i/* ${workDir}/sub-0$cnt
  cnt=$(($cnt+1))
done
```
{{% /tab %}}

{{% tab name="自动转换脚本" %}}
```shell
#!/bin/bash
# 这将使该nifti数据结合bvec和bval文件，创建一个新的原始扩散加权图像（mif）
set -e
workDir=Fixel_Analysis/subjects
cd ${workDir}

for i in sub-01 sub-02 sub-03; do
    for j in AP PA;do
        mrconvert -force $i/ses-preop/dwi/*$j*.nii.gz $i/ses-preop/dwi/dwi_$j.mif -fslgrad $i/ses-preop/dwi/*$j*.bvec $i/ses-preop/dwi/*$j*.bval
        rm  $i/ses-preop/dwi/*$j*.nii.gz #注意，该命令会删除原NIfTI数据
    done
done

```
{{% /tab %}}

{{% tab name="原ds001226-download数据集目录结构参考" %}}
```
...
├── sub-CON03
│   └── ses-preop
│       ├── anat
│       │   ├── sub-CON03_ses-preop_T1w.json
│       │   └── sub-CON03_ses-preop_T1w.nii.gz
│       ├── dwi
│       │   ├── sub-CON03_ses-preop_acq-AP_dwi.bval
│       │   ├── sub-CON03_ses-preop_acq-AP_dwi.bvec
│       │   ├── sub-CON03_ses-preop_acq-AP_dwi.json
│       │   ├── sub-CON03_ses-preop_acq-AP_dwi.nii.gz
│       │   ├── sub-CON03_ses-preop_acq-PA_dwi.bval
│       │   ├── sub-CON03_ses-preop_acq-PA_dwi.bvec
│       │   ├── sub-CON03_ses-preop_acq-PA_dwi.json
│       │   └── sub-CON03_ses-preop_acq-PA_dwi.nii.gz
│       └── func
│           ├── sub-CON03_ses-preop_task-rest_bold.json
│           └── sub-CON03_ses-preop_task-rest_bold.nii.gz
├── sub-CON04
│   └── ses-preop
│       ├── anat
│       │   ├── sub-CON04_ses-preop_T1w.json
│       │   └── sub-CON04_ses-preop_T1w.nii.gz
│       ├── dwi
│       │   ├── sub-CON04_ses-preop_acq-AP_dwi.bval
│       │   ├── sub-CON04_ses-preop_acq-AP_dwi.bvec
│       │   ├── sub-CON04_ses-preop_acq-AP_dwi.json
│       │   ├── sub-CON04_ses-preop_acq-AP_dwi.nii.gz
│       │   ├── sub-CON04_ses-preop_acq-PA_dwi.bval
│       │   ├── sub-CON04_ses-preop_acq-PA_dwi.bvec
│       │   ├── sub-CON04_ses-preop_acq-PA_dwi.json
│       │   └── sub-CON04_ses-preop_acq-PA_dwi.nii.gz
│       └── func
│           ├── sub-CON04_ses-preop_task-rest_bold.json
│           └── sub-CON04_ses-preop_task-rest_bold.nii.gz
├── sub-CON05
│   └── ses-preop
│       ├── anat
│       │   ├── sub-CON05_ses-preop_T1w.json
│       │   └── sub-CON05_ses-preop_T1w.nii.gz
│       ├── dwi
│       │   ├── sub-CON05_ses-preop_acq-AP_dwi.bval
│       │   ├── sub-CON05_ses-preop_acq-AP_dwi.bvec
│       │   ├── sub-CON05_ses-preop_acq-AP_dwi.json
│       │   ├── sub-CON05_ses-preop_acq-AP_dwi.nii.gz
│       │   ├── sub-CON05_ses-preop_acq-PA_dwi.bval
│       │   ├── sub-CON05_ses-preop_acq-PA_dwi.bvec
│       │   ├── sub-CON05_ses-preop_acq-PA_dwi.json
│       │   └── sub-CON05_ses-preop_acq-PA_dwi.nii.gz
│       └── func
│           ├── sub-CON05_ses-preop_task-rest_bold.json
│           └── sub-CON05_ses-preop_task-rest_bold.nii.gz
├── sub-CON06
│   └── ses-preop
│       ├── anat
│       │   ├── sub-CON06_ses-preop_T1w.json
│       │   └── sub-CON06_ses-preop_T1w.nii.gz
│       ├── dwi
│       │   ├── sub-CON06_ses-preop_acq-AP_dwi.bval
│       │   ├── sub-CON06_ses-preop_acq-AP_dwi.bvec
│       │   ├── sub-CON06_ses-preop_acq-AP_dwi.json
│       │   ├── sub-CON06_ses-preop_acq-AP_dwi.nii.gz
│       │   ├── sub-CON06_ses-preop_acq-PA_dwi.bval
│       │   ├── sub-CON06_ses-preop_acq-PA_dwi.bvec
│       │   ├── sub-CON06_ses-preop_acq-PA_dwi.json
│       │   └── sub-CON06_ses-preop_acq-PA_dwi.nii.gz
│       └── func
│           ├── sub-CON06_ses-preop_task-rest_bold.json
│           └── sub-CON06_ses-preop_task-rest_bold.nii.gz
...
```
{{% /tab %}}

{{% tab name="Fixel_Analysis目录结构参考" %}}
```
Fixel_Analysis
└── subjects
    ├── sub-01
    │   └── ses-preop
    │       ├── anat
    │       │   ├── sub-CON03_ses-preop_T1w.json
    │       │   └── sub-CON03_ses-preop_T1w.nii.gz
    │       ├── dwi
    │       │   ├── dwi_AP.mif
    │       │   ├── dwi_PA.mif
    │       │   ├── sub-CON03_ses-preop_acq-AP_dwi.bval
    │       │   ├── sub-CON03_ses-preop_acq-AP_dwi.bvec
    │       │   ├── sub-CON03_ses-preop_acq-AP_dwi.json
    │       │   ├── sub-CON03_ses-preop_acq-AP_dwi.nii.gz
    │       │   ├── sub-CON03_ses-preop_acq-PA_dwi.bval
    │       │   ├── sub-CON03_ses-preop_acq-PA_dwi.bvec
    │       │   ├── sub-CON03_ses-preop_acq-PA_dwi.json
    │       │   └── sub-CON03_ses-preop_acq-PA_dwi.nii.gz
    │       └── func
    │           ├── sub-CON03_ses-preop_task-rest_bold.json
    │           └── sub-CON03_ses-preop_task-rest_bold.nii.gz
    ├── sub-02
    │   └── ses-preop
    │       ├── anat
    │       │   ├── sub-CON04_ses-preop_T1w.json
    │       │   └── sub-CON04_ses-preop_T1w.nii.gz
    │       ├── dwi
    │       │   ├── dwi_AP.mif
    │       │   ├── dwi_PA.mif
    │       │   ├── sub-CON04_ses-preop_acq-AP_dwi.bval
    │       │   ├── sub-CON04_ses-preop_acq-AP_dwi.bvec
    │       │   ├── sub-CON04_ses-preop_acq-AP_dwi.json
    │       │   ├── sub-CON04_ses-preop_acq-AP_dwi.nii.gz
    │       │   ├── sub-CON04_ses-preop_acq-PA_dwi.bval
    │       │   ├── sub-CON04_ses-preop_acq-PA_dwi.bvec
    │       │   ├── sub-CON04_ses-preop_acq-PA_dwi.json
    │       │   └── sub-CON04_ses-preop_acq-PA_dwi.nii.gz
    │       └── func
    │           ├── sub-CON04_ses-preop_task-rest_bold.json
    │           └── sub-CON04_ses-preop_task-rest_bold.nii.gz
    └── sub-03
        └── ses-preop
            ├── anat
            │   ├── sub-CON05_ses-preop_T1w.json
            │   └── sub-CON05_ses-preop_T1w.nii.gz
            ├── dwi
            │   ├── dwi_AP.mif
            │   ├── dwi_PA.mif
            │   ├── sub-CON05_ses-preop_acq-AP_dwi.bval
            │   ├── sub-CON05_ses-preop_acq-AP_dwi.bvec
            │   ├── sub-CON05_ses-preop_acq-AP_dwi.json
            │   ├── sub-CON05_ses-preop_acq-AP_dwi.nii.gz
            │   ├── sub-CON05_ses-preop_acq-PA_dwi.bval
            │   ├── sub-CON05_ses-preop_acq-PA_dwi.bvec
            │   ├── sub-CON05_ses-preop_acq-PA_dwi.json
            │   └── sub-CON05_ses-preop_acq-PA_dwi.nii.gz
            └── func
                ├── sub-CON05_ses-preop_task-rest_bold.json
                └── sub-CON05_ses-preop_task-rest_bold.nii.gz

17 directories, 42 files
```

{{% /tab %}}
{{< /tabs >}}






#### 预处理步骤
接下来的预处理步骤可以在[这个网页上找到](https://mrtrix.readthedocs.io/en/latest/fixel_based_analysis/mt_fibre_density_cross-section.html)，在MRtrix网站上。他们详细解释了每一个步骤，所以我们不会在这里重复所有的解释。让我们专注于第一个预处理步骤，去噪，以说明for_each命令的工作原理。例如，如果我们在subject目录下运行以下代码：


##### 1. Denoising and unringing

```shell
$ for_each -h
MRtrix 3.0.4                        for_each

     for_each: part of the MRtrix3 package

SYNOPSIS

     Perform some arbitrary processing step for each of a set of inputs

USAGE

     for_each [ options ] inputs colon command

        inputs       Each of the inputs for which processing should be run

        colon        Colon symbol (":") delimiting the for_each inputs &
                     command-line options from the actual command to be executed

        command      The command string to run for each input, containing any
                     number of substitutions listed in the Description section
...
       - IN:   The full matching pattern, including leading folders. For
     example, if the target list contains a file "folder/image.mif", any
     occurrence of "IN" will be substituted with "folder/image.mif".

        - NAME: The basename of the matching pattern. For example, if the target
     list contains a file "folder/image.mif", any occurrence of "NAME" will be
     substituted with "image.mif".

        - PRE:  The prefix of the input pattern (the basename stripped of its
     extension). For example, if the target list contains a file
     "folder/my.image.mif.gz", any occurrence of "PRE" will be substituted with
     "my.image".

        - UNI:  The unique part of the input after removing any common prefix
     and common suffix. For example, if the target list contains files:
     "folder/001dwi.mif", "folder/002dwi.mif", "folder/003dwi.mif", any
     occurrence of "UNI" will be substituted with "001", "002", "003".
...

# 如果咱们想调试for_each代码，那么可以加入debug或者info参数:for_each -debug * :  <your cmd>，举例：for_each -debug * : echo IN 或者for_each -info * : echo IN

$ for_each -debug * : dwidenoise IN/ses-preop/dwi/dwi_AP.mif IN/ses-preop/dwi/dwi_AP_denoised.mif
for_each -debug * : dwidenoise IN/ses-preop/dwi/dwi_AP.mif IN/ses-preop/dwi/dwi_AP_denoised.mif
for_each: [DEBUG] execute() (from app.py:199): All inputs: ['sub-01', 'sub-02', 'sub-03']
for_each: [DEBUG] execute() (from app.py:199): Command: dwidenoise IN/ses-preop/dwi/dwi_AP.mif IN/ses-preop/dwi/dwi_AP_denoised.mif
for_each: [DEBUG] execute() (from app.py:199): CMDSPLIT: ['dwidenoise', 'IN/ses-preop/dwi/dwi_AP.mif', 'IN/ses-preop/dwi/dwi_AP_denoised.mif']
for_each: [DEBUG] execute() (from app.py:199): Common prefix: sub-0
for_each: [DEBUG] execute() (from app.py:199): No common suffix
for_each: [DEBUG] __init__() (from for_each:192): Input text: sub-01
for_each: [DEBUG] __init__() (from for_each:192): Substitutions: {'IN': 'sub-01', 'NAME': 'sub-01', 'PRE': 'sub-01', 'UNI': '1'}
for_each: [DEBUG] __init__() (from for_each:192): Resulting command: ['dwidenoise', 'sub-01/ses-preop/dwi/dwi_AP.mif', 'sub-01/ses-preop/dwi/dwi_AP_denoised.mif']
for_each: [DEBUG] __init__() (from for_each:192): Input text: sub-02
for_each: [DEBUG] __init__() (from for_each:192): Substitutions: {'IN': 'sub-02', 'NAME': 'sub-02', 'PRE': 'sub-02', 'UNI': '2'}
for_each: [DEBUG] __init__() (from for_each:192): Resulting command: ['dwidenoise', 'sub-02/ses-preop/dwi/dwi_AP.mif', 'sub-02/ses-preop/dwi/dwi_AP_denoised.mif']
for_each: [DEBUG] __init__() (from for_each:192): Input text: sub-03
for_each: [DEBUG] __init__() (from for_each:192): Substitutions: {'IN': 'sub-03', 'NAME': 'sub-03', 'PRE': 'sub-03', 'UNI': '3'}
for_each: [DEBUG] __init__() (from for_each:192): Resulting command: ['dwidenoise', 'sub-03/ses-preop/dwi/dwi_AP.mif', 'sub-03/ses-preop/dwi/dwi_AP_denoised.mif']
for_each: [  0%] 0/3 jobs completed sequentially...
Command:  dwidenoise sub-01/ses-preop/dwi/dwi_AP.mif sub-01/ses-preop/dwi/dwi_AP_denoised.mif
          dwidenoise: [INFO] opening image "sub-01/ses-preop/dwi/dwi_AP.mif"...
          dwidenoise: [INFO] image "sub-01/ses-preop/dwi/dwi_AP.mif" opened with dimensions 96x96x60x102, voxel spacing 2.5x2.5x2.5x8.6999999999999993, datatype Int16LE
          dwidenoise: [INFO] selected patch size: 5 x 5 x 5.
          dwidenoise: [INFO] select real float32 for processing
          dwidenoise: [100%] preloading data for "sub-01/ses-preop/dwi/dwi_AP.mif"
          dwidenoise: [INFO] creating image "sub-01/ses-preop/dwi/dwi_AP_denoised.mif"...
          dwidenoise: [INFO] image "sub-01/ses-preop/dwi/dwi_AP_denoised.mif" created with dimensions 96x96x60x102, voxel spacing 2.5x2.5x2.5x8.6999999999999993, datatype Float32LE
          dwidenoise: [100%] running MP-PCA denoising
for_each: [ 33%] 1/3 jobs completed sequentially...
Command:  dwidenoise sub-02/ses-preop/dwi/dwi_AP.mif sub-02/ses-preop/dwi/dwi_AP_denoised.mif
          dwidenoise: [INFO] opening image "sub-02/ses-preop/dwi/dwi_AP.mif"...
          dwidenoise: [INFO] image "sub-02/ses-preop/dwi/dwi_AP.mif" opened with dimensions 96x96x60x102, voxel spacing 2.5x2.5x2.5x8.6999999999999993, datatype Int16LE
          dwidenoise: [INFO] selected patch size: 5 x 5 x 5.
          dwidenoise: [INFO] select real float32 for processing
          dwidenoise: [100%] preloading data for "sub-02/ses-preop/dwi/dwi_AP.mif"
          dwidenoise: [INFO] creating image "sub-02/ses-preop/dwi/dwi_AP_denoised.mif"...
          dwidenoise: [INFO] image "sub-02/ses-preop/dwi/dwi_AP_denoised.mif" created with dimensions 96x96x60x102, voxel spacing 2.5x2.5x2.5x8.6999999999999993, datatype Float32LE
          dwidenoise: [100%] running MP-PCA denoising
for_each: [ 67%] 2/3 jobs completed sequentially...
Command:  dwidenoise sub-03/ses-preop/dwi/dwi_AP.mif sub-03/ses-preop/dwi/dwi_AP_denoised.mif
          dwidenoise: [INFO] opening image "sub-03/ses-preop/dwi/dwi_AP.mif"...
          dwidenoise: [INFO] image "sub-03/ses-preop/dwi/dwi_AP.mif" opened with dimensions 96x96x60x102, voxel spacing 2.5x2.5x2.5x8.6999999999999993, datatype Int16LE
          dwidenoise: [INFO] selected patch size: 5 x 5 x 5.
          dwidenoise: [INFO] select real float32 for processing
          dwidenoise: [100%] preloading data for "sub-03/ses-preop/dwi/dwi_AP.mif"
          ...
for_each:
for_each: Script reported successful completion for all inputs         
```
冒号(:)左边的代码意味着要循环处理由*通配符捕获的每一个项目。(注意，如果这个目录中除了被试文件夹之外还有其他文件，比如文本文件，那么这个命令就会出错，因为它正试图将预处理步骤应用于非扩散文件）。每个项目都被加载到IN关键字中；例如，对于第一个被试，这将扩展为：
```
dwidenoise sub-01/dwi.mif sub-01/dwi_denoised.mif
```
该命令获取sub-01的dwi.mif文件，对其进行去噪处理，并将输出文件dwi_denoised.mif放在sub-01文件夹中。这也是本教程中列出的所有其他for_each命令的程序。我们查看帮助文档，还应该注意的一个变量是`PRE`关键字，它是将输入项的扩展名剥离出来。


如果如果包括Gibbs环消除，则加入下述代码:
```shell
$ for_each * : medegibbs IN/ses-preop/dwi/dwi_AP_denoised.mif IN/ses-preop/dwi/dwi_AP_denoised_unringed.mif -axes 0,1
```


##### 2. Motion and Distortion correction
`dwifslpreproc`命令处理DWI数据的运动和失真校正（包括涡流失真和可选的易感性引起的EPI失真）。尽管该命令与其他MRtrix3命令无缝衔接，但它实际上是一个脚本，与FSL软件包接口，执行其大部分核心功能和算法。**为了使这个命令发挥作用，需要安装FSL（包括EDD）。**还要记得引用与具体算法有关的文章（见[dwifslpreproc](https://mrtrix.readthedocs.io/en/latest/reference/commands/dwifslpreproc.html#dwifslpreproc)帮助页）。

{{% notice note %}}
`dwifslpreproc`的`-pe_dir`选项是用来指定采集的相位编码方向的。上面例子中的`-pe_dir AP`指的是前后相位编码方向，这在采集人类数据时是比较常用的。对于典型的人类数据，将其改为`-pe_dir LR`，表示左-右相位编码方向，或`-pe_dir SI`，表示上-下相位编码方向。
{{% /notice %}}
```shell
$ for_each subjects/* : dwifslpreproc IN/${InterPath}dwi_${Direction}_denoised_unringed.mif IN/${InterPath}dwi_${Direction}_denoised_unringed_preproc.mif -rpe_none -pe_dir AP
```


##### 3. Bias field correction
multi-tissue FBA pipeline在后面的`mtnormalise`步骤中对偏置场进行校正（并联合进行全局强度归一化）。在这个阶段运行`dwibiascorrect`（不那么稳健和准确）的唯一动机是为了改善大脑掩膜的估计（在后面的`dwi2mask`步骤中，如果数据中存在严重的偏场）。然而，也有报道说，在这个阶段运行`dwibiascorrect`会导致后来的脑掩膜(Brain Mask)估计效果差。这可能是在数据中没有那么强烈的偏差场的情况下更有可能发生。在这个阶段是否运行`dwibiascorrect`对后面的`mtnormalise`的性能没有任何重大影响。

如果在这个阶段进行DWI偏置场校正，首先从DWI b=0数据中估计偏置场，然后应用偏置场校正所有DW容积，这是在MRtrix3的dwibiascorrect脚本中使用ants算法单步完成。该脚本使用ANTs中可用的偏置场校正算法（N4算法）。在这个基于fixel的分析管道中，不要用这个脚本的fsl算法。要对DW图像进行偏场校正，请运行：
```
$ for_each subjects/* : dwibiascorrect ants IN/${InterPath}dwi_${Direction}_denoised_unringed_preproc.mif IN/${InterPath}dwi_${Direction}_denoised_unringed_preproc_unbiased.mif
```

##### 4. Computing (average) tissue response functions
[Dhollander2016b](https://mrtrix.readthedocs.io/en/latest/reference/references.html#dhollander2016b)提出了从数据本身获得代表单纤维白质、灰质和脑脊液的3个组织反应函数的稳健和完全自动化的无监督方法，经过[Dhollander2019](https://mrtrix.readthedocs.io/en/latest/reference/references.html#dhollander2019)的改进，运行方式如下：
```shell
# 其中${unbias}用于确认Bias field correction是否做过。
$ for_each subjects/* : dwi2response dhollander IN/${InterPath}dwi_${Direction}_denoised_unringed_preproc${unbias}.mif IN/${InterPath}${Direction}_response_wm.txt IN/${InterPath}${Direction}_response_gm.txt IN/${InterPath}${Direction}_response_csf.txt
```

对于基于fixel的分析来说，关键是只使用一套唯一的（三个）响应函数（response function）来对所有受试者进行（3个组织）球形反卷积：由于（3个组织）球形反卷积结果将用这套响应函数的函数来表示，它们可以（以抽象的方式）被视为最终表观纤维密度指标和模型中估计的其他区段的单位。获得一套唯一的响应函数的一个可能的方法是对从所有受试者获得的每种组织类型的响应函数进行平均：
```shell
responsemean subjects/*/${InterPath}/${Direction}_response_wm.txt group_average_response_wm.txt
responsemean subjects/*/${InterPath}/${Direction}_response_gm.txt group_average_response_gm.txt
responsemean subjects/*/${InterPath}/${Direction}_response_csf.txt group_average_response_csf.txt
```

然而，没有严格要求最终的响应函数集必须是每个组织类型的所有受试者响应函数的平均值（或者事实上，它甚至不一定是平均值本身）。在某些非常特殊的情况下（病理对大脑产生全面影响，），如果不能可靠地获得响应函数，建议不考虑这些受试者。


##### 5. Unsampling DW images
在计算FODs之前对DWI数据进行上采样，可以增加解剖数据的对比度，改善之后的模板建立、配准、纤维束成像和统计。我们建议对人脑的各向同性体素大小进行上采样（如果你的原始分辨率已经很高，你可以跳过这一步）：
```shell
for_each subjects/* : mrgrid IN/${InterPath}dwi_${Direction}_denoised_unringed_preproc${unbias}.mif regrid -vox 1.24 IN/${InterPath}dwi_${Direction}_denoised_unringed_preproc${unbias}_unsampled.mif
```


##### 6. Compute upsampled brain mask images
从上采样的DW图像计算全脑掩膜：
```shell
for_each subjects/* : dwi2mask IN/${InterPath}dwi_${Direction}_denoised_unringed_preproc${unbias}_upsampled.mif IN/${InterPath}dwi_${Direction}_mask_upsampled.mif
```
{{% notice warning %}}
**在这一阶段，检查所有单个受试者的掩模是否包括要分析的大脑的所有区域是绝对关键的**。纤维方向分布(FOD)将只在这些掩模内计算；在以后的步骤中（在模板空间），分析掩模将被限制在所有掩模的交集上，所以任何排除某个区域的单个受试者掩模将导致该区域被排除在整个分析之外（除非遵循更先进的管道；详见[Mitigating the effects of brain cropping](https://mrtrix.readthedocs.io/en/latest/fixel_based_analysis/mitigating_brain_cropping.html#mitigating-brain-cropping)）。一般来说，在这个阶段，看起来过于宽松或包括非大脑区域的掩膜不应该引起任何关注。因此，如果有疑问，建议在这个阶段总是偏向于包含（区域）。
{{% /notice %}}

{{% notice note %}}
早期的`dwibiascorrect`步骤在基于多组织fixel的分析管道中并不重要，因为后期的`mtnormalise`步骤表现得更加稳健（如果包括`dwibiascorrect`，`mtnormalise`通常会在后期进一步改善结果）。虽然执行较早的`dwibiascorrect`步骤通常会提高`dwi2mask`的性能，但也有观察到相反的情况（通常是如果数据只包含弱的偏置场）。如果需要的话，可以通过在pipeline中加入或不加入`dwibiascorrect`来试验最佳的`dwi2mask`结果，并在必要时手动修正掩码（通过添加`dwi2mask`未能包括的区域）。
{{% /notice %}}

##### 7. Fibre Orientation Distribution estimation (multi-tissue spherical deconvolution)
在进行基于fixel的分析时，应使用之前获得的唯一一套（平均）组织响应函数进行多组织约束的球形反卷积：
```shell
for_each subjects/* : dwi2fod msmt_csd IN/${InterPath}dwi_${Direction}_denoised_unringed_preproc${unbias}_unsampled.mif group_average_response_wm.txt  IN/${InterPath}wmfod.mif group_average_response_gm.txt IN/${InterPath}gm.mif group_average_response_csf.txt IN/${InterPath}csf.mif -mask IN/${InterPath}dwi_${Direction}_mask_upsampled.mif
```

##### 8. Joint bias field correction and intensity normalisation
要对多组织区间参数进行联合偏置场校正和全局强度归一化，请使用`mtnormalise`：
```shell
for_each subjects/* : mtnormalise IN/${InterPath}wmfod.mif IN/${InterPath}wmfod_norm.mif IN/${InterPath}gm.mif IN/${InterPath}gm_norm.mif IN/${InterPath}csf.mif IN/${InterPath}csf_norm.mif -mask IN/${InterPath}dwi_${Direction}_mask_upsampled.mif
```

如果所有受试者的多组织CSD是用同一组（三个）组织反应函数进行的，那么`mtnormalise`的输出结果使这些受试者之间的绝对振幅也是可比的。**请注意，这一步在FBA管道中是至关重要的**，即使先前使用`dwibiascorrect`进行了偏置场校正，因为`dwibiascorrect`并不校正受试者之间的整体强度差异。`mtnormalise`的性能不会因为之前是否运行过`dwibiascorrect`而受到明显影响。如果事先在pipeline中运行了偏置场校正，`mtnormalise`将进一步校正残留的强度不均匀性。


{{% notice warning %}}
`mtnormalise`的结果可能对包含非脑体素的掩膜敏感。尽管不包含脑组织，但基础算法会试图将这些体素的组织体积之和来进行到归一化，如果这些体素的数量很大，就会导致错误的偏置场校正。出于这个原因，我们建议在`mtnormalise`步骤中使用保守的（即较少的空间扩展）掩码。与第6步不同的是，在第6步中，我们鼓励包括所有的脑体素，甚至以包括一些非脑体素为代价，对于偏置场的估计，排除非脑体素比包括所有的脑体素更优先。
{{% /notice %}}


##### 9. Generate a study-specific unbiased FOD template
**群体模板(Population template)的创建是基于fixel分析中最耗时的步骤之一**。如果你的研究中有非常多的受试者，你可以选择从30-40人的有限子集中创建模板。通常情况下，受试者的选择是为了使生成的模板能够代表你的群体（例如，类似数量的病人和对照组，但要避免与其他群体相比有过度异常的病人）。为了建立一个模板，把所有的FOD图像放在一个文件夹里，把一组相应的掩膜图像（与FOD图像的前缀相同）放在另一个文件夹里（使用掩膜可以大大加快配准速度）：


```shell
mkdir -p ../template/fod_input
mkdir ../template/mask_input

# 将所有 FOD 图像（和掩码）软链接到单个输入文件夹中。要使用整个群体来构建模板：
# 注意：这里的PRE是相对于for_each的变量，会被替换；
for_each * : ln -sr IN/wmfod_norm.mif ../template/fod_input/PRE.mif
for_each * : ln -sr IN/dwi_mask_upsampled.mif ../template/mask_input/PRE.mif

# 如果你选择从有限的子集（如30-40个）受试者中创建模板，而你的研究有多个组，那么你可以争取每个组的受试者数量相似，使模板更能代表整个人群。假设受试者目录标签可以用来识别每个组的成员
for_each `ls -d *patient | sort -R | tail -20` : ln -sr IN/wmfod_norm.mif ../template/fod_input/PRE.mif ";" ln -sr IN/dwi_mask_upsampled.mif ../template/mask_input/PRE.mif
for_each `ls -d *control | sort -R | tail -20` : ln -sr IN/wmfod_norm.mif ../template/fod_input/PRE.mif ";" ln -sr IN/dwi_mask_upsampled.mif ../template/mask_input/PRE.mif

# 运行FOD 模板建立命令
population_template ../template/fod_input -mask_dir ../template/mask_input ../template/wmfod_template.mif -voxel_size 1.25
```
voxel大小通常被设置为与输入的FOD图像的voxel大小一样（在这个pipeline中，通常是指pipeline道的早期，预处理数据被上采样的分辨率）。


##### 10. Register all subject FOD images to the FOD template
将每个被试(或者叫受试者)的FOD图像配准到FOD模板上：
```shell
for_each * : mrregister IN/wmfod_norm.mif -mask1 IN/dwi_mask_upsampled.mif ../template/wmfod_template.mif -nl_warp IN/subject2template_warp.mif IN/template2subject_warp.mif
```

##### 11. Compute the template mask (intersection of all subject masks in template space)
不同的受试者有不同的大脑覆盖。为确保后续分析在包含所有受试者数据的体素中进行，我们将所有受试者的掩膜扭曲（warp）到模板空间，并计算模板空间中所有受试者掩模的交集为模板掩膜。将所有的掩模扭曲到模板空间：
```shell
for_each * : mrtransform IN/dwi_mask_upsampled.mif -warp IN/subject2template_warp.mif -interp nearest -datatype bit IN/dwi_mask_in_template_space.mif

# 计算所有掩膜的交集
mrmath */dwi_mask_in_template_space.mif min ../template/template_mask.mif -datatype bit
```

{{% notice warning %}}
**在这一阶段，检查所产生的模板掩膜是否包括所有要分析的大脑区域是绝对关键的**。如果不是这样，可能的原因是个别受试者的掩膜不包括某个区域，或者是模板建立过程或个别受试者的配准在一个或多个受试者身上出了问题。建议回到这些步骤，在继续进行任何工作之前，找出并解决问题的原因。
{{% /notice %}}

{{% notice note %}}
在这一阶段，可以编辑模板掩膜，并删除某些不属于大脑的区域（尽管这些区域不太可能在开始的交叉步骤中存活，因为它们必须存在于所有受试者的掩膜中才会发生），或其他不感兴趣的区域。然而，**在这个阶段不建议将新的区域添加到掩模中**。如果确实需要这样做，请回到一开始导致这些区域被排除的相关步骤（也见上述警告）。
{{% /notice %}}

##### 12. Compute a white matter template analysis fixel mask
在这一步骤中，我们从FOD模板中分割出fixels。该结果是fixels的掩膜，定义了以后要进行统计分析的fixels（因此也定义了哪些fixels的统计数据可以通过基于连接的fixels增强机制（CFE,connectivity-based fixel enhancement）支持其他fixels[Raffelt2015](https://mrtrix.readthedocs.io/en/latest/reference/references.html#raffelt2015)）：

```shell
fod2fixel -mask ../template/template_mask.mif -fmls_peak_value 0.06 ../template/wmfod_template.mif ../template/fixel_mask
```
{{% notice note %}}
从这一步开始，出现在Pipeline中的Fixel图像使用[Fixel图像（目录）格式](https://mrtrix.readthedocs.io/en/latest/fixel_based_analysis/fixel_directory_format.html#fixel-format)进行存储，它将一个Fixel图像的所有Fixel数据存储在一个目录（即一个文件夹）中。
{{% /notice %}}

{{% notice warning %}}
这一步最终决定了将在其中进行统计分析的fixel mask，因此也决定了哪些fixel的统计数据可以通过CFE机制贡献给其他人；所以它可能对最终结果产生实质性影响。从本质上讲，如果通过`-fmls_peak_value`指定的阈值太高会排除真正的白质fixels，这就会导致不利的结果。这种风险在含有交叉纤维的体素中要高得多（在一个体素中交叉的纤维越多风险越大）。尽管已经观察到0.06是3-tissue CSD群体模板的一个不错的默认值，但**仍然强烈建议使用mrview可视化输出的fixel mask**。通过fixel绘图工具打开在`.../template/fixel_mask`目录中`index.mif`来做到这一点。如果在已知的或正常的解剖结构方面，fixel缺失（特别是注意交叉区域），则用提供给`-fmls_peak_value`选项的较低数值重新生成掩膜（当然，避免降低太多，因为可能会引入太多错误的或嘈杂的fixel）。对于一个成年人的大脑模板，使用1.25毫米的各向同性的模板体素尺寸，预计在fixel mask中会有几十万个fixels（你可以通过`mrinfo -size .../template/fixel_mask/directions.mif`检查，并查看图像沿第一维的尺寸）。
{{% /notice %}}

##### 13. Warp FOD images to template space
请注意，这里我们将FOD图像扭曲(warp)到模板空间，而不进行FOD的重新定位，因为重新定位将在随后的单独步骤中进行（在fixel分割之后）：
```shell
for_each * : mrtransform IN/wmfod_norm.mif -warp IN/subject2template_warp.mif -reorient_fod no IN/fod_in_template_space_NOT_REORIENTED.mif
```

##### 14. Segment FOD images to estimate fixels and their apparent fibre density (FD)
在这里，我们对每个FOD lobe进行分割，以确定每个体素中fixels的数量和方向。输出还包含每个fixel的表观纤维密度（AFD）值（估计为FOD lobe的积分）：

```shell
for_each * : fod2fixel -mask ../template/template_mask.mif IN/fod_in_template_space_NOT_REORIENTED.mif IN/fixel_in_template_space_NOT_REORIENTED -afd fd.mif
```
请注意，在下面的步骤中，我们将使用更通用的缩写词纤维密度（FD）来指代AFD指标。

##### 15. Reorient fixels
在此，我们根据之前使用的warp文件中每个体素的局部变换，对模板空间中所有受试者的fixels进行重新定位。
```shell
for_each * : fixelreorient IN/fixel_in_template_space_NOT_REORIENTED IN/subject2template_warp.mif IN/fixel_in_template_space
```
在这一步之后，`fixel_in_template_space_NOT_REORIENTED`文件夹可以被安全地删除。

##### 16. Assign subject fixels to template fixels
虽然每个受试者的数据已经（在空间上）被扭曲到共同的模板空间，受试者的fixels也相应地被重新定位(reoriented)，但仍然没有说明哪些fixels是匹配的（在受试者之间，以及受试者和模板fixels之间）。这一步正是通过将每个个体被试的fixels与单一的共同模板fixels集相匹配（这也内在地定义了它们在不同被试间的匹配方式）来建立的。对于模板fiexels掩码中的每个fixel，通过识别受试者图像匹配体素中的相应fixel，并将此相应受试者fixel的FD值分配给模板空间中的fixel来实现。如果在受试者中不存在或找不到与给定的模板fixel相对应的fixel，那么该fixel的值就被分配为零（因为在这个阶段没有受试者fixels很可能是由于FD很低，甚至是零）。这一步骤的执行情况如下：

```shell
for_each * : fixelcorrespondence IN/fixel_in_template_space/fd.mif ../template/fixel_mask ../template/fd PRE.mif
```

请注意，输出的fixel目录`.../template/fd`对所有被试都是一样的。因为在这一操作之后，只剩下一组fixels（即模板fixels），以及从每个被试获得的相应的FD值。由此产生的目录`.../template/fd`将这些数据作为单独的fixel数据文件来存储：每个受试者一个，都与一组相应的模板fixels有关。这种存储整个群体的FD数据的方式，为以后输入`fixelcfestats`做好了准备。

##### 17. Compute the fibre cross-section (FC) metric
如上所述，纤维密度指标在没有任何调制的情况下直接映射到fixel模板空间，只对每个体素中的轴突内空间的原始密度敏感。换句话说，它忽略了纤维束的横截面大小(cross-sectional size)，这是另一个属性，会影响到纤维束在其整个横截面范围内的总轴突内空间，从而影响其携带信息的总能力。在某些情况下，例如，萎缩可能会影响这个横截面的大小，但本身并不影响局部纤维密度指标。

在这一步中，我们计算了一个与纤维横截面（FC）的形态学差异有关的基于fixel的度量，其中的信息完全来自于配准时产生的翘曲（更多信息见[Raffelt2017](https://mrtrix.readthedocs.io/en/latest/reference/references.html#raffelt2017)）：

```shell
for_each * : warp2metric IN/subject2template_warp.mif -fc ../template/fixel_mask ../template/fc IN.mif
```

然而，对于FC的群体统计分析，我们建议计算`log(FC)`，**以确保数据以零为中心，呈正态分布。**在这里，我们创建了一个单独的fixel目录来存储`log(FC)`数据，并将fixel索引和方向文件复制过来：
```shell
mkdir ../template/log_fc
cp ../template/fc/index.mif ../template/fc/directions.mif ../template/log_fc
for_each * : mrcalc ../template/fc/IN.mif -log ../template/log_fc/IN.mif
```

{{% notice note %}}
这里计算的FC（因此也包括log(FC)）是一个相对指标，表达了相对于本研究人口模板的局部fixel-wise截面大小。虽然这使得解释单个研究中的FC差异成为可能（因为研究中只使用了一个唯一的模板），**但FC值不应在不同的研究中进行比较**，因为每个研究都有自己的人口模板。报告FC的绝对数量，或FC的绝对效应大小，也没有提供什么信息；因为同样，它只对模板有意义。
{{% /notice %}}

##### 18. Compute a combined measure of fibre density and cross-section (FDC)
纤维束携带信息的总能力，既受体素（fixel）水平的局部纤维密度的调节，也受其横截面尺寸的调节。在这里，我们计算了一个综合指标，它考虑了FD和FC的影响，形成了一个纤维密度和横截面（FDC）指标：
```shell
mkdir ../template/fdc
cp ../template/fc/index.mif ../template/fdc
cp ../template/fc/directions.mif ../template/fdc
for_each * : mrcalc ../template/fd/IN.mif ../template/fc/IN.mif -mult ../template/fdc/IN.mif
```
这也是一个很好的例子，说明可以跨多个fixel数据文件进行计算。然而，请注意，这只有在这两个文件共享同一组原始fixels（在这种情况下，模板fixel掩码）时才有效。**因为fixels也必须以完全相同的顺序存储才能正常工作，所以必须非常小心**，在`mrcalc`命令中使用的所有输入fixel数据文件的`index.mif`文件（在本例中为`./template/fd`和.`/template/fc`文件夹）是彼此的精确拷贝。


##### 19. Perform whole-brain fibre tractography on the FOD template
使用基于连通性的fixel增强（CFE）的统计分析[Raffelt2015](https://mrtrix.readthedocs.io/en/latest/reference/references.html#raffelt2015)利用了来自概率纤维束图的局部连通性信息，它作为邻域定义，对局部聚类的统计值进行无阈值增强。从FOD模板中生成全脑纤维束图（注意从这里开始的其余步骤都是从模板目录中执行的）：

```shell
cd ../template
tckgen -angle 22.5 -maxlen 250 -minlen 10 -power 1.0 wmfod_template.mif -seed_image template_mask.mif -mask template_mask.mif -select 20000000 -cutoff 0.06 tracks_20_million.tck
```
{{% notice warning %}}
由于历史上的软件错误，FOD模板束图的适当的FOD振幅截止点在不同的数据集和不同版本的MRtrix3之间可能有很大的不同。虽然0.06的值被建议为多组织数据的一个合理值，但使用这个值首先生成一个较小数量的流线（如100,000），并目测确认生成的流线在白质通路的末端表现出适当的传播范围，然后再致力于生成密集的束状图，可能是有益的。
{{% /notice %}}

##### 20. Reduce biases in tractogram densities
进行[SIFT,Spherical-deconvolution informed filtering of Tractograms](https://mrtrix.readthedocs.io/en/latest/quantitative_structural_connectivity/sift.html)，以减少全脑牵引图中的牵引偏差：
```shell
tcksift tracks_20_million.tck wmfod_template.mif tracks_2_million_sift.tck -term_number 2000000
```

{{% notice note %}}
SIFT[Smith2013](https://mrtrix.readthedocs.io/en/latest/reference/references.html#smith2013)，即 "球面反卷积信息滤波的纤维束追踪"，是一种改善全脑流线重建的定量的新方法。通过重建，流线密度与整个白质的球面反卷积估计的纤维密度成正比，连接两个区域的流线数量成为连接这两个区域的纤维横截面积的比例估计。因此，我们希望这种方法能在一系列流线图的应用中得到应用。
{{% /notice %}}

##### 21. Generate fixel-fixel connectivity matrix
基于全脑流线图的fixel-fixel连接矩阵的生成过程如下：
```shell
fixelconnectivity fixel_mask/ tracks_2_million_sift.tck matrix/
```
输出目录应包含三个图像：index.mif、fixels.mif和values.mif；这些图像用于编码本质上是稀疏的fixel-fixel连接。

{{% notice warning %}}
运行`fixelconnectivity`需要相当大的内存（对于模板分析fixel掩码中的大约50万个fixels和定义fixels之间成对连接的典型的tractogram，32GB的RAM是典型的内存需求；你可以通过`mrinfo -size /fixel_template/directions.mif`检查模板分析fixel掩码中的fixels的数量,并查看沿第一维度的图像大小；还可以检查（并避免）纤维束成像中的严重假阳性连接，例如在不应该连接两个半球的大区域中的流线）。如果与硬件有关的限制迫使你减少固定点的数量，不建议改变阈值来得出白质模板分析fixel mask，因为它可能会删除白质深处的关键fixels（例如在交叉区域）。相反，可以考虑略微增加模板体素的大小，或者在空间上从模板掩膜中去除不感兴趣的区域（在这个Pipeline中，早期得到的是模板空间中所有被试掩膜的交点）。
{{% /notice %}}

##### 22. Smooth fixel data using fixel-fixel connectivity
根据稀疏的fixel-fixel连接矩阵对fxiel数据进行平滑处理：
```shell
fixelfilter fd smooth fd_smooth -matrix matrix/
fixelfilter log_fc smooth log_fc_smooth -matrix matrix/
fixelfilter fdc smooth fdc_smooth -matrix matrix/
```
依次对每个fixel目录调用`fixelfilter`命令，平滑过滤器将被应用于每个目录中的所有fixel数据文件；没有必要对每个单独的fixel数据文件分别调用 `fixelfilter`命令。

##### 23. Perform statistical analysis of FD, FC, and FDC
使用CFE的统计分析是针对每个指标（FD、log(FC)和FDC）分别进行的，具体如下：
```shell
fixelcfestats fd_smooth/ files.txt design_matrix.txt contrast_matrix.txt matrix/ stats_fd/
fixelcfestats log_fc_smooth/ files.txt design_matrix.txt contrast_matrix.txt matrix/ stats_log_fc/
fixelcfestats fdc_smooth/ files.txt design_matrix.txt contrast_matrix.txt matrix/ stats_fdc/
```
输入文件`files.txt`是一个文本文件，包含输入fixel目录内要分析的每个文件的文件名（不是完整的路径），每个文件名在单独一行。行的顺序应与design_matrix.txt文件中的行相对应。

{{% notice note %}}
虽然以前的`fixelcfestats`版本在输入fixel数据时对这些数据进行了平滑处理，但现在不再是这样了。我们希望提供给`fixelcfestats`的fixel数据已经被适当地平滑了；例如，使用`fixelfilter`。
{{% /notice %}}


{{% notice note %}}
与其他一些提供GLM的软件包（如FSL `randomise`）不同，一列1（对应于 "全局截距"：当所有其他设计矩阵因素为零时的平均图像值）将不会被自动包含。如果你的实验模型需要这样一个因素，就有必要在设计矩阵中明确包括它。
{{% /notice %}}




##### 24. Visualise the results
为了查看结果，在mrview中加载population FOD模板图像，并使用矢量图工具叠加fixel图像。注意，p值图像被保存为（1-p值）。因此，要想在p<0.05的阈值下观察所有结果，在mrview fixel绘图工具中，应用一个较低的阈值0.95。

#### 脚本汇总
我们可以根据数据结构来调整MRtrix教程中的命令，或者假设把受试者组织起来，每个文件夹中都有一个`dvi.mif`文件，你可以复制和粘贴下面的代码（**注意，这省略了偏置场校正，根据我的经验，这有时会导致后来的大脑掩码估计更糟糕。**）,[**下述代码详细讲解参考官网文档**](https://mrtrix.readthedocs.io/en/latest/fixel_based_analysis/mt_fibre_density_cross-section.html)。

{{< tabs >}}
{{% tab name="shell" %}}
```shell
# Denoising:遍历所有被试，对相关的dwi.mif文件去噪。
for_each * : dwidenoise IN/dwi.mif IN/dwi_denoised.mif
# Unringing: 
for_each * : mrdegibbs IN/dwi_denoised.mif IN/dwi_denoised_unringed.mif -axes 0,1

for_each * : dwifslpreproc IN/dwi_denoised_unringed.mif IN/dwi_denoised_unringed_preproc.mif -rpe_none -pe_dir AP

for_each * : dwi2response dhollander IN/dwi_denoised_unringed_preproc.mif IN/response_wm.txt IN/response_gm.txt IN/response_csf.txt

responsemean */response_wm.txt ../group_average_response_wm.txt
responsemean */response_gm.txt ../group_average_response_gm.txt
responsemean */response_csf.txt ../group_average_response_csf.txt

for_each * : mrgrid IN/dwi_denoised_unringed_preproc_unbiased.mif regrid -vox 1.25 IN/dwi_denoised_unringed_preproc_unbiased_upsampled.mif

for_each * : dwi2mask IN/dwi_denoised_unringed_preproc_unbiased_upsampled.mif IN/dwi_mask_upsampled.mif

for_each * : dwi2fod msmt_csd IN/dwi_denoised_unringed_preproc_unbiased_upsampled.mif ../group_average_response_wm.txt IN/wmfod.mif ../group_average_response_gm.txt IN/gm.mif  ../group_average_response_csf.txt IN/csf.mif -mask IN/dwi_mask_upsampled.mif

for_each * : mtnormalise IN/wmfod.mif IN/wmfod_norm.mif IN/gm.mif IN/gm_norm.mif IN/csf.mif IN/csf_norm.mif -mask IN/dwi_mask_upsampled.mif

mkdir -p ../template/fod_input
mkdir ../template/mask_input

for_each * : ln -sr IN/wmfod_norm.mif ../template/fod_input/PRE.mif

for_each * : ln -sr IN/dwi_mask_upsampled.mif ../template/mask_input/PRE.mif population_te mplate ../template/fod_input -mask_dir ../template/mask_input ../template/wmfod_template.mif -voxel_size 1.25

for_each * : mrregister IN/wmfod_norm.mif -mask1 IN/dwi_mask_upsampled.mif ../template/wmfod_template.mif -nl_warp IN/subject2template_warp.mif IN/template2subject_warp.mif

for_each * : mrtransform IN/dwi_mask_upsampled.mif -warp IN/subject2template_warp.mif -interp nearest -datatype bit IN/dwi_mask_in_template_space.mif

mrmath */dwi_mask_in_template_space.mif min ../template/template_mask.mif -datatype bit

fod2fixel -mask ../template/template_mask.mif -fmls_peak_value 0.06 ../template/wmfod_template.mif ../template/fixel_mask

for_each * : mrtransform IN/wmfod_norm.mif -warp IN/subject2template_warp.mif -reorient_fod no IN/fod_in_template_space_NOT_REORIENTED.mif

for_each * : fod2fixel -mask ../template/template_mask.mif IN/fod_in_template_space_NOT_REORIENTED.mif IN/fixel_in_template_space_NOT_REORIENTED -afd fd.mif

for_each * : fixelreorient IN/fixel_in_template_space_NOT_REORIENTED IN/subject2template_warp.mif IN/fixel_in_template_space

for_each * : fixelcorrespondence IN/fixel_in_template_space/fd.mif ../template/fixel_mask ../template/fd PRE.mif

for_each * : warp2metric IN/subject2template_warp.mif -fc ../template/fixel_mask ../template/fc IN.mif
mkdir ../template/log_fc

cp ../template/fc/index.mif ../template/fc/directions.mif ../template/log_fc
for_each * : mrcalc ../template/fc/IN.mif -log ../template/log_fc/IN.mif

mkdir ../template/fdc
cp ../template/fc/index.mif ../template/fdc
cp ../template/fc/directions.mif ../template/fdc

for_each * : mrcalc ../template/fd/IN.mif ../template/fc/IN.mif -mult ../template/fdc/IN.mif

cd ../template

tckgen -angle 22.5 -maxlen 250 -minlen 10 -power 1.0 wmfod_template.mif -seed_image template_mask.mif -mask template_mask.mif -select 20000000 -cutoff 0.06 tracks_20_million.tck

tcksift tracks_20_million.tck wmfod_template.mif tracks_2_million_sift.tck -term_number 2000000

fixelconnectivity fixel_mask/ tracks_2_million_sift.tck matrix/

fixelfilter fd smooth fd_smooth -matrix matrix/
fixelfilter log_fc smooth log_fc_smooth -matrix matrix/
fixelfilter fdc smooth fdc_smooth -matrix matrix/

```
{{% /tab %}}
{{< /tabs >}}


{{% notice warning %}}
在Macintosh操作系统上，命令`ln -sr`可能不起作用（在大多数Linux系统上应该起作用）。在这种情况下，将`wmfod*.mif`文件复制到模板`/fod_input`文件夹中，并将`dvi_mask_upsampled*.mif`文件复制到模板`/mask_input`文件夹中.
{{% /notice %}}


{{% notice note %}}
有时`dwi2mask`命令可能无法覆盖整个大脑，特别是脑脊液袋部分(especially pockets of cerebrospinal fluid)。在这种情况下，你可以用FSL的`bet2`命令代替`dwi2mask`命令，这需要将掩膜转换为NIFTI格式，然后再转换回.mif格式：

```
mrconvert -force dwi_denoised_unringed_preproc_upsampled.mif tmp.nii
bet2 tmp.nii tmp -m -f 0.2
mrconvert -force tmp_mask.nii.gz dwi_mask_upsampled.mif
rm tmp*
```
请确保像我们想前几个教程中那样检查掩膜（mask），以确保掩膜上没有漏洞。你可能需要改变-f选项后的数值，以生成一个覆盖大脑所有体素的良好的全脑屏蔽。使用这种方法可以解决`mtnormalise`的任何 "平衡因子(balance factor)"错误，特别是在一种或多种组织类型为空的情况下。
{{% /notice %}}


#### 创建GLM模型
教程中的最后几步需要一个设计矩阵和对比矩阵，类似于用FSL创建的那些矩阵（例子见本页）。对于一个在所有参与者中平均的简单效应，基于Fixel的指标，我们将创建一个全部为1的列，如以下所示:

{{< tabs >}}
{{% tab name="design_matrix.txt" %}}
```
1
1
1
```
{{% /tab %}}

{{% tab name="contrast_matrix.txt" %}}
```
1
```
{{% /tab %}}

{{< /tabs >}}

并将其保存到一个名为`contrast_matrix.txt`的文件中，在这个例子中。上述文本文件规定，我们对所有的受试者进行加权处理，并给他们一个对比度值1，以平均他们对每个fixel的所有数值。然后，我们可以对每个基于fixel的指标--FD、FC和FDC--进行这些简单的效果分析：


```shell
fixelcfestats fd_smooth/ files.txt design_matrix.txt contrast_matrix.txt matrix/ stats_fd/ 
fixelcfestats log_fc_smooth/ files.txt design_matrix.txt contrast_matrix.txt matrix/ stats_log_fc/ 
fixelcfestats fdc_smooth/ files.txt design_matrix.txt contrast_matrix.txt matrix/ stats_fdc/
```

这将生成每个指标对应的`stats`文件夹，每个文件夹都包含代表不同统计数据的文件。例如，我们可以在`mrview`中加载文件`wmfod_template.mif`作为底层，然后点击`Tool ->Fixel plot`来加载，例如`Zstat.mif`文件，点击Fixel绘图面板顶部的文件夹`icron`，导航到`stats_fdc`，然后选择文件`Zstat.mif`。你应该看到类似这样的东西：
![Zstat_FDC](/mrtrix/images/08_FDC_Directions.png)

默认情况下，观察面板中会有一个色标条，显示该图像中的最小和最大Z统计量；在我们的例子中，最大Z统计量为5.33，表明FDC值的Z统计量最高。如果我们放大到一个区域，如左上纵筋，我们可以看到每个fixel由三个正交方向组成，当我们沿着不同的纤维束移动时，方向会发生变化，fixel的每个向量按其强度进行颜色编码：

![FDC_Direction](/mrtrix/images/08_FDC_Directions.png)

你也可以加载文件`fwe_1mpvalue.mif`，它将显示一个有意义的fixel的1-p图，可以以0.95为阈值，只显示那些通过p=0.05的意义阈值的fixel。鉴于我们只有三个受试者，我们不太可能有任何明显的固定点，而且无论如何，它们对简单的效应分析没有什么意义。另一方面，为了观察组间的对比，我们将在一个计算集群上分析整个数据集。



#### [Fixel-Based Analysis on the Supercomputing Cluster](https://andysbrainbook.readthedocs.io/en/latest/MRtrix/MRtrix_Course/MRtrix_11_FixelBasedAnalysis.html#fixel-based-analysis-on-the-supercomputing-cluster)


#### 参考资料
- https://mrtrix.readthedocs.io/en/latest/fixel_based_analysis/mt_fibre_density_cross-section.html
- https://andysbrainbook.readthedocs.io/en/latest/MRtrix/MRtrix_Course/MRtrix_11_FixelBasedAnalysis.html
- [Fixel based analysis of white matter alterations in early stage cerebral small vessel disease](https://www.nature.com/articles/s41598-022-05665-2)


