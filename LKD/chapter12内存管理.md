# 内存管理

![内存分配函数](D:\笔记\Notes\image\内存分配函数.png)

## 页

内核把物理页作为内存管理的基本单元。内核用struct page结构表示系统中的每个**物理页**， 在头文件<linux/mm_types.h>定义。**page结构与物理页相关，而非与虚拟页相关。这个数据结构的目的是描述物理内存本身，而不是描述包含在其中的数据。**

> The important point to understand is that the  page  structure is associated with physical pages, not virtual pages.Therefore, what the structure describes is transient at best. Even if the data contained in the page continues to exist, it might not always be associated with the same  page  structure because of swapping and so on.The kernel uses this data structure to describe the associated physical page.The data structure’s goal is to describe physical memory, not the data contained therein.

```c
struct page { 
	unsigned long         flags;//表示页的状态，每一位表示一种状态
	atomic_t              _count; //引用计数，表示这一页被引用了多少次
	atomic_t              _mapcount; 
	unsigned long         private; 
	struct address_space  *mapping; 
	pgoff_t               index; 
	struct list_head      lru; 
	void                  *virtual;//页的虚拟地址
};
```

## 区

由于硬件的限制，内核并不能对所有的页一视同仁，有些页位于内存中特定的物理地址上，不能将其用于一些特定的任务，因此内核把页分为不同的区。内核使用区对具有相似特性的页进行分组。linux必须处理如下两种由于硬件缺陷引起的内存寻址问题：

- 一些硬件只能用某些特定的内存地址来执行DMA
- 一些体系结构的内存的物理寻址范围比虚拟地址范围大得多，这样就有一些内存不能永久地映射到内核空间上。

linux主要使用了四种区

- ZONE_DMA——这个区包含的页能用来执行DMA操作
- ZONE_DMA32——和ZONE_DMA类似，该区的页可以执行DMA操作，但是只能被32位的设备访问。在某些体系结构中，该区比ZONE_DMA更大。
- ZONE_NORMAL——包含的都是能够正常映射的页。
- ZONE_HIGHMEM——高端内存，其中的页不能永久地映射到内核地址空间。

区的实际使用和分布是与体系结构相关的。linux把系统的页划分为区，形成不同的内存池，这样就可以根据用途进行分配了。

## 获得页

```c
//获得2^order个连续的物理页，并返回一个指针，该指针指向第一个页的结构体，如果出错返回NULL。
struct page *alloc_pages(gfp_t gfp_mask, unsigned int order)；
void __free_pages(struct page *page, unsigned int order);

//把给定的页转换为逻辑地址
void *page_address(struct page *page);

//分配2^order个连续物理页，直接返回请求的第一个页的逻辑地址。
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);
//释放页
void free_page(unsigned long addr, unsigned int order);
//如果只需一页，可以使用这两个函数
struct page *alloc_page(gfp_t gfp_mask)；
unsigned long __get_free_page(gfp_t gfp_mask);
void free_page(unsigned long addr);
//获取内容全为0的页
unsigned long get_zeroed_page(unsigned int gfp_mask);
```



## kmalloc()

kmalloc与用户空间的malloc非常类似，只不过多了一个flags参数，用它可以获得以字节为单位的内核内存。

它在<linux/slab.h>中定义。

```c
//返回一个指向内存块的指针，其内存块至少有size大小
void * kmalloc(size_t size, gfp_t flags)
   
void kfree(const void *ptr);
```



### gfp_mask标志

gfp_mask标志可以分为三类：行为修饰符、区修饰符及类型修饰符。

