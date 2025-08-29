---
title: 🎉 异构内存
summary: 异构内存调研
date: 2025-08-29

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Image credit: [**Unsplash**](https://unsplash.com)'

authors:
  - admin

tags:
  - Linux
  - memory
---
# problems of using a device specific memory allocator

像GPU这样有大量板载内存的设备，用特定的设备API（驱动）来管理它的内存。这样就会导致，它通过驱动分配的内存和计算机系统分配的的应用内存（application memory）之间产生一个disconnect。把这个情况称之为分裂的地址空间（split address space），用共享地址空间（shared address space）代表相反情况，即：任何的设备都能透明地使用应用的地址空间（application memory）。

分裂的地址空间的根源在于，设备只能通过设备驱动访问地址空间。从设备的角度出发，所有的memory objects不平等，这会让那些依赖大量库的程序复杂化。

**具体来说，这意味着，如果想要管理像GPU这样的设备，那么需要在计算机系统分配的内存 (malloc, mmap private, mmap share)和设备驱动分配的内存 (this still ends up with an mmap but of the device file)之间来回复制。**

**对于平面数据集（数组，图像，grid）来说，这很简单，但是对于复杂的数据集（list,tree等）这就很难保持正确，复制一个复杂的数据集，需要在他们每个元素之间重新映射所有的指针。这很容易出错，而且很难debug。**

分裂地址空间也意味着，库不能透明的使用从他们的核心程序或者其他库中获得的数据，因此，每一个库都必须要用特定的设备驱动复制他们的数据集。大量的项目都因此受损。

复制每个库API以接受每个设备特定分配器分配的输入或输出内存是不可行的。这将导致库入口点的组合爆炸。

最后，随着高级语言的进步，现在可以没有编译的知识也能使用GPU和其他设备。一些编译器定义好的模式，只在共享地址空间的条件下可用，那么使用共享地址空间去解决所有其他的模式也是可行的。

# I/O bus, device memory characteristics

由于一些限制，IO总线削弱了共享地址空间。大多数IO总线仅仅允许从设备到主存的基本的内存访问。甚至cache的一致性都是可选的。从CPU到设备的内存访问更是受限制。通常情况下，缓存都是不一致的。

如果我们仅仅考虑PCIE总线，那么一个设备可以访问主存（通常通过IOMMU）并与CPU缓存一致。但是，它仅仅允许设备对主存的一些有限的atomic operations。从另一个维度上看更糟糕：CPU仅仅可以访问一组有限范围的设备内存，不能在上面进行atomic operations。因此，从内核角度来说，设备内存与常规内存也不同。

另一个严重的因素是带宽有限（PCIE 4.0,16个通道，32B/s）比最快的GPU内存（1TB/S）慢33倍。最后的限制是延迟，从设备访问主存的延迟比设备访问自己的主存高一个数量级。

一些平台在开发新的I/O总线，或者对PCIE进行一些添加/修改以解决其中的一些限制（OpenCAPI,CCIX）。他们主要允许CPU和设备之间的双向缓存一致性，并允许所有的架构支持的atomic operations。遗憾的是，并非所有的平台都遵循这一趋势，并且一些主要架构没有针对这些问题的硬件解决方案。

因此为了使共享地址空间有意义，我们不仅必须允许设备访问任何内存，而且还必须允许 当设备在使用内存时，这些内存都能够被迁移到设备内存中（当迁移发生时，阻止CPU访问）。

# 共享内存以及迁移

HMM试图提供两种主要的features：

第一个：通过复制CPU page table到设备page table中来共享地址空间，因此，在进程的地址空间中的任何有效主存地址，相同的地址指针指向相同的物理内存。

为了实现这一点，HMM提供了一组helpers来填充设备的page table当CPU page table更新的时候。设备page table的更新不像CPU page table更新那样简单，要更新设备的page table，你必须分配buffer（或者使用一个预先分配的缓冲区池），然后将GPU特定的指令写入其中来实现更新（unmap, cache invalidations, and flush ...）.但是这一步用common code是做不到将所有设备都覆盖到的。 Hence why HMM provides helpers to factor out everything that can be while leaving the hardware specific details to the device driver.

第二个：HMM提供了一种新的ZONE_DEVICE 内存，这种内存允许为device内存的每一个page都分配一个struct page。这些pages是特别的，因为CPU不能够映射他们，但是他们允许使用现有的迁移机制将主存迁移到设备内存中，从CPU的角度来看，所有的东西都看起来都像一页能够被swapped out到disk中的page一样。使用一个struct page是最简单最简洁的与现有的mm机制集成的方法。HMM仅仅提供一个helper，第一个是为了device内存提供一个新的ZONE_DEVICE热插拔内存；第二个是为了执行迁移。什么时候迁移以及迁移什么都由设备驱动来决定。

注意：任何CPU对device page的访问以及迁移回主存时都会触发一个page fault。例如：当一个指定CPU访问的page A从主存page迁移到device page时。然后任何CPU对A的访问都会触发page fault，并且发起一个向主存的迁移。

凭借着两个特性，HMM不仅允许设备镜像进程的地址空间，并保持CPU和设备page fault同步，而且通过迁移部分在设备中正在使用的内存使用设备内存。

# Address space mirroring implementation and API

地址空间镜像的主要目标是允许将一些CPU的page table复制到设备page table中；HMM有助于保持两者的同步。如果一个设备驱动想要镜像一个进程的地址空间，那么它必须要从mmu_interval_notifier_insert开始：

```c
int mmu_interval_notifier_insert(struct mmu_interval_notifier *interval_sub,
                                 struct mm_struct *mm, unsigned long start,
                                 unsigned long length,
                                 const struct mmu_interval_notifier_ops *ops);
```

在 ops->invalidate()回调期间，设备驱动程序必须对the range进行更新操作（将range标记为只读，或者fully unmap等等）。这个设备必须在设备的回调返回之前完成更新。

当设备驱动想要填充一个一个虚拟地址range时，它可以使用：

```c
int hmm_range_fault(struct hmm_range *range);
```

如果请求写访问，它将在missing or read-only entries触发一个fage fault。page fault通常走mm page fault代码路径。就像CPU page fault一样。

这两个函数都将CPU page table entries复制到他们的pfns数组参数中。在这个数组中的每一个entry都对应于虚拟range中的一个地址。HMM提供了一系列flags去帮助驱动识别特殊的CPU page table entries.

为了保持things正确的同步，在sync_cpu_device_pagetables()回调函数中之内进行上锁是驱动程序必须且最重要要做的事。使用模式如下：

```c
int driver_populate_range(...)
{
     struct hmm_range range;
     ...

     range.notifier = &interval_sub;
     range.start = ...;
     range.end = ...;
     range.hmm_pfns = ...;

     if (!mmget_not_zero(interval_sub->notifier.mm))
         return -EFAULT;

again:
     range.notifier_seq = mmu_interval_read_begin(&interval_sub);
     mmap_read_lock(mm);
     ret = hmm_range_fault(&range);
     if (ret) {
         mmap_read_unlock(mm);
         if (ret == -EBUSY)
                goto again;
         return ret;
     }
     mmap_read_unlock(mm);

     take_lock(driver->update);
     if (mmu_interval_read_retry(&ni, range.notifier_seq) {
         release_lock(driver->update);
         goto again;
     }

     /* Use pfns array content to update device page table,
      * under the update lock */

     release_lock(driver->update);
     return 0;
}
```

driver->update 锁和驱动在其 invalidate() 回调函数中使用的锁是一致的。该锁必须在调用 mmu_interval_read_retry() 之前保持，以避免与并发 CPU 页表更新发生任何竞争。

# Leverage default_flags and pfn_flags_mask

 hmm_range 结构体有两个成员， default_flags以及pfn_flags_mask，他们指定整个range的fault和snapshot策略，而不是为pfns数组中的每个条目设置他们。

例如，如果设备驱动想要至少具有读取权限range的pages，可以设置：

```c
range->default_flags = HMM_PFN_REQ_FAULT;
range->pfn_flags_mask = 0;
```

然后调用上面说到的hmm_range_fault() ，这个函数将填充至少具有读取权限的range内的所有pages。

现假设驱动程序想要做同样的事情，但是它还想要在这整个range中得到一个拥有写权限的页面。驱动可以设置：

```c
range->default_flags = HMM_PFN_REQ_FAULT;
range->pfn_flags_mask = HMM_PFN_REQ_WRITE;
range->pfns[index_of_write] = HMM_PFN_REQ_WRITE;
```

有了这个，HMM will fault in all pages with at least read并且对于address = range->start + (index_of_write << PAGE_SHIFT)的斜面，it will fault with write permission。即：如果 CPU pte 没有设置写权限，那么HMM将调用handle_mm_fault()。

在 hmm_range_fault()完成后，标志位被设置为表的当前状态，即如果页面可写，那么HMM_PFN_VALID | HMM_PFN_WRITE将被设置。

# Represent and manage device memory from core kernel point of view

一些不同的设计都被尝试来支持设备内存，第一个使用一个特定于设备的数据结构来保存有关迁移内存的信息。HMM将自身hook到mm代码中的各个位置，以处理设备内存支持的任何地址访问。事实证明，这最终复制了大多struct page的字段，并且还需要增加许多内核代码路径才能理解这种新的内存，

大多数内核代码路径从来不会 access the memory behind a page，而仅仅只关心struct page的内容。因此，HMM换了个思路，直接使用设备内存的struct page，这能够让大多数内核代码路径察觉不到变化，我们仅仅需要确保，没人视图从CPU端映射这些页面就行。

# Migration to and from device memory

因为CPU不能够直接访问设备内存，设备驱动必须使用硬件DMA或者特定设备的load/store指令来迁移数据，migrate_vma_setup()，migrate_vma_pages()，migrate_vma_finalize()函数被设计用来使驱动程序更易于编写并集中跨驱动程序的通用代码。

在将page迁移到设备私有内存之前，需要创建特殊的设备私有struct page。这些将被用作特殊的“swap” page table entries，以便如果CPU进程尝试访问已经被迁移到设备私有内存的page时，它会产生中断。

这些可以通过以下方式分配和释放。

```c
struct resource *res;
struct dev_pagemap pagemap;

res = request_free_mem_region(&iomem_resource, /* number of bytes */,
                              "name of driver resource");
pagemap.type = MEMORY_DEVICE_PRIVATE;
pagemap.range.start = res->start;
pagemap.range.end = res->end;
pagemap.nr_range = 1;
pagemap.ops = &device_devmem_ops;
memremap_pages(&pagemap, numa_node_id());

memunmap_pages(&pagemap);
release_mem_region(pagemap.range.start, range_len(&pagemap.range));
```

当资源可以被绑定时，还有 [`devm_request_free_mem_region()`](https://www.kernel.org/doc/html/latest/core-api/kernel-api.html#c.devm_request_free_mem_region), devm_memremap_pages(), devm_memunmap_pages(), and devm_release_mem_region()

所有的迁移步骤类似于在系统内存中迁移NUMA页面，但是步骤被分成了设备驱动特有代码以及共享公共代码。

1. mmap_read_lock()

备驱动程序必须将 `struct vm_area_struct` 传递给migrate_vma_setup()，因此需要在迁移期间保留 mmap_read_lock() 或 mmap_write_lock()。

2. migrate_vma_setup(struct migrate_vma *args)

设备驱动初始化了 `struct migrate_vma` 的字段，并将该指针传递给 migrate_vma_setup()。`args->flags` 字段是用来过滤哪些源页面应该被迁移。例如：设置MIGRATE_VMA_SELECT_SYSTEM将只迁移系统内存，MIGRATE_VMA_SELECT_DEVICE_PRIVATE将仅仅迁移驻留在设备私有内存中的pages。如果后者被设置，args->pgmap_owner字段被用来识别驱动所拥有的设备 私有页。这避免了将驻留在别的设备中的pages迁移。下面早，只有匿名的私有VMAranges可以在系统内存和设备私有内存中来回切换。

migrate_vma_setup()所做的第一步是用`mmu_notifier_invalidate_range_start()` 和 `mmu_notifier_invalidate_range_end()` 调用来遍历设备周围的页表，使 其他设备的MMU无效，以便在 `args->src` 数组中填写要迁移的PFN。invalidate_range_start()回调函数被传递一个 `struct mmu_notifier_range` ，其event字段被设置为MMU_NOTIFY_MIGRATE，owner字段被设置为args->pgmap_owner传递给migrate_vma_setup()。这允许设备驱动跳过无 效化回调，只无效化那些实际正在迁移的设备私有MMU映射。这一点将在下一节详细解释。

当遍历页表时，一个pte_none()` or `is_zero_pfn() entry导致一个储存在args->src数组中的有效的“zero"PFN。这让驱动分配设备分配设备私有内存，并且clear it，而不是复制一页的0。 系统内存或设备私有结构页的有效PTE条目将被 `lock_page()``锁定，`，与LRU隔离（如果系统内存和设备私有页不在LRU上），从进程中取消映射，一个特殊的迁移PTE代替原来的PTE。migrate_vma_setup() 还clears了`args->dst` 数组.

3. The device driver allocates destination pages and copies source pages to destination pages.

驱动检查每一个src entry来看是否MIGRATE_PFN_MIGRATE位被设置了，并且跳过哪些没有被迁移的entries。设备驱动还可以通过不填充页面的dst数组来选择跳过迁移一个page。

驱动然后要么分配一个设备私有的struct page或者一个系统page，用lock_page()上锁，然后用如下函数填充dst数组entry

```c
dst[i] = migrate_pfn(page_to_pfn(dpage));
```

既然驱动知道，这个page正在被迁移，它可以使设备私有MMU映射吴骁并将设备私有内存复制到系统内存或另一个设备私有page中。由于核心Linux内核会处理CPU page table invalidations，所以设备驱动仅仅只需要使自己的MMU映射失效即可。

驱动可以使用migrate_pfn_to_page(src[i])来获得原设备的struct page然后，要么复制源pages到目的地如果指针为NULL，意味着原页面没有被填充到系统内存中，则clear目的设备私有内存。

4. migrate_vma_pages()

这一步使迁移实际被committed的地方

如果袁pages是一个pte_none()` or `is_zero_pfn()page,这里就是新配分page被插入到CPU page table中的地方。这一步可能会失败，因为CPU线程在同样的页面上faults。但是，page table被锁住了，所以仅仅只有一个新的pages会被插入。如果竞争失败，设备驱动将会发现MIGRATE_PFN_MIGRATE位被清楚了。

如果原Page被上锁了，隔离了等等，原struct page信息现在被复制到了目的struct page以实现CPU层面的隔离。

5. 设备驱动对哪些仍然在迁移的页面更新MMU page tables，回滚哪些没有被迁移的。

如果src entry仍然被设置了MIGRATE_PFN_MIGRATE位，设备驱动可以更新设备MMU，并且如果MIGRATE_PFN_WRITE位被设置的话，将设置写启用位。

6. migrate_vma_finalize()

这一步用新页的页表项替换特殊的迁移页表项，并释放对源和目的 `struct page` 的引用。

7. mmap_read_unlock()

释放锁。

# Exclusive access memory

一些设备具有如原子PTE位的功能，可以用来实现对系统内存的原子访问。为了在共享虚拟内存page中也支持这样的原子操作，这样的设备需要对页的访问时排他的，而不是来此CPU的任何用户空间访问。make_device_exclusive_range()函数可以用来使得内存range拒绝从用户空间的访问。

这将用特殊的swap entries替换给定范围的所有页的映射。任何试图访问swap entry的请求都将导致一个fault，该fault会通过原始映射替换该entry而得到恢复。驱动程序会被告知，映射已经被MMU通知器改变，之后它将不再有对该页的独占访问。独占访问被保证持续到驱动程序放弃页面锁和页面引用为止。这时页面上的任何CPU异常都可以正常进行。



# Memory cgroup (memcg) and rss accounting

目前来说，设备内存被当作在RSS计数器中的任何一个常规page计算，（如果设备page是匿名的，那么就被统计为匿名页；如果设备page被用于file backed page，那么统计为file；如果设备page被用作共享内存，就被标记为为shmem）这是为了保持现有应用程序的故意选择，因为这些应用程序可能在不知情的情况下开始使用设备内存，能够让运行不受影响。

**一个缺点是OOM killer可能会杀掉使用大量设备内存的应用，而不是使用大量常规系统内存的应用程序。因此不会释放太多的系统内存。我们希望收集大量关于应用程序和系统再存在设备内存的情况下再内存压力下如果反应的实际实验。**

对于memory cgroup做了同样的决定。设备内存pages被当作常规的内存计算。这确实简化了进出设备设备内存的迁移。这也意味着从设备内存迁移会常规内存不能够失败，因为它可能会超过memory cgroup的限制。一旦我们对设备内存的使用方式及其对内存资源控制的影响有了更多的了解，我们可能会在后面重新考虑这个选择。

请注意，设备内存永远不能由驱动程序或者GPU绑定，因为不由他们绑定，所以这样的内存总是在进程退出后被free掉，或者在共享内存或文件支持内存的情况下，当删除最后一个应用的时候被free掉。

想法：

1.如果系统内存被迁移到了设备内存，能不能把统计的mem减少，如果能够减少，就可以不停的迁移设备内存。直到把设备的占满，然后由于本身cgroup的限制，可以不停的申请内存，让设备内存没办法迁移回来。







首先理解对设备内存的使用搞清楚，存在异构设备情况下，之前的攻击是否还存在？

如果存在的话——体系结构问题
