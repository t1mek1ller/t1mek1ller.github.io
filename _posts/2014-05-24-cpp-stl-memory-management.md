---
layout: post
title: STL之内存管理
description: "STL之内存管理"
modified: 2014-05-24
tags: [C++, STL]
categories: [C++]
---

C++可以说由四大部分组成[1]:

1. 其包含的C子集;
2. 面向对象部分（类，封装，继承，多态，etc.);
3. 模板（泛型编程或是称之为元编程(meta programming)）;
4. STL（Standard Template Library）, 其中STL作为C++的标准模板库，包含基本的数据结构和算法，能在保证性能的情况下大大提高编程效率。

STL包含六大组件[2]:

1. 内存配置器（allocators），用于内存管理，进行内存的分配和释放
2. 容器（containers），一些基本的数据结构模板，包括vector，list，deque，(multi)set，(multi)map，unordered_set，unordered_map
3. 适配器（adapters），容器适配器，对容器进行了一层封装，包括stack，queue
4. 迭代器（iterators），具有指针行为的对象，每一种容器都有其特定的迭代器，也用于紧密结合算法和容器
5. 算法（algorithms），常用算法
6. 函数对象（functors），行为类似函数的对象，重载了函数调用操作符operator()

其中内存管理是整个STL的基石，一般内存管理都需要考虑到小块内存造成的内存碎片问题，这通常都是通过内存池实现，预先分配一大块内存，然后再进行内存分配和回收，这样可以防止小块内存的过度申请和释放造成效率低下。STL提供了双层内存配置器，当申请的内存超过128字节时，调用第一级配置器，当小于128字节时则调用第二级配置器。C++对象的构造与析构分为两步：

```C++
class Foo {...}
Foo *p = new Foo();//分为两步，先调用::operator new配置内存，然后调用Foo::Foo()构造对象
delete p;//也分为两步，先调用Foo::~Foo()析构对象，然后调用::operator delete释放内存
```

STL的内存配置和释放也分为两步，构造对象时先分配内存，再构造对象，析构对象时先析构对象，再释放内存。对象的构造和析构利用construct和destroy这两个函数：

```C++
template <class T1, class T2>
inline void construct(T1* p, const T2& value)
{
    new (p) T1(value);//c++的placement new操作符，在p所指向的内存上调用T1的构造函数进行对象的构造
}

//一种destroy版本
template <class T>
inline void destroy(T* p)
{
    p->~T();//直接调用
}
```

这里的destroy对于POD（plain old data）型别（拥有trivil constructor/destructor/copy/assignment函数）的对象效率很低，STL会通过__type_traits判断对象是否是POD型别，若是则什么也不做。

## 第一级内存配置器
用于分配申请内存大于128字节，其分配过程如下所示：

```C++
template <int __inst> //template 非型别参数
class __malloc_alloc_template {

private:

  static void* _S_oom_malloc(size_t);
  static void* _S_oom_realloc(void*, size_t);

#ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG
  static void (* __malloc_alloc_oom_handler)();
#endif

public:

  static void* allocate(size_t __n)
  {
    void* __result = malloc(__n);//没有用C++的new handler机制，而是采用了C的malloc/free
    if (0 == __result) __result = _S_oom_malloc(__n);//内存不足时调用用户定义的OOM函数
    return __result;
  }

  static void deallocate(void* __p, size_t /* __n */)
  {
    free(__p);
  }

  static void* reallocate(void* __p, size_t /* old_sz */, size_t __new_sz)
  {
    void* __result = realloc(__p, __new_sz);
    if (0 == __result) __result = _S_oom_realloc(__p, __new_sz);
    return __result;
  }

//可以设置用户定义的OOM处理函数，模拟了C++的set_new_handler()函数
  static void (* __set_malloc_handler(void (*__f)()))()
  {
    void (* __old)() = __malloc_alloc_oom_handler;
    __malloc_alloc_oom_handler = __f;
    return(__old);
  }

};

// malloc_alloc out-of-memory handling
#ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG
template <int __inst>
void (* __malloc_alloc_template<__inst>::__malloc_alloc_oom_handler)() = 0;
#endif

template <int __inst>
void*
__malloc_alloc_template<__inst>::_S_oom_malloc(size_t __n)
{
    void (* __my_malloc_handler)();
    void* __result;

    for (;;) {//不断尝试释放，配置，再释放，再配置
        __my_malloc_handler = __malloc_alloc_oom_handler;
        if (0 == __my_malloc_handler) { __THROW_BAD_ALLOC; }//未设定内存处理函数时直接抛出bad_alloc异常
        (*__my_malloc_handler)();//调用OOM处理函数
        __result = malloc(__n);//再次尝试配置内存
        if (__result) return(__result);
    }
}
```

