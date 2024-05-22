## ThreadLocal简介
 ThreadLocal是一个**在多线程中为每一个线程创建单独的变量副本的类**; 当使用ThreadLocal来维护变量时, ThreadLocal会为每个线程创建单独的变量副本, 避免因多线程操作共享变量而导致的数据不一致的情况。
## ThreadLocal原理
### ThreadLocal的内部结构
#### jdk8前：
每个ThreadLocal都创建一个Map，然后用线程作为Map的key，要存储的局部变量作为Map的value，这样就能达到各个线程的局部变量隔离的效果。这是最简单的设计方法，JDK最早期的ThreadLocal 确实是这样设计的，但现在早已不是了。![20210124115504330.png](https://raw.githubusercontent.com/danmuking/image/main/e9bd0f1eed997751b6b4bd0419db3fd9.png)
#### jdk8后：
每个Thread维护一个ThreadLocalMap，这个Map的key是ThreadLocal实例本身，value才是真正要存储的值Object。
![20210124120125266.png](https://raw.githubusercontent.com/danmuking/image/main/e7a1a8e068b0511aa8f0b0d10bbf3e76.png)
优点：

1. 这样设计之后每个Map存储的Entry数量就会变少。因为之前的存储数量由Thread的数量决定，现在是由ThreadLocal的数量决定。在实际运用当中，往往ThreadLocal的数量要少于Thread的数量。
2. 当Thread销毁之后，对应的ThreadLocalMap也会随之销毁，能减少内存的使用。
## ThreadLocal造成内存泄露的问题
内存泄漏跟Entry中使用了弱引用的key有关系？没有 
**弱引用相关概念**
Java中的引用有4种类型： 强、软、弱、虚。当前这个问题主要涉及到强引用和弱引用：

- 强引用（“Strong” Reference），就是我们最常见的普通对象引用，只要还有强引用指向一个对象，就能表明对象还“活着”，垃圾回收器就不会回收这种对象。
- 弱引用（WeakReference），垃圾回收器一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。

**如果key使用强引用**
此时ThreadLocal的内存图（实线表示强引用）如下：
![20210124144330949.png](https://raw.githubusercontent.com/danmuking/image/main/081644df331d4e261767d19f8839c9c8.png)
假设在业务代码中使用完ThreadLocal ，ThreadLocal Ref被回收了。
但是因为threadLocalMap的Entry强引用了threadLocal，造成threadLocal无法被回收。
在没有手动删除这个Entry以及CurrentThread依然运行的前提下，始终有强引用链 threadRef->currentThread->threadLocalMap->entry，Entry就不会被回收（Entry中包括了ThreadLocal实例和value），导致Entry内存泄漏。
也就是说，ThreadLocalMap中的key使用了强引用， 是无法完全避免内存泄漏的。
**如果key使用弱引用**
此时ThreadLocal的内存图（实线表示强引用，虚线表示弱引用）如下：![20210124144945153.png](https://raw.githubusercontent.com/danmuking/image/main/3850d07be5f4b5fad501164f5ccb9141.png)
同样假设在业务代码中使用完ThreadLocal ，threadLocal Ref被回收了。
 由于ThreadLocalMap只持有ThreadLocal的弱引用，没有任何强引用指向threadlocal实例, 所以threadlocal就可以顺利被gc回收，此时Entry中的key=null。
 但是在没有手动删除这个Entry以及CurrentThread依然运行的前提下，也存在有强引用链 threadRef->currentThread->threadLocalMap->entry -> value ，value不会被回收， 而这块value永远不会被访问到了，导致value内存泄漏。
 也就是说，ThreadLocalMap中的key使用了弱引用， 也有可能内存泄漏。
#### 出现内存泄漏的真实原因
**ThreadLocal内存泄漏的根源是**：由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key就会导致内存泄漏。**尤其是在使用线程池的时候线程将不会被销毁。**
#### 为什么使用弱引用
避免内存泄漏有两种方式：

1. 使用完ThreadLocal，调用其remove方法删除对应的Entry
2. 使用完ThreadLocal，当前Thread也随之运行结束

在ThreadLocalMap中的set/getEntry方法中，会对key为null（也即是ThreadLocal为null）进行判断，如果为null的话，那么是会对value置为null的。
这就意味着使用完ThreadLocal，CurrentThread依然运行的前提下，就算忘记调用remove方法，弱引用比强引用可以**多一层保障**：弱引用的ThreadLocal会被回收，对应的value在下一次ThreadLocalMap调用set,get,remove中的任一方法的时候会被清除，从而避免内存泄漏。
## hash冲突的解决
ThreadLocalMap使用线性探测法来解决哈希冲突的。
该方法一次探测下一个地址，直到有空的地址后插入，若整个空间都找不到空余的地址，则产生溢出。

# 参考资料：
[由浅入深，全面解析ThreadLocal_LeslieGuGu的博客-CSDN博客](https://blog.csdn.net/weixin_44050144/article/details/113061884)[12-ThreadLocalMap中hash冲突的解决_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1N741127FH?p=12&spm_id_from=pageDriver&vd_source=be6ff5cf4681fcfe41b6e1ff363e2a71)
[Java 并发 - ThreadLocal详解](https://pdai.tech/md/java/thread/java-thread-x-threadlocal.html)
[Java面试必问：ThreadLocal终极篇 淦！ - 敖丙 - 博客园](https://www.cnblogs.com/aobing/p/13382184.html)
