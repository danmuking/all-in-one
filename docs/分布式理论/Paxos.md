Paxos算法在分布式领域具有非常重要的地位。但是Paxos算法有两个比较明显的缺点：1.难以理解 2.工程实现更难。
网上有很多讲解Paxos算法的文章，但是质量参差不齐。看了很多关于Paxos的资料后发现，学习Paxos最好的资料是论文《Paxos Made Simple》，其次是中、英文版维基百科对Paxos的介绍。本文试图带大家一步步揭开Paxos神秘的面纱。
## Paxos是什么
Paxos算法是基于**消息传递**且具有**高度容错特性**的**一致性算法**，是目前公认的解决**分布式一致性**问题**最有效**的算法之一。
Google Chubby的作者Mike Burrows说过这个世界上**只有一种**一致性算法，那就是Paxos，其它的算法都是**残次品**。
虽然Mike Burrows说得有点夸张，但是至少说明了Paxos算法的地位。然而，Paxos算法也因为晦涩难懂而臭名昭著。本文的目的就是带领大家深入浅出理解Paxos算法，不仅理解它的执行流程，还要理解算法的推导过程，作者是怎么一步步想到最终的方案的。只有理解了推导过程，才能深刻掌握该算法的精髓。而且理解推导过程对于我们的思维也是非常有帮助的，可能会给我们带来一些解决问题的思路，对我们有所启发。
## 问题产生的背景
在常见的分布式系统中，总会发生诸如**机器宕机**或**网络异常**（包括消息的延迟、丢失、重复、乱序，还有网络分区）等情况。Paxos算法需要解决的问题就是如何在一个可能发生上述异常的分布式系统中，快速且正确地在集群内部对**某个数据的值**达成**一致**，并且保证不论发生以上任何异常，都不会破坏整个系统的一致性。
注：这里**某个数据的值**并不只是狭义上的某个数，它可以是一条日志，也可以是一条命令（command）。。。根据应用场景不同，**某个数据的值**有不同的含义。
![](https://raw.githubusercontent.com/danmuking/image/main/74871dbb70ba2da7bc39b3cc45bc262a.webp)
## 相关概念
在Paxos算法中，有三种角色：

- **Proposer**
- **Acceptor**
- **Learners**

在具体的实现中，一个进程可能**同时充当多种角色**。比如一个进程可能**既是Proposer又是Acceptor又是Learner**。
还有一个很重要的概念叫**提案（Proposal）**。最终要达成一致的value就在提案里。
**注：**

- **暂且**认为『**提案=value**』，即提案只包含value。在我们接下来的推导过程中会发现如果提案只包含value，会有问题，于是我们再对提案**重新设计**。
- **暂且**认为『**Proposer可以直接提出提案**』。在我们接下来的推导过程中会发现如果Proposer直接提出提案会有问题，需要增加一个学习提案的过程。

Proposer可以提出（propose）提案；Acceptor可以接受（accept）提案；如果某个提案被选定（chosen），那么该提案里的value就被选定了。
回到刚刚说的『对某个数据的值达成一致』，指的是Proposer、Acceptor、Learner都认为同一个value被选定（chosen）。那么，Proposer、Acceptor、Learner分别在什么情况下才能认为某个value被选定呢？

- Proposer：只要Proposer发的提案被Acceptor接受（刚开始先认为只需要一个Acceptor接受即可，在推导过程中会发现需要半数以上的Acceptor同意才行），Proposer就认为该提案里的value被选定了。
- Acceptor：只要Acceptor接受了某个提案，Acceptor就认为该提案里的value被选定了。
- Learner：Acceptor告诉Learner哪个value被选定，Learner就认为那个value被选定。

![](https://raw.githubusercontent.com/danmuking/image/main/2e1c6748e9c5268429cba27ce5dda89e.webp)
## 问题描述
假设有一组可以**提出（propose）value**（value在提案Proposal里）的**进程集合**。一个一致性算法需要保证提出的这么多value中，**只有一个**value被选定（chosen）。如果没有value被提出，就不应该有value被选定。如果一个value被选定，那么所有进程都应该能**学习（learn）**到这个被选定的value。对于一致性算法，**安全性（safaty）**要求如下：

- 只有被提出的value才能被选定。
- 只有一个value被选定，并且
- 如果某个进程认为某个value被选定了，那么这个value必须是真的被选定的那个。

我们不去精确地定义其**活性（liveness）**要求。我们的目标是保证**最终有一个提出的value被选定**。当一个value被选定后，进程最终也能学习到这个value。
Paxos的目标：保证最终有一个value会被选定，当value被选定后，进程最终也能获取到被选定的value。
假设不同角色之间可以通过发送消息来进行通信，那么：

- 每个角色以任意的速度执行，可能因出错而停止，也可能会重启。一个value被选定后，所有的角色可能失败然后重启，除非那些失败后重启的角色能记录某些信息，否则等他们重启后无法确定被选定的值。
- 消息在传递过程中可能出现任意时长的延迟，可能会重复，也可能丢失。但是消息不会被损坏，即消息内容不会被篡改（拜占庭将军问题）。
## 推导过程
### 最简单的方案——只有一个Acceptor
假设只有一个Acceptor（可以有多个Proposer），只要Acceptor接受它收到的第一个提案，则该提案被选定，该提案里的value就是被选定的value。这样就保证只有一个value会被选定。
但是，如果这个唯一的Acceptor宕机了，那么整个系统就**无法工作**了！
因此，必须要有**多个Acceptor**！
![](https://raw.githubusercontent.com/danmuking/image/main/31471009a8a272198b14f97c593fb436.webp)
### 多个Acceptor
多个Acceptor的情况如下图。那么，如何保证在多个Proposer和多个Acceptor的情况下选定一个value呢？
![](https://raw.githubusercontent.com/danmuking/image/main/9795f7a1c028701e13420a1ccfee8292.webp)
下面开始寻找解决方案。
如果我们希望即使只有一个Proposer提出了一个value，该value也最终被选定。
那么，就得到下面的约束：
P1：一个Acceptor必须接受它收到的第一个提案。
但是，这又会引出另一个问题：如果每个Proposer分别提出不同的value，发给不同的Acceptor。根据P1，Acceptor分别接受自己收到的value，就导致不同的value被选定。出现了不一致。如下图：
![](https://raw.githubusercontent.com/danmuking/image/main/26bb1dba9a3ea18553be111a34bd5d4d.webp)
刚刚是因为『一个提案只要被一个Acceptor接受，则该提案的value就被选定了』才导致了出现上面不一致的问题。因此，我们需要加一个规定：
规定：一个提案被选定需要被**半数以上**的Acceptor接受
这个规定又暗示了：『一个Acceptor必须能够接受不止一个提案！』不然可能导致最终没有value被选定。比如上图的情况。v1、v2、v3都没有被选定，因为它们都只被一个Acceptor的接受。
最开始讲的『**提案=value**』已经不能满足需求了，于是重新设计提案，给每个提案加上一个提案编号，表示提案被提出的顺序。令『**提案=提案编号+value**』。
虽然允许多个提案被选定，但必须保证所有被选定的提案都具有相同的value值。否则又会出现不一致。
于是有了下面的约束：
P2：如果某个value为v的提案被选定了，那么每个编号更高的被选定提案的value必须也是v。
一个提案只有被Acceptor接受才可能被选定，因此我们可以把P2约束改写成对Acceptor接受的提案的约束P2a。
P2a：如果某个value为v的提案被选定了，那么每个编号更高的被Acceptor接受的提案的value必须也是v。
只要满足了P2a，就能满足P2。
但是，考虑如下的情况：假设总的有5个Acceptor。Proposer2提出[M1,V1]的提案，Acceptor2-5（半数以上）均接受了该提案，于是对于Acceptor2-5和Proposer2来讲，它们都认为V1被选定。Acceptor1刚刚从宕机状态恢复过来（之前Acceptor1没有收到过任何提案），此时Proposer1向Acceptor1发送了[M2,V2]的提案（V2≠V1且M2>M1），对于Acceptor1来讲，这是它收到的第一个提案。根据P1（一个Acceptor必须接受它收到的第一个提案。）,Acceptor1必须接受该提案！同时Acceptor1认为V2被选定。这就出现了两个问题：

1. Acceptor1认为V2被选定，Acceptor2-5和Proposer2认为V1被选定。出现了不一致。
2. V1被选定了，但是编号更高的被Acceptor1接受的提案[M2,V2]的value为V2，且V2≠V1。这就跟P2a（如果某个value为v的提案被选定了，那么每个编号更高的被Acceptor接受的提案的value必须也是v）矛盾了。

![](https://raw.githubusercontent.com/danmuking/image/main/0990598eb71dee1d4641f2b8c04c446d.webp)
所以我们要对P2a约束进行强化！
P2a是对Acceptor接受的提案约束，但其实提案是Proposer提出来的，所有我们可以对Proposer提出的提案进行约束。得到P2b：
P2b：如果某个value为v的提案被选定了，那么之后任何Proposer提出的编号更高的提案的value必须也是v。
由P2b可以推出P2a进而推出P2。
那么，如何确保在某个value为v的提案被选定后，Proposer提出的编号更高的提案的value都是v呢？
只要满足P2c即可：
P2c：对于任意的N和V，如果提案[N, V]被提出，那么存在一个半数以上的Acceptor组成的集合S，满足以下两个条件中的任意一个：

- S中每个Acceptor都没有接受过编号小于N的提案。
- S中Acceptor接受过的最大编号的提案的value为V。
### Proposer生成提案
为了满足P2b，这里有个比较重要的思想：Proposer生成提案之前，应该先去**『学习』**已经被选定或者可能被选定的value，然后以该value作为自己提出的提案的value。如果没有value被选定，Proposer才可以自己决定value的值。这样才能达成一致。这个学习的阶段是通过一个**『Prepare请求』**实现的。
于是我们得到了如下的**提案生成算法**：

1. Proposer选择一个**新的提案编号N**，然后向**某个Acceptor集合**（半数以上）发送请求，要求该集合中的每个Acceptor做出如下响应（response）。
(a) 向Proposer承诺保证**不再接受**任何编号**小于N的提案**。
(b) 如果Acceptor已经接受过提案，那么就向Proposer响应**已经接受过**的编号小于N的**最大编号的提案**。我们将该请求称为**编号为N**的**Prepare请求**。
2. 如果Proposer收到了**半数以上**的Acceptor的**响应**，那么它就可以生成编号为N，Value为V的**提案[N,V]**。这里的V是所有的响应中**编号最大的提案的Value**。如果所有的响应中**都没有提案**，那 么此时V就可以由Proposer**自己选择**。
生成提案后，Proposer将该**提案**发送给**半数以上**的Acceptor集合，并期望这些Acceptor能接受该提案。我们称该请求为**Accept请求**。（注意：此时接受Accept请求的Acceptor集合**不一定**是之前响应Prepare请求的Acceptor集合）
### Acceptor接受提案
Acceptor**可以忽略任何请求**（包括Prepare请求和Accept请求）而不用担心破坏算法的**安全性**。因此，我们这里要讨论的是什么时候Acceptor可以响应一个请求。
我们对Acceptor接受提案给出如下约束：
P1a：一个Acceptor只要尚**未响应过**任何**编号大于N**的**Prepare请求**，那么他就可以**接受**这个**编号为N的提案**。
如果Acceptor收到一个编号为N的Prepare请求，在此之前它已经响应过编号大于N的Prepare请求。根据P1a，该Acceptor不可能接受编号为N的提案。因此，该Acceptor可以忽略编号为N的Prepare请求。当然，也可以回复一个error，让Proposer尽早知道自己的提案不会被接受。
因此，一个Acceptor**只需记住**：1. 已接受的编号最大的提案 2. 已响应的请求的最大编号。
![](https://raw.githubusercontent.com/danmuking/image/main/08b38cb4381b351dae4ad58877ada1fd.webp)
### Paxos算法描述
经过上面的推导，我们总结下Paxos算法的流程。
Paxos算法分为**两个阶段**。具体如下：

- **阶段一：**(a) Proposer选择一个**提案编号N**，然后向**半数以上**的Acceptor发送编号为N的**Prepare请求**。(b) 如果一个Acceptor收到一个编号为N的Prepare请求，且N**大于**该Acceptor已经**响应过的**所有**Prepare请求**的编号，那么它就会将它已经**接受过的编号最大的提案（如果有的话）**作为响应反馈给Proposer，同时该Acceptor承诺**不再接受**任何**编号小于N的提案**。
- **阶段二：**(a) 如果Proposer收到**半数以上**Acceptor对其发出的编号为N的Prepare请求的**响应**，那么它就会发送一个针对**[N,V]提案**的**Accept请求**给**半数以上**的Acceptor。注意：V就是收到的**响应**中**编号最大的提案的value**，如果响应中**不包含任何提案**，那么V就由Proposer**自己决定**。(b) 如果Acceptor收到一个针对编号为N的提案的Accept请求，只要该Acceptor**没有**对编号**大于N**的**Prepare请求**做出过**响应**，它就**接受该提案**。

![](https://raw.githubusercontent.com/danmuking/image/main/8b665a8b14d10507f84b254b40e4eab0.webp)
## Learner学习被选定的value
Learner学习（获取）被选定的value有如下三种方案：
![](https://raw.githubusercontent.com/danmuking/image/main/bfd91ec9bf82c9f3c3aa2335c312dc3b.webp)
## 如何保证Paxos算法的活性
![](https://raw.githubusercontent.com/danmuking/image/main/6104cebd593752883758ef8a302690df.webp)
通过选取**主Proposer**，就可以保证Paxos算法的活性。至此，我们得到一个**既能保证安全性，又能保证活性**的**分布式一致性算法**——**Paxos算法**。
# WIKI
**Paxos算法**是[莱斯利·兰伯特](https://zh.wikipedia.org/wiki/%E8%8E%B1%E6%96%AF%E5%88%A9%C2%B7%E5%85%B0%E4%BC%AF%E7%89%B9)（英语：Leslie Lamport，[LaTeX](https://zh.wikipedia.org/wiki/LaTeX)中的“La”）于1990年提出的一种基于消息传递且具有高度容错特性的共识（consensus）算法。[[1]](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95#cite_note-1)
需要注意的是，Paxos常被误称为“一致性算法”。但是“[一致性（consistency）](https://zh.wikipedia.org/wiki/%E7%BA%BF%E6%80%A7%E4%B8%80%E8%87%B4%E6%80%A7)”和“共识（consensus）”并不是同一个概念。Paxos是一个共识（consensus）算法。[[2]](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95#cite_note-2)
## 问题和假设[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=1)]
分布式系统中的节点通信存在两种模型：[共享内存](https://zh.wikipedia.org/wiki/%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98)（Shared memory）和[消息传递](https://zh.wikipedia.org/wiki/%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92)（Messages passing）。基于消息传递通信模型的分布式系统，不可避免的会发生以下错误：进程可能会慢、被杀死或者重启，消息可能会延迟、丢失、重复。在最普通的 Paxos 场景中，先不考虑可能出现“消息篡改”（即[拜占庭错误](https://zh.wikipedia.org/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98)的情况）。Paxos 算法解决的问题是在一个可能发生前述异常（即排除消息篡改之外的其他任何异常）的[分布式系统](https://zh.wikipedia.org/wiki/%E5%88%86%E5%B8%83%E5%BC%8F%E8%AE%A1%E7%AE%97)中，如何对某个值的看法相同，保证无论发生以上任何异常，都不会破坏决议的共识机制。一个典型的场景是，在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。为保证每个节点执行相同的命令序列，需要在每一条指令上执行一个“共识算法”以保证每个节点看到的指令一致。一个通用的共识算法可以应用在许多场景中，是分布式计算中的重要问题。因此从20世纪80年代起对于共识算法的研究就没有停止过。
为描述Paxos算法，Lamport虚拟了一个叫做Paxos的[希腊城邦](https://zh.wikipedia.org/wiki/%E5%B8%8C%E8%87%98%E5%9F%8E%E9%82%A6)，这个岛按照议会民主制的政治模式制订法律，但是没有人愿意将自己的全部时间和精力放在这种事情上。所以无论是议员，议长或者传递纸条的服务员都不能承诺别人需要时一定会出现，也无法承诺批准决议或者传递消息的时间。但是这里假设没有[拜占庭将军问题](https://zh.wikipedia.org/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98)（Byzantine failure，即虽然有可能一个消息被传递了两次，但是绝对不会出现错误的消息）；只要等待足够的时间，消息就会被传到。另外，Paxos岛上的议员是不会反对其他议员提出的决议的。
对应于分布式系统，议员对应于各个节点，制定的法律对应于系统的状态。各个节点需要进入一个一致的状态，例如在独立[Cache](https://zh.wikipedia.org/wiki/Cache)的[对称多处理器](https://zh.wikipedia.org/w/index.php?title=%E5%AF%B9%E7%A7%B0%E5%A4%9A%E5%A4%84%E7%90%86%E5%99%A8&action=edit&redlink=1)系统中，各个处理器读内存的某个字节时，必须读到同样的一个值，否则系统就违背了一致性的要求。一致性要求对应于法律条文只能有一个版本。议员和服务员的不确定性对应于节点和消息传递通道的不可靠性。
## 算法[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=2)]
### 算法的提出与证明[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=3)]
首先将议员的角色分为 proposers，acceptors，和 learners（允许身兼数职）。proposers 提出提案，提案信息包括提案编号和提议的 value；acceptor 收到提案后可以接受（accept）提案，若提案获得多数派（majority）的 acceptors 的接受，则称该提案被批准（chosen）；learners 只能“学习”被批准的提案。划分角色后，就可以更精确的定义问题：

1. 决议（value）只有在被 proposers 提出后才能被批准（未经批准的决议称为“提案（proposal）”）；
2. 在一次 Paxos 算法的执行实例中，只批准（chosen）一个 value；
3. learners 只能获得被批准（chosen）的 value。
```
在 Leslie Lamport 之后发表的paper中将 majority 替换为更通用的 quorum 概念，但在描述classic paxos的论文  Paxos made simple（页面存档备份，存于互联网档案馆） 中使用的还是majority的概念。
```
另外还需要保证 progress。这一点以后再讨论。
作者通过不断加强上述3个约束（主要是第二个）获得了 Paxos 算法。
批准 value 的过程中，首先 proposers 将 value 发送给 acceptors，之后 acceptors 对 value 进行接受（accept）。为了满足只批准一个 value 的约束，要求经“多数派（majority）”接受的 value 成为正式的决议（称为“批准”决议）。这是因为无论是按照人数还是按照权重划分，两组“多数派”至少有一个公共的 acceptor，如果每个 acceptor 只能接受一个 value，约束2就能保证。
于是产生了一个显而易见的新约束：
```
P1：一个 acceptor 必须接受（accept）第一次收到的提案。
```
注意 P1 是不完备的。如果恰好一半 acceptor 接受的提案具有 value A，另一半接受的提案具有 value B，那么就无法形成多数派，无法批准任何一个 value。
约束2并不要求只批准一个提案，暗示可能存在多个提案。只要提案的 value 是一样的，批准多个提案不违背约束2。于是可以产生约束 P2：
```
P2：一旦一个具有 value v 的提案被批准（chosen），那么之后批准（chosen）的提案必须具有 value v。
```
注：通过某种方法可以为每个提案分配一个编号，在提案之间建立一个全序关系，所谓“之后”都是指所有编号更大的提案。
如果 P1 和 P2 都能够保证，那么约束2就能够保证。
批准一个 value 意味着多个 acceptor 接受（accept）了该 value。因此，可以对 P2 进行加强：
```
P2a：一旦一个具有 value v 的提案被批准（chosen），那么之后任何 acceptor 再次接受（accept）的提案必须具有 value v。
```
由于通信是异步的，P2a 和 P1 会发生冲突。如果一个 value 被批准后，一个 proposer 和一个 acceptor 从休眠中苏醒，前者提出一个具有新的 value 的提案。根据 P1，后者应当接受，根据 P2a，则不应当接受，这种场景下 P2a 和 P1 有矛盾。于是需要换个思路，转而对 proposer 的行为进行约束：
```
P2b：一旦一个具有 value v 的提案被批准（chosen），那么以后任何 proposer 提出的提案必须具有 value v。
```
由于 acceptor 能接受的提案都必须由 proposer 提出，所以 P2b 蕴涵了 P2a，是一个更强的约束。
但是根据 P2b 难以提出实现手段。因此需要进一步加强 P2b。
假设一个编号为 m 的 value v 已经获得批准（chosen），来看看在什么情况下对任何编号为 n（n>m）的提案都含有 value v。因为 m 已经获得批准（chosen），显然存在一个 acceptors 的多数派 C，他们都接受（accept）了v。考虑到任何多数派都和 C 具有至少一个公共成员，可以找到一个蕴涵 P2b 的约束 P2c：
```
P2c：如果一个编号为 n 的提案具有 value v，该提案被提出（issued），那么存在一个多数派，要么他们中所有人都没有接受（accept）编号小于 n 
的任何提案，要么他们已经接受（accept）的所有编号小于 n 的提案中编号最大的那个提案具有 value v。
```
可以用[数学归纳法](https://zh.wikipedia.org/wiki/%E6%95%B0%E5%AD%A6%E5%BD%92%E7%BA%B3%E6%B3%95)证明 P2c 蕴涵 P2b：
假设具有value v的提案m获得批准，当n=m+1时，采用反证法，假如提案n不具有value v，而是具有value w，根据P2c，则存在一个多数派S1，要么他们中没有人接受过编号小于n的任何提案，要么他们已经接受的所有编号小于n的提案中编号最大的那个提案是value w。由于S1和通过提案m时的多数派C之间至少有一个公共的acceptor，所以以上两个条件都不成立，导出矛盾从而推翻假设，证明了提案n必须具有value v；
若（m+1）..（N-1）所有提案都具有value v，采用反证法，假如新提案N不具有value v，而是具有value w',根据P2c，则存在一个多数派S2，要么他们没有接受过m..（N-1）中的任何提案，要么他们已经接受的所有编号小于N的提案中编号最大的那个提案是value w'。由于S2和通过m的多数派C之间至少有一个公共的acceptor，所以至少有一个acceptor曾经接受了m，从而也可以推出S2中已接受的所有编号小于n的提案中编号最大的那个提案的编号范围在m..（N-1）之间，而根据初始假设，m..（N-1）之间的所有提案都具有value v，所以S2中已接受的所有编号小于n的提案中编号最大的那个提案肯定具有value v，导出矛盾从而推翻新提案N不具有value v的假设。根据数学归纳法，我们证明了若满足P2c，则P2b一定满足。
P2c是可以通过消息传递模型实现的。另外，引入了P2c后，也解决了前文提到的P1不完备的问题。
### 算法的内容[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=4)]
要满足P2c的约束，proposer提出一个提案前，首先要和足以形成多数派的acceptors进行通信，获得他们进行的最近一次接受（accept）的提案（prepare过程），之后根据回收的信息决定这次提案的value，形成提案开始投票。当获得多数acceptors接受（accept）后，提案获得批准（chosen），由acceptor将这个消息告知learner。这个简略的过程经过进一步细化后就形成了Paxos算法。
在一个paxos实例中，每个提案需要有不同的编号，且编号间要存在全序关系。可以用多种方法实现这一点，例如将序数和proposer的名字拼接起来。如何做到这一点不在Paxos算法讨论的范围之内。
如果一个没有chosen过任何proposer提案的acceptor在prepare过程中回答了一个proposer针对提案n的问题，但是在开始对n进行投票前，又接受（accept）了编号小于n的另一个提案（例如n-1），如果n-1和n具有不同的value，这个投票就会违背P2c。因此在prepare过程中，acceptor进行的回答同时也应包含承诺：不会再接受（accept）编号小于n的提案。这是对P1的加强：
```
P1a：当且仅当acceptor没有回应过编号大于n的prepare请求时，acceptor接受（accept）编号为n的提案。
```
现在已经可以提出完整的算法了。
#### 决议的提出与批准[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=5)]
通过一个决议分为两个阶段：

1. prepare阶段：
   1. proposer选择一个提案编号n并将prepare请求发送给acceptors中的一个多数派；
   2. acceptor收到prepare消息后，如果提案的编号大于它已经回复的所有prepare消息(回复消息表示接受accept)，则acceptor将自己上次接受的提案回复给proposer，并承诺不再回复小于n的提案；
2. 批准阶段：
   1. 当一个proposer收到了多数acceptors对prepare的回复后，就进入批准阶段。它要向回复prepare请求的acceptors发送accept请求，包括编号n和根据P2c决定的value（如果根据P2c没有已经接受的value，那么它可以自由决定value）。
   2. 在不违背自己向其他proposer的承诺的前提下，acceptor收到accept请求后即批准这个请求。

这个过程在任何时候中断都可以保证正确性。例如如果一个proposer发现已经有其他proposers提出了编号更高的提案，则有必要中断这个过程。因此为了优化，在上述prepare过程中，如果一个acceptor发现存在一个更高编号的提案，则需要通知proposer，提醒其中断这次提案。
#### 实例[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=6)]
用实际的例子来更清晰地描述上述过程：
有A1, A2, A3, A4, A5 5位议员，就税率问题进行决议。议员A1决定降税率,因此它向所有人发出一个草案。这个草案的内容是：
```
现有的税率是什么?如果没有决定，我来决定一下. 提出时间：本届议会第3年3月15日;提案者：A1
```
在最简单的情况下，没有人与其竞争;信息能及时顺利地传达到其它议员处。
于是, A2-A5回应：
```
我已收到你的提案，等待最终批准
```
而A1在收到2份回复后就发布最终决议：
```
税率已定为10%,新的提案不得再讨论本问题。
```
这实际上退化为[二阶段提交](https://zh.wikipedia.org/wiki/%E4%BA%8C%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4)协议。
现在我们假设在A1提出提案的同时, A5决定将税率定为20%:
```
现有的税率是什么?如果没有决定，我来决定一下.时间：本届议会第3年3月16日;提案者：A5
```
草案要通过侍从送到其它议员的案头. A1的草案将由4位侍从送到A2-A5那里。现在，负责A2和A3的侍从将草案顺利送达，负责A4和A5的侍从则不上班. A5的草案则顺利的送至A4和A3手中。
现在, A1, A2, A3收到了A1的提案; A4, A3, A5收到了A5的提案。按照协议, A1, A2, A4, A5将接受他们收到的提案，侍从将拿着
```
我已收到你的提案，等待最终批准
```
的回复回到提案者那里。
而A3的行为将决定批准哪一个。
在讨论之前我们要明确一点，提案是全局有序的。在这个示例中，是说每个提案提出的日期都不一样。即第3年3月15日只有A1的提案；第3年3月16日只有A5的提案。不可能在某一天存在两个提案。
##### 情况一[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=7)]
假设A1的提案先送到A3处，而A5的侍从决定放假一段时间。于是A3接受并派出了侍从. A1等到了两位侍从，加上它自己已经构成一个多数派，于是税率10%将成为决议. A1派出侍从将决议送到所有议员处：
```
税率已定为10%,新的提案不得再讨论本问题。
```
A3在很久以后收到了来自A5的提案。由于税率问题已经讨论完毕，开始讨论某些议员在第3年3月17日提出的议案。因此这个3月16日提出的议案他不去理会。他自言自语地抱怨一句：
```
这都是老问题了，没有必要讨论了。
```
##### 情况二[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=8)]
依然假设A1的提案先送到A3处，但是这次A5的侍从不是放假了，只是中途耽搁了一会。这次, A3依然会将"接受"回复给A1.但是在决议成型之前它又收到了A5的提案。则：
1.如果A5提案的提出时间比A1的提案更晚一些，这里确实满足这种情况，因为3月16日晚于3月15日。，则A3回复：
```
我已收到您的提案，等待最终批准，但是您之前有人提出将税率定为10%,请明察。
```
于是, A1和A5都收到了足够的回复。这时关于税率问题就有两个提案在同时进行。但是A5知道之前有人提出税率为10%.于是A1和A5都会向全体议员广播：
```
税率已定为10%,新的提案不得再讨论本问题。
```
共识到了保证。
2. 如果A5提案的提出时间比A1的提案更早一些。假设A5的提案是3月14日提出，则A3直接不理会。
```
A1不久后就会广播税率定为10%.
```
#### 决议的发布[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=9)]
一个显而易见的方法是当acceptors批准一个value时，将这个消息发送给所有learners。但是这个方法会导致消息量过大。
由于假设没有Byzantine failures，learners可以通过别的learners获取已经通过的决议。因此acceptors只需将批准的消息发送给指定的某一个learner，其他learners向它询问已经通过的决议。这个方法降低了消息量，但是指定learner失效将引起系统失效。
因此acceptors需要将accept消息发送给learners的一个子集，然后由这些learners去通知所有learners。
但是由于消息传递的不确定性，可能会没有任何learner获得了决议批准的消息。当learners需要了解决议通过情况时，可以让一个proposer重新进行一次提案。注意一个learner可能兼任proposer。
#### Progress的保证[[编辑](https://zh.wikipedia.org/w/index.php?title=Paxos%E7%AE%97%E6%B3%95&action=edit&section=10)]
根据上述过程当一个proposer发现存在编号更大的提案时将终止提案。这意味着提出一个编号更大的提案会终止之前的提案过程。如果两个proposer在这种情况下都转而提出一个编号更大的提案，就可能陷入活锁，违背了Progress的要求。一般活锁可以通过 **随机睡眠-重试** 的方法解决。这种情况下的解决方案是选举出一个leader，仅允许leader提出提案。但是由于消息传递的不确定性，可能有多个proposer自认为自己已经成为leader。Lamport在[The Part-Time Parliament](http://research.microsoft.com/users/lamport/pubs/lamport-paxos.pdf)（[页面存档备份](https://web.archive.org/web/20070418160712/http://research.microsoft.com/users/lamport/pubs/lamport-paxos.pdf)，存于[互联网档案馆](https://zh.wikipedia.org/wiki/%E4%BA%92%E8%81%94%E7%BD%91%E6%A1%A3%E6%A1%88%E9%A6%86)）一文中描述并解决了这个问题。
## 参考资料

- 论文《Paxos Made Simple》
- 论文《The Part-Time Parliament》
- 英文版维基百科的Paxos
- [https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95)
- 书籍《从Paxos到ZooKeeper》
- [https://www.cnblogs.com/linbingdong/p/6253479.html](https://www.cnblogs.com/linbingdong/p/6253479.html)


 
