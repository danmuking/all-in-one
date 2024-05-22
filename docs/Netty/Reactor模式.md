## 传统I/O服务模型
![](https://raw.githubusercontent.com/danmuking/image/main/1e4524bc08c8cc6d2be3b20ee6ef8832.webp)
传统阻塞I/O服务模型
**模型特点：**
1、采用阻塞I/O模式获取输入数据
2、每个连接都需要独立的线程完成数据的输入，业务处理，数据返回
**问题分析：**
1、当并发数很大，就会创建大量线程，占用大量的系统资源
2、连接创建后，如果当前线程暂时没有数据可读，该线程会阻塞在read操作，造成线程资源浪费
## Reactor模式
![](https://raw.githubusercontent.com/danmuking/image/main/2bf4f105ff46f993499f8b3b56fbac2e.webp)
reactor模式工作原理图
## 说明
Reactor模式称为反应器模式或应答者模式，是基于事件驱动的设计模式，拥有一个或多个并发输入源，有一个服务处理器和多个请求处理器，服务处理器会同步的将输入的请求事件以多路复用的方式分发给相应的请求处理器。

Reactor设计模式是一种为处理并发服务请求，并将请求提交到一个或多个服务处理程序的事件设计模式。当客户端请求抵达后，服务处理程序使用多路分配策略，由一个非阻塞的线程来接收所有请求，然后将请求派发到相关的工作线程并进行处理的过程。

在事件驱动的应用中，将一个或多个客户端的请求分离和调度给应用程序，同步有序地接收并处理多个服务请求。对于高并发系统经常会使用到Reactor模式，用来替代常用的多线程处理方式以节省系统资源并提高系统的吞吐量。
## 
单Reactor单线程
![](https://raw.githubusercontent.com/danmuking/image/main/d1de296e7504911362536792c88514a7.webp)
单Reactor单线程
**
优缺点分析**
**优点：**模型简单，没有多线程、进程通信和竞争的问题，全部都在一个线程中完成。
**缺点：**
1、性能问题，只有一个线程，无法发挥多核CPU的性能，Handler在处理某个连接业务时，整个进程无法处理其他连接事件，很容易导致性能瓶颈。
2、可靠性问题，线程意外终止或进入死循环，会导致整个系统通信模块不可用，不能接收和处理外部消息，造成节点故障。
## 单Reactor多线程
![](https://raw.githubusercontent.com/danmuking/image/main/fe9a67a1f78bb505f9ed6982bee62918.webp)
单Reactor多线程
**工作流程**
1、Reactor对象通过select监听客户端请求事件，收到事件后，通过dispatch进行分发。
2、如果建立连接请求，则Acceptor通过accept处理连接请求，然后创建一个Handler对象处理完成连接后的各种事件。
3、如果不是连接请求，则由reactor分发调用连接对应的handler来处理。
4、handler只负责相应事件，不做具体的业务处理，通过read读取数据后，会分发给后面的worker线程池的某个线程处理业务。
5、worker线程池会分配独立线程完成真正的业务，并将结果返回给handler。
6、handler收到响应后，通过send分发将结果返回给client。
**优点：可以充分利用多核cpu的处理能力**
**缺点：多线程数据共享和访问比较复杂，rector处理所有的事件的监听和响应，在单线程运行，在高并发应用场景下，容易出现性能瓶颈。**
## **主从Reactor多线程**
![](https://raw.githubusercontent.com/danmuking/image/main/67402a64d31f714ddb8bed8e00a05ec7.webp)
主从Reactor多线程工作原理图
**工作流程**
1、Reactor主线程MainReactor对象通过select监听连接事件，收到事件后，通过Acceptor处理连接事件。
2、当Acceptor处理连接事件后，MainReactor将连接分配给SubAcceptor。
3、SubAcceptor将连接加入到连接队列进行监听，并创建handler进行各种事件处理。
4、当有新事件发生时，SubAcceptor就会调用对应的handler进行各种事件处理。
5、handler通过read读取数据，分发给后面的work线程处理。
6、work线程池分配独立的work线程进行业务处理，并返回结果。
7、handler收到响应的结果后，再通过send返回给client。
**注意：Reactor主线程可以对应多个Reactor子线程，即SubAcceptor。**
## **Reactor模式总结**
**3种模式用生活案例来理解**
![](https://raw.githubusercontent.com/danmuking/image/main/16713e1ce47d301a1f34879a5571cd7b.webp)
1、单reactor单线程，前台接待员、服务员时同一个人，全程为顾客服务。
2、单reactor多线程，1个前台接待，多个服务员，接待员只负责接待。
3、主从reactor多线程，多个前台接待，多个服务员。
**Reactor模式的优点**
1、响应块，不必为单个同步时间所阻塞，虽然Reactor本身依然时同步的。
2、可以最大程度的避免复杂的多线程及同步问题，并且避免多线程/进程的切换开销。
3、扩展性好，可以方便的通过增加Reactor实例个数来充分利用CPU资源。
4、复用性好，Reactor模式本身与具体事件处理逻辑无关，具有很高的复用性。

## 参考资料
[Reactor模式](https://zhuanlan.zhihu.com/p/347779760)
