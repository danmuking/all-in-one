## 0x00. 消息的发送流程
一条消息从生产到被消费，将会经历三个阶段：
![](https://raw.githubusercontent.com/danmuking/image/main/43e722b34f9fda1d4c6df8fafd1b52f2.jpeg)

- 生产阶段，Producer 新建消息，然后通过网络将消息投递给 MQ Broker
- 存储阶段，消息将会存储在 Broker 端磁盘中
- 消息阶段， Consumer 将会从 Broker 拉取消息

以上任一阶段都可能会丢失消息，我们只要找到这三个阶段丢失消息原因，采用合理的办法避免丢失，就可以彻底解决消息丢失的问题。
## 0x01. 生产阶段
生产者（Producer） 通过网络发送消息给 Broker，当 Broker 收到之后，将会返回确认响应信息给 Producer。所以生产者只要接收到返回的确认响应，就代表消息在生产阶段未丢失。
RocketMQ 发送消息示例代码如下：
```java
DefaultMQProducer mqProducer=new DefaultMQProducer("test");
// 设置 nameSpace 地址
mqProducer.setNamesrvAddr("namesrvAddr");
mqProducer.start();
Message msg = new Message("test_topic" /* Topic */,
        "Hello World".getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
);
// 发送消息到一个Broker
try {
    SendResult sendResult = mqProducer.send(msg);
} catch (RemotingException e) {
    e.printStackTrace();
} catch (MQBrokerException e) {
    e.printStackTrace();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```
send 方法是一个同步操作，只要这个方法不抛出任何异常，就代表消息已经**发送成功**。
消息发送成功仅代表消息已经到了 Broker 端，Broker 在不同配置下，可能会返回不同响应状态:

- SendStatus.SEND_OK
- SendStatus.FLUSH_DISK_TIMEOUT
- SendStatus.FLUSH_SLAVE_TIMEOUT
- SendStatus.SLAVE_NOT_AVAILABLE

引用官方状态说明：
![](https://raw.githubusercontent.com/danmuking/image/main/026a62d1435dedbdee552ab832ec73ff.jpeg)
上图中不同 broker 端配置将会在下文详细解释
另外 RocketMQ 还提供异步的发送的方式，适合于链路耗时较长，对响应时间较为敏感的业务场景。
```java
DefaultMQProducer mqProducer = new DefaultMQProducer("test");
// 设置 nameSpace 地址
mqProducer.setNamesrvAddr("127.0.0.1:9876");
mqProducer.setRetryTimesWhenSendFailed(5);
mqProducer.start();
Message msg = new Message("test_topic" /* Topic */,
        "Hello World".getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
);

try {
    // 异步发送消息到，主线程不会被阻塞，立刻会返回
    mqProducer.send(msg, new SendCallback() {
        @Override
        public void onSuccess(SendResult sendResult) {
            // 消息发送成功，
        }

        @Override
        public void onException(Throwable e) {
            // 消息发送失败，可以持久化这条数据，后续进行补偿处理
        }
    });
} catch (RemotingException e) {
    e.printStackTrace();
} catch (InterruptedException e) {
    e.printStackTrace();
}
```
异步发送消息一定要**注意重写**回调方法，在回调方法中检查发送结果。
不管是同步还是异步的方式，都会碰到网络问题导致发送失败的情况。针对这种情况，我们可以设置合理的重试次数，当出现网络问题，可以自动重试。设置方式如下：
```java
// 同步发送消息重试次数，默认为 2
mqProducer.setRetryTimesWhenSendFailed(3);
// 异步发送消息重试次数，默认为 2
mqProducer.setRetryTimesWhenSendAsyncFailed(3);
```
## 0x02. Broker 存储阶段
默认情况下，消息只要到了 Broker 端，将会优先保存到内存中，然后立刻返回确认响应给生产者。随后 Broker 定期批量的将一组消息从内存异步刷入磁盘。
这种方式减少 I/O 次数，可以取得更好的性能，但是如果发生机器掉电，异常宕机等情况，消息还未及时刷入磁盘，就会出现丢失消息的情况。
若想保证 Broker 端不丢消息，保证消息的可靠性，我们需要将消息保存机制修改为同步刷盘方式，即消息**存储磁盘成功**，才会返回响应。
修改 Broker 端配置如下：
```
## 默认情况为 ASYNC_FLUSH 
flushDiskType = SYNC_FLUSH
```
若 Broker 未在同步刷盘时间内（**默认为 5s**）完成刷盘，将会返回 SendStatus.FLUSH_DISK_TIMEOUT 状态给生产者。
**集群部署**
为了保证可用性，Broker 通常采用一主（**master**）多从（**slave**）部署方式。为了保证消息不丢失，消息还需要复制到 slave 节点。
默认方式下，消息写入 **master** 成功，就可以返回确认响应给生产者，接着消息将会异步复制到 **slave** 节点。
注：master 配置：flushDiskType = SYNC_FLUSH
此时若 master 突然**宕机且不可恢复**，那么还未复制到 **slave** 的消息将会丢失。
为了进一步提高消息的可靠性，我们可以采用同步的复制方式，**master** 节点将会同步等待 **slave** 节点复制完成，才会返回确认响应。
异步复制与同步复制区别如下图：
![](https://raw.githubusercontent.com/danmuking/image/main/880c359056b617a056470545cdadad40.jpeg)
注： 大家不要被上图误导，broker master 只能配置一种复制方式，上图只为解释同步复制的与异步复制的概念。
Broker master 节点 同步复制配置如下：
```
## 默认为 ASYNC_MASTER 
brokerRole=SYNC_MASTER
```
如果 **slave** 节点未在指定时间内同步返回响应，生产者将会收到 SendStatus.FLUSH_SLAVE_TIMEOUT 返回状态。
**小结**
结合生产阶段与存储阶段，若需要**严格保证消息不丢失**，broker 需要采用如下配置：
```
## master 节点配置
flushDiskType = SYNC_FLUSH
brokerRole=SYNC_MASTER

## slave 节点配置
brokerRole=slave
flushDiskType = SYNC_FLUSH
```
同时这个过程我们还需要生产者配合，判断返回状态是否是 SendStatus.SEND_OK。若是其他状态，就需要考虑补偿重试。
虽然上述配置提高消息的高可靠性，但是会**降低性能**，生产实践中需要综合选择。
## 0x03. 消费阶段
消费者从 broker 拉取消息，然后执行相应的业务逻辑。一旦执行成功，将会返回 ConsumeConcurrentlyStatus.CONSUME_SUCCESS 状态给 Broker。
如果 Broker 未收到消费确认响应或收到其他状态，消费者下次还会再次拉取到该条消息，进行重试。这样的方式有效避免了消费者消费过程发生异常，或者消息在网络传输中丢失的情况。
消息消费的代码如下：
```java
// 实例化消费者
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("test_consumer");

// 设置NameServer的地址
consumer.setNamesrvAddr("namesrvAddr");

// 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息
consumer.subscribe("test_topic", "*");
// 注册回调实现类来处理从broker拉取回来的消息
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        // 执行业务逻辑
        // 标记该消息已经被成功消费
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
// 启动消费者实例
consumer.start();
```
以上消费消息过程的，我们需要**注意返回消息状态**。只有当业务逻辑真正执行成功，我们才能返回 ConsumeConcurrentlyStatus.CONSUME_SUCCESS。否则我们需要返回 ConsumeConcurrentlyStatus.RECONSUME_LATER，稍后再重试。
## 0x04. 总结
看完 RocketMQ 不丢消息处理办法，回头再看这篇 [kafka](https://mp.weixin.qq.com/s/qttczGROYoqSulzi8FLXww)，有没有发现，两者解决思路是一样的，区别就是参数配置不一样而已。
所以下一次，面试官再问你 XX 消息队列如何保证不丢消息？如果你没用过这个消息队列，也不要哭，微笑面对他，从容给他分析那几步会丢失，然后大致解决思路。
最后我们还可以说出我们的思考，虽然提高消息可靠性，但是可能导致消息重发，重复消费。所以对于消费客户端，需要注意保证**幂等性**。
![](https://raw.githubusercontent.com/danmuking/image/main/555e91c4e378e6384b322fd7370baf61.jpeg)
但是要注意了，这时面试官可能就会跟你的话题，让你来聊聊如何保证幂等性，一定先想好再说哦。

## 参考资料
[https://www.cnblogs.com/goodAndyxublog/p/12563813.html](https://www.cnblogs.com/goodAndyxublog/p/12563813.html)
