---
title: "FreeSurfer教程 #3. 组分析"
date: 2023-02-06T17:02:06+08:00
draft: true
---
- [知识点汇总](#知识点汇总)
- [背景介绍](#背景介绍)
- [构建FSGD和对比文件](#构建fsgd和对比文件)
- [基于`recon-all`和`mris_preproc`完成皮层重建和数据合并](#基于recon-all和mris_preproc完成皮层重建和数据合并)
- [输出结果查看](#输出结果查看)
- [参考资料](#参考资料)


#### 知识点汇总
1. 构建FSGD文件(*.fsgd)
2. 构建对比文件(*.mtx)
3. 构建广义线性模型

#### 背景介绍
组分析准备工作:
- 首先，使用`recon-all`对数据进行预处理，在每个顶点计算不同的结构测量值;
- 其次，我们创建了一个**FSGD文件**(\*.fsgd)和一个对比文件(*.mtx)，表明我们想要相互比较哪些组。


#### 构建FSGD和对比文件

**1.构建FSGD(FreeSurfer Group Descriptor file)文件**
FSGD格式使用标签（tags）来识别信息，如下图所示:
{{< tabs >}}
{{% tab name="FSGD format" %}}
```
  Example of a legal file:
  ------------------------- cut here ------------------
  GroupDescriptorFile 1
  Title MyTitle
  Class Class1 plus blue
  Class Class2 circle green
  Variables             Age  Weight   IQ
  Input subjid1 Class1   10    100   1000
  Input subjid2 Class2   20    200   2000
  #Input subjid3 Class2   20    200   2000
  DefaultVariable Age
  ------------------------- cut here ------------------
```
{{% /tab %}}
{{< /tabs >}}

**注意**：
- 文件首行必须是` GroupDescriptorFile 1`
- `Title`不是必须的，仅用于展示；
- `Class`仅需要类名称，后续的两个仅用于展示；
- `Input`标签
- `#` 用于注释
- `DefaultVariable`用于默认的变量（variables）展示，且必须是Variables的列变量（比如这里的Age\Weight\IQ）；

**一般规则**：
- `Tags`标签不区分大小写，`Class`和`CLASS`是同一个标签；
- `Lables`是区分大小写的；
- 使用空格来分隔；
- 被试ID不能重复；
- [更多详见官网](https://surfer.nmr.mgh.harvard.edu/fswiki/FsTutorial/CreateFsgdFile)

**2.构建对比文件(*.mtx,stands for contrast matrix)**
创建一个对比文件的作用是为我们模型中的每个回归量指定对比权重（contrast weights）。
举例，HC表示正常受试组，CB表示异常受试组
{{< tabs >}}
{{% tab name="HC-CB.mtx" %}}
```
1 -1
```
{{% /tab %}}

{{% tab name="CB-HC.mtx" %}}
```
-1 1
```
{{% /tab %}}
{{< /tabs >}}

- `CB-HC.mtx`表示CB组减去HC组。
- `HC-CB.mtx`表示HC组减去BC组。


- [更多详见](https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_07_FSGD.html)
- [更多详见2](https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_10_CorrelationAnalysis.html#fs-10-correlationanalysis)
- [官网FSGD文件参考](https://surfer.nmr.mgh.harvard.edu/fswiki/FsgdExamples)


#### 基于`recon-all`和`mris_preproc`完成皮层重建和数据合并

![Example output from qcache](/freesurfer/images/07_qcache_output.webp?width=60pc)

建议在运行`recon-all`时使用`qcache`选项。这将以几种不同的平滑尺寸(FWHM)生成厚度(thickness)、体积(volume)和曲率图(curvature)，例如0mm、10mm和25mm全宽半最大内核。

{{% notice warning %}}
在组分析之前，需要决定使用哪个平滑核（FWHM），然后在后续使用时保持一致。
{{% /notice %}}


![mrispreproc_concatenation](/freesurfer/images/08_mrispreproc_concatenation.gif?width=60pc)

- 为了进行组分析，我们需要将所有个体的结构图（individual structural maps）合并到一个数据集中。
- 该数据也被重新采样到fsaverage模板，该模板位于MNI空间。每当我们进行任何类型的组分析(比较组、兴趣区域分析等等)时，每个被试的数据必须具有相同的维度和体素分辨率。
  
通过`mris_preproc`来完成上述步骤：
{{< tabs >}}
{{% tab name="shell" %}}
```
#!/bin/tcsh

setenv study $argv[1]

foreach hemi (lh rh)  # 遍历lh rh，即左半球和右半球都完成下述计算。
  foreach smoothing (10) # FWHM的平滑尺寸设置为10mm
    foreach meas (volume thickness) # 遍历计算volume thichkness
      mris_preproc --fsgd FSGD/{$study}.fsgd \
        --cache-in {$meas}.fwhm{$smoothing}.fsaverage \
        --target fsaverage \
        --hemi {$hemi} \
        --out {$hemi}.{$meas}.{$study}.{$smoothing}.mgh
    end
  end
end

```
{{% /tab %}}
{{< /tabs >}}

- An FSGD file (indicated by the `--fsgd` option);
- A template to resample to (`--target`);
- An indication of which hemisphere to resample (`--hemi`);
- A label for the output file (`--out`).
- `--cache-in` option to specify which smoothed images we want to use in the analysis.



{{% notice note %}}
如果在`recall`期间没有使用`-qcache`选项，你仍然可以平滑数据，而不必重新运行所有的预处理步骤;例如,
```
recon-all -s <subject name> -qcache
```

指定一个特定的平滑核，可使用`-fwhm`参数，例如：
```
recon-all -s <subject name> -qcache -fwhm 10```

{{% /notice %}}


#### 基于`mri_glmfit`的广义线性模型
参数选项如下：
- `--y`:The concatenated dataset containing all of the subjects’ structural maps;  
- `--fsgd`: The FSGD file;
- `--C`:A list of contrasts (each contrast specified by a different line containing );
-`--surf`: The hemisphere of the template to analyze;
- `--cortex`:A mask to restrict our analysis only to the cortex;
- `--glmdir`:An output label for the directory containing the results.

{{< tabs >}}
{{% tab name="shell" %}}
```Bash
#!/bin/tcsh

set study = $argv[1]

foreach hemi (lh rh)
  foreach smoothness (10)
    foreach meas (volume thickness)
        mri_glmfit \
        --y {$hemi}.{$meas}.{$study}.{$smoothness}.mgh \
        --fsgd FSGD/{$study}.fsgd \
        --C Contrasts/CB-HC.mtx \
        --C Contrasts/HC-CB.mtx \
        --surf fsaverage {$hemi}  \
        --cortex  \
        --glmdir {$hemi}.{$meas}.{$study}.{$smoothness}.glmdir
    end
  end
end
```
{{% /tab %}}
{{< /tabs >}}


#### 输出结果查看
如果脚本运行成功后，可在目录下查看到下述相关文件夹：
```
lh.thickness.CannabisStudy.10.glmdir
lh.volume.CannabisStudy.10.glmdir
rh.thickness.CannabisStudy.10.glmdir
rh.volume.CannabisStudy.10.glmdir
```

进入其中一个，比如`lh.volume.CannabisStudy.10.glmdir`,我们可以看到下述文件:
```
CB-HC  
HC-CB 
X.mat  
beta.mgh  
fwhm.dat  
mri_glmfit.log  
rvar.mgh  
surface
Xg.dat  
dof.dat  
mask.mgh  
rstd.mgh  
sari.mgh  
y.fsgd
```
- 目录CB-HC和HC-CB包含mri_glmfit中指定的每种对比的对比数据。
- y.fsgd是用于运行分析的FSGD文件的副本;
- mri_glmfit.log包含为当前分析运行的代码的日志;
- mask.mgh是用于分析的掩模;
- beta.mgh是由分析创建的个体beta权重的级联数据集。
[可以通过输入mri_glmfit并查看命令行参数下的部分来阅读其他输出的描述。](https://surfer.nmr.mgh.harvard.edu/fswiki/mri_glmfit)


{{% notice note %}}
FreeSurfer使用-log10(p)表示法;换句话说，sig.mgh映射中的值1表示p值为0.1，值2表示p值为0.01，依此类推。
{{% /notice %}}

#### 参考资料
- https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_08_GroupAnalysis.html
- https://surfer.nmr.mgh.harvard.edu/fswiki/FsTutorial/GroupAnalysis
- [FSGD文件](https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_07_FSGD.html)



