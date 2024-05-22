## ConcurrentHashMap 1.8
### 存储结构
![](https://raw.githubusercontent.com/danmuking/image/main/35bb2a5f86ccadda1932d71e7f94289c.png)
Java8 ConcurrentHashMap 存储结构（图片来自 javadoop）
可以发现 Java8 的 ConcurrentHashMap 相对于 Java7 来说变化比较大，不再是之前的 **Segment 数组 + HashEntry 数组 + 链表**，而是 **Node 数组 + 链表 / 红黑树**。当冲突链表达到一定长度时，链表会转换成红黑树。
### ConcurrentHashMap几个重要概念 
下面是几个重要的属性
```java
private static final int MAXIMUM_CAPACITY = 1 << 30;
private static final int DEFAULT_CAPACITY = 16;
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
static final int MOVED     = -1; // 表示正在转移
static final int TREEBIN   = -2; // 表示已经转换成树
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
transient volatile Node<K,V>[] table;//默认没初始化的数组，用来保存元素
private transient volatile Node<K,V>[] nextTable;//转移的时候用的数组
/**
 * 用来控制表初始化和扩容的，默认值为0，当在初始化的时候指定了大小，这会将这个大小保存在sizeCtl中
 *  -1 说明正在初始化
    -N 说明有 N-1 个线程正在进行扩容
     0 默认状态，表明table没有初始化
    >0 表示 table 扩容的阈值，如果 table 已经初始化。
 */
private transient volatile int sizeCtl;
```
### 初始化
```java
// 这构造函数里，什么都不干
public ConcurrentHashMap() {
}
// 指定初始化容量
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```
### 初始化 initTable
```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //　如果 sizeCtl < 0 ,说明另外的线程执行CAS 成功，正在进行初始化。
        if ((sc = sizeCtl) < 0)
            // 让出 CPU 使用权
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // sc为0.75数组大小，也就是sc为扩容阈值
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}

```
### put 过程分析
#### 流程图

![](https://raw.githubusercontent.com/danmuking/image/main/c82bd841963554626af907476c3d6943.jpeg)
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 得到 hash 值
    int hash = spread(key.hashCode());
    // 用于记录相应链表的长度
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组"空"，进行数组初始化
        if (tab == null || (n = tab.length) == 0)
            // 初始化数组，后面会详细介绍
            tab = initTable();

        // 找该 hash 值对应的数组下标，得到第一个节点 f
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果数组该位置为空，
            //    用一次 CAS 操作将这个新值放入其中即可，这个 put 操作差不多就结束了，可以拉到最后面了
            //          如果 CAS 失败，那就是有并发操作，进到下一个循环就好了
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // hash 居然可以等于 MOVED，这个需要到后面才能看明白，不过从名字上也能猜到，肯定是因为在扩容
        else if ((fh = f.hash) == MOVED)
            // 帮助数据迁移，这个等到看完数据迁移部分的介绍后，再理解这个就很简单了
            tab = helpTransfer(tab, f);

        else { // 到这里就是说，f 是该位置的头节点，而且不为空

            V oldVal = null;
            // 获取数组该位置的头节点的监视器锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // 头节点的 hash 值大于 0，说明是链表
                        // 用于累加，记录链表的长度
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果发现了"相等"的 key，判断是否要进行值覆盖，然后也就可以 break 了
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 到了链表的最末端，将这个新值放到链表的最后面
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        // 调用红黑树的插值方法插入新节点
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
                // 判断是否要将链表转换为红黑树，临界值和 HashMap 一样，也是 8
                if (binCount >= TREEIFY_THRESHOLD)
                    // 如果当前数组的长度小于 64，那么会选择进行数组扩容，而不是转换为红黑树
                    //    具体源码我们就不看了，扩容部分后面说
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 
    addCount(1L, binCount);
    return null;
}
```
### 扩容: tryPresize
这里的扩容也是做翻倍扩容的，扩容后数组容量为原来的 2 倍。
```java
// 首先要说明的是，方法参数 size 传进来的时候就已经翻了倍了
private final void tryPresize(int size) {
    // c: size 的 1.5 倍，再加 1，再往上取最近的 2 的 n 次方。
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;

        // 这个 if 分支和之前说的初始化数组的代码基本上是一样的，在这里，我们可以不用管这块代码
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2); // 0.75 * n
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            // 扩容戳，为一个负值
            // 高16为为扩容标记
            // 低16为为扩容线程数+1，从2开始是因为-1已经作为初始化标记
            int rs = resizeStamp(n);
        	 /*
             * 如果正在扩容Table的话，则帮助扩容
             * 否则的话，开始新的扩容
             * 在transfer操作，将第一个参数的table中的元素，移动到第二个参数的table中去，
             * 虽然此时第二个参数设置的是null，但是，在transfer方法中，当第二个参数为null的时候，
             * 会创建一个两倍大小的table
             */
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 2. 用 CAS 将 sizeCtl 加 1，然后执行 transfer 方法
                //    此时 nextTab 不为 null
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 1. 将 sizeCtl 设置为 (rs << RESIZE_STAMP_SHIFT) + 2)
            //     我是没看懂这个值真正的意义是什么? 不过可以计算出来的是，结果是一个比较大的负数
            //  调用 transfer 方法，此时 nextTab 参数为 null
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```
### 数据迁移: transfer
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;

    // stride 在单核下直接等于 n，多核模式下为 (n>>>3)/NCPU，最小值是 16
    // stride 可以理解为”步长“，有 n 个位置是需要进行迁移的，
    //   将这 n 个任务分为多个任务包，每个任务包有 stride 个任务
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range

    // 如果 nextTab 为 null，先进行一次初始化
    //    前面我们说了，外围会保证第一个发起迁移的线程调用此方法时，参数 nextTab 为 null
    //       之后参与迁移的线程调用此方法时，nextTab 不会为 null
    if (nextTab == null) {
        try {
            // 容量翻倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // nextTable 是 ConcurrentHashMap 中的属性
        nextTable = nextTab;
        // transferIndex 也是 ConcurrentHashMap 的属性，用于控制迁移的位置
        transferIndex = n;
    }

    int nextn = nextTab.length;

    // ForwardingNode 翻译过来就是正在被迁移的 Node
    // 这个构造方法会生成一个Node，key、value 和 next 都为 null，关键是 hash 为 MOVED
    // 后面我们会看到，原数组中位置 i 处的节点完成迁移工作后，
    //    就会将位置 i 处设置为这个 ForwardingNode，用来告诉其他线程该位置已经处理过了
    //    所以它其实相当于是一个标志。
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);


    // advance 指的是做完了一个位置的迁移工作，可以准备做下一个位置的了
    boolean advance = true;
    // 迁移工作是否全部完成
    boolean finishing = false; // to ensure sweep before committing nextTab

    /*
     * 下面这个 for 循环，最难理解的在前面，而要看懂它们，应该先看懂后面的，然后再倒回来看
     * 
     */

    // i 是位置索引，bound 是边界，注意是从后往前
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;

        // 下面这个 while 真的是不好理解
        // advance 为 true 表示可以进行下一个位置的迁移了
        //   简单理解结局: i 指向了 transferIndex，bound 指向了 transferIndex-stride
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;

            // 将 transferIndex 值赋给 nextIndex
            // 这里 transferIndex 一旦小于等于 0，说明原数组的所有位置都有相应的线程去处理了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                // 看括号中的代码，nextBound 是这次迁移任务的边界，注意，是从后往前
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                // 所有的迁移操作已经完成
                nextTable = null;
                // 将新的 nextTab 赋值给 table 属性，完成迁移
                table = nextTab;
                // 重新计算 sizeCtl: n 是原数组长度，所以 sizeCtl 得出的值将是新数组长度的 0.75 倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }

            // 之前我们说过，sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2
            // 然后，每有一个线程参与迁移就会将 sizeCtl 加 1，
            // 这里使用 CAS 操作对 sizeCtl 进行减 1，代表做完了属于自己的任务
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 任务结束，方法退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;

                // 到这里，说明 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT，
                // 也就是说，所有的迁移任务都做完了，也就会进入到上面的 if(finishing){} 分支了
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode ”空节点“
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 该位置处是一个 ForwardingNode，代表该位置已经迁移过了
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 对数组该位置处的结点加锁，开始处理数组该位置处的迁移工作
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 头节点的 hash 大于 0，说明是链表的 Node 节点
                    if (fh >= 0) {
                        // 下面这一块和 Java7 中的 ConcurrentHashMap 迁移是差不多的，
                        // 需要将链表一分为二，
                        //   找到原链表中的 lastRun，然后 lastRun 及其之后的节点是一起进行迁移的
                        //   lastRun 之前的节点需要进行克隆，然后分到两个链表中
                        int runBit = fh & n;
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
                        // 其中的一个链表放在新数组的位置 i
                        setTabAt(nextTab, i, ln);
                        // 另一个链表放在新数组的位置 i+n
                        setTabAt(nextTab, i + n, hn);
                        // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                        //    其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                        setTabAt(tab, i, fwd);
                        // advance 设置为 true，代表该位置已经迁移完毕
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        // 红黑树的迁移
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
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
                        // 如果一分为二后，节点数小于等于6，那么将红黑树转换回链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;

                        // 将 ln 放置在新数组的位置 i
                        setTabAt(nextTab, i, ln);
                        // 将 hn 放置在新数组的位置 i+n
                        setTabAt(nextTab, i + n, hn);
                        // 将原数组该位置处设置为 fwd，代表该位置已经处理完毕，
                        //    其他线程一旦看到该位置的 hash 值为 MOVED，就不会进行迁移了
                        setTabAt(tab, i, fwd);
                        // advance 设置为 true，代表该位置已经迁移完毕
                        advance = true;
                    }
                }
            }
        }
    }
}
```
### **扩容戳有什么用**
当满足扩容条件之后，首先会先调用一个方法来获取扩容戳，这个扩容戳比较有意思，要理解扩容戳，必须从二进制的角度来分析。resizeStamp 方法就一句话，其中 `RESIZE_STAMP_BITS`是一个默认值 16。
```java
static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }
```
这里面关键就是 Integer.numberOfLeadingZeros(n) 这个方法，这个方法源码就不贴出来了，实际上这个方法就是做一件事，那就是**获取当前数据转成二进制后的最高非 0 位前的 0 的个数**。
这句话有点拗口，我们举个例子，就以 16 为准，16 转成二进制是 10000，最高非 0 位是在第 5 位，因为 int 类型是 32 位，所以他前面还有 27 位，而且都是 0，那么这个方法得到的结果就是 27（1 的前面还有 27 个 0）。
然后 1 << (RESIZE_STAMP_BITS - 1) 在当前版本就是 1<<15，也就是得到一个二进制数 1000000000000000，这里也是要做一件事，把这个 **1 移动到第 16 位**。最后这两个数通过 | 操作一定得到的结果就是第 16 位是 1，因为 int 是 32 位，最多也就是 32 个 0，而且因为 n 的默认大小是 16（ConcurrentHashMap 默认大小），所以实际上最多也就是 27（11011），也就是说这个数最高位的 1 也只是在第五位，执行 | 运算最多也就是影响低 5 位的结果。
注意：这里之所以要保证第 16 位为 1，是为了保证 sizeCtl 变量为负数，因为前面我们提到，这个变量为负数才代表当前有线程在扩容，至于这个变量和 sizeCtl 的关系后面会介绍。
### **首次扩容为什么计数要 +2 而不是 +1**
首次扩容一定不会走前面两个条件，而是走的最后一个红框内条件，这个条件通过 CAS 操作将 rs 左移了 16（RESIZE_STAMP_SHIFT）位，然后加上一个 2，这个代表什么意思呢？为什么是加 2 呢？
要回答这个问题我们先回答另一个问题，上面通过方法获得的扩容戳 rs 究竟有什么用？实际上这个扩容戳代表了两个含义：

- 高 16 位代表当前扩容的标记，可以理解为一个纪元。
- 低 16 位代表了扩容的线程数。

知道了这两个条件就好理解了，因为 rs 最终是要赋值给 sizeCtl 的，而 sizeCtl 负数才代表扩容，而将 rs 左移 16 位就刚好使得最高位为 1，此时低 16 位全部是 0，而因为低 16 位要记录扩容线程数，所以应该 +1，但是这里是 +2，原因是 sizeCtl 中 -1 这个数值已经被使用了，用来代替当前有线程准备扩容，所以如果直接 +1 是会和标志位发生冲突。
所以继续回到上图中的第二个红框，就是正常继续 +1 了，只有初始化第一次记录扩容线程数的时候才需要 +2。
### **扩容条件**
接下来我们继续看上图中第一个红框，这里面有 5 个条件，代表是满足这 5 个条件中的任意一个，则不进行扩容：

1. (sc >>> RESIZE_STAMP_SHIFT) != rs 这个条件实际上有 bug，在 JDK12 中已经换掉。
2. sc == rs + 1 表示最后一个扩容线程正在执行首位工作，也代表扩容即将结束。
3. sc == rs + MAX_RESIZERS 表示当前已经达到最大扩容线程数，所以不能继续让线程加入扩容。
4. 扩容完成之后会把 nextTable（扩容的新数组） 设为 null。
5. transferIndex <= 0 表示当前可供扩容的下标已经全部分配完毕，也代表了当前线程扩容结束。
### 多并发下如何实现扩容
在多并发下如何实现扩容才不会冲突呢？可能大家都想到了采用分而治之的思想，在 ConcurrentHashMap 中采用的是分段扩容法，即每个线程负责一段，默认最小是 16，也就是说如果 ConcurrentHashMap 中只有 16 个槽位，那么就只会有一个线程参与扩容。如果大于 16 则根据当前 CPU 数来进行分配，最大参与扩容线程数不会超过 CPU 数。
![](https://raw.githubusercontent.com/danmuking/image/main/8aa5c4157ef64efb2b705f3f751cd8e3.webp)
扩容空间和 HashMap 一样，每次扩容都是将原空间大小左移一位，即扩大为之前的两倍。注意这里的 transferIndex 代表的就是推进下标，默认为旧数组的大小。
### 扩容时的数据迁移如何保证安全性
初始化好了新的数组，接下来就是要准备确认边界。也就是要确认当前线程负责的槽位，确认好之后会从大到小开始往前推进，比如线程一负责 1-16，那么对应的数组边界就是 0-15，然后会从最后一位 15 开始迁移数据：
![](https://raw.githubusercontent.com/danmuking/image/main/7fc960a7c55c728b1f6b21e8a96e35a2.webp)
这里面有三个变量比较关键：

- fwd 节点，这个代表的是占位节点，最关键的就是这个节点的 hash 值为 -1，所以一旦发现某一个节点中的 hash 值为 -1 就可以知道当前节点已经被迁移了。
- advance：代表是否可以继续推进下一个槽位，只有当前槽位数据被迁移完成之后才可以设置为 true
- finishing：是否已经完成数据迁移。

知道了这几个变量，再看看上面的代码，第一次一定会进入 while 循环，因为默认 advance 为 true，第一次进入循环的目的为了确认边界，因为边界值还没有确认，所以会直接走到最后一个分支，通过 CAS 操作确认边界。
确认边界这里直接表述很难理解，我们通过一个例子来说明：
假设说最开始的空间为 16，那么扩容后的空间就是 32，此时 transferIndex 为旧数组大小 16，而在第二个 if判断中，transferIndex 赋值给了 nextIndex，所以 nextIndex 为 1，而 stride 代表的是每个线程负责的槽位数，最小就是 16，所以 stride 也是 16，所以 nextBound= nextIndex > stride ? nextIndex - stride : 0 皆可以得到：nextBound=0 和 i=15 了，也就是当前线程负责 0-15 的数组下标，且从 0 开始推进，确认边界后立刻将 advance 设置为 false，也就是会跳出 while 循环，从而执行下面的数据迁移部分逻辑。
PS：因为 nextBound=0，所以 CAS 操作实际上也是把 transferIndex 变成了 0，表示当前扩容的数组下标已经全部分配完毕，这也是前面不满足扩容的第 5 个条件。
数据迁移时，会使用 synchronized 关键字对当前节点进行加锁，也就是说锁的粒度精确到了每一个节点，可以说大大提升了效率。加锁之后的数据迁移和 HashMap 基本一致，也是通过区分高低位两种情况来完成迁移，在本文就不重复讲述。
当前节点完成数据迁移之后，advance 变量会被设置为 true，也就是说可以继续往前推进节点了，所以会重新进入上面的 while 循环的前面两个分支，把下标 i 往前推进之后再次把 advance 设置为 false，然后重复操作，直到下标推进到 0 完成数据迁移。
![](https://raw.githubusercontent.com/danmuking/image/main/0065ee5d116a7eca39da8cb343b57d56.webp)
while 循环彻底结束之后，会进入到下面这个 if 判断，红框中就是当前线程自己完成了迁移之后，会将扩容线程数进行递减，递减之后会再次通过一个条件判断，这个条件其实就是前面进入扩容前条件的反推，如果成立说明扩容已经完成，扩容完成之后会将 nextTable 设置为 null，所以上面不满足扩容的第 4 个条件就是在这里设置的。
## 参考资料
[助力面试之ConcurrentHashMap面试灵魂拷问，你能扛多久](https://zhuanlan.zhihu.com/p/355565143)
[JUC集合: ConcurrentHashMap详解](https://pdai.tech/md/java/thread/java-thread-x-juc-collection-ConcurrentHashMap.html#%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84-1)
