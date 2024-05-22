哈喽，大家哈，我是 **DanMu**。今天在刷博客的时候刷到一个挺有意思的面试题：“ Redis **是单线程的吗？**”在平时看的文章中都会说 Redis 是单线程的。但是 Redis 中同样存在着异步执行的操作，比如`bgsave` 命令，它它允许在后台异步地将当前数据持久化到磁盘，既然是异步操作，那么必然存在多个线程，为什么还说 Redis 是单线程的呢？
![NB-Panda (1).png](https://cdn.nlark.com/yuque/0/2024/png/29091075/1715610631202-30193877-23d6-42c2-a831-c1fbd820d2f4.png#averageHue=%23b7b7b7&clientId=u788567c0-8795-4&from=paste&height=191&id=u470b797a&originHeight=450&originWidth=450&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=105016&status=done&style=none&taskId=u48056c7f-599b-43e5-89db-1bad74a647d&title=&width=191)
其实 Redis 在4.0之后就已经部分引入了多线程机制，我们平常说的 Redis 是单线程的，主要指的是** Redis 对外提供键值对存储服务**的主要功能，也就是**网络请求**和**数据操作**是由一个线程完成的。而其他例如持久化存储模块、集群支撑模块等均可以由多线程完成。在 Redis 6.0 版本中，网络请求处理部分迎来了一次重大更新，引入了多线程模型。这意味着 Redis 在接收网络请求时，能够利用多个线程并行处理，从而大大提高了并发性能。
> 即使是在数据操作中，也并不是完全的单线程，例如 unlink 命令，作用就是将 key-value 进行异步删除。

##  Redis 为什么使用单线程
“单线程”这个词汇，在很多场景中常与“低效”相提并论。确实，Mysql以及大量中间件都积极拥抱多线程技术，那么为什么 Redis 特立独行的采用单线程模型，而不用多线程来提高效率呢？要回答这个问题其实只需要三个字：**没必要**！
![](https://cdn.nlark.com/yuque/0/2024/webp/29091075/1715612337695-d7dec7ce-a3d3-4400-9824-6c87bf471038.webp#averageHue=%23c5c5c5&clientId=u788567c0-8795-4&from=paste&id=u1a39b736&originHeight=198&originWidth=198&originalType=url&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&taskId=u721ab88b-9090-4c2d-b203-abb8148809f&title=)
哈哈哈哈哈哈，但是作为一个有节操的作者，还是要深入分析一下 Redis 使用单线程的具体原因。要说清楚这点，就需要先了解多线程的使用场景。
### 多线程的适用场景
一个计算机程序在执行的过程中，主要需要进行两种操作分别是读写操作和计算操作。其中读写操作主要是涉及到的就是 IO 操作，其中包括网络 IO 和磁盘 IO ，而计算操作主要涉及到 CPU 。而多线程的目的，就是通过并发的方式来**提升 IO 的利用率和 CPU 的利用率**。
对于 Redis 来说， CPU 资源根本就不是 Redis 的性能瓶颈，所以 Redis 不需要通过多线程技术来提升 CPU 利用率。而 Redis 的操作基本都是基于内存的，因此磁盘 IO 也很难成为瓶颈， Redis 面临的最大挑战其实是**网络 IO **，早期设计者认为通过** IO 多路复用技术**就足以应对大部分场景，但是随着使用者对于 Redis 的要求越来越高，在6.0版本中不得不引入多线程来应对日益增长的网络性能要求，这个我们后面再说。
### 多线程的弊端
任何技术都是一体两面，采用多线程可以帮助我们提升 CPU 和I/O的利用率，但是多线程带来的并发问题也会带来**框架设计上的复杂性**。而且，多线程模型中，多个线程的上下文切换也会带来一定的**性能开销**。因此，Redis 的作者认为引入多线程并不能为 Redis 带来性能上的提升，反而会带来设计上的复杂性，因此 Redis 最终选择了简单的单线程模型。
##  Redis 为什么使用单线程还那么快呢
 Redis  为什么使用单线程还那么快呢？这也是面试中经常出现的一个问题，要回答这个主要需要从四个部分入手：

1.  Redis 是**单线程**的，这主要指的是 Redis 的单线程模型避免了不同线程之间对于共享资源的竞争，同时减少了线程切换的开销。
2.  Redis 具有**高效的数据结构**， Redis 的数据结构都经过了精心的优化，在效率和空间上都具有极高的效率，当然这不是一两句话能够说清的，有机会我会另外写一篇文章来说说 Redis 的数据结构。
3.  Redis 是**基于内存**的，对于数据的操作都在内存中进行，因此所需要极少的 IO 时间
4.  Redis 采用** IO 多路复用**来加快 IO 利用率，通过一个线程就可以监听多个 IO 事件。
> **IO 多路复用**技术是一种利用单个线程监听多个IO接口的技术，通常由操作系统提供。

## 为什么 Redis 6.0又引入了多线程
> 需要注意的是， Redis 6.0中的引入的多线程，通常指针对处理**网络请求**过程引入的多线程。

这主要是由于 Redis 面对的场景越来越复杂。根据测算， Redis  将所有数据放在内存中，内存的响应时长大约为 100 纳秒，对于小数据包， Redis  服务器可以处理 80,000 到 100,000 QPS，这么高的对于 80% 的公司来说，单线程的 Redis 已经足够使用了。
但随着越来越复杂的业务场景，有些公司动不动就上亿的交易量，因此需要更大的 QPS。为了提升QPS，很多公司的做法是部署 Redis 集群，并且尽可能提升 Redis 机器数。但是这种做法的资源消耗是巨大的。
经过分析，限制 Redis 的性能的主要瓶颈出现在网络 IO 的处理上，虽然之前采用了**多路复用**技术。但是**多路复用的 IO 模型本质上仍然是同步阻塞型 IO 模型**。这里简单解释一下**同步阻塞型 IO 模型**
![Redis单线程-第 4 页.drawio.png](https://cdn.nlark.com/yuque/0/2024/png/29091075/1715763851219-5c4f6a2d-a8c4-4092-b84e-5286de1f97e5.png#averageHue=%23f8f8f8&clientId=u26e8f9c1-c4a2-4&from=paste&height=4921&id=u83cecaf1&originHeight=4921&originWidth=6307&originalType=binary&ratio=1&rotation=0&showTitle=false&size=3169514&status=done&style=none&taskId=u87efad07-24fe-4abe-a2ab-20db799fe2f&title=&width=6307)
从上图我们可以看到，在多路复用的 IO 模型中，在处理网络请求时，调用 select的过程是阻塞的，也就是说这个过程会阻塞线程，如果并发量很高，此处可能会成为瓶颈。
虽然现在很多服务器都是多个 CPU 核的，但是对于 Redis 来说，因为使用了单线程，在一次数据操作的过程中，有大量的 CPU 时间片是耗费在了网络 IO 的同步处理上的，并没有充分的发挥出多核的优势。
**如果能采用多线程，使得网络处理的请求并发进行，就可以大大的提升性能。多线程除了可以减少由于网络 I/O 等待造成的影响，还可以充分利用  CPU  的多核优势。**
##  Redis 的网络模型
既然提到了 Redis 的多线程，作为一个认真的作者，自然不能浅尝辄止。接下来，让我们来深入理解一下 Redis 的网络 IO 模型。
 Redis 的网络模型依然基于鼎鼎大名的Reactor模式，大量知名的开源中间件例如Netty、Nginx都使用了这一技术。其实相较于Reactor模式，我更喜欢叫他Dispatch模式，也就是分发器模式，这更接近于它的本质。根据分发器和处理线程的数量，通常又可以分为三种模式：

1. 单分发器单线程
2. 单分发器多线程
3. 多分发器多线程

在这里我们主要讨论与 Redis 相关的前两种。
### 单分发器单线程![Redis单线程-第 1 页.drawio.png](https://cdn.nlark.com/yuque/0/2024/png/29091075/1715763269212-4b26a321-20aa-48fe-a0e1-20a053bf67e0.png#averageHue=%23f3d8b4&clientId=u1aecb49f-2b37-4&from=paste&height=2493&id=BXZie&originHeight=2493&originWidth=5013&originalType=binary&ratio=1&rotation=0&showTitle=false&size=8175694&status=done&style=none&taskId=u1505505d-ed1d-4c84-9a6f-b680d4282c3&title=&width=5013)
Reactor 对象通过 select （ IO  多路复用接口） 监听事件，收到事件后通过 dispatch 进行分发，具体分发给 Acceptor 对象还是 Handler 对象，还要看收到的事件类型

- 如果是连接建立的事件，则交由 Acceptor 对象进行处理，Acceptor 对象会通过 accept 方法 获取连接，并创建一个 Handler 对象来处理后续的响应事件
- 如果不是连接建立事件， 则交由当前连接对应的 Handler 对象来进行响应

在 Redis 6.0之前，使用的便是这种模式，当然，这种模式的缺点同样十分明显，所有的操作都在同一个线程中完成，一旦出现一个高耗时的业务，后续的业务甚至是连接事件都需要等待。
### 单分发器多线程
为了克服单分发器单线程的缺点，就需要引入多进程，方案的示意图如下图所示
![Redis单线程-第 2 页.drawio.png](https://cdn.nlark.com/yuque/0/2024/png/29091075/1715763556450-8742547b-1b2e-412f-b414-5b2bd5f95b35.png#averageHue=%23fcfaf6&clientId=u26e8f9c1-c4a2-4&from=paste&height=2958&id=u61f24c5b&originHeight=2958&originWidth=3262&originalType=binary&ratio=1&rotation=0&showTitle=false&size=8748953&status=done&style=none&taskId=uc157f5b1-8358-4d62-90c2-477e2057a55&title=&width=3262)

- Reactor 对象通过 select （ IO  多路复用接口） 监听事件，收到事件后通过 dispatch 进行分发，具体分发给 Acceptor 对象还是 Handler 对象，还要看收到的事件类型；
- 如果是连接建立的事件，则交由 Acceptor 对象进行处理，Acceptor 对象会通过 accept 方法 获取连接，并创建一个 Handler 对象来处理后续的响应事件；
- 如果不是连接建立事件， 则交由当前连接对应的 Handler 对象来进行响应；

上面的三个步骤和单 Reactor 单线程方案是一样的，接下来的步骤就开始不一样了：

- Handler 对象不再负责业务处理，只负责数据的接收和发送，Handler 对象通过 read 读取到数据后，会将数据发给子线程里的 Processor 对象进行业务处理；
- 子线程里的 Processor 对象就进行业务处理，处理完后，将结果发给主线程中的 Handler 对象，接着由 Handler 通过 send 方法将响应结果发送给 client；

这个方案的有点在于能够充分利用多线程的能力，但是也存在问题，由于子线程只负责进行业务的处理，所有的**发送依然需要主线程负责**，这就这不仅会导致子线程对主线程的竞争，同时当并发量大的时候，主线程同样会成为瓶颈，因此 Redis 并没有直接使用这个方案，而是进行了一定的修改。
###  Redis 的多线程方案
![Redis单线程-第 3 页.drawio.png](https://cdn.nlark.com/yuque/0/2024/png/29091075/1715763761734-461b635f-c323-4c92-b116-04aa9199fae6.png#averageHue=%23faf8f4&clientId=u26e8f9c1-c4a2-4&from=paste&height=4956&id=uc5833aca&originHeight=4956&originWidth=5467&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10242095&status=done&style=none&taskId=u179a92be-ee77-4237-8533-e5de630fe99&title=&width=5467)
 Redis 对单分发器多线程的方案进行了一定的改进，从上图中我们可以看到，主要是有两点不同：

1. 将数据的接收和发送都纳入到了子线程当中，这样可以避免在高并发时，主线程的处理能力成为瓶颈
2. 业务依然在主线程中执行，这是因为 Redis 原始的设计便是一个单线程模型，这样可以尽量保证整体结构的一致性，减少修改成本。  
## 点关注，不迷路
> 好了，以上就是这篇文章的全部内容了，如果你能看到这里，**非常感谢你的支持！**
> 如果你觉得这篇文章写的还不错， 求**点赞**👍 求**关注**❤️ 求**分享**👥 对暖男我来说真的 **非常有用！！！**
> 白嫖不好，创作不易，各位的支持和认可，就是我创作的最大动力，我们下篇文章见！
> 如果本篇博客有任何错误，请批评指教，不胜感激 ！

> 最后推荐我的**IM项目DiTing**（[https://github.com/danmuking/DiTing-Go](https://github.com/danmuking/DiTing-Go)），致力于成为一个初学者友好、易于上手的 IM 解决方案，希望能给你的学习、面试带来一点帮助，如果人才你喜欢，给个Star⭐叭！


## 参考资料
[https://segmentfault.com/a/1190000041275783](https://segmentfault.com/a/1190000041275783)
[https://zhuanlan.zhihu.com/p/452981967](https://zhuanlan.zhihu.com/p/452981967)
