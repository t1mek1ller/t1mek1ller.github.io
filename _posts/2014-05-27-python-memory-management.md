---
layout: post
title: Python之内存管理
description: "Python之内存管理"
modified: 2014-05-27
tags: [Python]
categories: [Python]
---

之前分析过[C++中STL的内存管理](http://atimekiller.github.io/cpp-stl-memory-management/)，在读完《Python源码剖析》之后决定再来总结一下Python中的内存管理机制，以进一步体会内存管理这个主题。

一般内存管理都要涉及到小块内存的管理，这点一般都采用内存池来实现，而另一方面，是如何归还向系统申请的内存，这一点C++是通过RAII(Resource Acquisition Is Initialization)来进行的，也即是资源分配即初始化，保证在任何情况下，先构造对象，再析构对象，是一种基于对象的资源管理办法，建立在C++对构造函数、析构函数、拷贝函数的基础之上（见effective c++条款13），而Python的内存管理则采用和Java类似的垃圾回收机制（Garbage Collection），这里暂不讨论RAII和GC孰优孰劣（个人更倾向于RAII，因为它是确定的，而且“对象在，资源在，对象没，资源没” 的理念难道不是更符合认知吗？）。下面仅仅分析Python中比STL中更为复杂和精巧的内存池的实现。

## 内存管理架构
先看一下Python的内存管理架构，Python中内存管理分为四个层次：

1. 最底层的C函数库层（malloc, free）
2. PyMem_API，对C内存管理函数进行了一层封装，以兼容各个平台，提供可移植性
3. PyObj_API，提供更高抽象层次的内存管理接口，以PyObject_Malloc，PyObject_Realloc，PyObject_Free示人
4. 针对不同类型对象（int，string，list，dict...）的对象缓冲池机制

Python中的对小内存区块的管理大部分工作就是实现PyObj_API这一层接口，相当于STL中的第二级配置器，而PyMem_API就是STL第一级配置器。

## 内存池

### 内存池全景
Python源码剖析（本文所有图均截自此文献）给出了Python处理小块内块的内存池全景，如下图所示（看起来好复杂-.-）：

{% capture images %}
    /images/python-memory-1.png
{% endcapture %}
{% include gallery images=images caption="Python内存池全景" cols=1 %}

可以从图中看到主要包括三层结构和一个usedpool缓冲池，由下到上分为block，pool，arena。pool管理着一堆block，arena管理着一堆pool。

### block
block内存配置的最小单位，容量是8的整数倍，最小为8字节，最大为256字节的内存块，也就是说Python中对于256字节以下的采用PyObj函数族，256字节以上的交给PyMem函数族。这点和STL中类似（STL中用数组维护了类似的链表，最大为128字节）。

### pool
pool管理着一堆固定大小的内存块block，一个pool的大小通常是一个系统内存页（一般为4KB），其结构定义如下：

{% highlight C++ %}
     /* When you say memory, my mind reasons in terms of (pointers to) blocks */
typedef uchar block;

/* Pool for small blocks. */
struct pool_header {
    union { block *_padding ;
            uint count ; } ref ;          /* number of allocated blocks    */
    block *freeblock ;                   /* pool's free list head         */
    struct pool_header *nextpool ;       /* next pool of this size class  */
    struct pool_header *prevpool ;       /* previous pool       ""        */
    uint arenaindex ;                    /* index into arenas of base adr */
    uint szidx;                         /* block size class index        */
    uint nextoffset ;                    /* bytes to virgin block         */
    uint maxnextoffset ;                 /* largest valid nextoffset      */
};
{% endhighlight %}

需要注意的pool_header和其所管理的内存是连续的，下面是其结构图：

{% capture images %}
    /images/python-memory-2.png
{% endcapture %}
{% include gallery images=images caption="pool_header结构图" cols=1 %}

对于这个结构其实很好理解，每次分配一个block出去，更新偏移量和freeblock直到到达最大偏移量。其实这个结构中最巧妙的就是freeblock指针，它是用来指向pool中一个未被使用的自由的block，关键是内存释放之后会导致pool中出现大量离散的自由block，Python必须建立一种机制将这些离散的自由block组织起来，再次使用。Python就是利用这个freeblock指针实现一个离散自由block链表。那么如何实现的呢？其实考虑到未分配的block其实还可以用作其他用途，即存储其下一个未分配的block的地址Σ(っ °Д °;)っ！那么freeblock就是指向这个离散自由链表的表头~，这种思想可以说在Python实现中处处可见。可以看一下释放时的代码：

{% highlight C++ %}
void
PyObject_Free(void *p)
{
    poolp pool;
    block *lastfree;
    poolp next, prev;
    uint size;
#ifndef Py_USING_MEMORY_DEBUGGER
    uint arenaindex_temp;
#endif

    if (p == NULL)      /* free(NULL) has no effect */
        return;

#ifdef WITH_VALGRIND
    if (UNLIKELY(running_on_valgrind > 0))
        goto redirect;
#endif

    pool = POOL_ADDR(p);
    if (Py_ADDRESS_IN_RANGE(p, pool)) {
        /* We allocated this address. */
        LOCK();
        /* Link p to the start of the pool's freeblock list.  Since
         * the pool had at least the p block outstanding, the pool
         * wasn't empty (so it's already in a usedpools[] list, or
         * was full and is in no list -- it's not in the freeblocks
         * list in any case).
         */
        assert(pool->ref.count > 0);            /* else it was empty */
        *(block **)p = lastfree = pool->freeblock;//这一句把p内容变为当前freeblock内容
        pool->freeblock = (block *)p;//更新freeblock，改为指向p
{% endhighlight %}

这样，产生的就是如下图所示的离散自由freeblock链表：

{% capture images %}
    /images/python-memory-3.png
{% endcapture %}
{% include gallery images=images caption="离散自由链表" cols=1 %}

### arena
一个arena管理多个pool，默认管理大小为256KB，一般也就是256/4=64个pool，其arena_object结构体如下：

{% highlight C++ %}
/* Record keeping for arenas. */
struct arena_object {
    /* The address of the arena, as returned by malloc.  Note that 0
     * will never be returned by a successful malloc, and is used
     * here to mark an arena_object that doesn't correspond to an
     * allocated arena.
     */
    uptr address ;

    /* Pool-aligned pointer to the next pool to be carved off. */
    block* pool_address ;

    /* The number of available pools in the arena:  free pools + never-
     * allocated pools.
     */
    uint nfreepools ;

    /* The total number of pools in the arena, whether or not available. */
    uint ntotalpools ;

    /* Singly-linked list of available pools. */
    struct pool_header * freepools ;

    /* Whenever this arena_object is not associated with an allocated
     * arena, the nextarena member is used to link all unassociated
     * arena_objects in the singly-linked `unused_arena_objects` list.
     * The prevarena member is unused in this case.
     *
     * When this arena_object is associated with an allocated arena
     * with at least one available pool, both members are used in the
     * doubly-linked `usable_arenas` list, which is maintained in
     * increasing order of `nfreepools` values.
     *
     * Else this arena_object is associated with an allocated arena
     * all of whose pools are in use.  `nextarena` and `prevarena`
     * are both meaningless in this case.
     */
    struct arena_object * nextarena ;
    struct arena_object * prevarena ;
};
{% endhighlight %}

python运行时会维护一个arena_object的集合（数组），在此数组上又维护了两类链表（这种技巧经常出现啊~~~），一种是未使用的unused_arena_objects单链表（未与pool建立联系）, 另一种是可用的usable_arenas双向链表（已与pool建立联系），需要注意的是在配置arenas的内存时并没有配置其管理的pool的内存，只有当从未使用的unused_arena_objects单链表中取出一个变成可用状态时才会分配ARENA_SIZE(256KB)的内存。

### usedpool数组
arena集合是小块内存池，但是当Python申请内存时，最基本的操作单元并不是arena而是pool, pool一共有三种状态：

1. used状态：pool中至少有一个block已被使用，由usedpool数组维护
2. full状态：所有的block都被使用了，这种状态的pool在arena中，但不在arena的freepools链表中
3. empty状态：pool中所有的block都未被使用，处于这个状态的pool通过pool_header中的nextpool构成一个链表，这个链表的表头是arena_object中的freepools

{% highlight C++ %}
#define PTA(x)  ((poolp )((uchar *)&(usedpools[2*(x)]) - 2*sizeof(block *)))
#define PT(x)   PTA(x), PTA(x)

static poolp usedpools[2 * ((NB_SMALL_SIZE_CLASSES + 7) / 8) * 8] = {
    PT(0), PT(1), PT(2), PT(3), PT(4), PT(5), PT(6), PT(7)
#if NB_SMALL_SIZE_CLASSES > 8
    , PT(8), PT(9), PT(10), PT(11), PT(12), PT(13), PT(14), PT(15)
......
};
{% endhighlight %}

这个usedpool数组也非常精巧，索引下标sizeindex是对于大小的pool，数组初始的元素值是指向前面8个字节（如下图所示），当判断特定sizeindex为 i 的pool是否有可用数组时 直接判断是否 usedpool[i+i]->nextpool == pool，如下图所示：

{% capture images %}
    /images/python-memory-4.png
{% endcapture %}
{% include gallery images=images caption="usedpool数组" cols=1 %}

## 总结
这里只分析了内存池里各个数据结构，其实内部如何实现配置内存，释放内存，以及pool各个状态的转换，各式各样的链表的维护都是非常非常复杂，精细和繁琐的....关键还是用C开发的........真是应了那句话，细节是魔鬼！
本文大部分参考了Python源码分析，这是一本好书！