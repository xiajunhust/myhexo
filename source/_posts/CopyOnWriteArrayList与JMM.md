---
title: CopyOnWriteArrayList与JMM
date: 2016-08-06 16:37:02
tags: Java
---

说明：本文代码均以JDK1.8的源码为准。

# 1 什么是CopyOnWriteArrayList

关于CopyOnWriteArrayList是什么以及基本用法，在这里不多说，网上可以搜到大量这方面的文章。在这里只做简要说明：CopyOnWriteArrayList相当于线程安全的ArrayList，是一个可变数组。它具有如下特性： 
 
- 是线程安全的
- 写操作会复制整个基础数组，因此写操作开销很大
- 适用于如下情况：数组大小较小，并且读操作比写操作多很多的情形

<!-- more -->

# 2 CopyOnWriteArrayList的设计原理与JMM

下面我们分析CopyOnWriteArrayList的设计原理，结合JMM的基础知识，分析CopyOnWriteArrayList是如何保证线程安全的。

首先看用来实际保存数据的数组：

```
/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;
```

可以看到array数组前面使用了volatile变量来修饰。volatile主要用来解决内存可见性问题。关于volatile的详细实现原理可以参考《[深入理解java内存模型.pdf](http://o8sltkx20.bkt.clouddn.com/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.pdf)》以及[Java并发编程：volatile关键字解析-博客园-海子](http://www.cnblogs.com/dolphin0520/p/3920373.html)。

## 2.1 CopyOnWriteArrayList的读方法

读方法比较简单，直接从array中获取对应索引的值。

```
/**
 * {@inheritDoc}
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E get(int index) {
    return get(getArray(), index);
}

@SuppressWarnings("unchecked")
private E get(Object[] a, int index) {
    return (E) a[index];
}

/**
 * Gets the array.  Non-private so as to also be accessible
 * from CopyOnWriteArraySet class.
 */
final Object[] getArray() {
    return array;
}
```

## 2.2 CopyOnWriteArrayList的写方法

- set方法  
源码如下：

```
/**
 * Replaces the element at the specified position in this list with the
 * specified element.
 *
 * @throws IndexOutOfBoundsException {@inheritDoc}
 */
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}

/**
 * Sets the array.
 */
final void setArray(Object[] a) {
    array = a;
}
```

set方法的功能是将对应索引的元素置为一个新值。执行流程：  
（1）加锁  
（2）获取对应索引已有的值  
（3）比较已有的值和新值，如果不相等，转4，否则转5  
（4）创建新的数组，复制原数组的元素，并将对应索引置为新值。然后将新数组赋给array（setArray）  
（5）setArray-将array赋给array

这里有一个比较奇怪的点，为什么已有的值和新值相等的时候，还要执行setArray呢？本质上setArray也没有做什么事情。  

这段代码混合使用了锁以及volatile。锁的用法比较容易理解，它在使用同一个锁的不同线程之间保证内存顺序性，代码结尾的释放锁的操作提供了本线程和其他欲获取相同的锁的线程之间的happens-before语义。但是CopyOnWriteArrayList类中其他代码，不一定会使用到这把锁，因此，前面所述的锁带来的内存模型含义对这部分代码执行是不适用的。

其他没用到这把锁的代码，读写是volatile读和volatile写（因为array前面使用volatile关键字修饰）。由volatile来保证happens-before语义。

---

<b>volatile的特性及原理</b>

volatile 变量自身具有下列特性:  （1）可见性。对一个volatile 变量的读,总是能看到(任意线程)对这个volatile变量最后的写入。  （2）原子性:对任意单个volatile 变量的读/写具有原子性,但类似于volatile++这种复合操作不具有原子性。

volatile 写和锁的释放有相同的内存语义。

为了实现 volatile 的内存语义,编译器在生成字节码时,会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。

---

这里调用setArray的原因是，确保set方法对array执行的永远是volatile写。这就和其他对array执行volatile读的线程之间建立了happens-before语义。非常重要的一点：volatile读/写语义针对的是读写操作，而不是使用volatile修饰的变量本身。这样说更直白一点：在一个volatile写操作之前的对其他非volatile变量的写，happens-before于同一个volatile变量读操作之后的对其他变量的读。这句话比较绕，看下面一个例子就比较易懂了。

```
// initial conditions
int nonVolatileField = 0;
CopyOnWriteArrayList<String> list = /* a single String */

// Thread 1
nonVolatileField = 1;                 // (1)
list.set(0, "x");                     // (2)

// Thread 2
String s = list.get(0);               // (3)
if (s == "x") {
    int localVar = nonVolatileField;  // (4)
}
```

现在假设原始数组中无元素“x”，这样(2)成功设置了元素"x"，(3)处可以成功获取到元素"x"。这种情况下，(4)一定会读取到(1)处设置的值1.因为(2)处的volatile写以及在此之前的任何写操作都happens-before(3)处的读以及之后的所有读。

但是，假设一开始数组中就有了元素"x"，如果else不调用setArray，那么(2)处的写就不是volatile写，(4)处的读就不一定能读到(1)处设置的值！

很显然我们不想让内存可见性依赖于list中已有的值，为了确保任何情况下的内存可见性，set方法必须永远都是一个volatile写，这就是为何要在else代码块中调用setArray的原因。

其他写方法（add、remove）比较易懂，在此不详述。


# 3 参考资料

- [Java多线程系列--“JUC集合”02之 CopyOnWriteArrayList](http://www.cnblogs.com/skywang12345/p/3498483.html)
- [Why setArray() method call required in CopyOnWriteArrayList](http://stackoverflow.com/questions/28772539/why-setarray-method-call-required-in-copyonwritearraylist#)
- [深入理解java内存模型.pdf](http://o8sltkx20.bkt.clouddn.com/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Java%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.pdf)
- [Java并发编程：volatile关键字解析-博客园-海子](http://www.cnblogs.com/dolphin0520/p/3920373.html)