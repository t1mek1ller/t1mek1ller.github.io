---
layout: post
title: Netty数据结构之FastThreadLocal
description: "The Data Structure of Netty - FastThreadLocal"
modified: 2019-08-01
tags: [Netty]
categories: [Netty]
---


ThreadLocal是JDK中线程局部变量的一种实现。顾名思义，线程局部变量是每个线程单独维护的变量，线程间不共享，也就不存在竞争关系。

ThreadLocal解决了什么问题？对象的成员变量的生命周期取决于对象的生命周期，同样地，把线程看做对象，其局部变量的生命周期也是在线程之内。有些场景下，我们需要设置线程局部变量，比如：

* 保存线程上下文
* 简化开发，省去了线程之间的同步操作
* 提高性能，避免了线程同步带来的性能消耗

ThreadLocal给开发者提供了一种设置线程局部变量的方法。

Netty为了追求极致的性能，并没有直接使用JDK的ThreadLocal，而是实现了名为FastThreadLocal的方法，解决了ThreadLocal潜在的性能问题。下面分别深入两种ThreadLocal的实现来一探究竟。


## JDK的ThreadLocal
我们可以先看下JDK的ThreadLocal使用样例：

{% highlight Java %}
   import java.util.concurrent.atomic.AtomicInteger;
  
   public class ThreadId {
       // Atomic integer containing the next thread ID to be assigned
       private static final AtomicInteger nextId = new AtomicInteger(0);
  
       // Thread local variable containing each thread's ID
       private static final ThreadLocal<Integer> threadId =
           new ThreadLocal<Integer>() {
               @Override protected Integer initialValue() {
                   return nextId.getAndIncrement();
           }
       };
  
       // Returns the current thread's unique ID, assigning it if necessary
       public static int get() {
           return threadId.get();
       }
   }
{% endhighlight %}

线程局部变量是如何初始化的？在上述示例中，`threadId`是一个ThreadLocal对象，其模板参数是Integer，表示这是一个Integer类型的线程局部变量，并且重写了`initialValue`方法给出了对象初始化值。所以ThreadLocal对象并不是用户关心的线程局部变量，而是规定了线程局部变量的初始化方式。用户可以通过ThreadLocal对象获取线程局部变量。

线程局部变量是如何获取的？在上述示例中，直接调用`threadId`的`get`函数可以得到，但`get`方法并没有指定参数，所以具体获取到的值取决于调用线程，这个get方法揭示了整个ThreadLocal的设计思路，具体实现如下：

{% highlight Java %}
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
{% endhighlight %}

从上述实现中，我们可以看到其实每个线程持有一个ThreadLocalMap对象，保存了所有属于该线程的局部变量。这个map的key是ThreadLocal对象，value是真实的局部变量。这个实现说明每个线程在首次调用get前都会初始化真实的局部变量。

所以一般情况下ThreadLocal对象是被声明为`private static`的，避免创建过多的ThreadLocal对象引起不免要的内存分配。如果拥有ThreadLocal对象的实例是单例的情况下，不用`static`修饰也可以。

ThreadLocalMap的实现是基于线性探测的哈希表，其特别之处在于HashTable中的Entry是针对key的弱引用，实现如下：
{% highlight Java %}
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
{% endhighlight %}

采用弱引用的原因在于如果一个ThreadLocal对象只被线程引用，说明是可以被回收的，因为在线程之外没有任何方法可以获取到此变量。这种情况下，可以通过弱引用的方式实现，因为当对象只有弱引用时，会被GC回收，此时ThreadLocalMap会清理这些过期的keys。


## Netty的FastThreadLocal
上面说到，JDK的Thread线程持有的`ThreadLocalMap`是基于线性探测的哈希表，在大量使用ThreadLocal场景下，进行查询操作时由于hash值冲突会有额外的性能损失。

Netty的FastThreadLocal没有采用哈希表的方案，而是用数组的方案，线程内用数组存储所有局部对象，FastThreadLocal持有全局唯一的下标索引来获取对应的局部对象。FastThreadLocal部分实现如下所示：

{% highlight Java %}
public class FastThreadLocal<V> {

	...
	
