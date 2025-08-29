---
title: 🎉 Cgroup | 学习笔记2
summary: Cgroup v2 cpu controller
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
`cpuacct子系统`（CPU accounting）会自动生成报告来显示`cgroup中`任务所使用的`CPU`资源，其中包括子群组任务。报告有两大类：

- `usage`: 统计`cgroup`中进程使用 CPU 的时间，单位为纳秒。
- `stat`: 统计`cgroup`中进程使用 CPU 的时间，单位为`USER_HZ`。

### usage

- `cpuacct.usage` : 报告一个`cgroup`中所有任务（包括其子孙层级中的所有任务）使用`CPU`的总时间（纳秒）,该文件时可以写入`0`值的，用来进行重置统计信息。
- `cpuacct.usage_percpu`: 报告一个`cgroup`中所有任务（包括其子孙层级中的所有任务）在每个`CPU`使用`CPU`的时间（纳秒）。
- `cpuacct.usage_user`: 报告一个`cgroup`中所有任务（包括其子孙层级中的所有任务）使用用户态`CPU`的总时间（纳秒）。
- `cpuacct.usage_percpu_user` 报告一个`cgroup`中所有任务（包括其子孙层级中的所有任务）在每个`CPU`上使用用户态`CPU`的时间（纳秒）。
- `cpuacct.usage_sys`: 报告一个`cgroup`中所有任务（包括其子孙层级中的所有任务）使用内核态`CPU`的总时间（纳秒）。
- `cpuacct.usage_percpu_sys`：报告一个`cgroup`中所有任务（包括其子孙层级中的所有任务）在每个`CPU`上使用内核态`CPU`的时间（纳秒）。
- `cpuacct.usage_all`：详细输出文件`cpuacct.usage_percpu_user`和`cpuacct.usage_percpu_sys`的内容。

### stat

- ```
  cpuacct.stat
  ```

  ：报告 cgroup 的所有任务（包括其子孙层级中的所有任务）使用的用户和系统 CPU 时间，方式如下：

  - `user`——用户模式中任务使用的 CPU 时间
  - `system`——系统模式中任务使用的 CPU 时间
  - 其单位为`USER_HZ`

## 内核实现

### 结构体 struct cpuacct

`cpuacct`的内核实现中，对`cpu`时间的统计结果都存放到数据结构`struct cpuacct`中，数据结构定义如下：

```c
/* track CPU usage of a group of tasks and its child groups */
struct cpuacct {
	struct cgroup_subsys_state	css;
	/* cpuusage holds pointer to a u64-type object on every CPU */
	struct cpuacct_usage __percpu	*cpuusage;
	struct kernel_cpustat __percpu	*cpustat;
};
```

除了 css 外，其他两个成员都是`__percpu`类型。

### 结构体 struct cpuacct_usage

数据结构定义如下：

```c
struct cpuacct_usage {
	u64	usages[CPUACCT_STAT_NSTATS];
};

/* Time spent by the tasks of the CPU accounting group executing in ... */
enum cpuacct_stat_index {
	CPUACCT_STAT_USER,	/* ... user mode */
	CPUACCT_STAT_SYSTEM,	/* ... kernel mode */

	CPUACCT_STAT_NSTATS,
};
```

### 结构体 struct kernel_cpustat

数据结构定义如下：

```c
struct kernel_cpustat {
	u64 cpustat[NR_STATS];
};

enum cpu_usage_stat {
	CPUTIME_USER,
	CPUTIME_NICE,
	CPUTIME_SYSTEM,
	CPUTIME_SOFTIRQ,
	CPUTIME_IRQ,
	CPUTIME_IDLE,
	CPUTIME_IOWAIT,
	CPUTIME_STEAL,
	CPUTIME_GUEST,
	CPUTIME_GUEST_NICE,
	NR_STATS,
};
```

`cpuacct.stat`中的统计时间主要来源于该结构体，其中

- user 时间包括：CPUTIME_USER + CPUTIME_NICE
- system 时间包括：CPUTIME_IRQ + CPUTIME_SOFTIRQ + CPUTIME_SYSTEM

