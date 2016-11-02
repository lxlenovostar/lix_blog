---
layout: default
title: 理解cgroup实现 
---
# 理解cgroup实现

## cgroup 概述
cgroups是Linux内核提供的一种机制，这种机制可以根据特定的行为，把一系列系统任务及其子任务整合（或分隔）   
到按资源划分等级的不同组内，从而为系统资源管理提供一个统一的框架。    
实现cgroups的主要目的是为不同用户层面的资源管理，提供一个统一化的接口。从单个进程的资源控制到操作系统   
层面的虚拟化。Cgroups提供了以下四大功能。
1. 资源限制（Resource Limitation）：cgroups可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，   
一旦超过这个配额就发出OOM（Out of Memory）。
2. 优先级分配（Prioritization）：通过分配的CPU时间片数量及硬盘IO带宽大小，实际上就相当于控制了进程运行的优先级。
3. 资源统计（Accounting）： cgroups可以统计系统的资源使用量，如CPU使用时长、内存用量等等，这个功能非常适用于计费。
4. 进程控制（Control）：cgroups可以对进程组执行挂起、恢复等操作。

## cgroup 术语表
1. task（任务）：cgroups的术语中，task就表示系统的一个进程。
2. cgroup（控制组）：cgroups 中的资源控制都以cgroup为单位实现。cgroup表示按某种资源控制标准划分而成的任务组，   
包含一个或多个子系统。一个任务可以加入某个cgroup，也可以从某个cgroup迁移到另外一个cgroup。
3. subsystem（子系统）：cgroups中的subsystem就是一个资源调度控制器（Resource Controller）。比如CPU子系统可以   
控制CPU时间分配，内存子系统可以限制cgroup内存使用量。
4. hierarchy（层级树）：hierarchy由一系列cgroup以一个树状结构排列而成，每个hierarchy通过绑定对应的subsystem   
进行资源调度。hierarchy中的cgroup节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个hierarchy。

## cgroup的基本规则
大家在namespace技术的讲解中已经了解到，传统的Unix进程管理，实际上是先启动init进程作为根节点，   
再由init节点创建子进程作为子节点，而每个子节点由可以创建新的子节点，如此往复，形成一个树状结构。    
而cgroups也是类似的树状结构，子节点都从父节点继承属性。它们最大的不同在于，系统中cgroup构成的   
hierarchy可以允许存在多个。如果进程模型是由init作为根节点构成的一棵树的话，那么cgroups的模型则是    
由多个hierarchy构成的森林。这样做的目的也很好理解，如果只有一个hierarchy，那么所有的task都要受到   
绑定其上的subsystem的限制，会给那些不需要这些限制的task造成麻烦。   






