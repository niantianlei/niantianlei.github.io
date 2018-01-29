---
layout:     post
title:      "ConcurrentHashMap源码分析"
subtitle:   " \"ConcurrentHashMap\""

author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
header-mask: 0.4
catalog:    true
tags:
    - Java
---

`Hashtable`在并发环境下性能差主要是由于所有操作必须竞争同一把锁，假如容器中有多把锁，每一把锁用于锁一部分数据，这样在多线程访问时不同段的数据时，就不会存在锁竞争了，这样便可以有效地提高并发效率。这就是`ConcurrentHashMap`所采用的锁分段技术。  
首先将数据分成一段一段地存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。  


## Java7中的实现
采用`Segment`数组和`HashEntry`数组实现，`Segment`继承了`ReentrantLock`，是一种可重入锁；`HashEntry`用于存储键值对数据。  
一个`ConcurrentHashMap`维护着一个`Segment`数组，一个`Segment`包含一个`HashEntry`数组，每个`HashEntry`是一个链表结构的元素，每个Segment守护着一个`HashEntry`数组里的元素，当对`HashEntry`数组的数据进行修改时，必须首先获得与它对应的`Segment`锁。  
```
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;
    ...
}
```
value以及next字段都被`volatile`修饰，是为了保证可见性。  
而`Segment`类似于`HashMap`，也有负载因子`loadFactor`,`threshold`等属性。   

#### 初始化
```
public ConcurrentHashMap(int initialCapacity,
                               float loadFactor, int concurrencyLevel) {
      if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
          throw new IllegalArgumentException();
      //MAX_SEGMENTS 为1<<16=65536，也就是最大并发数为65536
      if (concurrencyLevel > MAX_SEGMENTS)
          concurrencyLevel = MAX_SEGMENTS;
      //2的sshif次方等于ssize，例:ssize=16,sshift=4;ssize=32,sshif=5
     int sshift = 0;
     //ssize 为segments数组长度，根据concurrentLevel计算得出
     int ssize = 1;
     while (ssize < concurrencyLevel) {
         ++sshift;
         ssize <<= 1;
     }
     this.segmentShift = 32 - sshift;
     this.segmentMask = ssize - 1;
     if (initialCapacity > MAXIMUM_CAPACITY)
         initialCapacity = MAXIMUM_CAPACITY;
     //计算cap的大小，即Segment中HashEntry的数组长度，cap也一定为2的n次方.
     int c = initialCapacity / ssize;
     if (c * ssize < initialCapacity)
         ++c;
     int cap = MIN_SEGMENT_TABLE_CAPACITY;
     while (cap < c)
         cap <<= 1;
     //创建segments数组并初始化第一个Segment，其余的Segment延迟初始化
     Segment<K,V> s0 =
         new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                          (HashEntry<K,V>[])new HashEntry[cap]);
     Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
     UNSAFE.putOrderedObject(ss, SBASE, s0); 
     this.segments = ss;
 }
```
初始化构造器有三个参数，默认值：`initialCapacity`为16，`loadFactor`为0.75，`concurrentLevel`为16。  

从上面的代码可以看出来，`Segment`数组的长度`ssize`是由`concurrentLevel`来决定的，为了能通过按位与的散列算法来定位`segments`数组的索引，`ssize`一定是大于或等于`concurrentLevel`的最小的2的幂。比如：默认情况下`concurrentLevel`是16，则`ssize`为16；若`concurrentLevel`为14，`ssize`为16。  
`concurrentLevel`的最大值是65535，意味着`segments`数组的最大长度为65536。  

`segmentShift`和`segmentMask`这两个变量在定位segment时会用到，`sshift`等于`ssize`从1向左移的次数（默认长度16，则需左移4次,`sshift`则为4）。`segmentShift`用于定位参与散列运算的位数，等于`32-sshift`，默认情况为28，这里用32是因为`hash()`方法输出的最大数是32位的。  
`segmentMask`是散列运算的掩码，等于`ssize-1`，即15，二进制下各位都是1。  

