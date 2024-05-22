## 线程安全分析
HashMap 是线程不安全的。
JDK 1.7 HashMap 采用数组 + 链表的数据结构，多线程背景下，在数组扩容的时候，存在 Entry 链死循环和数据丢失问题。
JDK 1.8 HashMap 采用数组 + 链表 + 红黑二叉树的数据结构，优化了 1.7 中数组扩容的方案，解决了 Entry 链死循环和数据丢失问题。但是多线程背景下，put 方法存在数据覆盖的问题。
### 1.7 中扩容引发的线程不安全分析
```java
void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```
这段代码是 HashMap 的扩容操作，重新定位每个桶的下标，并采用头插法将元素迁移到新数组中。头插法会将链表的顺序翻转，这也是形成死循环的关键点。
假设现在有两个线程A、B同时对下面这个 HashMap 进行扩容操作：
![](https://raw.githubusercontent.com/danmuking/image/main/3a08a1f88f9007ec43e8e815dd9ef4e7.png)
正常扩容后的结果是下面这样的：
![](https://raw.githubusercontent.com/danmuking/image/main/734ae42dfcd41da2c44936869e0b6762.png)
但是当线程A执行到上面 transfer 函数的第11行代码时，CPU 时间片耗尽，线程A被挂起。即如下图中位置所示：
![](https://raw.githubusercontent.com/danmuking/image/main/b659661a80b2c904939efb642fddd222.png)
此时线程A中：e=3、next=7、e.next=null
![](https://raw.githubusercontent.com/danmuking/image/main/b860b99cbf9a973d80082427e2207bd9.png)
当线程A的时间片耗尽后，CPU 开始执行线程B，并在线程B中成功的完成了数据迁移：
![](https://raw.githubusercontent.com/danmuking/image/main/eec813ab65cb00b4d951ea7e1bada475.png)
重点来了，根据 Java 内存模式可知，线程B执行完数据迁移后，此时主内存中 newTable 和 table 都是最新的，也就是说：7.next=3、3.next=null
随后线程A获得CPU时间片继续执行 newTable[i] = e，将3放入新数组对应的位置，执行完此轮循环后线程A的情况如下：
![](https://raw.githubusercontent.com/danmuking/image/main/80f899c93c12e51642e5476e537248c9.png)
接着继续执行下一轮循环，此时 e=7，从主内存中读取 e.next 时发现主内存中 7.next=3，于是乎next=3，并将 7 采用头插法的方式放入新数组中，并继续执行完此轮循环，结果如下：
![](https://raw.githubusercontent.com/danmuking/image/main/95349877e246d7edee095111347cc365.png)
执行下一次循环可以发现，next=e.next=null，所以此轮循环将会是最后一轮循环。接下来当执行完e.next=newTable[i]即3.next=7后，3和7之间就相互连接了，当执行完newTable[i]=e后，3被头插法重新插入到链表中，执行结果如下图所示：
![](https://raw.githubusercontent.com/danmuking/image/main/dbb5862c805e40136b8e4e5833a4b5fc.png)
上面说了此时 e.next=null 即 next=null，当执行完 e=null 后，将不会进行下一轮循环。到此线程A、B的扩容操作完成，很明显当线程A执行完后，HashMap 中出现了环形结构，当在以后对该 HashMap 进行操作时会出现死循环。
并且从上图可以发现，元素5在扩容期间被莫名的丢失了，这就发生了数据丢失的问题。
### 1.8 中 put 方法数据覆盖问题分析
根据上面JDK1.7出现的问题，在JDK1.8中已经得到了很好的解决，如果你去阅读1.8的源码会发现找不到 transfer 函数，因为 JDK1.8 直接在 resize 函数中完成了数据迁移。另外说一句，JDK1.8在进行元素插入时使用的是尾插法。
为什么说JDK1.8会出现数据覆盖的情况喃，我们来看一下下面这段JDK1.8中的put操作代码：
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null) // 如果没有hash碰撞则直接插入元素
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
其中第六行代码是判断是否出现 hash 碰撞，假设两个线程A、B都在进行 put 操作，并且 hash 函数计算出的插入下标是相同的，当线程A 执行完第六行代码后由于时间片耗尽导致被挂起，而线程B得到时间片后在该下标处插入了元素，完成了正常的插入，然后线程A获得时间片，由于之前已经进行了 hash 碰撞的判断，所有此时不会再进行判断，而是直接进行插入，这就导致了线程B插入的数据被线程A覆盖了，从而线程不安全。
除此之前，还有就是代码的第38行处有个 ++size，我们这样想，还是线程A、B，这两个线程同时进行 put 操作时，假设当前 HashMap 的zise大小为10，当线程A执行到第38行代码时，从主内存中获得size的值为10后准备进行+1操作，但是由于时间片耗尽只好让出CPU，线程B快乐的拿到CPU还是从主内存中拿到size的值10进行+1操作，完成了put操作并将size=11写回主内存，然后线程A再次拿到CPU并继续执行(此时size的值仍为10)，当执行完put操作后，还是将size=11写回内存，此时，线程A、B都执行了一次put操作，但是size的值只增加了1，所有说还是由于数据覆盖又导致了线程不安全。

## 参考资料
[https://www.cnblogs.com/jmcui/p/11422083.html](https://www.cnblogs.com/jmcui/p/11422083.html)