`/sys/fs/cgroup/cpuacct`下所有统计文件就是通过`cpuacct`结构体中的统计值来输出信息的。

而`cpu`时间信息的更新则由如下函数完成,用于更新 cpuusage, 该函数更新所有的`cpuacct cgroup`，包括根`root cpuacct cgroup`

### `cpuacct_charge`

```c
void cpuacct_charge(struct task_struct *tsk, u64 cputime)
{
	struct cpuacct *ca;
	int index = CPUACCT_STAT_SYSTEM;
	struct pt_regs *regs = task_pt_regs(tsk);

	if (regs && user_mode(regs))
		index = CPUACCT_STAT_USER;

	rcu_read_lock();

	for (ca = task_ca(tsk); ca; ca = parent_ca(ca))
		this_cpu_ptr(ca->cpuusage)->usages[index] += cputime;

	rcu_read_unlock();
}
```

### `cpuacct_account_field`

用于更新 cpustat，该函数更新所有的`cpuacct cgroup`，但不包括`root cpuacct cgroup`

```c
void cpuacct_account_field(struct task_struct *tsk, int index, u64 val)
{
	struct cpuacct *ca;

	rcu_read_lock();
	for (ca = task_ca(tsk); ca != &root_cpuacct; ca = parent_ca(ca))
		this_cpu_ptr(ca->cpustat)->cpustat[index] += val;
	rcu_read_unlock();
}
/* Return CPU accounting group to which this task belongs */
static inline struct cpuacct *task_ca(struct task_struct *tsk)
{
	return css_ca(task_css(tsk, cpuacct_cgrp_id));
}
```

如下函数会去调用`cpuacct_charge`:

- `update_curr(struct cfs_rq *cfs_rq)`->`cgroup_account_cputime`->`cpuacct_charge`
- `update_curr_rt(struct rq *rq)`->`cgroup_account_cputime`->`cpuacct_charge`
- `update_curr_dl(struct rq *rq)`->`cgroup_account_cputime`->`cpuacct_charge`
- `put_prev_task_stop(struct rq *rq, struct task_struct *prev)`->`cgroup_account_cputime`->`cpuacct_charge`

如下函数会去调用`cpuacct_account_field`:

- `account_process_tick`->`account_user_time`->`task_group_account_field`->`cgroup_account_cputime_field`->`cpuacct_account_field`
- `account_process_tick`->`irqtime_account_process_tick`->`account_user_time`->`task_group_account_field`->`cgroup_account_cputime_field`->`cpuacct_account_field`
- `__context_tracking_exit`->`vtime_user_exit`->`account_user_time`->`task_group_account_field`->`cgroup_account_cputime_field`->`cpuacct_account_field`
- `account_process_tick`->`account_system_time`->`account_system_index_time`->`task_group_account_field`->`cgroup_account_cputime_field`->`cpuacct_account_field`
- `irqtime_account_process_tick`->`account_system_index_time`->`task_group_account_field`->`cgroup_account_cputime_field`->`cpuacct_account_field`
- `__vtime_account_system`->`account_system_time`->`account_system_index_time`->`task_group_account_field`->`cgroup_account_cputime_field`->`cpuacct_account_field`

除此之外，还有如下函数会更新`cpustat`

- `account_idle_time`
- `account_steal_time`
- `account_guest_time`

### 总结一下

更新cpuusage得接口有如下几个：

- update_curr
- update_curr_rt
- update_curr_dl
- put_prev_task_stop

更新 cpustat 的接口有如下几个：

- `account_user_time`
- `account_system_time`
- `irqtime_account_process_tick`
- `account_idle_time`
- `account_steal_time`
- `account_guest_time`

## 分析更新 cpustat 的接口实现

### account_user_time

该接口根据`task_nice(p)`是否为真，更新`CPUTIME_NICE`或者`CPUTIME_NICE`

```c
void account_user_time(struct task_struct *p, u64 cputime)
{
	int index;

	/* Add user time to process. */
	p->utime += cputime;
	account_group_user_time(p, cputime);

	index = (task_nice(p) > 0) ? CPUTIME_NICE : CPUTIME_USER;

	/* Add user time to cpustat. */
	task_group_account_field(p, index, cputime);

	/* Account for user time used */
	acct_account_cputime(p);
}
```