变量`cap`就是`segment`里`HashEntry`数组的长度，它等于`initialCapacity / ssize`的倍数`c`，如果`c>1`，就会取大于等于`c`的2的N次方，所以`cap`不是1，就是2的N次方。  
`segment`的容量`threshold=cap*loadFactor`，默认情况下`initialCapacity`为16，`loadFactor`等于0.75，则`cap`等于1，`threshold`等于0。  

#### 定位Segment
首先对元素的hashCode值进行一次再散列，这样做的目的是让数字的每一位都参与到哈希运算，以减少哈希冲突，使元素均匀分布在不同的`Segment`上，从而提高存取效率。  
hash值无符号右移`segmentShift`位与段掩码进行与运算，定位`Segment`。  

#### get方法
```
 public V get(Object key) {
    Segment<K,V> s; 
    HashEntry<K,V>[] tab;
    //上一小节提到的再哈希过程
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    //先定位Segment，再定位HashEntry
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```
get方法无需加锁，除非读到的值是空才会加锁重读。**由于其中涉及到的共享变量都使用volatile修饰**，volatile可以保证内存可见性，所以不会读取到过期数据。  
其中定位到`HashEntry`是直接使用再散列值与上tab长度-1而非像`Segment`使用高位，目的是避免两次散列后的值一样。  

#### put方法
```
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
        /*tryLock不成功时会遍历定位到的HashEnry位置的链表（遍历主要是为了使CPU缓存链表），
        若找不到，则创建HashEntry。tryLock一定次数后（MAX_SCAN_RETRIES变量决定），则lock。
        若遍历过程中，由于其他线程的操作导致链表头结点变化，则需要重新遍历。*/
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;/*定位HashEntry，可以看到，
        这个hash值在定位Segment时和在Segment中定位HashEntry都会用到，
        只不过定位Segment时只用到高几位。*/
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
　　　　　　　　/*若c超出阈值threshold，需要扩容并rehash。
		扩容后的容量是当前容量的2倍。这样可以最大程度避免之前散列好的entry重新散列， 
		扩容并rehash的这个过程是比较消耗资源的。*/
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```
put方法需要对共享变量进行写入操作，所以为了线程安全，必须加锁。put方法首先定位到`Segment`，然后在`Segment`里进行插入操作。  
插入操作会先判断是否需要对`HashEntry`数组扩容，然后才将元素添加到正确的位置。  

假如两个线程同时执行put方法，又定位到同一个`Segment`。  
1、线程A执行tryLock()方法成功获取锁，则把`HashEntry`对象插入到相应的位置；  
2、线程B获取锁失败，则执行scanAndLockForPut()方法，在scanAndLockForPut方法中，会通过重复执行tryLock()方法尝试获取锁，在多处理器环境下，重复次数为64，单处理器重复次数为1，当执行tryLock()方法的次数超过上限时，则执行lock()方法挂起线程B；  
3、当线程A执行完插入操作时，会通过unlock()方法释放锁，接着唤醒线程B继续执行。  
#### size实现
计算元素数量时，首先想到的方法就是：`Segment`里的全局变量`count`是一个volatile变量，那么在多线程场景下，能不能直接把所有`Segment`的`count`相加得到整个`ConcurrentHashMap`大小了呢?  
结果不一定准确，这是因为统计后面`Segment`元素数量时，已经加过的`Segment`可能有数据的增、删。  
所以最安全的做法，是在统计size的时候把所有Segment的put，remove和clean方法全部锁住，但是这种做法显然非常低效。  

因为在累加count操作过程中，之前累加过的`count`发生变化的几率非常小，所以`ConcurrentHashMap`的做法是先尝试2次通过不锁住`Segment`的方式来统计各个`Segment`大小，如果统计的过程中，容器的`count`发生了变化，则再采用加锁的方式来统计所有Segment的大小。  