## 第二级内存配置器
而对于128字节以下的内存配置则采用更为复杂和精巧的第二级内存配置器。其主要结构是采用一个 free_list维护一大块内存，free_list维护了16种内存大小的区块，每种都为8的倍数，最小为8字节，最大为128字节。这个free_list其实并不是链表，而是一个16个元素大小的数组，数组元素类型是obj * （每个元素指向一个链表的头结点），而obj的结构如下：

```C++
union obj {
    union obj *free_list_link;
    char client_data[1];
};
```

这个节点结构非常精巧，节点的第一种字段是指向下一个obj的指针，而第二个字段则是指向真正的用户内存块。也就是说这种节点是一物二用，在内存还未分配给用户时通过第一个字段将这些取自内存池的相同大小的内存块一一链接起来形成一个链表；在分配出去之后，用户则直接可以通过强制类型转换转成用户需要的类型。下面看一下其allocate函数吧：

```C++
//获取free_list的对应下标以适应用户申请的内存块大小  
static  size_t _S_freelist_index(size_t __bytes) {
        return (((__bytes) + (size_t)_ALIGN-1)/(size_t)_ALIGN - 1);
  }


/* __n must be > 0      */
  static void* allocate(size_t __n)
  {
    void* __ret = 0;

    if (__n > (size_t) _MAX_BYTES) {//大于128调用第一级配置器
      __ret = malloc_alloc::allocate(__n);
    }
    else {
      _Obj* __STL_VOLATILE* __my_free_list
          = _S_free_list + _S_freelist_index(__n);//获取适合用户申请内存块大小在free_list中的下标
      // Acquire the lock here with a constructor call.
      // This ensures that it is released in exit or during stack
      // unwinding.
#     ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;//多线程模式下需要加锁
#     endif
      _Obj* __RESTRICT __result = *__my_free_list;
      if (__result == 0)//没有可用块，则调用refill从内存池中获取一定的内存块重新填充free_list
        __ret = _S_refill(_S_round_up(__n));
      else {
        *__my_free_list = __result -> _M_free_list_link;//将free_list指向下一个可用内存块
        __ret = __result;
      }
    }

    return __ret;
  };
```

关键就在于上述函数中的_S_refill函数，这个函数会默认从内存池中默认取20个n字节的内存块，然后再将它们通过obj中的free_list_link字段一一链接起来，实现如下代码段所示，

```C++
/* Returns an object of size __n, and optionally adds to size __n free list.*/
/* We assume that __n is properly aligned.                                */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
void*
__default_alloc_template<__threads, __inst>::_S_refill(size_t __n)
{
    int __nobjs = 20;//默认取20个，但不一定能取到，要看内存池的情况
    char* __chunk = _S_chunk_alloc(__n, __nobjs);//从内存池中取__nobjs个__n字节的内存块
    _Obj* __STL_VOLATILE* __my_free_list;
    _Obj* __result;
    _Obj* __current_obj;
    _Obj* __next_obj;
    int __i;

    if (1 == __nobjs) return(__chunk);
    __my_free_list = _S_free_list + _S_freelist_index(__n);

    /* Build free list in chunk */
      __result = (_Obj*)__chunk;
      *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);
      for (__i = 1; ; __i++) {//这就是进行链接的过程
        __current_obj = __next_obj;
        __next_obj = (_Obj*)((char*)__next_obj + __n);
        if (__nobjs - 1 == __i) {
            __current_obj -> _M_free_list_link = 0;
            break;
        } else {
            __current_obj -> _M_free_list_link = __next_obj;
        }
      }
    return(__result);
}
```

