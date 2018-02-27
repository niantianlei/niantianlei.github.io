---
layout:     post
title:      "ThreadLocal源码分析"
subtitle:   " \"ThreadLocal\""
date:   2017-12-19 10:05:00 +0800
author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
header-mask: 0.4
tags:
    - Java
---

## ThreadLocal类详解
目的：解决多线程对同一变量的访问冲突。  
ThreadLocal提供了get、set、remove等方法，可以为每个使用该变量的线程都存有一份独立的副本，因此get总是返回由当前执行线程在调用set时设置的最新值。  
当线程终止后，这些值会作为垃圾回收。  
对于多线程资源共享问题。同步机制采用了“以时间换空间”的方式，而ThreadLocal采用了“以空间换时间”的方式。前者仅提供一份变量，让不同的线程排队访问；后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。  
### 源码分析
下面进一步了解ThreadLocal类方法的实现。  
set()用来设置当前线程中变量的副本，  
get()方法是用来获取ThreadLocal在当前线程中保存的变量副本，  
remove()用来移除当前线程中变量的副本。  
##### set
![a]({{ "/img/post/ThreadLocal/set.png" | prepend: site.baseurl }} )  
其中，getMap是获取当前线程的ThreadLocalMap。  
```
ThreadLocalMap getMap(Thread t) {  
    return t.threadLocals;  
}  
```
首先获取当前线程，通过getMap方法拿到一个ThreadLocalMap实例，然后将变量的值设置到这个ThreadLocalMap对象中，当然如果拿到的ThreadLocalMap对象为null，就通过createMap方法创建一个新的ThreadLocalMap实例存储变量。  
再看看ThreadLocalMap是什么，  
![b]({{ "/img/post/ThreadLocal/ThreadLocalMap.png" | prepend: site.baseurl }} )  
![c]({{ "/img/post/ThreadLocal/ThreadLocalMap-constructor.png" | prepend: site.baseurl }} )  
ThreadLocalMap是ThreadLocal的一个静态内部类，内部维护一个Entry（继承了WeakReference）数组来储存信息，Entry数组table初始长度为16。ThreadLocalMap中的Entry以ThreadLocal为key，以变量值为value。  
线程隔离的原理就在于此，它实现了键值对的设置和获取，每个线程中都有一个独立的ThreadLocalMap副本，它所存储的值，只能被当前线程读取和修改。ThreadLocal类通过操作每一个线程特有的ThreadLocalMap副本，从而实现了变量访问在不同线程中的隔离。因为每个线程的变量都是自己特有的，完全不会有并发错误。  
##### get
![d]({{ "/img/post/ThreadLocal/get.png" | prepend: site.baseurl }} )  
同样地，首先通过当前线程拿到对应的ThreadLocalMap变量map。判断map是否为空：不为空，利用当前ThreadLocal实例this，从map中获取对应的Entry实例，Entry的key就是ThreadLocal实例this，value为此线程的变量值；如果Entry实例不为null，直接返回其储存的value，并进行类型转换；如果map为空，调用setInitialValue()方法返回value。  
![e]({{ "/img/post/ThreadLocal/setInitialValue.png" | prepend: site.baseurl }} )  
可见，setInitialValue方法的实现基本同set，只是将value值设为默认值null。  

##### 哈希冲突
Entry中是没有链表结构的，所以不像HashMap那样处理哈希冲突，  而是采用线性探测法处理。有冲突就插在下一个位置。

### 内存泄露问题
因为是以空间换时间，需要考虑内存泄漏问题。ThreadLocalMap中的Entry使用了弱引用，若没有任何强引用会被下一次GC回收，无论内存是否充足，但此时value不会被GC回收；而value没有被回收，就会出现key为null的Entry。这时没有办法访问key为null的value，那么到这个线程结束以前，就会一直存在一条强引用链，只有当线程终止时，才会回收，这一阶段会存在内存泄漏。而ThreadLocal的get(),set(),remove()方法都会清除线程ThreadLocalMap里所有key为null的value。  
因此，最好在使用完ThreadLocal里的对象后调用remove方法，或者set成null。  

那如果不用弱引用，使用强引用，如果线程没有结束，引用ThreadLocal的对象被回收，ThreadLocalMap仍持有ThreadLocal的强引用，就会发生内存泄漏。