那么ConcurrentHashMap是如何判断在统计的时候容器是否发生了变化呢？使用modCount变量，在put , remove和clean方法里操作元素前都会将变量modCount进行加1，那么在统计size前后比较modCount是否发生变化，从而得知容器的大小是否发生变化。  
```
try {
    for (;;) {
        if (retries++ == RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                ensureSegment(j).lock(); // force creation
        }
        sum = 0L;
        size = 0;
        overflow = false;
        for (int j = 0; j < segments.length; ++j) {
            Segment<K,V> seg = segmentAt(segments, j);
            if (seg != null) {
                sum += seg.modCount;
                int c = seg.count;
                if (c < 0 || (size += c) < 0)
                    overflow = true;
            }
        }
        if (sum == last)
            break;
        last = sum;
    }
} finally {
    if (retries > RETRIES_BEFORE_LOCK) {
        for (int j = 0; j < segments.length; ++j)
            segmentAt(segments, j).unlock();
    }
}
```


## Java8中的实现
这个版本摒弃了Java7中Segment复杂的设计，主要做了两点改进：  

1.取消segments字段，直接采用`transient volatile Node<K,V>[] table;`保存数据，采用table数组元素作为锁，从而实现了对每一行数据进行加锁，进一步减少并发冲突的概率。  
2.将底层数据结构由数组+单向链表变更为数组+单向链表+红黑树。因此，对于个数超过8(默认值)的列表，jdk1.8中采用了红黑树进行存储，查找的时间复杂度可以降低到O(logN)，改进了性能。  

```
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    ...
```
和`HashMap`一样，都是内部维护了一个Node类型的数组，区别是这里的Node类中的value
和next字段用了volatile修饰，仍然是为了保证可见性。  

#### put方法
```
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```
如果key或value是null则抛出空指针异常。  
首先一个for死循环，插入成功，才会跳出。  
如果table为空，进行初始化，  
根据hash值计算出在table里面的位置，   
如果这个位置没有元素，直接新建一个Node节点保存数据并用CAS操作进行插入。  
数组索引`i`的计算过程如下：    
```
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
其中`HASH_BITS = 0x7fffffff`，意味着只取后31位。  
所以计算数组索引的过程是：key.hashCode()异或其右移16位的值，与上`HASH_BITS`取后31位，最后与上table长度-1。  

如果table[i]不为null，进一步判断其hash值是否为MOVED(值为-1)，如果是，说明该链表正在进行扩容，进行协助扩容。  
如果不是，就要针对table[i]的首节点加`synchronized`锁，如果该节点的hash不小于0，表明这是个链表，则遍历链表更新节点或在末尾插入新节点。  
如果首节点为TreeBin类型，说明为红黑树结构，在红黑树中插入节点。  

如果put操作对数据产生了影响，判断当前链表元素个数达到8个，则将链表转化为红黑树（table长度小于64时仅扩容），如果旧value不为空，返回旧值。  

**注：**链表转树时，并不会直接转，只是把这些节点包装成TreeNode放到TreeBin中， 
再由TreeBin来转化红黑树。  

##### 三个重要的无锁方法
```
@SuppressWarnings("unchecked")
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```
其中，Unsafe.getObjectVolatile可以直接获取指定内存的数据  
tabAt获得在i位置上的Node节点。  

casTabAt利用CAS算法设置i位置上的Node节点。将tab[i]和c作比较，相等则将tab[i]设置为v，否则不修改。   

setTabAt设置节点位置的值，仅在上锁区被调用。  


##### 初始化表
另外，初始化函数如下：  
```
private final Node<K,V>[] initTable() {  
    Node<K,V>[] tab; int sc;  
    while ((tab = table) == null || tab.length == 0) {  
        // 如果sizeCtl为负数，则说明已经有其它线程正在进行扩容，即正在初始化或初始化完成，此时该线程放弃CPU调度  
        if ((sc = sizeCtl) < 0)  
            Thread.yield(); // lost initialization race; just spin  
        // 利用CAS方法设置sizeCtl为-1，表示table正在初始化  
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {  
            try {  
                // 再次确认tab是否已经初始化  
                if ((tab = table) == null || tab.length == 0) {  
                    // 设置n为DEFAULT_CAPACITY（16）  
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;  
                    @SuppressWarnings("unchecked")  
                    /* 
                     * 创建大小为n的node数组，并传给table 
                    */  
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];  
                    table = tab = nt;  
                    // sc=0.75n  
                    sc = n - (n >>> 2);  
                }  
            } finally {  
                // sizeCtl = 0.75*Capacity,为扩容门限  
                sizeCtl = sc;  
            }  
            break;  
        }  
    }  
    return tab;  
}  
```
其中`sizeCtl`是控制标识符，负数代表正在进行初始化或扩容操作，其中-1代表正在初始化，-N表示有N-1个线程正在进行扩容操作；正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，类似于扩容阈值。  

#### get方法
```
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
1、根据key计算hash值；并根据hash值计算出在table中的位置i；  
2、如果table为空或table[i]为null，返回null；  
3、首先判断table表中首节点的hash值是否为待查询的key的hash值，如果是进一步判断key是否相等，如果相等直接返回value。再判断首节点的hash值是否小于0，小于0说明该节点正在扩容，则调用节点的find()(find()是Node类的方法实际上也是遍历寻找过程)方法进行寻找；  
4、都没找到，则进行遍历查找。  

