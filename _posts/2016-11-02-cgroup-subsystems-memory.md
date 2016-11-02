---
layout: default
title: cgroup 内存子系统 
---
# cgroup 内存子系统 

## 内存按用途分类
1. 用户空间的匿名映射页（Anonymous pages in User Mode address spaces），比如调用malloc分配的内存，   
以及使用MAP_ANONYMOUS的mmap；当系统内存不够时，内核可以将这部分内存交换出去。   
2. 用户空间的文件映射页（Mapped pages in User Mode address spaces），包含map file和map tmpfs；   
前者比如指定文件的mmap，后者比如IPC共享内存；当系统内存不够时，内核可以回收这些页，但回收之前   
可能需要与文件同步数据。   
3. 文件缓存（page in page cache of disk file）；发生在程序通过普通的read/write读写文件时，当系统   
内存不够时，内核可以回收这些页，但回收之前可能需要与文件同步数据。   
4. buffer pages，属于page cache；比如读取块设备文件。   
*其中1和2是算作进程的RSS，3和4属于page cache。*

## 内存子系统的作用 
1. Accounting memory usage under cgroup, Memory here is physical memory.   
2. Limit memory usage under user specified value.   
3. If necessary, cull(pageout) memory under it.   

## 内存子系统的内核实现
memory子系统是通过linux的resource counter机制实现的。resource counter是内核为子系统提供的一种   
资源管理机制。这个机制的实现包括了用于记录资源的数据结构和相关函数。Resource counter定义了一个   
res_counter的结构体来管理特定资源，定义如下：   
```
struct res_counter {   
	//Usage 用于记录当前已使用的资源 
	unsigned long long usage;
	//max_usage 用于记录使用过的最大资源量 
	unsigned long long max_usage;
	//limit 用于设置资源的使用上限，进程组不能使用超过这个限制的资源 
	unsigned long long limit;
	//soft_limit 用于设定一个软上限，进程组使用的资源可以超过这个限制 
	unsigned long long soft_limit;
	//failcnt 用于记录资源分配失败的次数，管理可以根据这个记录，调整上限值。 
	unsigned long long failcnt; 
	spinlock_t lock;
	//Parent 指向父节点，这个变量用于处理层次性的资源管理。
	struct res_counter *parent;
};
```
相关函数：
```
void res_counter_init(struct res_counter *counter, struct res_counter *parent)
int res_counter_charge(struct res_counter *counter, unsigned long val,struct res_counter **limit_fail_at)
void res_counter_uncharge(struct res_counter *counter, unsigned long val)
```
当资源将要被分配的时候，资源就要被记录到相应的res_counter里。这个函数作用就是记录进程组使用的资源。   
在这个函数中有:   
```
for (c = counter; c != NULL; c = c->parent) {
	spin_lock(&c->lock);
	ret = res_counter_charge_locked(c, val);
	spin_unlock(&c->lock);
	if (ret < 0) {
		*limit_fail_at = c;
		goto undo;
	}
}
```
在这个循环里，从当前res_counter开始，从下往上逐层增加资源的使用量。我们来看一下   
res_counter_charge_locked这个函数，这个函数顾名思义就是在加锁的情况下增加使用   
量。实现如下:   
```
    if (counter->usage + val > counter->limit) {
        counter->failcnt++;
        ret = -ENOMEM;
        if (!force)
            return ret;
    }   

    counter->usage += val;
    if (counter->usage > counter->max_usage)
        counter->max_usage = counter->usage;
```
首先判断是否已经超过使用上限，如果是的话就增加失败次数，返回相关代码；否则就增加   
使用量的值，如果这个值已经超过历史最大值，则更新最大值。   

## Charge/Uncharge
Memory cgroup accounts usage of memory. There are roughly 2 operations, charge/uncharge.   
1. Charge
- (Memory) Usage += PAGE_SIZE
- Free/cull memory if usage hit limits
- Check a page as “This page is charged”
 
2. Uncharge
- (Memory) Usage -= PAGE_SIZE
- Remove the check

## LRU
## Performance

# 需要理解的问题：
1. 内存子系统如何和进程联系起来？ 
2. 内存子系统中的LRU有什么用？ 
3. charge在具体内存分配中的应用举例？参考github的博客。
4. res_counter 和内存子系统的联系？ 
5. 内核/proc统计内存的方法？
6. 命名空间 和 如何实现Docker ?  