    private final int index;

    public FastThreadLocal() {
        index = InternalThreadLocalMap.nextVariableIndex();
    }

    /**
     * Returns the current value for the current thread
     */
    @SuppressWarnings("unchecked")
    public final V get() {
        InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.get();
        Object v = threadLocalMap.indexedVariable(index);
        if (v != InternalThreadLocalMap.UNSET) {
            return (V) v;
        }

        return initialize(threadLocalMap);
    }

    /**
     * Returns the initial value for this thread-local variable.
     */
    protected V initialValue() throws Exception {
        return null;
    }    

    ...
}

{% endhighlight %}

我们可以看到FastThreadLocal在初始化的时候会调用InternalThreadLocalMap获取下一个全局索引，用于标志当前FastThreadLocal关联的局部对象在InternalThreadLocalMap维护的数组中的下标。

InternalThreadLocalMap维护的数组在其父类中：
{% highlight Java %}
class UnpaddedInternalThreadLocalMap {

	...

	Object[] indexedVariables;

	...

}

{% endhighlight %}


这里UnpaddedInternalThreadLocalMap是没有做padding的，而子类InternalThreadLocalMap会添加padding字段以消除伪共享的影响，充分利用CPU缓存行。添加的padding字段如下所示：

{% highlight Java %}
public final class InternalThreadLocalMap extends UnpaddedInternalThreadLocalMap {

	...

    // Cache line padding (must be public)
    // With CompressedOops enabled, an instance of this class should occupy at least 128 bytes.
    public long rp1, rp2, rp3, rp4, rp5, rp6, rp7, rp8, rp9;	

    ...
}

{% endhighlight %}

有一点需要说明的是，这里所有成员变量加上对象头总共是136B，并不能整除64B，原因是一些历史修改并没有兼顾到padding，参见[这个issue](https://github.com/netty/netty/issues/9284)。关于使用缓存行padding是否真的有效果，需要做充分的benchmark，这里不做展开。


相应的，如果需要用到FastThreadLocal的特性，对应的线程也需要用继承自Thread的FastThreadLocalThread，持有了InternalThreadLocalMap对象：

{% highlight Java %}
public class FastThreadLocalThread extends Thread {

	...

	private InternalThreadLocalMap threadLocalMap;

	...

}

{% endhighlight %}


为什么InternalThreadLocalMap中的数组没有用弱引用？Netty实现了FastThreadLocalRunnable重写了Runnable方法，在run方法的finally快中调用了`FastThreadLocal.removeAll`来清理所有的数据，实现如下：

{% highlight Java %}
final class FastThreadLocalRunnable implements Runnable {
    private final Runnable runnable;

    private FastThreadLocalRunnable(Runnable runnable) {
        this.runnable = ObjectUtil.checkNotNull(runnable, "runnable");
    }

    @Override
    public void run() {
        try {
            runnable.run();
        } finally {
            FastThreadLocal.removeAll();
        }
    }

    static Runnable wrap(Runnable runnable) {
        return runnable instanceof FastThreadLocalRunnable ? runnable : new FastThreadLocalRunnable(runnable);
    }
}

{% endhighlight %}

如果线程都终结了，那么为什么还要清除相关数据呢？因为应用是可能运行在容器之中的，这种情况下线程资源是共享的，所以需要清理相关数据。



# 总结

Netty的FastThrealLocal主要快在用数组替代线性探测的哈希表，避免哈希冲突造成的额外性能损失，至于使用缓存行padding，消除伪共享是否真的有效，需要提供更多的benchmark测试。

另外，关于FastThrealLocal的全局index也让很多人疑惑，比如[这里](https://github.com/netty/netty/issues/8271)，[这里](https://github.com/netty/netty/issues/9328)，还有[这里](https://github.com/netty/netty/issues/7992)。主要是因为index是全局的，那么某个线程如果只获取一个很大的index的局部对象，那就需要开辟很大数组，浪费空间。但其实FastThrealLocal的设计前提是不会有那么多的FastThreadLocal对象，而且空余的数组槽位只是引用了同一个Object对象，并没有占用很多内存，相比起带来的查询性能的提升，这是值得优化的。