### account_system_time

该接口根据不同的情况可能更新`CPUTIME_IRQ` 或者 `CPUTIME_SOFTIRQ` 或者 `CPUTIME_SYSTEM`

> 注意，该接口还有可能通过接口`account_guest_time`进行时间的更新。

```c
void account_system_time(struct task_struct *p, int hardirq_offset, u64 cputime)
{
	int index;

	if ((p->flags & PF_VCPU) && (irq_count() - hardirq_offset == 0)) {
		account_guest_time(p, cputime);
		return;
	}

	if (hardirq_count() - hardirq_offset)
		index = CPUTIME_IRQ;
	else if (in_serving_softirq())
		index = CPUTIME_SOFTIRQ;
	else
		index = CPUTIME_SYSTEM;

	account_system_index_time(p, cputime, index);
}
```

### account_idle_time

该接口更新了`CPUTIME_IDLE`或者`CPUTIME_IOWAIT`，只更新了`root cpuacct`，即`idle`时间不是`cgroup aware`的。

```c
/*
 * Account for idle time.
 * @cputime: the CPU time spent in idle wait
 */
void account_idle_time(u64 cputime)
{
	u64 *cpustat = kcpustat_this_cpu->cpustat;
	struct rq *rq = this_rq();

	if (atomic_read(&rq->nr_iowait) > 0)
		cpustat[CPUTIME_IOWAIT] += cputime;
	else
		cpustat[CPUTIME_IDLE] += cputime;
}

```

### account_steal_time

该接口更新了`CPUTIME_STEAL`，只更新了`root cpuacct`，即`steal`时间不是`cgroup aware`的。

```c
/*
 * Account for involuntary wait time.
 * @cputime: the CPU time spent in involuntary wait
 */
void account_steal_time(u64 cputime)
{
	u64 *cpustat = kcpustat_this_cpu->cpustat;

	cpustat[CPUTIME_STEAL] += cputime;
}
```

### account_guest_time

该接口根据`task_nice(p)`是否为真，更新`CPUTIME_NICE and CPUTIME_GUEST_NICE`或者`CPUTIME_USER and CPUTIME_GUEST`, 只更新了`root cpuacct`，即`guest`时间不是`cgroup aware`的。

```c
/*
 * Account guest CPU time to a process.
 * @p: the process that the CPU time gets accounted to
 * @cputime: the CPU time spent in virtual machine since the last update
 */
void account_guest_time(struct task_struct *p, u64 cputime)
{
	u64 *cpustat = kcpustat_this_cpu->cpustat;

	/* Add guest time to process. */
	p->utime += cputime;
	account_group_user_time(p, cputime);
	p->gtime += cputime;

	/* Add guest time to cpustat. */
	if (task_nice(p) > 0) {
		cpustat[CPUTIME_NICE] += cputime;
		cpustat[CPUTIME_GUEST_NICE] += cputime;
	} else {
		cpustat[CPUTIME_USER] += cputime;
		cpustat[CPUTIME_GUEST] += cputime;
	}
}
```

### root cpuacct 的数据来源？

- `root cpuacct cgroup`中`usage`来源于变量`root_cpuacct_cpuusage`，在`cpuacct_charge`中会更新它的值。
- `root cpuacct cgroup`中`cpustat`来源于变量`kernel_cpustat`,`cpuacct_account_field`并不会更新它，而是有系统上其它部分去更新。

```c
static DEFINE_PER_CPU(struct cpuacct_usage, root_cpuacct_cpuusage);
static struct cpuacct root_cpuacct = {
	.cpustat	= &kernel_cpustat,
	.cpuusage	= &root_cpuacct_cpuusage,
};
```

### /proc/stat 中 cpu 相关统计信息来自哪里？

- `/proc/stat`中 cpu 相关统计数据来自于变量`kernel_cpustat`,这个跟`root cpuacct cgroup`的数据来源是一样的。