#### size实现
```
/**
 * A padded cell for distributing counts.  Adapted from LongAdder
 * and Striped64.  See their internal docs for explanation.
 */
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

/**
 * Base counter value, used mainly when there is no contention,
 * but also as a fallback during table initialization
 * races. Updated via CAS.
 */
 //ConcurrentHashMap中元素个数,基于CAS无锁更新,但返回的不一定是当前Map的真实元素个数
private transient volatile long baseCount;

/**
 * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
 */
private transient volatile int cellsBusy;

/**
 * Table of counter cells. When non-null, size is a power of 2.
 */
private transient volatile CounterCell[] counterCells;
```
事实上，put和remove方法的最后都会调用addCount方法利用CAS更新`baseCount`。  
初始化时`counterCells`为空，在并发量很高时，如果存在两个线程同时执行CAS修改`baseCount`值，则失败的线程会使用CounterCell记录元素个数的变化；  

如果`CounterCell`数组counterCells为空，调用fullAddCount()方法进行初始化，并插入对应的记录数，通过CAS设置cellsBusy字段，只有设置成功的线程才能初始化`CounterCell`数组；  


如果通过CAS设置cellsBusy字段失败的话，则继续尝试通过CAS修改baseCount字段，如果修改baseCount字段成功的话，就退出循环，否则继续循环插入CounterCell对象；  

```
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}


final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
使用`volatile`类型的变量`baseCount`记录元素数量，部分元素的变化个数保存在CounterCell数组中。  
通过累加baseCount和CounterCell数组中的数量，即可得到元素的总个数。  

#### 扩容transfer
比起`HashMap`，`ConcurrentHashMap`的扩容复杂很多，因为需要支持并发环境下进行。多线程进行扩容操作，而并不加锁，目的是利用并发处理去减少扩容带来的时间影响。  

扩容的时机：  
1、新增节点后，链表元素大于8，且数组长度小于64，则调用tryPresize方法将数组长度扩大到原来的两倍，并使用transfer方法调整位置。  
2、新增节点后，addCount方法记录元素数量，数组元素超过阈值时也会调用transfer方法。  

整个扩容操作分为两个部分：  
第一部分是构建一个`nextTable`，它的容量是原来的两倍，这个操作是单线程完成的。  
第二个部分就是将原来table中的元素复制到`nextTable`中，这里允许多线程进行操作。
先来看一下单线程是如何完成的：  

大体思想就是遍历、复制的过程。首先根据运算得到需要遍历的次数i，然后利用tabAt方法获得i位置的元素，对某个node复制到新表时要上锁。  

如果这个位置为空，就在原table中的i位置放入ForwardingNode节点，表示这个槽位处理完毕；  
如果位置i的元素hash值为-1将advance设为true表示已完成；  
否则，给头节点增加synchronized锁，  
如果这个位置是Node节点（hash>=0），如果它是一个链表的头节点，就构造两个子链表，把他们分别放在nextTable的i和i+n的位置上，位置的判断同HashMap；  
如果这个位置是TreeBin节点，也做一个反序处理，并且判断是否需要untreefi，把处理的结果分别放在nextTable的i和i+n的位置上；  
最后在table的i位置上插入forwardNode节点  表示已经处理过该节点；  
遍历过所有的节点以后就完成了复制工作，这时让nextTable作为新的table，完成扩容。  

多线程的处理：  
如果遍历到的节点是forward节点，就向后继续遍历，再加上给节点上锁的机制，就完成了多线程的控制。多线程遍历节点，处理完一个节点，就将对应位置设为ForwardingNode节点`setTabAt(tab, i, fwd);`，另一个线程看到forward，就向后遍历。这样交叉就完成了复制工作。   
```
/**
* 一个过渡的table表  只有在扩容的时候才会使用
*/
private transient volatile Node<K,V>[] nextTable;