1. 行为修饰符表示内核应当如何分配所需的内存。

   | 标志              | 描述                                                         |
   | ----------------- | ------------------------------------------------------------ |
   | __GFP_WAIT        | The allocator can sleep.                                     |
   | __GFP_HIGH        | The allocator can  access emergency pools.                   |
   | __GFP_IO          | The allocator can start disk I/O.                            |
   | __GFP_FS          | The allocator can   start filesystem I/O.                    |
   | __GFP_COLD        | The allocator should   use cache cold pages.                 |
   | __GFP_NOWARN      | The allocator does not print failure   warnings.             |
   | __GFP_REPEAT      | The allocator repeats   the allocation if it fails, but the allocation can potentially fail. |
   | __GFP_NOFAIL      | The allocator indefinitely repeats the   allocation. The allocation cannot fail. |
   | __GFP_NORETRY     | The allocator never   retries if the allocation fails.       |
   | __GFP_NOMEMALLOC  | The allocator does not fall back on   reserves.              |
   | __GFP_HARDWALL    | The allocator   enforces “hardwall” cpuset boundaries.       |
   | __GFP_RECLAIMABLE | The allocator marks   the pages reclaimable.                 |
   | __GFP_COMP        | The allocator adds compound page metadata   (used internally by the hugetlb code). |

2. 区修饰符表示内存区应当从何如分配

   | 标志          | 描述                                         |
   | ------------- | -------------------------------------------- |
   | __GFP_DMA     | Allocates only from   ZONE_DMA               |
   | __GFP_DMA32   | Allocates only from   ZONE_DMA32             |
   | __GFP_HIGHMEM | Allocates from   ZONE_HIGHMEM or ZONE_NORMAL |

3. 类型标志指定所需的行为和区描述符以完成特定类型的处理。

   | 标志         | 描述                                                         |
   | ------------ | ------------------------------------------------------------ |
   | GFP_ATOMIC   | The   allocation is high priority and must not sleep. This is the flag to use in   interrupt handlers, in bottom halves, while holding a spin-lock, and in other   situations where you cannot sleep. |
   | GFP_NOWAIT   | Like GFP_ATOMIC, except that the call will   not fallback on emergency memory pools. This increases the liklihood of the   memory allocation failing. |
   | GFP_NOIO     | This   allocation can block, but must not initiate disk I/O. This is the flag to use   in block I/O code when you cannot cause more disk |
   | GFP_NOFS     | This allocation can block and can initiate   disk I/O, if it must, but it will not initiate a filesystem operation. This   is the flag to use in filesystem code when you cannot start another   filesystem operation. |
   | GFP_KERNEL   | This is a normal allocation and might block.   This is the flag to use in process context code when it is safe to sleep. The   kernel will do whatever it has to do to obtain the memory requested by the   caller. This flag should be your default choice. |
   | GFP_USER     | This   is a normal allocation and might block. This flag is used to allocate memory   for user-space processes. |
   | GFP_HIGHUSER | This   is an allocation from ZONE_HIGHMEM and might block. This flag is used to   allocate memory for user-space processes. |
   | GFP_DMA      | This   is an allocation from ZONE_DMA. Device drivers that need DMA-able memory use   this flag, usually in combination with one of the preceding flags. |



## vmalloc()

vmalloc()与kmalloc()类似，只不过vmalloc分配的内存虚拟地址是连续的，而物理地址则无需连续。kmalloc确保页在物理地址上是连续的。vmalloc只确保页在虚拟地址空间内是连续的。

```c
void * vmalloc(unsigned long size)；
void vfree(const void *addr)；
```

vmalloc为了把物理上不连续的页转换为虚拟地址空间上连续的页，必须专门建立页表项，因此性能会不如kmalloc。

## slab层

分配和释放数据结构是所有内核中最普遍的操作之一，为了加速这些操作，linux提供了slab层，slab分配器扮演了**通用数据结构缓存层**的角色。

### slab层的设计

slab层把不同的对象划分为所谓的高速缓存组，其中每个高速缓存组都存放不同类型的对象。每个对象类型对应一个高速缓存。kmalloc()接口就是建立在slab层之上，使用了一组通用高速缓存。

高速缓存又被划分为slab，slab由一个或多个物理上连续的页组成，一般情况下，slab也仅仅由一页组成。每个高速缓存可以由多个slab组成。

每个slab都包含一些对象成员，这里的对象时指被缓存对的数据结构。每个slab处于三种状态之一：满、部分满或空。

