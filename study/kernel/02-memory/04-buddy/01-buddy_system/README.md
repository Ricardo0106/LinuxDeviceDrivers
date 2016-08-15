服务器体系与共享存储器架构
=======

| 日期 | 内核版本 | 架构| 作者 | GitHub| CSDN |
| ------- |:-------:|:-------:|:-------:|:-------:|:-------:|
| 2016-06-14 | [Linux-4.7](http://lxr.free-electrons.com/source/?v=4.7) | X86 & arm | [gatieme](http://blog.csdn.net/gatieme) | [LinuxDeviceDrivers](https://github.com/gatieme/LDD-LinuxDeviceDrivers) | [Linux内存管理](http://blog.csdn.net/gatieme/article/category/6225543) |




在内核初始化完成之后, 内存管理的责任就由伙伴系统来承担. 伙伴系统基于一种相对简单然而令人吃惊的强大算法.

Linux内核使用二进制伙伴算法来管理和分配物理内存页面, 该算法由Knowlton设计, 后来Knuth又进行了更深刻的描述.

伙伴系统是一个结合了2的方幂个分配器和空闲缓冲区合并计技术的内存分配方案, 其基本思想很简单. 内存被分成含有很多页面的大块, 每一块都是2个页面大小的方幂. 如果找不到想要的块, 一个大块会被分成两部分, 这两部分彼此就成为伙伴. 其中一半被用来分配, 而另一半则空闲. 这些块在以后分配的过程中会继续被二分直至产生一个所需大小的块. 当一个块被最终释放时, 其伙伴将被检测出来, 如果伙伴也空闲则合并两者.

*	内核如何记住哪些内存块是空闲的

*	分配空闲页面的方法

*	影响分配器行为的众多标识位

*	内存碎片的问题和分配器如何处理碎片




#2	伙伴系统的结构
-------


##2.1	伙伴系统数据结构
-------

系统内存中的每个物理内存页（页帧），都对应于一个struct page实例, 每个内存域都关联了一个struct zone的实例，其中保存了用于管理伙伴数据的主要数数组


```cpp
//  http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L324
struct zone
{
	 /* free areas of different sizes */
	struct free_area        free_area[MAX_ORDER];
};
```

struct free_area是一个伙伴系统的辅助数据结构, 它定义在[include/linux/mmzone.h?v=4.7, line 88](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L88)

```cpp
struct free_area {
	struct list_head        free_list[MIGRATE_TYPES];
unsigned long           nr_free;
};
```

| 字段 | 描述 |
|:-----:|:-----:|
| free_list | 是用于连接空闲页的链表. 页链表包含大小相同的连续内存区 |
| nr_free | 指定了当前内存区中空闲页块的数目（对0阶内存区逐页计算，对1阶内存区计算页对的数目，对2阶内存区计算4页集合的数目，依次类推 |



伙伴系统的分配器维护空闲页面所组成的块, 这里每一块都是2的方幂个页面, 方幂的指数称为**阶**.


阶是伙伴系统中一个非常重要的术语. 它描述了内存分配的数量单位. 内存块的长度是$2^order$, 其中order的范围从0到MAX_ORDER


zone->free_area[MAX_ORDER]数组中阶作为各个元素的索引, 用于指定对应链表中的连续内存区包含多少个页帧.

*	数组中第0个元素的阶为0, 它的free_list链表域指向具有包含区为单页($2^0=1$)的内存页面链表

*	数组中第1个元素的free_list域管理的内存区为两页($2^1=2$)

*	第3个管理的内存区为4页, 依次类推.

*	直到$2^{MAX_ORDER-1}$个页面大小的块

![空闲页快](../images/order_free_list.png)


##2.2	最大阶MAX_ORDER与FORCE_MAX_ZONEORDER配置选项
-------


一般来说`MAX_ORDER`默认定义为11, 这意味着一次分配可以请求的页数最大是2^11=2048, 参见[include/linux/mmzone.h?v=4.7, line 22](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L22)

```cpp
/* Free memory management - zoned buddy allocator.  */
#ifndef CONFIG_FORCE_MAX_ZONEORDER
#define MAX_ORDER 11
#else
#define MAX_ORDER CONFIG_FORCE_MAX_ZONEORDER
#endif
#define MAX_ORDER_NR_PAGES (1 << (MAX_ORDER - 1))
```



但如果特定于体系结构的代码设置了`FORCE_MAX_ZONEORDER`配置选项, 该值也可以手工改变

例如，IA-64系统上巨大的地址空间可以处理`MAX_ORDER = 18`的情形，而ARM或v850系统则使用更小的值(如8或9). 但这不一定是由计算机支持的内存数量比较小引起的，也可能是内存对齐方式的要求所导致


可以参考一些架构的Kconfig文件如下

| arm | arm64 |
|:----:|:-----:|:-------:|
| [arch/arm/Kconfig?v=4.7, line 1696](http://lxr.free-electrons.com/source/arch/arm/Kconfig?v=4.7#L1696) | [arch/arm64/Kconfig?v=4.7, line 679](http://lxr.free-electrons.com/source/arch/arm64/Kconfig?v=4.7#L679)


比如[arm64体系结构的Kconfig配置文件的描述](http://lxr.free-electrons.com/source/arch/arm64/Kconfig?v=4.7#L679) |

```cpp
config FORCE_MAX_ZONEORDER
int
default "14" if (ARM64_64K_PAGES && TRANSPARENT_HUGEPAGE)
default "12" if (ARM64_16K_PAGES && TRANSPARENT_HUGEPAGE)
default "11"`
```






##2.3	内存区是如何连接的
-------


内存区中第1页内的链表元素, 可用于将内存区维持在链表中。因此，也不必引入新的数据结构来管理物理上连续的页，否则这些页不可能在同一内存区中. 如下图所示


![伙伴系统中相互连接的内存区](../images/buddy_node_connect.png)


伙伴不必是彼此连接的. 如果一个内存区在分配其间分解为两半, 内核会自动将未用的一半加入到对应的链表中.

如果在未来的某个时刻, 由于内存释放的缘故, 两个内存区都处于空闲状态, 可通过其地址判断其是否为伙伴. 管理工作较少, 是伙伴系统的一个主要优点.

基于伙伴系统的内存管理专注于某个结点的某个内存域, 例如, DMA或高端内存域. 但所有内存域和结点的伙伴系统都通过备用分配列表连接起来.

下图说明了这种关系.

![伙伴系统和内存域／结点之间的关系](../images/buddy_and_node_zone.png)


最后要注意, 有关伙伴系统和当前状态的信息可以在/proc/buddyinfo中获取

![伙伴系统和当前状态的信息](../images/buddy_info.png)


上述输出给出了各个内存域中每个分配阶中空闲项的数目, 从左至右, 阶依次升高. 上面给出的信息取自4 GiB物理内存的AMD64系统.

#传统伙伴系统算法
-------

在内核分配内存时, 必须记录页帧的已分配或空闲状态, 以免两个进程使用同样的内存区域. 由于内存分配和释放非常频繁, 内核还必须保证相关操作尽快完成. 内核可以只分配完整的页帧. 将内存划分为更小的部分的工作, 则委托给用户空间中的标准库. 标准库将来源于内核的页帧拆分为小的区域, 并为进程分配内存.


内核中很多时候要求分配连续页. 为快速检测内存中的连续区域, 内核采用了一种古老而历经检验的技术: **伙伴系统**

系统中的空闲内存块总是两两分组, 每组中的两个内存块称作伙伴. 伙伴的分配可以是彼此独立的. 但如果两个伙伴都是空闲的, 内核会将其合并为一个更大的内存块, 作为下一层次上某个内存块的伙伴.

下图示范了该系统, 图中给出了一对伙伴, 初始大小均为8页. 即系统中所有的页面都是8页的.

![伙伴系统](../images/buddy_system.png)

内核对所有大小相同的伙伴（1、2、4、8、16或其他数目的页），都放置到同一个列表中管理. 各有8页的一对伙伴也在相应的列表中.

如果系统现在需要8个页帧, 则将16个页帧组成的块拆分为两个伙伴. 其中一块用于满足应用程序的请求, 而剩余的8个页帧则放置到对应8页大小内存块的列表中.

如果下一个请求只需要2个连续页帧, 则由8页组成的块会分裂成2个伙伴, 每个包含4个页帧. 其中一块放置回伙伴列表中，而另一个再次分裂成2个伙伴, 每个包含2页。其中一个回到伙伴系统，另一个则传递给应用程序.

在应用程序释放内存时, 内核可以直接检查地址, 来判断是否能够创建一组伙伴, 并合并为一个更大的内存块放回到伙伴列表中, 这刚好是内存块分裂的逆过程。这提高了较大内存块可用的可能性.

在系统长期运行时，服务器运行几个星期乃至几个月是很正常的，许多桌面系统也趋向于长期开机运行，那么会发生称为碎片的内存管理问题。频繁的分配和释放页帧可能导致一种情况：系统中有若干页帧是空闲的，但却散布在物理地址空间的各处。换句话说，系统中缺乏连续页帧组成的较大的内存块，而从性能上考虑，却又很需要使用较大的连续内存块。通过伙伴系统可以在某种程度上减少这种效应，但无法完全消除。如果在大块的连续内存中间刚好有一个页帧分配出去，很显然这两块空闲的内存是无法合并的.

在内核版本2.6.24之后, 增加了一些有效措施来防止内存碎片.


#3	避免碎片
-------

在第1章给出的简化说明中, 一个双链表即可满足伙伴系统的所有需求. 在内核版本2.6.23之前, 的确是这样. 但在内核2.6.24开发期间, 内核开发者对伙伴系统的争论持续了相当长时间. 这是因为伙伴系统是内核最值得尊敬的一部分，对它的改动不会被大家轻易接受


##3.1	内存碎片


伙伴系统的基本原理已经在第1章中讨论过，其方案在最近几年间确实工作得非常好。但在Linux内存管理方面，有一个长期存在的问题：在系统启动并长期运行后，物理内存会产生很多碎片。该情形如下图所示

![物理内存的碎片](../images/physical_memory_fragmentation.png)

假定内存由60页组成，这显然不是超级计算机，但用于示例却足够了。左侧的地址空间中散布着空闲页。尽管大约25%的物理内存仍然未分配，但最大的连续空闲区只有一页. 这对用户空间应用程序没有问题：其内存是通过页表映射的，无论空闲页在物理内存中的分布如何，应用程序看到的内存
似乎总是连续的。右图给出的情形中，空闲页和使用页的数目与左图相同，但所有空闲页都位于一个连续区中。

但对内核来说，碎片是一个问题. 由于(大多数)物理内存一致映射到地址空间的内核部分, 那么在左图的场景中, 无法映射比一页更大的内存区. 尽管许多时候内核都分配的是比较小的内存, 但也有时候需要分配多于一页的内存. 显而易见, 在分配较大内存的情况下, 右图中所有已分配页和空闲页都处于连续内存区的情形，是更为可取的.

很有趣的一点是, 在大部分内存仍然未分配时, 就也可能发生碎片问题. 考虑图3-25的情形.

只分配了4页，但可分配的最大连续区只有8页，因为伙伴系统所能工作的分配范围只能是2的幂次.


![物理内存的碎片](../images/some_memory_fragmentation.png)

我提到内存碎片只涉及内核，这只是部分正确的。大多数现代CPU都提供了使用巨型页的可能性，比普通页大得多。这对内存使用密集的应用程序有好处。在使用更大的页时，地址转换后备缓冲器只需处理较少的项，降低了TLB缓存失效的可能性。但分配巨型页需要连续的空闲物理内存！

很长时间以来，物理内存的碎片确实是Linux的弱点之一。尽管已经提出了许多方法，但没有哪个方法能够既满足Linux需要处理的各种类型工作负荷提出的苛刻需求，同时又对其他事务影响不大。




##3.2	依据可移动性组织页
-------


在内核2.6.24开发期间，防止碎片的方法最终加入内核。在我讨论具体策略之前，有一点需要澄清。

文件系统也有碎片，该领域的碎片问题主要通过碎片合并工具解决。它们分析文件系统，重新排序已分配存储块，从而建立较大的连续存储区. 理论上，该方法对物理内存也是可能的，但由于许多物理内存页不能移动到任意位置，阻碍了该方法的实施。因此，内核的方法是反碎片(anti-fragmentation), 即试图从最初开始尽可能防止碎片.

<font color=0x00ffff>
反碎片的工作原理如何?
</font>


为理解该方法，我们必须知道内核将已分配页划分为下面3种不同类型。


| 页面类型 | 描述 | 举例 |
|:---------:|:-----:|:-----:|
| 不可移动页 | 在内存中有固定位置, **不能移动**到其他地方. | 核心内核分配的大多数内存属于该类别 |
| 可移动页 | **可以随意地移动**. | 属于用户空间应用程序的页属于该类别. 它们是通过页表映射的<br>如果它们复制到新位置，页表项可以相应地更新，应用程序不会注意到任何事 |
| 可回收页 | **不能直接移动, 但可以删除, 其内容可以从某些源重新生成**. | 例如，映射自文件的数据属于该类别<br>kswapd守护进程会根据可回收页访问的频繁程度，周期性释放此类内存. , 页面回收本身就是一个复杂的过程. 内核会在可回收页占据了太多内存时进行回收, 在内存短缺(即分配失败)时也可以发起页面回收. |



页的可移动性，依赖该页属于3种类别的哪一种. 内核使用的**反碎片技术**, 即基于将具有相同可移动性的页分组的思想.

<font color=0x00ffff>
为什么这种方法有助于减少碎片?
</font>

由于页无法移动, 导致在原本几乎全空的内存区中无法进行连续分配. 根据页的可移动性, 将其分配到不同的列表中, 即可防止这种情形. 例如, 不可移动的页不能位于可移动内存区的中间, 否则就无法从该内存区分配较大的连续内存块.


想一下, 上图中大多数空闲页都属于可回收的类别, 而分配的页则是不可移动的. 如果这些页聚集到两个不同的列表中, 如下图所示. 在不可移动页中仍然难以找到较大的连续空闲空间, 但对可回收的页, 就容易多了.


![减少内存碎片](../images/little_memory_fragmentation.png)


但要注意, 从最初开始, 内存并未划分为可移动性不同的区. 这些是在运行时形成的. 内核的另一种方法确实将内存分区, 分别用于可移动页和不可移动页的分配, 我会下文讨论其工作原理. 但这种划分对这里描述的方法是不必要的



##3.3	避免碎片数据结构
-------



###3.3.1	迁移类型
-------


尽管内核使用的反碎片技术卓有成效，它对伙伴分配器的代码和数据结构几乎没有影响。内核定义了一些枚举常量(早期用宏来实现)来表示不同的迁移类型, 参见[include/linux/mmzone.h?v=4.7, line 38](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L38)

```cpp
enum {
        MIGRATE_UNMOVABLE,
        MIGRATE_MOVABLE,
        MIGRATE_RECLAIMABLE,
        MIGRATE_PCPTYPES,       /* the number of types on the pcp lists */
        MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
#ifdef CONFIG_CMA
        /*
         * MIGRATE_CMA migration type is designed to mimic the way
         * ZONE_MOVABLE works.  Only movable pages can be allocated
         * from MIGRATE_CMA pageblocks and page allocator never
         * implicitly change migration type of MIGRATE_CMA pageblock.
         *
         * The way to use it is to change migratetype of a range of
         * pageblocks to MIGRATE_CMA which can be done by
         * __free_pageblock_cma() function.  What is important though
         * is that a range of pageblocks must be aligned to
         * MAX_ORDER_NR_PAGES should biggest page be bigger then
         * a single pageblock.
         */
        MIGRATE_CMA,
#endif
#ifdef CONFIG_MEMORY_ISOLATION
        MIGRATE_ISOLATE,        /* can't allocate from here */
#endif
        MIGRATE_TYPES
};
```


|  宏  | 类型 |
|:----:|:-----:|
| MIGRATE_UNMOVABLE | 不可移动页 |
| MIGRATE_MOVABLE | 可移动页 |
| MIGRATE_RECLAIMABLE | 可回收页 |
| MIGRATE_PCPTYPES | 是per_cpu_pageset, 即用来表示每CPU页框高速缓存的数据结构中的链表的迁移类型数目 |
| MIGRATE_HIGHATOMIC |  = MIGRATE_PCPTYPES, 在罕见的情况下，内核需要分配一个高阶的页面块而不能休眠.如果向具有特定可移动性的列表请求分配内存失败，这种紧急情况下可从MIGRATE_HIGHATOMIC中分配内存 |
| MIGRATE_CMA | Linux内核最新的连续内存分配器(CMA), 用于避免预留大块内存 |
| MIGRATE_ISOLATE | 是一个特殊的虚拟区域, 用于跨越NUMA结点移动物理内存页. 在大型系统上, 它有益于将物理内存页移动到接近于使用该页最频繁的CPU. |
| MIGRATE_TYPES | 只是表示迁移类型的数目, 也不代表具体的区域 |

对于MIGRATE_CMA类型, 其中在我们使用ARM等嵌入式Linux系统的时候, 一个头疼的问题是GPU, Camera, HDMI等都需要预留大量连续内存，这部分内存平时不用，但是一般的做法又必须先预留着. 目前, Marek Szyprowski和Michal Nazarewicz实现了一套全新的Contiguous Memory Allocator. 通过这套机制, 我们可以做到不预留内存，这些内存平时是可用的，只有当需要的时候才被分配给Camera，HDMI等设备. 参照[宋宝华--Linux内核最新的连续内存分配器(CMA)——避免预留大块内存](http://21cnbao.blog.51cto.com/109393/898846/), 内核为此提供了函数is_migrate_cma来检测当前类型是否为MIGRATE_CMA, 该函数定义在[include/linux/mmzone.h?v=4.7, line 69](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L69)

```cpp
/* In mm/page_alloc.c; keep in sync also with show_migration_types() there */
extern char * const migratetype_names[MIGRATE_TYPES];

#ifdef CONFIG_CMA
#  define is_migrate_cma(migratetype) unlikely((migratetype) == MIGRATE_CMA)
#else
#  define is_migrate_cma(migratetype) false
#endif
```




对伙伴系统数据结构的主要调整, 是将空闲列表分解为MIGRATE_TYPE个列表, 可以参见free_area的定义[include/linux/mmzone.h?v=4.7, line 88](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L88)

```cpp
struct free_area
{
	struct list_head        free_list[MIGRATE_TYPES];
unsigned long           nr_free;
};
```

*	nr_free统计了所有列表上空闲页的数目，而每种迁移类型都对应于一个空闲列表


宏for_each_migratetype_order(order, type)可用于迭代指定迁移类型的所有分配阶

```cpp
#define for_each_migratetype_order(order, type) \
        for (order = 0; order < MAX_ORDER; order++) \
                for (type = 0; type < MIGRATE_TYPES; type++)
```


###3.3.2	迁移备用列表fallbacks
-------


<font color = 0x00ffff>
如果内核无法满足针对某一给定迁移类型的分配请求, 会怎么样?
</font>


此前已经出现过一个类似的问题, 即特定的NUMA内存域无法满足分配请求时. 我们需要从其他内存域中选择一个代价最低的内存域完成内存的分配, 因此内核在内存的结点pg_data_t中提供了一个备用内存域列表zonelists.

内核在内存迁移的过程中处理这种情况下的做法是类似的. 提供了一个备用列表fallbacks, 规定了在指定列表中无法满足分配请求时. 接下来应使用哪一种迁移类型, 定义在[mm/page_alloc.c?v=4.7, line 1799](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L1799)


```cpp
/*
 * This array describes the order lists are fallen back to when
 * the free lists for the desirable migrate type are depleted
 * 该数组描述了指定迁移类型的空闲列表耗尽时
 * 其他空闲列表在备用列表中的次序
 */
static int fallbacks[MIGRATE_TYPES][4] = {
	//  分配不可移动页失败的备用列表
    [MIGRATE_UNMOVABLE]   = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE,   MIGRATE_TYPES },
    //  分配可回收页失败时的备用列表
    [MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE,   MIGRATE_MOVABLE,   MIGRATE_TYPES },
    //  分配可移动页失败时的备用列表
    [MIGRATE_MOVABLE]     = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE, MIGRATE_TYPES },
#ifdef CONFIG_CMA
    [MIGRATE_CMA]     = { MIGRATE_TYPES }, /* Never used */
#endif
#ifdef CONFIG_MEMORY_ISOLATION
    [MIGRATE_ISOLATE]     = { MIGRATE_TYPES }, /* Never used */
#endif
};
```

>该数据结构大体上是自明的 :
>
>每一行对应一个类型的备用搜索域的顺序, 在内核想要分配不可移动页`MIGRATE_UNMOVABLE`时, 如果对应链表为空, 则遍历fallbacks[MIGRATE_UNMOVABLE], 首先后退到可回收页链表`MIGRATE_RECLAIMABLE`, 接下来到可移动页链表`MIGRATE_MOVABLE`, 最后到紧急分配链表`MIGRATE_TYPES`.


##3.3.3	pageblock_order变量
-------


全局变量和辅助函数尽管页可移动性分组特性总是编译到内核中，但只有在系统中有足够内存可以分配到多个迁移类型对应的链表时，才是有意义的。由于每个迁移链表都应该有适当数量的内存，内核需要定义"适当"的概念. 这是通过两个全局变量pageblock_order和pageblock_nr_pages提供的. 第一个表示内核认为是"大"的一个分配阶, pageblock_nr_pages则表示该分配阶对应的页数。如果体系结构提供了巨型页机制, 则pageblock_order通常定义为巨型页对应的分配阶. 定义在[include/linux/pageblock-flags.h?v=4.7, line 44](http://lxr.free-electrons.com/source/include/linux/pageblock-flags.h?v=4.7#L42)


```cpp
#ifdef CONFIG_HUGETLB_PAGE

    #ifdef CONFIG_HUGETLB_PAGE_SIZE_VARIABLE

        /* Huge page sizes are variable */
        extern unsigned int pageblock_order;

    #else /* CONFIG_HUGETLB_PAGE_SIZE_VARIABLE */

    /* Huge pages are a constant size */
        #define pageblock_order         HUGETLB_PAGE_ORDER

    #endif /* CONFIG_HUGETLB_PAGE_SIZE_VARIABLE */

#else /* CONFIG_HUGETLB_PAGE */

    /* If huge pages are not used, group by MAX_ORDER_NR_PAGES */
    #define pageblock_order         (MAX_ORDER-1)

#endif /* CONFIG_HUGETLB_PAGE */

#define pageblock_nr_pages      (1UL << pageblock_order)
```

在IA-32体系结构上, 巨型页长度是4MB, 因此每个巨型页由1024个普通页组成, 而HUGETLB_PAGE_ORDER则定义为10. 相比之下, IA-64体系结构允许设置可变的普通和巨型页长度, 因此HUGETLB_PAGE_ORDER的值取决于内核配置.

如果体系结构不支持巨型页, 则将其定义为第二高的分配阶, 即`MAX_ORDER - 1`

```cpp
/* If huge pages are not used, group by MAX_ORDER_NR_PAGES */
#define pageblock_order         (MAX_ORDER-1)
```

如果各迁移类型的链表中没有一块较大的连续内存, 那么页面迁移不会提供任何好处, 因此在可用内存太少时内核会关闭该特性. 这是在build_all_zonelists函数中检查的, 该函数用于初始化内存域列表. 如果没有足够的内存可用, 则全局变量[`page_group_by_mobility_disabled`](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L79)设置为0, 否则设置为1.

内核如何知道给定的分配内存属于何种迁移类型?

我们将在以后讲解, 有关各个内存分配的细节都通过分配掩码指定.

内核提供了两个标志，分别用于表示分配的内存是可移动的(\__GFP_MOVABLE)或可回收的(\__GFP_RECLAIMABLE).



###3.3.4	gfpflags_to_migratetype函数
-------


如果这些标志都没有设置, 则分配的内存假定为不可移动的. 辅助函数gfpflags_to_migratetype可用于转换分配标志及对应的迁移类型, 该函数定义在[include/linux/gfp.h?v=4.7, line 266](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L266)

```cpp
static inline int gfpflags_to_migratetype(const gfp_t gfp_flags)
{
    VM_WARN_ON((gfp_flags & GFP_MOVABLE_MASK) == GFP_MOVABLE_MASK);
    BUILD_BUG_ON((1UL << GFP_MOVABLE_SHIFT) != ___GFP_MOVABLE);
    BUILD_BUG_ON((___GFP_MOVABLE >> GFP_MOVABLE_SHIFT) != MIGRATE_MOVABLE);

    if (unlikely(page_group_by_mobility_disabled))
        return MIGRATE_UNMOVABLE;

    /* Group based on mobility */
    return (gfp_flags & GFP_MOVABLE_MASK) >> GFP_MOVABLE_SHIFT;
}
```

>linux-2.6.x的内核中转换分配标志及对应的迁移类型的辅助函数为allocflags_to_migratetype, 这个名字会有歧义的, 让我们误以为参数的标识中有alloc flags, 但是其实并不然, 因此后来的内核中将该函数更名为gfpflags_to_migratetype, 参见[Rename it to gfpflags_to_migratetype()](https://patchwork.kernel.org/patch/4291831)

在2.6.25中为如下接口

```cpp
/* Convert GFP flags to their corresponding migrate type */
static inline int allocflags_to_migratetype(gfp_t gfp_flags)
{
    WARN_ON((gfp_flags & GFP_MOVABLE_MASK) == GFP_MOVABLE_MASK);

    if (unlikely(page_group_by_mobility_disabled))
        return MIGRATE_UNMOVABLE;

    /* Group based on mobility */
    return (((gfp_flags & __GFP_MOVABLE) != 0) << 1) |
        ((gfp_flags & __GFP_RECLAIMABLE) != 0);
}
````

如果停用了页面迁移特性, 则所有的页都是不可移动的. 否则. 该函数的返回值可以直接用作free_area.free_list的数组索引.

###3.3.5	pageblock_flags变量与其函数接口

最后要注意, 每个内存域都提供了一个特殊的字段, 可以跟踪包含pageblock_nr_pages个页的内存区的属性. 即zone->pageblock_flags字段, 当前只有与页可移动性相关的代码使用, 参见[include/linux/mmzone.h?v=4.7, line 367](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L367)


```cpp
struct zone
{
#ifndef CONFIG_SPARSEMEM
    /*
     * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
     * In SPARSEMEM, this map is stored in struct mem_section
     */
    unsigned long       *pageblock_flags;
#endif /* CONFIG_SPARSEMEM */
};
```

在初始化期间, 内核自动确保对内存域中的每个不同的迁移类型分组, 在pageblock_flags中都分配了足够存储NR_PAGEBLOCK_BITS个比特位的空间。当前，表示一个连续内存区的迁移类型需要3个比特位, 参见[include/linux/pageblock-flags.h?v=4.7, line 28](http://lxr.free-electrons.com/source/include/linux/pageblock-flags.h?v=4.7#L28)

```cpp
/* Bit indices that affect a whole block of pages */
enum pageblock_bits {
    PB_migrate,
    PB_migrate_end = PB_migrate + 3 - 1,
            /* 3 bits required for migrate types */
    PB_migrate_skip,/* If set the block is skipped by compaction */

    /*
     * Assume the bits will always align on a word. If this assumption
     * changes then get/set pageblock needs updating.
     */
    NR_PAGEBLOCK_BITS
};
```


内核提供`set_pageblock_migratetype`负责设置以page为首的一个内存区的迁移类型, 该函数定义在[mm/page_alloc.c?v=4.7, line 458](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L458), 如下所示


```cpp
void set_pageblock_migratetype(struct page *page, int migratetype)
{
    if (unlikely(page_group_by_mobility_disabled &&
             migratetype < MIGRATE_PCPTYPES))
        migratetype = MIGRATE_UNMOVABLE;

    set_pageblock_flags_group(page, (unsigned long)migratetype,
                    PB_migrate, PB_migrate_end);
}
```



`migratetype`参数可以通过上文介绍的`gfpflags_to_migratetype`辅助函数构建. 请注意很重要的一点, 页的迁移类型是预先分配好的, 对应的比特位总是可用, 与页是否由伙伴系统管理无关. 在释放内存时，页必须返回到正确的迁移链表。这之所以可行，是因为能够从`get_pageblock_migratetype`获得所需的信息. 参见[include/linux/mmzone.h?v=4.7, line 84](http://lxr.free-electrons.com/source/include/linux/mmzone.h?v=4.7#L84)


```cpp
#define get_pageblock_migratetype(page)                                 \
        get_pfnblock_flags_mask(page, page_to_pfn(page),                \
                        PB_migrate_end, MIGRATETYPE_MASK)
```


最后请注意, 在各个迁移链表之间, 当前的页面分配状态可以从`/proc/pagetypeinfo`获得.


![proc/pagetypeinfo]()




##初始化基于可移动性的分组
-------




在内存子系统初始化期间, memmap_init_zone负责处理内存域的page实例. 该函数定义在[mm/page_alloc.c?v=4.7, line 5139](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L5139), 该函数完成了一些不怎么有趣的标准初始化工作，但其中有一件是实质性的，即所有的页最初都标记为可移动的. 参见[mm/page_alloc.c?v=4.7, line 5224](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L5224)


```cpp
/*
 * Initially all pages are reserved - free ones are freed
 * up by free_all_bootmem() once the early boot process is
 * done. Non-atomic initialization, single-pass.
 */
void __meminit memmap_init_zone(unsigned long size, int nid, unsigned long zone,
        unsigned long start_pfn, enum memmap_context context)
{
    /*  ......  */

    for (pfn = start_pfn; pfn < end_pfn; pfn++) {
        /*  ......  */
not_early:
        if (!(pfn & (pageblock_nr_pages - 1))) {
            struct page *page = pfn_to_page(pfn);

            __init_single_page(page, pfn, zone, nid);
            set_pageblock_migratetype(page, MIGRATE_MOVABLE);
        } else {
            __init_single_pfn(pfn, zone, nid);
        }
    }
}
```


在分配内存时, 如果必须"盗取"不同于预定迁移类型的内存区, 内核在策略上倾向于"盗取"更大的内存区. 由于所有页最初都是可移动的, 那么在内核分配不可移动的内存区时, 则必须"盗取".

实际上, 在启动期间分配可移动内存区的情况较少, 那么分配器有很高的几率分配长度最大的内存区, 并将其从可移动列表转换到不可移动列表. 由于分配的内存区长度是最大的, 因此不会向可移动内存中引入碎片.

总而言之, 这种做法避免了启动期间内核分配的内存(经常在系统的整个运行时间都不释放)散布到物理内存各处, 从而使其他类型的内存分配免受碎片的干扰，这也是页可移动性分组框架的最重要的目标之一.


#4	分配器API
-------


##4.1	分配内存的接口
-------

就伙伴系统的接口而言, NUMA或UMA体系结构是没有差别的, 二者的调用语法都是相同的.

所有函数的一个共同点是 : 只能分配2的整数幂个页.

因此，接口中不像C标准库的malloc函数或bootmem和memblock分配器那样指定了所需内存大小作为参数. 相反, 必须指定的是分配阶, 伙伴系统将在内存中分配$2^order$页. 内核中细粒度的分配只能借助于slab分配器(或者slub、slob分配器), 后者基于伙伴系统


| 内存分配函数 | 功能 | 定义 |
|:-----:|:-----:|
| alloc_pages(mask, order) | 分配$2^order$页并返回一个struct page的实例，表示分配的内存块的起始页 | [NUMA-include/linux/gfp.h, line 466](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L466)<br>[UMA-include/linux/gfp.h?v=4.7, line 476](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L476) |
| alloc_page(mask) | 是前者在order = 0情况下的简化形式，只分配一页 |  [include/linux/gfp.h?v=4.7, line 483](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L483) |
| get_zeroed_page(mask) | 分配一页并返回一个page实例，页对应的内存填充0（所有其他函数，分配之后页的内容是未定义的） | [mm/page_alloc.c?v=4.7, line 3900](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L3900)| |
| [__get_free_pages(mask, order)](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L3883)<br>[__get_free_page(mask)](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L500) | 工作方式与上述函数相同，但返回分配内存块的虚拟地址，而不是page实例 |
| get_dma_pages(gfp_mask, order) | 用来获得适用于DMA的页. | [include/linux/gfp.h?v=4.7, line 503](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L503) |


在空闲内存无法满足请求以至于分配失败的情况下，所有上述函数都返回空指针(比如alloc_pages和alloc_page)或者0(比如get_zeroed_page、__get_free_pages和__get_free_page).

因此内核在各次分配之后都必须检查返回的结果. 这种惯例与设计得很好的用户层应用程序没什么不同, 但在内核中忽略检查会导致严重得多的故障


内核除了伙伴系统函数之外, 还提供了其他内存管理函数. 它们以伙伴系统为基础, 但并不属于伙伴分配器自身. 这些函数包括vmalloc和vmalloc_32, 使用页表将不连续的内存映射到内核地址空间中, 使之看上去是连续的.

还有一组kmalloc类型的函数, 用于分配小于一整页的内存区. 其实现
将在本章后续的几节分别讨论。

## 释放函数
-------

有4个函数用于释放不再使用的页，与所述函数稍有不同


| 内存释放函数 | 描述 |
|:--------------:|:-----:|
| [free_page(struct page *)](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L520)<br>[free_pages(struct page *, order)](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L3918) | 用于将一个或2order页返回给内存管理子系统。内存区的起始地址由指向该内存区的第一个page实例的指针表示 |
| [__free_page(addr)](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L519)<br>[__free_pages(addr, order)](http://lxr.free-electrons.com/source/mm/page_alloc.c?v=4.7#L3906) | 类似于前两个函数，但在表示需要释放的内存区时，使用了虚拟内存地址而不是page实例 |



##4.2	分配掩码
-------


前述所有函数中强制使用的mask参数，到底是什么语义?

我们知道Linux将内存划分为内存域. 内核提供了所谓的内存域修饰符(zone modifier)(在掩码的最低4个比特位定义), 来指定从哪个内存域分配所需的页.


参见[include/linux/gfp.h?v=4.7, line 12~374](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L12)

```cpp
//  http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7

//  line 12 ~ line 44
/* Plain integer GFP bitmasks. Do not use this directly. */
#define ___GFP_DMA              0x01u
#define ___GFP_HIGHMEM          0x02u
#define ___GFP_DMA32            0x04u
#define ___GFP_MOVABLE          0x08u
/*  ......  */

// line 46 ~ line 192
#define __GFP_DMA       ((__force gfp_t)___GFP_DMA)
#define __GFP_HIGHMEM   ((__force gfp_t)___GFP_HIGHMEM)
#define __GFP_DMA32     ((__force gfp_t)___GFP_DMA32)
#define __GFP_MOVABLE   ((__force gfp_t)___GFP_MOVABLE)  /* ZONE_MOVABLE allowed */
#define GFP_ZONEMASK    (__GFP_DMA|__GFP_HIGHMEM|__GFP_DMA32|__GFP_MOVABLE)

// line 194 ~ line 260
#define GFP_DMA         __GFP_DMA
#define GFP_DMA32       __GFP_DMA32
```


其中GFP缩写的意思为获取空闲页(get free page), __GFP_MOVABLE不表示物理内存域, 但通知内核应在特殊的虚拟内存域ZONE_MOVABLE进行相应的分配.


我们从注释中找到这样的信息, 可以作为参考

```cpp
bit       result
=================
0x0    => NORMAL
0x1    => DMA or NORMAL
0x2    => HIGHMEM or NORMAL
0x3    => BAD (DMA+HIGHMEM)
0x4    => DMA32 or DMA or NORMAL
0x5    => BAD (DMA+DMA32)
0x6    => BAD (HIGHMEM+DMA32)
0x7    => BAD (HIGHMEM+DMA32+DMA)
0x8    => NORMAL (MOVABLE+0)
0x9    => DMA or NORMAL (MOVABLE+DMA)
0xa    => MOVABLE (Movable is valid only if HIGHMEM is set too)
0xb    => BAD (MOVABLE+HIGHMEM+DMA)
0xc    => DMA32 (MOVABLE+DMA32)
0xd    => BAD (MOVABLE+DMA32+DMA)
0xe    => BAD (MOVABLE+DMA32+HIGHMEM)
0xf    => BAD (MOVABLE+DMA32+HIGHMEM+DMA)

GFP_ZONES_SHIFT must be <= 2 on 32 bit platforms.
```



很有趣的一点是，没有\__GFP_NORMAL常数，而内存分配的主要负担却落到ZONE_NORMAL内存域

内核考虑到这一点, 提供了一个函数gfp_zone来计算与给定分配标志兼容的最高内存域. 那么内存分配可以从该内存域或更低的内存域进行, 该函数定义在[include/linux/gfp.h?v=4.7, line 394](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L394)

```cpp
static inline enum zone_type gfp_zone(gfp_t flags)
{
    enum zone_type z;
    int bit = (__force int) (flags & GFP_ZONEMASK);

    z = (GFP_ZONE_TABLE >> (bit * GFP_ZONES_SHIFT)) &
                     ((1 << GFP_ZONES_SHIFT) - 1);
    VM_BUG_ON((GFP_ZONE_BAD >> bit) & 1);
    return z;
}
```

其中GFP_ZONES_SHIFT的定义如下, 在[include/linux/gfp.h?v=4.7, line 337](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L337)


```cpp
#if defined(CONFIG_ZONE_DEVICE) && (MAX_NR_ZONES-1) <= 4
/* ZONE_DEVICE is not a valid GFP zone specifier */
#define GFP_ZONES_SHIFT 2
#else
#define GFP_ZONES_SHIFT ZONES_SHIFT
#endif

#if 16 * GFP_ZONES_SHIFT > BITS_PER_LONG
#error GFP_ZONES_SHIFT too large to create GFP_ZONE_TABLE integer
#endif
```


由于内存域修饰符的解释方式不是那么直观, 表3-7给出了该函数结果的一个例子, 其中DMA和DMA32内存域相同. 假定在下文中没有设置\__GFP_MOVABLE修饰符.

| 修饰符 | 扫描的内存域 |
|:-------:|:--------------:|
| 无 | ZONE_NORMAL、ZONE_DMA |
| \__GFP_DMA | ZONE_DMA |
| \__GFP_DMA & \__GFP_HIGHMEM | ZONE_DMA |
| \__GFP_HIGHMEM | ZONE_HIGHMEM、ZONE_NORMAL、ZONE_DMA |

*	如果\__GFP_DMA和\__GFP_HIGHMEM都没有设置, 则首先扫描ZONE_NORMAL, 后面是ZONE_DMA

*	如果设置了\__GFP_HIGHMEM没有设置__GFP_DMA，则结果是从ZONE_HIGHMEM开始扫描所有3个内存域。=

*	如果设置了__GFP_DMA，那么\__GFP_HIGHMEM设置与否没有关系. 只有ZONE_DMA用于3种情形. 这是合理的, 因为同时使用\__GFP_HIGHMEM和__GFP_DMA没有意义. 高端内存从来都不适用于DMA


设置\__GFP_MOVABLE不会影响内核的决策，除非它与\__GFP_HIGHMEM同时指定. 在这种情况下, 会使用特殊的虚拟内存域ZONE_MOVABLE满足内存分配请求. 对前文描述的内核的反碎片策略而言, 这种行为是必要的.

除了内存域修饰符之外, 掩码中还可以设置一些标志. 

下图中给出了掩码的布局，以及与各个比特位置关联的常数. \__GFP_DMA32出现了几次，因为它可能位于不同的地方.


![GFP掩码的布局](../images/gfp_flag_mask.png)



与内存域修饰符相反, 这些额外的标志并不限制从哪个物理内存段分配内存, 但确实可以改变分配器的行为. 例如, 它们可以修改查找空闲内存时的积极程度. 内核源代码中定义的下列标志, 定义在[include/linux/gfp.h?v=4.7, line 23](http://lxr.free-electrons.com/source/include/linux/gfp.h?v=4.7#L23)


```cpp
#define ___GFP_HIGH             0x20u		/* 应该访问紧急分配池？ */
#define ___GFP_IO               0x40u		/* 可以启动物理IO？ */
#define ___GFP_FS               0x80u		/* 可以调用底层文件系统？ */
#define ___GFP_COLD             0x100u	   /* 需要非缓存的冷页 */
#define ___GFP_NOWARN           0x200u	   /* 禁止分配失败警告 */
#define ___GFP_REPEAT           0x400u	   /* 重试分配，可能失败 */
#define ___GFP_NOFAIL           0x800u	   /* 一直重试，不会失败 */
#define ___GFP_NORETRY          0x1000u	  /* 不重试，可能失败 */
#define ___GFP_MEMALLOC         0x2000u  	/* 不使用紧急分配链表 */
#define ___GFP_COMP             0x4000u
#define ___GFP_ZERO             0x8000u
#define ___GFP_NOMEMALLOC       0x10000u
#define ___GFP_HARDWALL         0x20000u
#define ___GFP_THISNODE         0x40000u	 /* 没有备用结点，没有策略 */
#define ___GFP_ATOMIC           0x80000u
#define ___GFP_ACCOUNT          0x100000u
#define ___GFP_NOTRACK          0x200000u
#define ___GFP_DIRECT_RECLAIM   0x400000u
#define ___GFP_OTHER_NODE       0x800000u
#define ___GFP_WRITE            0x1000000u
#define ___GFP_KSWAPD_RECLAIM   0x2000000u
```

