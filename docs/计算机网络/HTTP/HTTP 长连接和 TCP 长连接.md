大家好，我是小林。
之前有位读者私信我，他在字节面试时，被问到这两个问题：
![](https://raw.githubusercontent.com/danmuking/image/main/f694c7248e916ae6add17d864df14894.jpeg)
第一个问题：MySQL 的 NULL 值是怎么存放的？
第二个问题：HTTP 长连接和 TCP 长连接有什么区别？
第一个问题，主要是考核你是否清楚 MySQL 一条记录是怎么存储的，我在前几天已经写了一篇文章讲解了，还没看过的同学，可以去看这篇：字节一面:MySQL 的 NULL 值是怎么存放的?
第二问题，其实是在问 HTTP 的 Keep-Alive 和 TCP 的 Keepalive 有什么区别？
这是个好问题，应该有不少人都会搞混，因为这两个东西看上去太像了，很容易误以为是同一个东西。
如果认真读过我网站上图解网络系列文章的同学，应该这个问题你们都会，因为我之前就写过。
不过，应该也有不少同学，看过后忘记了，这次就带大家重新复习一波。
事实上，这两个完全是两样不同东西，实现的层面也不同：

- HTTP 的 Keep-Alive，是由应用层（用户态）实现的，称为 HTTP 长连接；
- TCP 的 Keepalive，是由TCP 层（内核态）实现的，称为 TCP 保活机制；

接下来，分别说说它们。
#### HTTP 的 Keep-Alive
HTTP 协议采用的是「请求-应答」的模式，也就是客户端发起了请求，服务端才会返回响应，一来一回这样子。
![](https://raw.githubusercontent.com/danmuking/image/main/665251cd3a6ebb898599267511e98f13.png)
请求-应答
由于 HTTP 是基于 TCP 传输协议实现的，客户端与服务端要进行 HTTP 通信前，需要先建立 TCP 连接，然后客户端发送 HTTP  请求，服务端收到后就返回响应，至此「请求-应答」的模式就完成了，随后就会释放 TCP 连接。
![](https://raw.githubusercontent.com/danmuking/image/main/6f454b233a09c1db40266fd311733b2f.png)
一个 HTTP 请求
如果每次请求都要经历这样的过程：建立 TCP -> 请求资源 -> 响应资源 -> 释放连接，那么此方式就是 HTTP 短连接，如下图：
![](https://raw.githubusercontent.com/danmuking/image/main/febcdc96b2db8d69c244e0fb5bbd62df.png)
HTTP 短连接
这样实在太累人了，一次连接只能请求一次资源。
能不能在第一个 HTTP 请求完后，先不断开 TCP 连接，让后续的 HTTP 请求继续使用此连接？
当然可以，HTTP 的 Keep-Alive 就是实现了这个功能，可以使用同一个 TCP 连接来发送和接收多个 HTTP 请求/应答，避免了连接建立和释放的开销，这个方法称为 HTTP 长连接。
![](https://raw.githubusercontent.com/danmuking/image/main/9981751947e5f6f936f67dcbd7480332.png)
HTTP 长连接
HTTP 长连接的特点是，只要任意一端没有明确提出断开连接，则保持 TCP 连接状态。
怎么才能使用 HTTP 的 Keep-Alive 功能？
在 HTTP 1.0 中默认是关闭的，如果浏览器要开启 Keep-Alive，它必须在请求的包头中添加：
复制
```
Connection: Keep-Alive

1.
```
然后当服务器收到请求，作出回应的时候，它也添加一个头在响应中：
复制
```
Connection: Keep-Alive

1.
```
这样做，连接就不会中断，而是保持连接。当客户端发送另一个请求时，它会使用同一个连接。这一直继续到客户端或服务器端提出断开连接。
从 HTTP 1.1 开始， 就默认是开启了 Keep-Alive，如果要关闭 Keep-Alive，需要在 HTTP 请求的包头里添加：
复制
```
Connection:close

1.
```
现在大多数浏览器都默认是使用 HTTP/1.1，所以 Keep-Alive 都是默认打开的。一旦客户端和服务端达成协议，那么长连接就建立好了。
HTTP 长连接不仅仅减少了 TCP 连接资源的开销，而且这给 HTTP 流水线技术提供了可实现的基础。
所谓的 HTTP 流水线，是客户端可以先一次性发送多个请求，而在发送过程中不需先等待服务器的回应，可以减少整体的响应时间。
举例来说，客户端需要请求两个资源。以前的做法是，在同一个 TCP 连接里面，先发送 A 请求，然后等待服务器做出回应，收到后再发出 B 请求。HTTP 流水线机制则允许客户端同时发出 A 请求和 B 请求。
![](https://raw.githubusercontent.com/danmuking/image/main/cfdce552e5aa603e049d36f2d9a832a6.png)
右边为 HTTP 流水线机制
但是服务器还是按照顺序响应，先回应 A 请求，完成后再回应 B 请求。
而且要等服务器响应完客户端第一批发送的请求后，客户端才能发出下一批的请求，也就说如果服务器响应的过程发生了阻塞，那么客户端就无法发出下一批的请求，此时就造成了「队头阻塞」的问题。
可能有的同学会问，如果使用了 HTTP 长连接，如果客户端完成一个 HTTP 请求后，就不再发起新的请求，此时这个 TCP 连接一直占用着不是挺浪费资源的吗？
对没错，所以为了避免资源浪费的情况，web 服务软件一般都会提供 keepalive_timeout 参数，用来指定 HTTP 长连接的超时时间。
比如设置了 HTTP 长连接的超时时间是 60 秒，web 服务软件就会启动一个定时器，如果客户端在完后一个 HTTP 请求后，在 60 秒内都没有再发起新的请求，定时器的时间一到，就会触发回调函数来释放该连接。
![](https://raw.githubusercontent.com/danmuking/image/main/cc8554df5263060ce0b240b999fb85a2.png)
HTTP 长连接超时
#### TCP 的 Keepalive
TCP 的 Keepalive 这东西其实就是 TCP 的保活机制，它的工作原理我之前的文章写过，这里就直接贴下以前的内容。
如果两端的 TCP 连接一直没有数据交互，达到了触发 TCP 保活机制的条件，那么内核里的 TCP 协议栈就会发送探测报文。

- 如果对端程序是正常工作的。当 TCP 保活的探测报文发送给对端, 对端会正常响应，这样TCP 保活时间会被重置，等待下一个 TCP 保活时间的到来。
- 如果对端主机崩溃，或对端由于其他原因导致报文不可达。当 TCP 保活的探测报文发送给对端后，石沉大海，没有响应，连续几次，达到保活探测次数后，TCP 会报告该 TCP 连接已经死亡。

所以，TCP 保活机制可以在双方没有数据交互的情况，通过探测报文，来确定对方的 TCP 连接是否存活，这个工作是在内核完成的。
![](https://raw.githubusercontent.com/danmuking/image/main/0ed0295f732477b6d6897709543f1589.png)
TCP 保活机制
注意，应用程序若想使用 TCP 保活机制需要通过 socket 接口设置 SO_KEEPALIVE 选项才能够生效，如果没有设置，那么就无法使用 TCP 保活机制。
#### 总结
HTTP 的 Keep-Alive 也叫 HTTP 长连接，该功能是由「应用程序」实现的，可以使得用同一个 TCP 连接来发送和接收多个 HTTP 请求/应答，减少了 HTTP 短连接带来的多次 TCP 连接建立和释放的开销。
TCP 的 Keepalive 也叫 TCP 保活机制，该功能是由「内核」实现的，当客户端和服务端长达一定时间没有进行数据交互时，内核为了确保该连接是否还有效，就会发送探测报文，来检测对方是否还在线，然后来决定是否要关闭该连接。
