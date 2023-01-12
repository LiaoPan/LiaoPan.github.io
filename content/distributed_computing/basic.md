---
title: "Basic"
date: 2023-01-08T21:11:11+08:00
draft: true
---

### [SLURM](https://slurm.schedmd.com/documentation.html)

SLURM （Simple Linux Utility for Resource Management）是一种可用于大型计算节点集群的高度可伸缩和容错的集群管理器和作业调度系统，被世界范围内的超级计算机和计算集群广泛采用。SLURM 维护着一个待处理工作的队列并管理此工作的整体资源利用。它以一种共享或非共享的方式管理可用的计算节点（取决于资源的需求），以供用户执行工作。SLURM 会为任务队列合理地分配资源，并监视作业至其完成。如今，SLURM 已经成为了很多最强大的超级计算机上使用的领先资源管理器，如天河二号上便使用了 SLURM 资源管理系统。


1. https://slurm.schedmd.com/documentation.html
2. https://docs.hpc.sjtu.edu.cn/job/slurm.html
3. https://docs.slurm.cn/master/quick-start-user-guide.-kuai-su-ru-men-yong-hu-zhi-nan


算法运行环境提供：
若每台计算节点部署相关软件运行环境，会相对繁琐难配置，且在后期使用或者维护中会容易被破坏，故建议使用singularity或者docker技术来封装整个运行环境。
相关的数据盘访问可使用NFS技术（Network File System，网络文件系统）来共享数据，让每个计算节点都可以方便访问到数据。当然，类似的技术还有GlusterFS等。

#### [Singularity](https://sylabs.io/singularity/)
基于 Singularity 的高性能计算容器技术，相比Docker等在云计算环境中使用的容器技术，Singularity 同时支持root用户和非root用户启动，且容器启动前后，用户上下文保持不变，这使得用户权限在容器内部和外部都是相同的。 此外，Singularity 强调容器服务的便捷性、可移植性和可扩展性，而弱化了容器进程的高度隔离性，因此量级更轻，内核namespace更少，性能损失更小。

#### [docker]()







### SGE