/**
* Moves and/or copies the nodes in each bin to new table. See
* above for explanation.
*/
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
   int n = tab.length, stride;
   if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
       stride = MIN_TRANSFER_STRIDE; // subdivide range
   if (nextTab == null) {            // initiating
       try {
           @SuppressWarnings("unchecked")
           Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];//构造一个nextTable对象 它的容量是原来的两倍
           nextTab = nt;
       } catch (Throwable ex) {      // try to cope with OOME
           sizeCtl = Integer.MAX_VALUE;
           return;
       }
       nextTable = nextTab;
       transferIndex = n;
   }
   int nextn = nextTab.length;
   ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);//构造一个连节点指针 用于标志位
   boolean advance = true;//并发扩容的关键属性 如果等于true 说明这个节点已经处理过
   boolean finishing = false; // to ensure sweep before committing nextTab
   for (int i = 0, bound = 0;;) {
       Node<K,V> f; int fh;
       //这个while循环体的作用就是在控制i--  通过i--可以依次遍历原hash表中的节点
       while (advance) {
           int nextIndex, nextBound;
           if (--i >= bound || finishing)
               advance = false;
           else if ((nextIndex = transferIndex) <= 0) {
               i = -1;
               advance = false;
           }
           else if (U.compareAndSwapInt
                    (this, TRANSFERINDEX, nextIndex,
                     nextBound = (nextIndex > stride ?
                                  nextIndex - stride : 0))) {
               bound = nextBound;
               i = nextIndex - 1;
               advance = false;
           }
       }
       if (i < 0 || i >= n || i + n >= nextn) {
           int sc;
           if (finishing) {
               //如果所有的节点都已经完成复制工作  就把nextTable赋值给table 清空临时对象nextTable
               nextTable = null;
               table = nextTab;
               sizeCtl = (n << 1) - (n >>> 1);//扩容阈值设置为原来容量的1.5倍  依然相当于现在容量的0.75倍
               return;
           }
           //利用CAS方法更新这个扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作
           if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
               if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                   return;
               finishing = advance = true;
               i = n; // recheck before commit
           }
       }
       //如果遍历到的节点为空 则放入ForwardingNode指针
       else if ((f = tabAt(tab, i)) == null)
           advance = casTabAt(tab, i, null, fwd);
       //如果遍历到ForwardingNode节点  说明这个点已经被处理过了 直接跳过  这里是控制并发扩容的核心
       else if ((fh = f.hash) == MOVED)
           advance = true; // already processed
       else {
               //节点上锁
           synchronized (f) {
               if (tabAt(tab, i) == f) {
                   Node<K,V> ln, hn;
                   //如果fh>=0 证明这是一个Node节点
                   if (fh >= 0) {
                       int runBit = fh & n;
                       //以下的部分在完成的工作是构造两个子链表的过程
                       Node<K,V> lastRun = f;
                       for (Node<K,V> p = f.next; p != null; p = p.next) {
                           int b = p.hash & n;
                           if (b != runBit) {
                               runBit = b;
                               lastRun = p;
                           }
                       }
                       if (runBit == 0) {
                           ln = lastRun;
                           hn = null;
                       }
                       else {
                           hn = lastRun;
                           ln = null;
                       }
                       for (Node<K,V> p = f; p != lastRun; p = p.next) {
                           int ph = p.hash; K pk = p.key; V pv = p.val;
                           if ((ph & n) == 0)
                               ln = new Node<K,V>(ph, pk, pv, ln);
                           else
                               hn = new Node<K,V>(ph, pk, pv, hn);
                       }
                       //在nextTable的i位置上插入一个链表
                       setTabAt(nextTab, i, ln);
                       //在nextTable的i+n的位置上插入另一个链表
                       setTabAt(nextTab, i + n, hn);
                       //在table的i位置上插入forwardNode节点  表示已经处理过该节点
                       setTabAt(tab, i, fwd);
                       //设置advance为true 返回到上面的while循环中 就可以执行i--操作
                       advance = true;
                   }
                   //对TreeBin对象进行处理  与上面的过程类似
                   else if (f instanceof TreeBin) {
                       TreeBin<K,V> t = (TreeBin<K,V>)f;
                       TreeNode<K,V> lo = null, loTail = null;
                       TreeNode<K,V> hi = null, hiTail = null;
                       int lc = 0, hc = 0;
                       //构造两个子链表
                       for (Node<K,V> e = t.first; e != null; e = e.next) {
                           int h = e.hash;
                           TreeNode<K,V> p = new TreeNode<K,V>
                               (h, e.key, e.val, null, null);
                           if ((h & n) == 0) {
                               if ((p.prev = loTail) == null)
                                   lo = p;
                               else
                                   loTail.next = p;
                               loTail = p;
                               ++lc;
                           }
                           else {
                               if ((p.prev = hiTail) == null)
                                   hi = p;
                               else
                                   hiTail.next = p;
                               hiTail = p;
                               ++hc;
                           }
                       }
                       //如果扩容后已经不再需要tree的结构 反向转换为链表结构
                       ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                           (hc != 0) ? new TreeBin<K,V>(lo) : t;
                       hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                           (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        //在nextTable的i位置上插入一个链表    
                       setTabAt(nextTab, i, ln);
                       //在nextTable的i+n的位置上插入另一个链表
                       setTabAt(nextTab, i + n, hn);
                        //在table的i位置上插入forwardNode节点  表示已经处理过该节点
                       setTabAt(tab, i, fwd);
                       //设置advance为true 返回到上面的while循环中 就可以执行i--操作
                       advance = true;
                   }
               }
           }
       }
   }
}
```

#### 补充：扩容时两个子链表的构造
拆分链表，构造两个子链表这一块代码，看了半天才看个大概  
不得不感概，Java8中`ConcurrentHashMap`设计的巧妙  

##### 情形一
首先在table数组的i处准备一个链表，用ABCDE表示五个哈希冲突的节点  
字母右边的0/1代表该节点的hash值&n的结果（其中，n为原table数组的长度），事实上，这个结果是看hash值的二进制形式在原数组长度的那一位是0还是1，&操作之后的结果是0或n  
为了方便我用0表示0，用1表示不为0(n)。  
构造的链表如下图所示：  
![123]({{ "/img/post/ConcurrentHashMap/123.png" | prepend: site.baseurl }} )  
首先记录首节点的那一位（0或1），为方便我将节点hash值的那一位记作hBit。`int runBit = fh & n;`，fh为首节点的hash；并记录首节点`Node<K,V> lastRun = f;`。  
```
for (Node<K,V> p = f.next; p != null; p = p.next) {
    int b = p.hash & n;
    if (b != runBit) {
        runBit = b;
        lastRun = p;
    }
}
```
然后，如以上代码所示，进入一个for循环遍历链表  
第一次，b记录B节点的hBit，`b != runBit`成立，更新runBit为节点B的hBit，更新lastRun为B节点；  
第二次，b记录C节点的hBit，`b != runBit`成立，更新runBit为节点C的hBit，更新lastRun为C节点；  
第三次，b记录D节点的hBit，`b != runBit`成立，更新runBit为节点D的hBit，更新lastRun为D节点；  
第四次，b记录E节点的hBit，此时`b != runBit`不成立，不更新runBit为及lastRun；  
第五次，p为null退出循环。  

最后的结果是lastRun为D节点，也就是说这段代码会找到链表最后几个hBit相等的节点的首节点，用lastRun表示，并用runBit记录这些节点的hBit  
我想这是为了后面构造子链表过程中减少节点插入的次数，遍历到lastRun即可停止，因为其后面的节点都属于一个子链表。   

```
if (runBit == 0) {
    ln = lastRun;
    hn = null;
}
else {
    hn = lastRun;
    ln = null;
}
```
如果最后几个节点的hBit为0，代表这些节点属于低位子链，则将ln(低位子链表头)设为lastRun；如果hBit不为0，那这些节点就属于高位子链表，则将hn(高位子链表头)设为lastRun。  
根据我构造的链表，runBit是不为0的，所以将ln设为null，hn设为lastRun即D。  

```
for (Node<K,V> p = f; p != lastRun; p = p.next) {
    int ph = p.hash; K pk = p.key; V pv = p.val;
    if ((ph & n) == 0)
        ln = new Node<K,V>(ph, pk, pv, ln);
    else
        hn = new Node<K,V>(ph, pk, pv, hn);
}
```
遍历链表直到lastRun  
第一次，p = A，A的hBit为0，则在低位链表的ln上采用头节点插入A，此时低位链表为`A -> null`；  
第二次，p = B，B的hBit不为0，则在高位链表的hn上采用头节点插入B，此时高位链表为`B -> D -> E -> null`；  
第三次，p = C，C的hBit为0，则在低位链表的ln上采用头节点插入C，此时低位链表为`C -> A -> null`；  
第四次，p = D，等于lastRun，退出循环。  

剩下的节点E就是D的后继节点，所以不需遍历。  
此时，低位链表为`C -> A -> null`，高位链表为`B -> D -> E -> null`

```
setTabAt(nextTab, i, ln);
setTabAt(nextTab, i + n, hn);
setTabAt(tab, i, fwd);
```
setTabAt方法必须在获取锁的条件下执行，这三行代码就是将ln放到新table的i处，hn放到新table的i+n处，然后原table的i处改为forwardingNode，其hash值为-1，目的是为了告诉其他线程该节点已经处理完毕。  

因此，最后的结果为（图中(·)代表在数组中的位置）  
![res1]({{ "/img/post/ConcurrentHashMap/res1.png" | prepend: site.baseurl }} )  

##### 情形二
![res2]({{ "/img/post/ConcurrentHashMap/res2.png" | prepend: site.baseurl }} )  
改变链表每个节点的hBit，0变成1，1变成0。  得到箭头上方的图，扩容后的结果位于箭头下方。  
这里ln为lastRun，hn为null。  

*综合情形一和情形二，我们可以看到，原链表从头遍历到lastRun，就可以得到一个完整的子链表。当然，如果lastRun的hBit为0，遍历到lastRun就可以得到高位子链表，又因为子链表的构造过程是头节点插入，所以高位子链表是一个反转的链表。例如情形二的`C -> A`；如果lastRun的hBit不为0，遍历到lastRun就可以得到低位子链表，同样也是反转的。例如情形一的`C -> A`*  

然后我看网上很多博客都说两个子链表是一个正序一个倒序，图都是一样的，估计是抄来的。  
事实上，我认为这种说法并不准确，我之前假设的两个情形确实符合这个说法。因此我再举个例子。  
##### 情形三
![res3]({{ "/img/post/ConcurrentHashMap/res3.png" | prepend: site.baseurl }} )  
假设在原数组的i处有这样的一个链表，节点数6<8。  
经过第一个循环过程后，lastRun为E，runBit不为0。  
hn设为E，ln为null。  
经过第二个循环，构造出两个子链表，低位链表为`D -> A`，高位链表为`C -> B -> E -> F`。  
结果为  
![res4]({{ "/img/post/ConcurrentHashMap/res4.png" | prepend: site.baseurl }} )  
可以得出结论，两个子链表一个是逆序，一个是自lastRun之后是正序，之前是倒序。如果之前只有一个节点那整个子链表就是正序的，多于一个节点时就不是了。  
