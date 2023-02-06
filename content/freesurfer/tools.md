---
title: "FreeSurfer教程 #x. 命令行汇总"
date: 2023-02-02T13:56:37+08:00
draft: true
weight: 10
---

- [`dcmunpack `](#dcmunpack-)
- [`recon-all`](#recon-all)
- [**`recon-all`的批处理脚本**](#recon-all的批处理脚本)
- [**`recon-all`的并行处理（Parallel）**](#recon-all的并行处理parallel)
    - [**`recon-all`的输出结果**](#recon-all的输出结果)
- [参考资料](#参考资料)


#### `dcmunpack `

`dcmunpack `:创建一个文本文件，其中显示扫描时的名称(正如您在扫描仪上所命名的那样)以及每次扫描所对应的dicom序列。
- 借此，我们可以知道dicom数据的模态（结构像、功能像等）
  
```shell
$ dcmunpack -src <dicom dir> -scanonly <text file with a list of scans>

For example

$ dcmunpack -src . -scanonly scan.log
$ cat scan.log
       ...
7      T1w_setter  ok   32  32  32   1 mri00152.dcm
8      T1w_MPR_vNav_4e err  192 192  23   1 mri00305.dcm
       ...
57     T2w_SPC_HCP_700um  ok  320 320 256   1 mri14468.dcm

```
-----


#### `recon-all`

`recon-all`： 皮层重建,从T1加权解剖图像中创建了一系列体积(volumes)和表面(surfaces)，并量化了大脑不同区域的灰质厚度和体积。
![Rescontruction](/freesurfer/images/01_reconstruction.webp)
- 将三维的解剖像（anatomical volume）转换到二维的皮层。

在使用FreeSurfer来处理进行皮层重建时，首先要确保`SUBJECTS_DIR`的变量已被设置，该变量指定了FreeSurfer的输出，并确保有足够的存储空间，官网推荐300MB以上。
通过`echo $SUBJECTS_DIR`查看变量是否已正常设置，若没有设置，其设置方法如下:
```shell
$ export SUBJECTS_DIR=<set_your_data_path>
注意：<set_your_data_path>表示需要替换的内容。
```
使用`recon-all`：
```shell
$ recon-all -all -i <one slice in the anatomical dicom series> -s  <subject id that you make up>
$ recon-all -all -i I50 -s  Subj001
```
`-all`表示跑完FreeSurfer的所有处理步骤。
`-i`表示指向数据输入的一个dicom文件（软件会自动检索到剩余的其他dicom文件），也可以将nifti作为输入。推荐最好全路径。
`-s`表示指定被试的名称或者ID，软件会根据该名称创建一个输出文件夹。
`-qcache`表示会在不同层级平滑数据，然后存在subject的输出目录中，方便后续的组分析。

`recon-all`成功跑完之后，我们可以在输出目录下发现`recon-all.log`日志文件，记录了算法运行的日志，若在最后出现` "recon-all exited without errors"`即表示算法运行成功并正常退出。

#### **`recon-all`的批处理脚本**
{{< tabs >}}
{{% tab name="Tcsh" %}}
```bash
#!/bin/tcsh
set s = $1
setenv SUBJECTS_DIR /path/to/your/data
set log = $SUBJECTS_DIR/recon-all-commands.log
set dcmdir = /path/to/your/dicoms
set subjid = echo $s |gawk -F- '{print $2}'

if (-e $dcmdir/$s/scan.log) then

 . echo "found scan.log, finding mprages" set dat = $dcmdir/$s/scan.log

else

 . echo "no scan.log"

endif

set mpr = (cat $dat | grep "256 256 128" |grep ok | awk '{print  $8}') echo "found mprages, $mpr"

echo recon-all -i $dcmdir/$s/$mpr -all -s $subjid >> $log
```
{{% /tab %}}

{{% tab name="Simple Script" %}}

```bash
cd your_study_group_data
setenv SUBJECTS_DIR $PWD
foreach i (*)
echo $i
recon-all -subjid $i -all
end
```
{{% /tab %}}
{{< /tabs >}}


[详见](https://surfer.nmr.mgh.harvard.edu/fswiki/FsTutorial/Scripts#BatchProcessingwithrecon-all)



#### **`recon-all`的并行处理（Parallel）**
![Parallel recon-all](/freesurfer/images/04_recon_parallel.webp)
为了加速`recon-all`的计算，我们可以使用[`parallel`](https://www.gnu.org/software/parallel/parallel_tutorial.html)命令来针对不同被试并行计算，以此达到解压时间的目的。


{{< tabs >}}
{{% tab name="Parallel recon-all" %}}
```Bash
$ export SUBJECTS_DIR=`pwd`
$ ls *.nii | parallel --jobs 8 recon-all -s {.} -i {} -all -qcache
```
{{% /tab %}}
{{< /tabs >}}
`{.}`表示去掉.nii的后缀；**扩展**：{.}，去掉扩展名；{/},去掉路径，只保留文件名；{//}，只保留路径；{/.}，同时去掉路径和扩展名；{#}，输出任务编号；**注意这些都是parallel的语法规则，建议自行搜索学习**；


[详见](https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_04_ReconAllParallel.html)





###### **`recon-all`的输出结果**

![recon-all results](/freesurfer/images/02_orig_white_pial.webp)
- 白质和灰质的交界边缘(黄色),更精细的交界边缘结果（蓝色），然后，这个精确的估计被用于协助检测灰质(红色)的交界边缘。

1. `recon-all`首先对解剖图像进行颅骨去除，得到**brainmask.mgz**(*.mgz为FreeSurfer软件的特定压缩格式，Massachusetts General Hospital)。所有的三维volumes数据都放在**mri**目录下。
2. `recon-all`估计两个半球白质和灰质之间的交界，这些表面估计被存储在名为**lh.orig**和**rh.orig**文件中。优化后的结果放在**lh.white**和**rh.white**文件中。
3. 然后将此边界用作基础，`recon-all`从该基础延伸触角以搜索灰质的边缘,到达此边缘后，将创建第三对数据集：**lh.pial** 和 **rh.pial**。 这些数据集代表软脑膜表面，就像包裹在灰质边缘的塑料薄膜。

![pial inflated](/freesurfer/images/03_pial_Inflated.webp)
- 将lh.pial转换到lh.inflated示意图

-----


{{%notice tip%}}
若想知道每个命令更多的使用方式，可以`<freesurfer_cmd> -help` 来查看更多的使用方法。比如`dcmunpack -help`
{{%/notice%}}



[更多细节，请参考官网](https://surfer.nmr.mgh.harvard.edu/fswiki/FsTutorial/PracticeV6.0)

#### 参考资料
1. [FreeSurfer命令行官网汇总](https://surfer.nmr.mgh.harvard.edu/fswiki/FreeSurferCommands)
2. [AndyBrainBook](https://andysbrainbook.readthedocs.io/en/latest/FreeSurfer/FS_ShortCourse/FS_03_ReconAll.html)
