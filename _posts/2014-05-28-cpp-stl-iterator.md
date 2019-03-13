---
layout: post
title: STL之迭代器
description: "STL之迭代器"
modified: 2014-05-28
tags: [C++, STL]
categories: [C++, STL]
---

除了[STL内存配置](http://atimekiller.github.io/cpp-stl-memory-management/)之外，我认为理解STL迭代器对理解STL的设计思想具有重大帮助，而且也能更高效的利用好STL。迭代器，其实就是行为类似指针的一种对象，是指针的更高一级的抽象和包装，它可以访问容器中的元素，更关键的是它将容器和算法有机统一起来，是泛型编程的绝好体现。下面总结一下STL迭代器的实现技术：

# 需要考虑的问题

1.  主要实现对 解引用操作符（operator*）、成员访问操作符（operator->）、前置\后置递增操作符（operator++）等操作符的重载，使其包含指针的所有行为
2.  获取迭代器相应型别，如内部所指之物的型别
3.  同等对待原生指针和迭代器

对于第一个问题，其实还比较简单，这里需要注意一下的是前置和后置递增操作符的重载方式：

{% highlight C++ %}
template<class T>
class MyIterator{
T* ptr;
...
T& operator *() {return *ptr;}
T* operator->() {return ptr;}

//pre-increment operator
MyIterator& operator++(){
    ptr = ptr->next();
    return *this;
}

//post-increment operator
MyIterator operator++(int){
    MyIterator temp = *this;
    ++*this;
    return temp;
}
...
};
{% endhighlight %}

# 函数模板的参数推导机制
这个其实是C++函数模板的特性，编译器会自动进行参数推导，如下程序所示：

{% highlight C++ %}
template<class I, class T>
void func_impl(I iter, T t){
    T tmp;//这里是就可以推导出 迭代器所指之物的真正类型
    ....
}

template <class I>
void func(I iter){//实际调用接口
    func_impl(iter, *iter);
}
{% endhighlight %}

但是这种推导机制只适用于函数参数，不适应于返回值，而且需要的不仅仅是迭代所指之物的型别，还有其他种类的型别。

# traits和偏特化
所谓的traits就是在定义迭代器的时候，通过typedef定义一些型别，在用到迭代器的地方直接就可以萃取出其内部定义的类型。而偏特化（partial specialization）则是提供另一份template定义式，其本身仍是模板化的（templatied）(见[Partial Specialization of Class Templates (C++)](http://msdn.microsoft.com/en-us/library/3967w96f.aspx))，这个主要解决原生指针不能内部自定义类型的问题。
如STL定义的基础迭代器，一般自己开发的迭代器都继承自它，以及萃取迭代的萃取类及其特化版本

{% highlight C++ %}
template <class _Category, class _Tp, class _Distance = ptrdiff_t,
          class _Pointer = _Tp*, class _Reference = _Tp&>
struct iterator {
  typedef _Category  iterator_category;//迭代器类型
  typedef _Tp        value_type;//迭代器所指之物类型
  typedef _Distance  difference_type;//迭代器距离类型
  typedef _Pointer   pointer;//指针类型
  typedef _Reference reference;//引用类型
};
template <class _Iterator>
struct iterator_traits {
 typedef typename _Iterator::iterator_category iterator_category;
 typedef typename _Iterator::value_type value_type;
 typedef typename _Iterator::difference_type difference_type;
 typedef typename _Iterator::pointer pointer;
 typedef typename _Iterator::reference reference;
};
template <class _Tp>
struct iterator_traits<_Tp*> {
 typedef random_access_iterator_tag iterator_category;
 typedef _Tp value_type;
 typedef ptrdiff_t difference_type;
 typedef _Tp* pointer;
 typedef _Tp& reference;
};

template <class _Tp>
struct iterator_traits<const _Tp*> {
 typedef random_access_iterator_tag iterator_category;
 typedef _Tp value_type;
 typedef ptrdiff_t difference_type;
 typedef const _Tp* pointer;
 typedef const _Tp& reference;
};
{% endhighlight %}

#  迭代器类型
迭代器内部定义型别之中有一项迭代器类型定义，主要区分迭代器所属类型，STL设定了五类迭代器：

{% highlight C++ %}
struct input_iterator_tag {};//只读迭代器
struct output_iterator_tag {};//只写迭代器
struct forward_iterator_tag : public input_iterator_tag {};//前向迭代器
struct bidirectional_iterator_tag : public forward_iterator_tag {};//双向迭代器
struct random_access_iterator_tag : public bidirectional_iterator_tag {};//随机迭代器
{% endhighlight %}

用继承关系来描述各个迭代器的好处在于通过继承可以不必再写单纯只作传递调用的函数，比如在参数为基类迭代器input_iterator_tag， 那么传入forward_iterator_tag的对象也是有效的。

# 总结
还是那句话，迭代器实现了容器和算法的有机统一，使算法和容器可以独立开来，是泛型编程的关键之处。另外一点需要注意的是对于每一种容器需要设计其对应的迭代器，因为不同容器的元素访问方式也是不同的。