## 5.8 Netty 模型
### 5.8.1 工作原理示意图1 - 简单版
Netty 主要基于主从 Reactors 多线程模型（如图）做了一定的改进，其中主从 Reactor 多线程模型有多个 Reactor
![image.png](https://raw.githubusercontent.com/danmuking/image/main/fe6f39823de22840ec3a1c2c6c957b79.png)
### 5.8.2 对上图说明

1. BossGroup 线程维护 Selector，只关注 Accecpt
2. 当接收到 Accept 事件，获取到对应的 SocketChannel，封装成 NIOScoketChannel 并注册到 Worker 线程（事件循环），并进行维护
3. 当 Worker 线程监听到 Selector 中通道发生自己感兴趣的事件后，就进行处理（就由 handler），注意 handler 已经加入到通道
### 5.8.3 工作原理示意图2 - 进阶版
![image.png](https://raw.githubusercontent.com/danmuking/image/main/394bbf976fc174217359326a2586560f.png)
### [5.8.4 工作原理示意图 - 详细版](https://dongzl.github.io/netty-handbook/#/_content/chapter05?id=_584-%e5%b7%a5%e4%bd%9c%e5%8e%9f%e7%90%86%e7%a4%ba%e6%84%8f%e5%9b%be-%e8%af%a6%e7%bb%86%e7%89%88)
![image.png](https://raw.githubusercontent.com/danmuking/image/main/ab32aa1cf703f2034f8c09cfb0876e88.png)
### 5.8.5 对上图的说明小结

1. Netty 抽象出两组线程池 BossGroup 专门负责接收客户端的连接，WorkerGroup 专门负责网络的读写
2. BossGroup 和 WorkerGroup 类型都是 NioEventLoopGroup
3. NioEventLoopGroup 相当于一个事件循环组，这个组中含有多个事件循环，每一个事件循环是 NioEventLoop
4. NioEventLoop 表示一个不断循环的执行处理任务的线程，每个 NioEventLoop 都有一个 Selector，用于监听绑定在其上的 socket 的网络通讯
5. NioEventLoopGroup 可以有多个线程，即可以含有多个 NioEventLoop
6. 每个 BossNioEventLoop 循环执行的步骤有 3 步
   - 轮询 accept 事件
   - 处理 accept 事件，与 client 建立连接，生成 NioScocketChannel，并将其注册到某个 worker NIOEventLoop 上的 Selector
   - 处理任务队列的任务，即 runAllTasks
7. 每个 Worker NIOEventLoop 循环执行的步骤
   - 轮询 read，write 事件
   - 处理 I/O 事件，即 read，write 事件，在对应 NioScocketChannel 处理
   - 处理任务队列的任务，即 runAllTasks
8. 每个 Worker NIOEventLoop 处理业务时，会使用 pipeline（管道），pipeline 中包含了 channel，即通过 pipeline 可以获取到对应通道，管道中维护了很多的处理器
