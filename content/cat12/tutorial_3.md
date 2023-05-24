---
title: "CAT12 Tutorial#3:使用脚本运行CAT12"
date: 2023-05-09T17:31:18+08:00
draft: true
weight: 3
---

#### 1.使用Batch Editor导出代码脚本
与SPM导出脚本是一样的操作。

#### 2. 无界面运行代码脚本
{{< tabs >}}
{{% tab name="run_matlab_cat12.sh" %}}
```
#!/bin/bash
set -e
# <your matlab bin path> -nodesktop -noFigureWindows -nosplash -r "<matlab code or M文件名称(注意不能带.m后缀)>>"

'/matlab/R2020a/bin/matlab' -nodisplay -nodesktop -noFigureWindows -nosplash -r "addpath '~/spm12'; <matlab code>; exit"
echo "Task finished without errors"
```

{{% /tab %}}
{{< /tabs >}}

如图所示，展示了在终端运行`run_matlab_cat12.sh`的日志。
![run_script](/cat12/images/4.run_script.png)