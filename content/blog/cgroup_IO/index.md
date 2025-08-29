https://zorrozou.github.io/docs/books/cgroup_linux_io_control_group.html

## 结论1

所有容器相关的IO，在容器退出的时候都会进行类似sync系统调用的数据落盘。

- 使用docker stop contaienr2无法停止容器，docker ps 依旧可以看到这个容器，但是容器内的进程都没有了，该容器对应的cgroup也不存在了，但是IO行为一直存在。
- 该过程中无法docker exec -it 进入容器。
- 当IO结束的时候，docker ps就看不到这个进程了。

可能存在的问题：这个过程container2的结束需要占用大量时间。

## 结论2

cgroup对于io的限制主要是限制提交io操作而不是限制fd的拥有者。

可能存在的绕过：

容器A不存在IO限制，容器B存在IO限制；容器B将fd传递给容器A，让容器A帮助容器B进行IO。尽管这个fd对应的文件容器A无法看见。

容器A,B共享一个目录：



命名管道，创建管道的过程。mkfifo,会创建一个文件。

挂载，读写文件，临时文件。tmp fs溢出了，会有落盘。



## cgroup能够限制网络IO吗？

结论：不能

“io.max” limits the maximum BPS and/or IOPS that a cgroup can consume on an IO device and is an example of this type.

cgroup v2 中 io 子系统等同于 v1 中的 blkio 子系统

blkcg_css









```c
static struct cgroup_subsys_state *blkcg_css(void)
{
	struct cgroup_subsys_state *css;

	css = kthread_blkcg();
	if (css)
		return css;
	return task_css(current, io_cgrp_id);
}
```

寻找线头：

```
.name = "stat",
```

block

```c
struct blkcg {
	struct cgroup_subsys_state	css;
	spinlock_t			lock;
	refcount_t			online_pin;

	struct radix_tree_root		blkg_tree;
	struct blkcg_gq	__rcu		*blkg_hint;
	struct hlist_head		blkg_list;

	struct blkcg_policy_data	*cpd[BLKCG_MAX_POLS];

	struct list_head		all_blkcgs_node;

	/*
	 * List of updated percpu blkg_iostat_set's since the last flush.
	 */
	struct llist_head __percpu	*lhead;

#ifdef CONFIG_BLK_CGROUP_FC_APPID
	char                            fc_app_id[FC_APPID_LEN];
#endif
#ifdef CONFIG_CGROUP_WRITEBACK
	struct list_head		cgwb_list;
#endif
};
```

