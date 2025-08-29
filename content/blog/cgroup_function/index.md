---
title: 🎉 Cgroup | 学习笔记
summary: Cgroup v2 各个控制器的作用
date: 2025-08-29

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
  - Cgroup
---

```shell
xz@xz:/sys/fs/cgroup/system.slice$ cat cgroup.controllers
cpuset cpu io memory hugetlb pids rdma
/sys/fs/cgroup # ls
cgroup.controllers        cpu.uclamp.min            hugetlb.2MB.max           memory.pressure
cgroup.events             cpu.weight                hugetlb.2MB.numa_stat     memory.reclaim
cgroup.freeze             cpu.weight.nice           hugetlb.2MB.rsvd.current  memory.stat
cgroup.kill               cpuset.cpus               hugetlb.2MB.rsvd.max      memory.swap.current
cgroup.max.depth          cpuset.cpus.effective     io.max                    memory.swap.events
cgroup.max.descendants    cpuset.cpus.partition     io.pressure               memory.swap.high
cgroup.pressure           cpuset.mems               io.stat                   memory.swap.max
cgroup.procs              cpuset.mems.effective     io.weight                 memory.swap.peak
cgroup.stat               hugetlb.1GB.current       memory.current            memory.zswap.current
cgroup.subtree_control    hugetlb.1GB.events        memory.events             memory.zswap.max
cgroup.threads            hugetlb.1GB.events.local  memory.events.local       pids.current
cgroup.type               hugetlb.1GB.max           memory.high               pids.events
cpu.idle                  hugetlb.1GB.numa_stat     memory.low                pids.max
cpu.max                   hugetlb.1GB.rsvd.current  memory.max                pids.peak
cpu.max.burst             hugetlb.1GB.rsvd.max      memory.min                rdma.current
cpu.pressure              hugetlb.2MB.current       memory.numa_stat          rdma.max
cpu.stat                  hugetlb.2MB.events        memory.oom.group
cpu.uclamp.max            hugetlb.2MB.events.local  memory.peak
```

### cpuset:

主要用于指定在哪个CPU上

### cpu:

指定一个CPU占用多少

### hugetlb

限制使用巨页的数量

### io

### 速率突破。

进程在退出的时候会取消提交或者尽快完成所有的IO请求。

## memory

可以利用

### pid

pid本身收到两个方面的限制：

```shell
xz@xz:/sys/fs/cgroup/system.slice$ ulimit -a   #非容器中
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 29555
max locked memory       (kbytes, -l) 65536
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024  
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 29555   #这里存在限制，但是默认情况下，
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited

/sys/fs/cgroup # ulimit -a   #容器中
-f: file size (blocks)             unlimited
-t: cpu time (seconds)             unlimited
-d: data seg size (kb)             unlimited
-s: stack size (kb)                8192
-c: core file size (blocks)        unlimited
-m: resident set size (kb)         unlimited
-l: locked memory (kb)             65536
-p: processes                      unlimited
-n: file descriptors               1048576
-v: address space (kb)             unlimited
-w: locks                          unlimited
-e: scheduling priority            0
-r: real-time priority             0
以及cgroup中的pid，
```

一、想办法直接通过cgroup迁移：

问题：

1. 得看到别的进程的pid。
2. /sys目录是只读挂载的，无法写入，即使共享pid namesapce，能够看到别的进程了，但是依旧无法写入。

这条路除非能够解决/sys非只读挂载。

二、pod中是否存在类似的功能：

目前没有发现。

## rdma

rdma并不存在namesapce隔离，只是说可以将这部分资源消耗完

RDMA 控制器可以核算以下资源。

- hca_handle
  - HCA 句柄的最大数量
- hca_object
  - HCA 对象的最大数量

```c
int ib_rdmacg_try_charge(struct ib_rdmacg_object *cg_obj,
			 struct ib_device *device,
			 enum rdmacg_resource_type resource_index)
{
	return rdmacg_try_charge(&cg_obj->cg, &device->cg_device,
				 resource_index);
}
```

ib_rdmacg_try_charge函数的调用都在driver中

## 同一个POD下共享的namespace

```shell
# ls -l /proc/5571/ns
total 0
lrwxrwxrwx 1 root root 0 1月  17 16:26 cgroup -> 'cgroup:[4026533020]'  #不同
lrwxrwxrwx 1 root root 0 1月  17 16:26 ipc -> 'ipc:[4026533016]'
lrwxrwxrwx 1 root root 0 1月  17 16:26 mnt -> 'mnt:[4026533018]'    	    #不同
lrwxrwxrwx 1 root root 0 1月  17 16:26 net -> 'net:[4026532954]'
lrwxrwxrwx 1 root root 0 1月  17 16:26 pid -> 'pid:[4026533019]'          #不同
lrwxrwxrwx 1 root root 0 1月  17 16:26 pid_for_children -> 'pid:[4026533019]'    #不同
lrwxrwxrwx 1 root root 0 1月  17 16:26 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 1月  17 16:26 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 1月  17 16:26 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 1月  17 16:26 uts -> 'uts:[4026533015]'			

# ls -l /proc/5539/ns
total 0
lrwxrwxrwx 1 65535 65535 0 1月  17 16:27 cgroup -> 'cgroup:[4026531835]'      #不同
lrwxrwxrwx 1 65535 65535 0 1月  17 16:24 ipc -> 'ipc:[4026533016]'
lrwxrwxrwx 1 65535 65535 0 1月  17 16:27 mnt -> 'mnt:[4026533014]'            #不同
lrwxrwxrwx 1 65535 65535 0 1月  17 16:24 net -> 'net:[4026532954]'
lrwxrwxrwx 1 65535 65535 0 1月  17 16:27 pid -> 'pid:[4026533017]'            #不同
lrwxrwxrwx 1 65535 65535 0 1月  17 16:27 pid_for_children -> 'pid:[4026533017]'    #不同
lrwxrwxrwx 1 65535 65535 0 1月  17 16:27 time -> 'time:[4026531834]'
lrwxrwxrwx 1 65535 65535 0 1月  17 16:27 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 65535 65535 0 1月  17 16:27 user -> 'user:[4026531837]'          
lrwxrwxrwx 1 root root 0 1月  17 16:26 uts -> 'uts:[4026533015]'			 
```

