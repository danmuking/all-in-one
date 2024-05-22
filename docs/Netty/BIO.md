## [2.1 I/O 模型](https://dongzl.github.io/netty-handbook/#/_content/chapter02?id=_21-io-%e6%a8%a1%e5%9e%8b)
### [2.1.1 模型基本说明](https://dongzl.github.io/netty-handbook/#/_content/chapter02?id=_211-%e6%a8%a1%e5%9e%8b%e5%9f%ba%e6%9c%ac%e8%af%b4%e6%98%8e)

1. I/O 模型简单的理解：就是用什么样的通道进行数据的发送和接收，很大程度上决定了程序通信的性能。
2. Java 共支持 3 种网络编程模型 I/O 模式：BIO、NIO、AIO。
3. Java BIO：同步并阻塞（传统阻塞型），服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销。【简单示意图】

![image.png](https://raw.githubusercontent.com/danmuking/image/main/e795d38d591b1a6fd2cc269ea0e6740a.png)

1. Java NIO：同步非阻塞，服务器实现模式为一个线程处理多个请求（连接），即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有 I/O 请求就进行处理。【简单示意图】

![image.png](https://raw.githubusercontent.com/danmuking/image/main/3ae9c9e7b917d834b31dd6a8aa131ef5.png)

1. Java AIO(NIO.2)：异步非阻塞，AIO 引入异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用。
2. 我们依次展开讲解。
## [2.2 BIO、NIO、AIO 使用场景分析](https://dongzl.github.io/netty-handbook/#/_content/chapter02?id=_22-bio%e3%80%81nio%e3%80%81aio-%e4%bd%bf%e7%94%a8%e5%9c%ba%e6%99%af%e5%88%86%e6%9e%90)

1. BIO 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4 以前的唯一选择，但程序简单易理解。
2. NIO 方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，弹幕系统，服务器间通讯等。编程比较复杂，JDK1.4 开始支持。
3. AIO 方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用 OS 参与并发操作，编程比较复杂，JDK7 开始支持。
## [2.3 Java BIO 基本介绍](https://dongzl.github.io/netty-handbook/#/_content/chapter02?id=_23-java-bio-%e5%9f%ba%e6%9c%ac%e4%bb%8b%e7%bb%8d)

1. Java BIO 就是传统的 Java I/O 编程，其相关的类和接口在 java.io。
2. BIO(BlockingI/O)：同步阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，可以通过线程池机制改善（实现多个客户连接服务器）。【后有应用实例】
3. BIO 方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4 以前的唯一选择，程序简单易理解。
## [2.4 Java BIO 工作机制](https://dongzl.github.io/netty-handbook/#/_content/chapter02?id=_24-java-bio-%e5%b7%a5%e4%bd%9c%e6%9c%ba%e5%88%b6)
![image.png](https://raw.githubusercontent.com/danmuking/image/main/840e6d5354819bacc681853b1fb52fb0.png)
对 BIO 编程流程的梳理

1. 服务器端启动一个 ServerSocket。
2. 客户端启动 Socket 对服务器进行通信，默认情况下服务器端需要对每个客户建立一个线程与之通讯。
3. 客户端发出请求后，先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝。
4. 如果有响应，客户端线程会等待请求结束后，在继续执行。
## [2.5 Java BIO 应用实例](https://dongzl.github.io/netty-handbook/#/_content/chapter02?id=_25-java-bio-%e5%ba%94%e7%94%a8%e5%ae%9e%e4%be%8b)
实例说明：

1. 使用 BIO 模型编写一个服务器端，监听 6666 端口，当有客户端连接时，就启动一个线程与之通讯。
2. 要求使用线程池机制改善，可以连接多个客户端。
3. 服务器端可以接收客户端发送的数据（telnet 方式即可）。
4. 代码演示：
```java
package com.atguigu.bio;

import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BIOServer {

    public static void main(String[] args) throws Exception {
        //线程池机制
        //思路
        //1. 创建一个线程池
        //2. 如果有客户端连接，就创建一个线程，与之通讯(单独写一个方法)
        ExecutorService newCachedThreadPool = Executors.newCachedThreadPool();
        //创建ServerSocket
        ServerSocket serverSocket = new ServerSocket(6666);
        System.out.println("服务器启动了");
        while (true) {
            System.out.println("线程信息id = " + Thread.currentThread().getId() + "名字 = " + Thread.currentThread().getName());
            //监听，等待客户端连接
            System.out.println("等待连接....");
            final Socket socket = serverSocket.accept();
            System.out.println("连接到一个客户端");
            //就创建一个线程，与之通讯(单独写一个方法)
            newCachedThreadPool.execute(new Runnable() {
                public void run() {//我们重写
                    //可以和客户端通讯
                    handler(socket);
                }
            });
        }
    }

    //编写一个handler方法，和客户端通讯
    public static void handler(Socket socket) {
        try {
            System.out.println("线程信息id = " + Thread.currentThread().getId() + "名字 = " + Thread.currentThread().getName());
            byte[] bytes = new byte[1024];
            //通过socket获取输入流
            InputStream inputStream = socket.getInputStream();
            //循环的读取客户端发送的数据
            while (true) {
                System.out.println("线程信息id = " + Thread.currentThread().getId() + "名字 = " + Thread.currentThread().getName());
                System.out.println("read....");
                int read = inputStream.read(bytes);
                if (read != -1) {
                    System.out.println(new String(bytes, 0, read));//输出客户端发送的数据
                } else {
                    break;
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            System.out.println("关闭和client的连接");
            try {
                socket.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```
## [2.6 Java BIO 问题分析](https://dongzl.github.io/netty-handbook/#/_content/chapter02?id=_26-java-bio-%e9%97%ae%e9%a2%98%e5%88%86%e6%9e%90)

1. 每个请求都需要创建独立的线程，与对应的客户端进行数据 Read，业务处理，数据 Write。
2. 当并发数较大时，需要创建大量线程来处理连接，系统资源占用较大。
3. 连接建立后，如果当前线程暂时没有数据可读，则线程就阻塞在 Read 操作上，造成线程资源浪费。
