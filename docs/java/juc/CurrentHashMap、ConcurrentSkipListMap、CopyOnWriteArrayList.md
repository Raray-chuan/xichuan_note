
## 一.CurrentHashMap


### 摘要
在涉及到Java多线程开发时，如果我们使用HashMap可能会导致死锁问题，使用HashTable效率又不高。而ConcurrentHashMap既可以保持同步也可以提高并发效率，所以这个时候ConcurrentHashmap是我们最好的选择。



### 为什么使用ConcurrentHashMap
* 在多线程环境中使用HashMap的put方法有可能导致程序死循环，因为多线程可能会导致HashMap形成环形链表，即链表的一个节点的next节点永不为null，就会产生死循环。这时，CPU的利用率接近100%，所以并发情况下不能使用HashMap。
* HashTable通过使用synchronized保证线程安全，但在线程竞争激烈的情况下效率低下。因为当一个线程访问HashTable的同步方法时，其他线程只能阻塞等待占用线程操作完毕。
* ConcurrentHashMap使用分段锁的思想，对于不同的数据段使用不同的锁，可以支持多个线程同时访问不同的数据段，这样线程之间就不存在锁竞争，从而提高了并发效率。




### 简介
在阅读ConcurrentHashMap的源码时，有一段相关描述。
The primary design goal of this hash table is to maintain concurrent readability(typically method get(), but also iterators and related methods) while minimizing update contention. Secondary goals are to keep space consumption about the same or better than java.util.HashMap, and to support high initial insertion rates on an empty table by many threads.
大致意思就是： ConcurrentHashMap的主要设计目的是保持并发的可读性（通常是指的get()方法的使用，同时也包括迭代器和相关方法），同时最小化更新征用（即在进行插入操作或者扩容时也可以保持其他数据段的访问）。第二个目标就是在空间利用方面保持与HashMap一致或者更好，并且支持多线程在空表的初始插入速率。




### Java7与Java8中的ConcurrentHashMap：
在ConcurrentHashMap中主要通过锁分段技术实现上述目标。
在Java7中，ConcurrentHashMap由Segment数组结构和HashEntry数组组成。Segment是一种可重入锁，是一种数组和链表的结构，一个Segment中包含一个HashEntry数组，每个HashEntry又是一个链表结构。正是通过Segment分段锁，ConcurrentHashMap实现了高效率的并发。
在Java8中，ConcurrentHashMap去除了Segment分段锁的数据结构，主要是基于CAS操作保证保证数据的获取以及使用synchronized关键字对相应数据段加锁实现了主要功能，这进一步提高了并发性。同时同时为了提高哈希碰撞下的寻址性能，Java 8在链表长度超过一定阈值(8)时将链表（寻址时间复杂度为O(N)）转换为红黑树（寻址时间复杂度为O(long(N)))。



### Java8中ConcurrentHashMap的结构
在Java8中，ConcurrentHashMap弃用了Segment类，但是保留了Segment属性，用于序列化。目前ConcurrentHashMap采用Node类作为基本的存储单元，每个键值对(key-value)都存储在一个Node中。同时Node也有一些子类，TreeNodes用于树结构中（当链表长度大于8时转化为红黑树）；TreeBins用于维护TreeNodes。当链表转树时，用于封装TreeNode。也就是说，ConcurrentHashMap的红黑树存放的是TreeBin，而不是treeNode；ForwordingNodes是一个重要的结构，它用于ConcurrentHashMap扩容时，是一个标志节点，内部有一个指向nextTable的属性，同时也提供了查找的方法；
```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;  //使用了volatile属性
        volatile Node<K,V> next;  //使用了volatile属性

        Node(int hash, K key, V val) {
            this.hash = hash;
            this.key = key;
            this.val = val;
        }
        Node(int hash, K key, V val, Node<K,V> next) {
            this(hash, key, val);
            this.next = next;
        }
        public final K getKey()     { return key; }
        public final V getValue()   { return val; }
        public final int hashCode() { return key.hashCode() ^ val.hashCode(); }
        public final String toString() {...}
        public final V setValue(V value) {...}

        public final boolean equals(Object o) {...}

        /**
         * Virtualized support for map.get(); overridden in subclasses.
         */
        Node<K,V> find(int h, Object k) {...}
    }
```