在内核中每个高速缓存都是用kmem_cache结构来标识，这个结构包含三个链表slabs_full,slabs_partitial和slabs_empty。slab描述符struct slab用来描述每个slab:

```c
struct {
			struct list_head list;
			unsigned long colouroff;
			void *s_mem;		/* including colour offset */
			unsigned int inuse;	/* num of objs active in slab */
			kmem_bufctl_t free;
			unsigned short nodeid;
};
```

### slab分配器的接口

1. 创建高速缓存

   ```c
   struct kmem_cache * kmem_cache_create(const char *name, //高速缓存的名字
                                           size_t size, //每个元素的大小
                                           size_t align, //slab内第一个元素的偏移，用来确保业内进行特定的对齐，通常为0.
                                           unsigned long flags, //可选设置项，用来控制高速缓存的行为
                                           void (*ctor)(void *));//高速缓存的构造函数，不使用，置为NULL即可。
   ```

   kmem_cache_create()在成功会返回一个指向所处创造高速缓存的指针，否则返回NULL。该函数可能会睡眠。

   撤销高速缓存，也可能会睡眠。使用前须确保高速缓存中所有slab都必须为空，在调用kmem_cache_destory()过程中不再访问这个高速缓存。

   ```c
   //成功返回0，否则返回非0值
   int kmem_cache_destroy(struct kmem_cache *cachep)
   ```

2. 从缓存中分配

   创建缓存之后就可以在缓存中获取对象

   ```c
   //获取对象
   void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags)
   //释放
   void kmem_cache_free(struct kmem_cache *cachep, void *objp);
   ```



## 在栈上静态分配

内核栈小而且固定，历史上每个进程有两个页的内核栈，因此32和64位系统内核栈大小分别是8KB和16KB。



## 高端内存的映射

由于高端内存中的页不能永久地映射到内核地址空间，因此alloc_pages()函数以__GFP_HIGHMEM标志获得的页不可能有逻辑地址。必须映射到内核的逻辑地址空间上。

1. 永久映射

   ```c
   //可能会睡眠，在高端内存或低端内存上都能用
   void *kmap(struct page *page)
   //解除映射
   void *kunmap(struct page *page);
   ```

2. 临时映射

   当必须创建一个映射而当前的上下文又不能睡眠，内核提供了临时映射（也即原子映射）。

   ```c
   //type描述了临时映射的目的
   void *kmap_atomic(struct page *page, enum km_type type);
   //取消临时映射，也不会阻塞。
   void *kunmap_atomic(struct page *page, enum km_type type);
   ```

## Per-CPU变量 

每CPU变量对于给定处理器其数据是唯一的。**除了当前处理器外，没有其他处理器可以解触到这个数据**，不存在并发访问的问题。

```c
unsigned long my_percpu[NR_CPUS];
int cpu;
cpu = get_cpu();//获得当前处理器，并禁止内核抢占。
my_percpu[cpu]++;
put_cpu();//激活内核抢占
```

### 编译时的每CPU变量

```c
//定义
DEFINE_PER_CPU(type, name)
//声明
DECLARE_PER_CPU(type, name) 
//返回当前处理器上的每CPU变量，同时禁止内核抢占    
get_cpu_var(var)
//重新激活内核抢占
put_cpu_var(var)
//获取指定处理器上的每cpu变量
per_cpu(name,cpu)++
```

### 运行时的每CPU变量

```c
//分配类型是type的per cpu变量，返回per cpu变量的地址（注意：不是各个CPU上的副本）
void *alloc_percpu(type)
//释放ptr指向的per cpu变量空间
void free_percpu(void __percpu *ptr)

//这个接口是和访问静态Per-CPU变量的get_cpu_var接口是类似的
get_cpu_ptr()
put_cpu_ptr()
//根据per cpu变量的地址和cpu number，返回指定CPU number上该per cpu变量的地址
per_cpu_ptr(ptr, cpu)
```

参考[Linux内核同步机制之（二）：Per-CPU变量](http://www.wowotech.net/kernel_synchronization/per-cpu.html)