而这个内存池到底是个什么东西呢，其实在这个第二级配置器中维护了三个指针：

```C++
  // Chunk allocation state.
  static char* _S_start_free;//内存池起始位置
  static char* _S_end_free;//内存池结束位置
  static size_t _S_heap_size;//内存池的大小，其实是所有从系统堆申请的内存大小，包括未分配的和已经分配的

//均初始化为0
template <bool __threads, int __inst>
char* __default_alloc_template<__threads, __inst>::_S_start_free = 0;

template <bool __threads, int __inst>
char* __default_alloc_template<__threads, __inst>::_S_end_free = 0;

template <bool __threads, int __inst>
size_t __default_alloc_template<__threads, __inst>::_S_heap_size = 0;

```

而真正从内存池中取指定大小的内存则是通过 _S_chunk_alloc函数:

```C++
/* We allocate memory in large chunks in order to avoid fragmenting     */
/* the malloc heap too much.                                            */
/* We assume that size is properly aligned.                             */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
char*
__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size, 
                                                            int& __nobjs)
{
    char* __result;
    size_t __total_bytes = __size * __nobjs;//需申请的内存大小
    size_t __bytes_left = _S_end_free - _S_start_free;//内存池中的剩余内存

    if (__bytes_left >= __total_bytes) {//内存池中有足够的剩余内存
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    } else if (__bytes_left >= __size) {//没有足够内存，但能提供一个指针内存块
        __nobjs = (int)(__bytes_left/__size);
        __total_bytes = __size * __nobjs;
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    } else {//一个内存块都不够了！Σ(っ °Д °;)っ
        size_t __bytes_to_get = 
	  2 * __total_bytes + _S_round_up(_S_heap_size >> 4);
        // Try to make use of the left-over piece.
        if (__bytes_left > 0) {//将内存零头分配给free_list中的相应链表中
            _Obj* __STL_VOLATILE* __my_free_list =
                        _S_free_list + _S_freelist_index(__bytes_left);

            ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;
            *__my_free_list = (_Obj*)_S_start_free;
        }
        //向系统堆申请空间
        _S_start_free = (char*)malloc(__bytes_to_get);
        if (0 == _S_start_free) {//系统堆空间不足，分配失败
            size_t __i;
            _Obj* __STL_VOLATILE* __my_free_list;
	    _Obj* __p;
            // Try to make do with what we have.  That can't
            // hurt.  We do not try smaller requests, since that tends
            // to result in disaster on multi-process machines.
            //这里就是在已经从内存池分配到大的内存块链表中寻找未用的内存块
            for (__i = __size;
                 __i <= (size_t) _MAX_BYTES;
                 __i += (size_t) _ALIGN) {
                __my_free_list = _S_free_list + _S_freelist_index(__i);
                __p = *__my_free_list;
                if (0 != __p) {
                    *__my_free_list = __p -> _M_free_list_link;
                    _S_start_free = (char*)__p;
                    _S_end_free = _S_start_free + __i;
                    return(_S_chunk_alloc(__size, __nobjs));
                    // Any leftover piece will eventually make it to the
                    // right free list.
                    //这里递归调用本函数，将一切残余内存都分配到相应内存链表中，真可谓无所不用其及的进行最大效率的内存管理
                }
            }
	    _S_end_free = 0;	// In case of exception.
            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);
            // This should either throw an
            // exception or remedy the situation.  Thus we assume it
            // succeeded.
            //出现意外时调用第一级内存配置器
        }
        _S_heap_size += __bytes_to_get;
        _S_end_free = _S_start_free + __bytes_to_get;
        return(_S_chunk_alloc(__size, __nobjs));
    }
}
```

至此，整个STL内存配置器的主要过程就分析完毕，可以看见内存管理永远是最复杂的模块之一，要想尽一切方法，不能让任何一块内存浪费掉~

## 参考文献:
[1]Meyers S. Effective C++: 55 specific ways to improve your programs and designs[M].2005.

[2]STL 源码剖析[M].2002.