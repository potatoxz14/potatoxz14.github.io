---
title: 🎉 Linux command free
summary: free 命令详解
date: 2025-08-28

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
  - 源码详解
---

参考链接：https://elinux.org/images/8/81/Elc2013_Garcia.pdf

https://zhuanlan.zhihu.com/p/39788032

跟踪脚本

```shell
#!/bin/sh

dir=/sys/kernel/debug/tracing
function="xxxx"
sysctl kernel.ftrace_enabled=1

echo $function > ${dir}/current_tracer   #可以进入event挑选
echo nop > ${dir}/trace
echo 1 > ${dir}/tracing_on
sleep 1    #do something
echo 0 > ${dir}/tracing_on
```

输入如下

```shell
# tracer: function
#
# entries-in-buffer/entries-written: 29571/29571   #P:2
#
#                           _-----=> irqs-off
#                           / _----=> need-resched
#                           | / _---=> hardirq/softirq
#                           || / _--=> preempt-depth
#                           ||| /   delay
#           TASK-PID   CPU#  ||||   TIMESTAMP  FUNCTION
#           | |     |   ||||    |       |
```

如果是kmem内存申请：

```shell
root@xz:/sys/kernel/debug/tracing/events/kmem# ls
enable  kmalloc           mm_page_alloc              mm_page_free          rss_stat
filter  kmem_cache_alloc  mm_page_alloc_extfrag      mm_page_free_batched
kfree   kmem_cache_free   mm_page_alloc_zone_locked  mm_page_pcpu_drain
```

```shell
#echo 1 > xxxx/enable
```

然后运行上述脚本