---
layout: post
title: cgroup 实验 
---
# cgroup 实验 

## 安装cgroup
在 centOS 6.5 上使用如下命令安装:
```
yum install cgroup
```

## 启动cgroup
```
service cgconfig start   #开启cgroups服务
chkconfig cgconfig on    #开机启动
```

## 创建控制组
```
mkdir -p /cgroup/cpu/limit_user
mkdir -p /cgroup/memory/limit_user
```

## 设置控制条件
```
echo 50000 > /cgroup/cpu/limit_user/cpu.cfs_quota_us
```
# TODO 需要再次理解　
以上的命令是指此控制组只可以使用50%的CPU。*cfs_period_us*表示控制组可以访问CPU的时间段,并微秒为单位，   
*cfs_quota_us*表示控制组在执行时间中的配额。如果让一个cgroup中的task可以执行0.2秒，那么就需要设置  
*cfs_quota_us*为200000 和 *cpu.cfs_period_us* 为 1000000.

```
echo 1048576 >  /cgroup/memory/limit_user/memory.limit_in_bytes
```
分配1MB的内存给cgroup

## 测试脚本:   
### 脚本一消耗CPU    
```
#! /bin/bash

x=0
while [ True ];do
    x=$x+1
done;
```
执行效果：   

![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-09-28-cgroup-test-1.jpg)

如上图所示CPU占用99%，执行以下命令：   
echo 30036 > /cgroup/cpu/limit_user/tasks
可以看到CPU的占用会被限制到50%。   

![](https://raw.githubusercontent.com/lxlenovostar/lix_blog/gh-pages/images/2016-09-28-cgroup-test-2.jpg)

### 脚本二消耗内存    
```
#! /bin/bash

echo $$ > /cgroup/memory/foo/tasks     #脚本主动将自身进程加入到cgroup

x="a"
while [ True ];do
    x=$x$x
done;
```
执行此脚本，当此进程试图占用的内存超过了cgroups的限制，会触发out of memory，导致进程被kill掉。

{% include references.md %}
