---
title: 🎉 Cgroup | 学习笔记4
summary: Cgroup v2 hugetlb controller
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
HugeTlb 记账与销账

```c
int hugetlb_cgroup_charge_cgroup(int idx, unsigned long nr_pages,
				 struct hugetlb_cgroup **ptr)
{
	return __hugetlb_cgroup_charge_cgroup(idx, nr_pages, ptr, false);
}
struct page *alloc_huge_page(struct vm_area_struct *vma,
				    unsigned long addr, int avoid_reserve)
```

通过对 `hugetlb_no_page` 函数进行精简后，主要完成3个工作：

1. 调用 `alloc_huge_page` 函数从空闲大内存页链表 `hugepage_freelists` 中申请一个大内存页。
2. 通过大内存页的物理地址生成页中间目录项的值。
3. 设置页中间目录项的值为上面生成的值。

 

hugetlb

```
echo  N  > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <stdint.h>
#include <errno.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdlib.h>

#define HUGE_SIZE   2048*1024   //2M
#define ALLOC_HUGE_SIZE (HUGE_SIZE*35) 
#define    ERR_DBG_PRINT(fmt, args...) \
    do \
    { \
        printf("ERR_DBG:%s(%d)-%s:\n"fmt": %s\n", __FILE__,__LINE__,__FUNCTION__,##args, strerror(errno)); \
    } while (0)
int main(int argc, char *argv[])
{
    key_t  our_key = -1;
    int shm_id = -1;
    char *huge_mem = NULL;

    our_key = ftok("/home/huge_mm_test.c", 7);
    if (-1 == (int)our_key)
    {
        ERR_DBG_PRINT("ftok fail:");
        return -1;
    }

    shm_id = shmget(our_key, ALLOC_HUGE_SIZE, IPC_CREAT|IPC_EXCL|SHM_HUGETLB|0666);
    if (-1 == shm_id)
    {
        ERR_DBG_PRINT("shmget fail:");
        return -1;
    }
    printf("shmget success!\n");
    huge_mem = shmat(shm_id, NULL, 0);
    if (-1 == (int)(unsigned long)huge_mem)
    {
        ERR_DBG_PRINT("shmat fail:");
        return -1;
    }
    memset(huge_mem, 0, ALLOC_HUGE_SIZE);
    printf("huge alloc write ok \n");

    sleep(10);
    shmdt(huge_mem);
   /* if (shmctl(shm_id, IPC_RMID, NULL)==-1)
        ERR_DBG_PRINT("shmctl fail:");
 */
    return 0;
}
```

