---
layout: post
title: Python之名字空间与作用域
description: "Python之名字空间与作用域"
modified: 2014-06-06
tags: [Python]
categories: [Python]
---

探讨Python作用域之前，先看一段Python代码：

{% highlight Python %}
a = 1

def f():
    print a
    a = 2
{% endhighlight %}

上述代码执行之后的结果完全在意料之外，程序没有输出 a的全局值1，而是抛出了异常，异常信息为 “UnboundLocalError: local variable 'a' referenced before assignment”，表示在变量赋值之前提前解引用了。产生这样令人大跌眼镜的结果归因于Python的名字空间和作用域规则。

首先，Python具有`静态作用域`，又称为`词法作用域`，这一点和大多数语言（C/C++, Java......）类似，所谓静态作用域就是变量的有效作用范围在代码运行之前就已经确定了。Python的作用域规则遵循`LEGB规则`，依次是Local作用域（使用def，class，lambda时会引入一个新的local作用域），Enclosing作用域（闭包），Global作用域（整个.py文件，称为一个module），Builtin作用域（内嵌作用域，包含一些builtin函数如dir，range等），Python会按照此顺序向上寻找某个变量。

而名字空间是一种特殊的作用域，包含了处于该作用域内的标识符。在Python中，名字空间在运行时会以一个字典对象（底层为PyDictObject实现）存在，也就是一系列key-value键值对，标识了该作用域内的所有符号。那么回到上面这个例子，其实在print a这条语句执行前，函数f所表示的名字空间中已经有了变量a，故表示在当前local作用域中可以找到a，但问题是程序在编译成字节码后，必须要等到a=2这个赋值语句完成后才会知道a的值，所以Python会在print a时抛出异常（有点别扭，为什么不继续向上寻找而是直接抛出异常？难道是要实现闭包的缘故？）。

Python中的闭包：

{% highlight Python %}
a = 1

def f():
    a = 2
    def g():
        print a
    return g

func = f()
func()
{% endhighlight %}

按照LEGB规则，上述程序输出结果为2。