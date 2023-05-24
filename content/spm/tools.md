---
title: "Tools"
date: 2023-02-07T12:54:02+08:00
draft: true
---

#### spm_vol读取nii的头信息
`spm_vol('your nii file')` ,Nifti格式可以为*.nii或者*.nii.gz（会自动解压）  
**返回**：文件名、影像维度信息; 
```
header = 

  struct with fields:

      fname: '/Datasets/flanker_task/sub-01/anat/sub-01_T1w.nii'
        dim: [176 256 256]
         dt: [4 0]
      pinfo: [3×1 double]
        mat: [4×4 double]
          n: [1 1]
    descrip: ''
    private: [1×1 nifti]
```

