---
title: "梯度和球体"
date: 2023-01-13T14:21:16+08:00
draft: true
pre: <b>1. </b>
weight: 1
---
#### 介绍
我们可从磁盘上加载你的b-values和b-vectors，然后可以创建自己的梯度表(gradient table)。但是这次我们假设我们是一个磁共振物理学家，我们想设计一个新的梯度方案，或者我们是一个想模拟多种不同梯度方案的科学家。

本例展示了如何使用DIPY创建梯度表和球体对象，创建一个multi-shell acquisition2-shells）,其中一个是$b=1000 s/mm^{2}$,另外一个是$b=2500 s/mm^{2}$。对于这两个shells，假设我们想要一个特定数量的梯度（64），并且我们想让球体上的点均匀地分布。




<!-- {{<jupyter dipy dipy_proc_gradients 1724>}} -->

#### 官网教程地址：
https://dipy.org/documentation/1.5.0/examples_built/gradients_spheres/#example-gradients-spheres
