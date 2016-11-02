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

### 规则1   
同一个hierarchy可以附加一个或多个subsystem。如下图，cpu和memory的subsystem附加到了一个hierarchy。      
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-11-02-understand-cgroup-1.png)   

### 规则2
一个subsystem可以附加到多个hierarchy，当且仅当这些hierarchy只有这唯一一个subsystem。如下图，小圈中的   
数字表示subsystem附加的时间顺序，CPU subsystem附加到hierarchy A的同时不能再附加到hierarchy B，因为   
hierarchy B已经附加了memory subsystem。如果hierarchy B与hierarchy A状态相同，没有附加过memory subsystem，   
那么CPU subsystem同时附加到两个hierarchy是可以的。   
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-11-02-understand-cgroup-2.png)   

### 规则3 
系统每次新建一个hierarchy时，该系统上的所有task默认构成了这个新建的hierarchy的初始化cgroup，这个cgroup   
也称为root cgroup。对于你创建的每个hierarchy，task只能存在于其中一个cgroup中，即一个task不能存在于同一个   
hierarchy的不同cgroup中，但是一个task可以存在在不同hierarchy中的多个cgroup中。如果操作时把一个task添加到    
同一个hierarchy中的另一个cgroup中，则会从第一个cgroup中移除。在下图中可以看到，httpd进程已经加入到hierarchy A     
中的/cg1而不能加入同一个hierarchy中的/cg2，但是可以加入hierarchy B中的/cg3。实际上不允许让任务加入同一个   
hierarchy中的多个cgroup是为了防止出现矛盾，如CPU subsystem为/cg1分配了30%，而为/cg2分配了50%，此时如果httpd   
在这两个cgroup中，就会出现矛盾。   
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-11-02-understand-cgroup-3.png)   

### 规则4   
进程（task）在fork自身时创建的子任务（child task）默认与原task在同一个cgroup中，但是child task允许被移动   
到不同的cgroup中。即fork完成后，父子进程间是完全独立的。如下图中，小圈中的数字表示task 出现的时间顺序，   
当httpd刚fork出另一个httpd时，在同一个hierarchy中的同一个cgroup中。但是随后如果PID为4840的httpd需要移动   
到其他cgroup也是可以的，因为父子任务间已经独立。总结起来就是：初始化时子任务与父任务在同一个cgroup，但是    
这种关系随后可以改变。   
![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-11-02-understand-cgroup-4.png)   

### cgroup实现
cgroups的实现本质上是给系统进程挂上钩子（hooks），当task运行的过程中涉及到某个资源时就会触发钩子上所附带
的subsystem进行检测，最终根据资源类别的不同使用对应的技术进行资源限制和优先级分配。   
![cgroup相关结构体](http://cdn2.infoqstatic.com/statics_s1_20161025-0357u2/resource/articles/docker-kernel-knowledge-cgroups-resource-isolation/zh/resources/0329014.png)   