处理Node之外，Node的一个子类ForwardingNodes也是一个重要的结构，它主要作为一个标记，在处理并发时起着关键作用，有了ForwardingNodes，也是ConcurrentHashMap有了分段的特性，提高了并发效率。
```java
 /**
     * A node inserted at head of bins during transfer operations.
     */
    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
	        //hash值默认为MOVED(-1)
            super(MOVED, null, null);
            this.nextTable = tab;
        }

        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            //在nextTable中查找，nextTable可以看做是当前hash表的一个副本
            outer: for (Node<K,V>[] tab = nextTable;;) {
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;
                for (;;) {
                    int eh; K ek;
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) {
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            return e.find(h, k);
                    }
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
    }
```



### ConcurrentHashMap中的原子操作
在ConcurrentHashMap中通过原子操作查找元素、替换元素和设置元素。这些原子操作起着非常关键的作用，你可以在所有ConcurrentHashMap的基本功能中看到它们的身影。
```java
 static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectAcquire(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSetObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }

    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectRelease(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```



### ConcurrentHashMap的功能实现
**1.ConcurrentHashMap初始化**
在介绍初始化之前先介绍一个重要的参数sizeCtl，含义如下：
```java
 /**
     * Table initialization and resizing control.  When negative,the
     * table is being initialized or resized: -1 for initialization,
     * else -(1 + the number of active resizing threads). Otherwise,
     * when table is null, holds the initial table size to use upon
     * creation, or 0 for default. After initialization, holds the
     * next element count value upon which to resize the table.
     * Hash表的初始化和调整大小的控制标志。为负数，Hash表正在初始化或者扩容;
     * (-1表示正在初始化,-N表示有N-1个线程在进行扩容)
     * 否则，当表为null时，保存创建时使用的初始化大小或者默认0;
     * 初始化以后保存下一个调整大小的尺寸。
     */
    private transient volatile int sizeCtl;
```

这个参数起到一个控制标志的作用，在ConcurrentHashMap初始化和扩容都有用到。 ConcurrentHashMap构造函数只是设置了一些参数，并没有对Hash表进行初始化。当在从插入元素时，才会初始化Hash表。在开始初始化的时候，首先判断sizeCtl的值，如果sizeCtl < 0，说明有线程在初始化，当前线程便放弃初始化操作。否则，将SIZECTL设置为-1，Hash表进行初始化。初始化成功以后，将sizeCtl的值设置为当前的容量值。
```java
 /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
	        //sizeCtl小于0，正在初始化
            if ((sc = sizeCtl) < 0)
                //调用yield()函数，使线程让出CPU资源
                Thread.yield(); // lost initialization race; just spin
            //设置SIZECTL为-1，表示正在初始化
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2); //n-(1/4)n,即默认的容量(n * loadFactor)
                    }
                } finally {
                    sizeCtl = sc; //重新设置sizeCtl
                }
                break;
            }
        }
        return tab;
    }
```

**2.确定元素在Hash表的索引**
通过hash算法可以将元素分散到哈希桶中。在ConcurrentHashMap中通过如下方法确定数组索引： 第一步：
```java
static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS; 
    }
```

第二步：(length-1) & (h ^ (h >>> 16)) & HASH_BITS);



### ConcurrentHashMap的put方法
* 如果key或者value为null，则抛出空指针异常；
* 如果table为null或者table的长度为0，则初始化table，调用initTable()方法。
* 计算当前键值的索引位置，如果Hash表中当前节点为null，则将元素直接插入。(注意，这里使用的就是前面锁说的CAS操作)
* 如果当前位置的节点元素的hash值为-1，说明这是一个ForwaringNodes节点，即正在进行扩容。那么当前线程加入扩容。
* 当前节点不为null，对当前节点加锁，将元素插入到当前节点。在Java8中，当节点长度大于8时，就将节点转为树的结构。
```java
	/** Implementation for put and putIfAbsent */
	final V putVal(K key, V value, boolean onlyIfAbsent) {
		//数据不合法，抛出异常
        if (key == null || value == null) throw new NullPointerException();
        //计算索引的第一步，传入键值的hash值
        int hash = spread(key.hashCode());
        int binCount = 0; //保存当前节点的长度
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable(); //初始化Hash表
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
	            //利用CAS操作将元素插入到Hash表中
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                    break;  // no lock when adding to empty bin(插入null的节点，无需加锁)
            }
            else if ((fh = f.hash) == MOVED) //f.hash == -1 
	            //正在扩容，当前线程加入扩容
                tab = helpTransfer(tab, f);
            else if (onlyIfAbsent && fh == hash &&  // check first node
                     ((fk = f.key) == key || fk != null && key.equals(fk)) &&
                     (fv = f.val) != null)
                return fv;
            else {
                V oldVal = null;
                //当前节点加锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                //插入的元素键值的hash值有节点中元素的hash值相同，替换当前元素的值
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
	                                    //替换当前元素的值
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                //没有相同的值，直接插入到节点中
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        //节点为树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
	                                //替换旧值
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
	                //如果节点长度大于8,转化为树
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



### ConcurrentHashMap的扩容机制

当ConcurrentHashMap中元素的数量达到cap * loadFactor时，就需要进行扩容。扩容主要通过transfer()方法进行，当有线程进行put操作时，如果正在进行扩容，可以通过helpTransfer()方法加入扩容。也就是说，ConcurrentHashMap支持多线程扩容，多个线程处理不同的节点。
* 开始扩容，首先计算步长，也就是每个线程分配到的扩容的节点数(默认是16)。这个值是根据当前容量和CPU的数量来计算(stride = (NCPU > 1) ? (n >>> 3) / NCPU : n)，最小是16。
* 接下来初始化临时的Hash表nextTable，如果nextTable为null，初始化nextTable长度为原来的2倍；
* 通过计算出的步长开始遍历Hash表，其中坐标是通过一个原子操作(compareAndSetInt)记录。通过一个while循环，如果在一个线程的步长内便跳过此节点。否则转下一步；
  如果当前节点为空，之间将此节点在旧的Hash表中设置为一个ForwardingNodes节点，表示这个节点已经被处理过了。
* 如果当前节点元素的hash值为MOVED(f.hash == -1)，表示这是一个ForwardingNodes节点，则直接跳过。否则，开始重新处理节点；
* 对当前节点进行加锁，在这一步的扩容操作中，重新计算元素位置的操作与HashMap中是一样的，即当前元素键值的hash与长度进行&操作，如果结果为0则保持位置不变，为1位置就是i+n。其中进行处理的元素是最后一个符合条件的元素，所以扩容后可能是一种倒序，但在Hash表中这种顺序也没有太大的影响。
* 最后如果是链表结构直接获得高位与低位的新链表节点，如果是树结构，同样计算高位与低位的节点，但是需要根据节点的长度进行判断是否需要转化为树的结构。
```java
   /**
     * Moves and/or copies the nodes in each bin to new table. See
     * above for explanation.
         */
          private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        //根据长度和CPU的数量计算步长，最小是16
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                //初始化新的Hash表，长度为原来的2倍
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        //初始化ForwardingNodes节点
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true; //是否跨过节点的标记
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            //根据步长判断是否需要跨过节点
            while (advance) {
                int nextIndex, nextBound;
                //到达没有处理的节点下标
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                //所有节点都已经接收处理
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSetInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                //更新下表transferIndex,在步长的范围内都忽略
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            //所有节点都被接收处理或者已经处理完毕
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                //处理完毕
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    //更新sizeCtl
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                //判断所有节点是否全部被处理
                if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            //如果节点为null,直接标记为已接收处理
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            //键值的hash为-1，表示这是一个ForwardingNodes节点，已经被处理
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
               //对当前节点进行加锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                           //索引位置是否改变的标志
                            int runBit = fh & n;
                            Node<K,V> lastRun = f; //最后一个元素
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                //重新计算更新直到最后一个元素
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            //runBit = 0,保持位置不变
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            //runBit = 1,位置时i+n
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            //重新遍历节点元素
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                //构建低位(位置不变)新的链表
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                    //构建高位(i+n)新的链表
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            //将新的链表设置到新的Hash表中相应的位置
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            //将原来的Hash表中相应位置的节点设置为ForwardingNodes节点
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        //如果节点是树的结构
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                //同样的方式计算新的索引位置
                                if ((h & n) == 0) {
                                   //构建新的链表结构
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                //构建新的链表结构
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            //判断是否需要转化为树
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            //判断是否需要转化为树
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            //将新的链表设置到新的Hash表中相应的位置
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            //将原来的Hash表中相应位置的节点设置为ForwardingNodes节点
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```



### ConcurrentHashMap的get方法

ConcurrentHashMap的get方法就是从Hash表中读取数据，而且与扩容不冲突。该方法没有同步锁。
* 通过键值的hash计算索引位置，如果满足条件，直接返回对应的值；
* 如果相应节点的hash值小于0 ，即该节点在进行扩容，直接在调用ForwardingNodes节点的find方法进行查找。
* 否则，遍历当前节点直到找到对应的元素。
```java
/**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code key.equals(k)},
     * then this method returns {@code v}; otherwise it returns
     * {@code null}.  (There can be at most one such mapping.)
     *
     * @throws NullPointerException if the specified key is null
     */
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        //满足条件直接返回对应的值
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            //e.hash<0，正在扩容
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            //遍历当前节点
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

### 总结
ConcurrentHashMap就介绍到这里，以上主要是对ConcurrentHashMap中经常使用到的特性进行分析，如果对其他内容感兴趣，可以阅读相应的源码。








## 二.ConcurrentSkipListMap、ConcurrentSkipListSet

 ### 2.1 ConcurrentSkipListMap
JDK1.6时，为了对高并发环境下的有序Map提供更好的支持，J.U.C新增了一个ConcurrentNavigableMap接口，ConcurrentNavigableMap很简单，它同时实现了NavigableMap和ConcurrentMap接口。

ConcurrentNavigableMap接口提供的功能也和NavigableMap几乎完全一致，很多方法仅仅是返回的类型不同。

NavigableMap接口，进一步扩展了SortedMap的功能，提供了根据指定Key返回最接近项、按升序/降序返回所有键的视图等功能。
J.U.C提供了基于ConcurrentNavigableMap接口的一个实现——ConcurrentSkipListMap。ConcurrentSkipListMap可以看成是并发版本的TreeMap，但是和TreeMap不同是，ConcurrentSkipListMap并不是基于红黑树实现的，其底层是一种类似跳表（Skip List）的结构。



ConcurrentSkipListMap是线程安全的有序的哈希表，适用于高并发的场景。
ConcurrentSkipListMap和TreeMap，它们虽然都是有序的哈希表。但是，第一，它们的线程安全机制不同，TreeMap是非线程安全的，而ConcurrentSkipListMap是线程安全的。第二，ConcurrentSkipListMap是通过跳表实现的，而TreeMap是通过红黑树实现的。

在4线程1.6万数据的条件下，ConcurrentHashMap 存取速度是ConcurrentSkipListMap 的4倍左右。

但ConcurrentSkipListMap有几个ConcurrentHashMap 不能比拟的优点：
1、ConcurrentSkipListMap 的key是有序的。
2、ConcurrentSkipListMap 支持更高的并发。ConcurrentSkipListMap 的存取时间是log（N），和线程数几乎无关。也就是说在数据量一定的情况下，并发的线程越多，ConcurrentSkipListMap越能体现出他的优势。

在非多线程的情况下，应当尽量使用TreeMap。此外对于并发性相对较低的并行程序可以使用Collections.synchronizedSortedMap将TreeMap进行包装，也可以提供较好的效率。对于高并发程序，应当使用ConcurrentSkipListMap，能够提供更高的并发度。

所以在多线程程序中，如果需要对Map的键值进行排序时，请尽量使用ConcurrentSkipListMap，可能得到更好的并发度。

注意，调用ConcurrentSkipListMap的size时，由于多个线程可以同时对映射表进行操作，所以映射表需要遍历整个链表才能返回元素个数，这个操作是个O(log(n))的操作。



栗子：
```java
import java.util.Map;
import java.util.concurrent.ConcurrentNavigableMap;
import java.util.concurrent.ConcurrentSkipListMap;

public class ConcurrentSkipListMapTest {
        public static void main(String[] args) {
            ConcurrentSkipListMap<String, Contact> map = new ConcurrentSkipListMap<>();
            Thread threads[]=new Thread[25];
            int counter=0;
            //创建和启动25个任务，对于每个任务指定一个大写字母作为ID
            for (char i='A'; i<'Z'; i++) {
                Task0 task=new Task0(map, String.valueOf(i));
                threads[counter]=new Thread(task);
                threads[counter].start();
                counter++;
            }
            //使用join()方法等待线程的结束
            for (int i=0; i<25; i++) {
                try {
                    threads[i].join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.printf("Size of the map: %d\n",map.size());
            Map.Entry<String, Contact> element;
            Contact contact;
           // 使用firstEntry()方法获取map的第一个实体，并输出。
            element=map.firstEntry();
            contact=element.getValue();
            System.out.printf("First Entry: %s: %s\n",contact.
                    getName(),contact.getPhone());
            //使用lastEntry()方法获取map的最后一个实体，并输出。
            element=map.lastEntry();
            contact=element.getValue();
            System.out.printf("Last Entry: %s: %s\n",contact.
                    getName(),contact.getPhone());
            //使用subMap()方法获取map的子map，并输出。

            System.out.printf("Submap from A1996 to B1002: \n");

            ConcurrentNavigableMap<String, Contact> submap=map.

            subMap("A1996", "B1001");

            do {
                element=submap.pollFirstEntry();
                if (element!=null) {
                    contact=element.getValue();
                    System.out.printf("%s: %s\n",contact.getName(),contact.
                            getPhone());
                }
            } while (element!=null);
        }

        }

class Contact {
    private String name;
    private String phone;

    public Contact(String name, String phone) {
        this.name = name;
        this.phone = phone;

    }

    public String getName() {
        return name;
    }

    public String getPhone() {
        return phone;
    }
}

class Task0 implements Runnable {

    private ConcurrentSkipListMap<String, Contact> map;
    private String id;

    public Task0(ConcurrentSkipListMap<String, Contact> map, String id) {
        this.id = id;
        this.map = map;
    }

    @Override
    public void run() {

        for (int i = 0; i < 1000; i++) {
            Contact contact = new Contact(id, String.valueOf(i + 1000));
            map.put(id + contact.getPhone(), contact);
        }
   }
}        
```



### 2.2 ConcurrentSkipListSet
ConcurrentSkipListSet，是JDK1.6时J.U.C新增的一个集合工具类，它是一种有序的SET类型。
ConcurrentSkipListSet实现了NavigableSet接口，ConcurrentSkipListMap实现了NavigableMap接口，以提供和排序相关的功能，维持元素的有序性，所以ConcurrentSkipListSet就是一种为并发环境设计的有序SET工具类。

栗子：
```java
import java.util.concurrent.ConcurrentSkipListSet;

public class ConcurrentSkipListSetTest {
    public static void main(String[] args) {
        ConcurrentSkipListSet<Contact1> set = new ConcurrentSkipListSet<>();
        Thread threads[]=new Thread[25];
        int counter=0;
        //创建和启动25个任务，对于每个任务指定一个大写字母作为ID
        for (char i='A'; i<'Z'; i++) {
            Task1 task=new Task1(set, String.valueOf(i));
            threads[counter]=new Thread(task);
            threads[counter].start();
            counter++;
        }
        //使用join()方法等待线程的结束
        for (int i=0; i<25; i++) {
            try {
                threads[i].join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.printf("Size of the set: %d\n",set.size());
        Contact1 contact;
        // 使用first方法获取set的第一个实体，并输出。
        contact=set.first();
        System.out.printf("First Entry: %s: %s\n",contact.
                getName(),contact.getPhone());
        //使用last方法获取set的最后一个实体，并输出。
        contact=set.last();
        System.out.printf("Last Entry: %s: %s\n",contact.
                getName(),contact.getPhone());

    }

}

class Contact1 implements Comparable<Contact1> {
    private String name;
    private String phone;

    public Contact1(String name, String phone) {
        this.name = name;
        this.phone = phone;

    }

    public String getName() {
        return name;
    }

    public String getPhone() {
        return phone;
    }

    @Override
    public int compareTo(Contact1 o) {
        return name.compareTo(o.name);
    }
}

class Task1 implements Runnable {

    private ConcurrentSkipListSet<Contact1> set;
    private String id;

    public Task1(ConcurrentSkipListSet<Contact1> set, String id) {
        this.id = id;
        this.set = set;
    }

    @Override
    public void run() {

        for (int i = 0; i < 100; i++) {
            Contact1 contact = new Contact1(id, String.valueOf(i + 100));
            set.add(contact);
        }
    }
}
```







## 三. CopyOnWriteArrayList、CopyOnWriteArraySet
### 3.1 CopyOnWriteArrayList

Copy-On-Write简称COW，是一种用于程序设计中的优化策略。其基本思路是，从一开始大家都在共享同一个内容，当某个人想要修改这个内容的时候，才会真正把内容Copy出去形成一个新的内容然后再改，这是一种延时懒惰策略。从JDK1.5开始Java并发包里提供了两个使用CopyOnWrite机制实现的并发容器,它们是CopyOnWriteArrayList和CopyOnWriteArraySet。

CopyOnWrite容器非常有用，可以在非常多的并发场景中使用到。什么是CopyOnWrite容器CopyOnWrite容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。

CopyOnWrite容器有很多优点，但是同时也存在两个问题，即内存占用问题和数据一致性问题。所以在开发的时候需要注意一下。内存占用问题。因为CopyOnWrite的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，旧的对象和新写入的对象（注意:在复制的时候只是复制容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，那么这个时候很有可能造成频繁的Yong GC和Full GC。之前我们系统中使用了一个服务由于每晚使用CopyOnWrite机制更新大对象，造成了每晚15秒的Full GC，应用响应时间也随之变长。针对内存占用问题，可以通过压缩容器中的元素的方法来减少大对象的内存消耗，比如，如果元素全是10进制的数字，可以考虑把它压缩成36进制或64进制。或者不使用CopyOnWrite容器，而使用其他的并发容器，如ConcurrentHashMap。数据一致性问题。CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以如果你希望写入的的数据，马上能读到，请不要使用CopyOnWrite容器。

源码：略。

使用：
```java
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

class ReadThread implements Runnable {
    private List<Integer> list;

    public ReadThread(List<Integer> list) {
        this.list = list;
    }

    @Override
    public void run() {
        System.out.print("size:="+list.size()+",::");
        for (Integer ele : list) {
            System.out.print(ele + ",");
        }
        System.out.println();
    }
}

class WriteThread implements Runnable {
    private List<Integer> list;

    public WriteThread(List<Integer> list) {
        this.list = list;
    }

    @Override
    public void run() {
        this.list.add(9);
    }
}


public class TestCopyOnWriteArrayListTest {

    private void test() {
        //1、初始化CopyOnWriteArrayList
        List<Integer> tempList = Arrays.asList(new Integer [] {1,2});
        CopyOnWriteArrayList<Integer> copyList = new CopyOnWriteArrayList<>(tempList);


        //2、模拟多线程对list进行读和写
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        executorService.execute(new ReadThread(copyList));
        executorService.execute(new WriteThread(copyList));
        executorService.execute(new WriteThread(copyList));
        executorService.execute(new WriteThread(copyList));
        executorService.execute(new ReadThread(copyList));
        executorService.execute(new WriteThread(copyList));
        executorService.execute(new ReadThread(copyList));
        executorService.execute(new WriteThread(copyList));
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        System.out.println("copyList size:"+copyList.size());
    }


    public static void main(String[] args) {
        new TestCopyOnWriteArrayList().test();
    }
}
```

### 3.2 CopyOnWriteArraySet
CopyOnWriteArraySet相对CopyOnWriteArrayList用来存储不重复的对象，是线程安全的。虽然继承了AbstractSet类，但CopyOnWriteArraySet与HashMap 完全不同，内部是用CopyOnWriteArrayList实现的，实现不重复的特性也是直接调用CopyOnWriteArrayList的方法实现的，感觉加的最有用的函数就是eq函数判断对象是否相同



来自：
[Java容器（二）-CurrentHashMap详解（JDK1.8）](https://blog.csdn.net/programerxiaoer/article/details/80040090)

[Java并发集合（二）-ConcurrentSkipListMap分析和使用](https://www.cnblogs.com/java-zzl/p/9767255.html)










