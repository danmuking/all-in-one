![image.png](https://raw.githubusercontent.com/danmuking/image/main/8200e548ab5c14b42dfbc7090491609c.png)
现在的社交平台不是都需要展示用户的归属地了嘛。虽然现在还没人强制要求抹茶聊天也这么做，但是我们也需要提早准备起来不是？
这个功能有两个主要功能点，1.ip获取2.ip转归属地。来看看我是怎么实现的。
## ip获取
### http请求
对于controller的请求，我们只需要写个拦截器，将用户的ip设置进上下文即可，非常方便。
```java
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        RequestInfo info = new RequestInfo();
        info.setUid(Optional.ofNullable(request.getAttribute(TokenInterceptor.ATTRIBUTE_UID)).map(Object::toString).map(Long::parseLong).orElse(null));
        info.setIp(ServletUtil.getClientIP(request));
        RequestHolder.set(info);
        return true;
    }
```
ip在请求头中都会携带。直接用hutool的工具类获取ip
```java
public static String getClientIP(HttpServletRequest request, String... otherHeaderNames) {
    String[] headers = {"X-Forwarded-For", "X-Real-IP", "Proxy-Client-IP", "WL-Proxy-Client-IP", "HTTP_CLIENT_IP", "HTTP_X_FORWARDED_FOR"};
    if (ArrayUtil.isNotEmpty(otherHeaderNames)) {
        headers = ArrayUtil.addAll(headers, otherHeaderNames);
    }

    return getClientIPByHeader(request, headers);
}
```
这里有点要注意，如果我们开启了nginx来带来请求，需要在nginx里面保存用户真实ip到`X-Real-IP`，否则你拿到的就是nginx的ip地址了。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/d09c674e9498bd5bf60609cd0237df08.png)
### websocket请求
对于websocket请求获取ip就会麻烦一些。
首先我们要有个概念，websocket初期会借助http来升级协议。所以我们需要在http升级之前就要获取ip，并且将用户ip保存起来。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/8633e52ee1a3b40aa4bee976aefd97ce.png)
在协议升级前，我们加入了`HttpHeadersHandler`处理器，这时候还是能拿到http的request的，想获取header很容易。
```java
public class HttpHeadersHandler extends ChannelInboundHandlerAdapter {
    private AttributeKey<String> key = AttributeKey.valueOf("Id");

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof FullHttpRequest) {
            HttpHeaders headers = ((FullHttpRequest) msg).headers();
            String ip = headers.get("X-Real-IP");
            if (Objects.isNull(ip)) {//如果没经过nginx，就直接获取远端地址
                InetSocketAddress address = (InetSocketAddress) ctx.channel().remoteAddress();
                ip = address.getAddress().getHostAddress();
            }
            NettyUtil.setAttr(ctx.channel(), NettyUtil.IP, ip);
        }
        ctx.fireChannelRead(msg);
    }
}
```
之后协议升级后，请求就不会再走这个处理器了，所以我们的ip需要保存起来。正好`channel`其实有个附件功能，我们可以直接把ip作为附件保存进channel。之后每次的websocket请求，都用的是同一个`channel`，从里面取ip就好了。
正好所有连接的用户，我们也会去保存uid和channel的映射关系，保存这个channel。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/cfbcf4f530bfcf6a71ebdf248b154966.png)
## ip的更新
ip的更新时机其实也是一个话题。总不可能用户每次请求，我们都要去做一次ip更新吧，那也太麻烦了。我们可以在用户首次认证去更新ip即可。
用户首次认证有两个场景：
1.用户浏览器里有token。前端拿它来后端认证下即可。
2.用户token失效重新扫码登录。
针对于第二种登录，扫码的时候是wx给我们的回调。我们通过回调的code，去找出code对应的连接`channel`。再从`channel`里找到用户信息以及ip
我们可以选用用户认证的时间点来触发ip的刷新。
## ip的保存
ip的信息其实是个比较复杂的数据类型，我们可以直接通过json格式存成user的扩展信息。
json格式需要mysql5.7+。
ip获取两个入口，http，websokcet。
```java
//注册时的ip
private String createIp;
//注册时的ip详情
private IpDetail createIpDetail;
//最新登录的ip
private String updateIp;
//最新登录的ip详情
private IpDetail updateIpDetail;
```
```java
//注册时的ip
private String ip;
//最新登录的ip
private String isp;
private String isp_id;
private String city;
private String city_id;
private String country;
private String country_id;
private String region;
private String region_id;
```
json格式在数据库里就是一个字符串，可以通过sql很方便的提取或更新其中的某个字段。通过mybatisplus获取实体类的时候，也可以自动帮忙反序列化。具体配置可看[json字段整合](https://www.yuque.com/snab/mallchat/ww7awri587zf2ua7)
## ip归属地解析
ip的归属地解析也是个很有意思的话题，本质就是利用ip解析出所属地区。
### 基于ip2region文件索引
github上有个开源的ip地址库`ip2region`，地址[https://github.com/lionsoul2014/ip2region](https://github.com/lionsoul2014/ip2region)
![image.png](https://raw.githubusercontent.com/danmuking/image/main/e8ddd3ea8dda3e9673cfbdff7e91a99c.png)
需要在项目启动的时候预加载文件数据，里面的地址不会实时更新，如果需要更新还需要根据指引写爬虫更新。
### 基于淘宝开放接口
淘宝有提供了ip地址库的查询接口，大家可以自己postman测试下。
淘宝的IP地址库API可以提供IP地址的详细信息，包括国家、省份、城市、经纬度等。使用淘宝的IP地址库API，可以轻松获取IP地址的详细信息，从而获取IP地址的地理位置。
```java
curl --request GET \
  --url 'https://ip.taobao.com/outGetIpInfo?ip=112.96.166.230&accessKey=alibaba-inc' \
  --header 'content-type: application/json' 
```
![image.png](https://raw.githubusercontent.com/danmuking/image/main/cf90c814284ddfd44e8296cac7e72ecb.png)
淘宝自己的地址库会一直更新，比较全。而且没有任何依赖，直接接口解析。它只有一个缺点，就是有频控o(╥﹏╥)o。
```jsx
{
  "msg": "the request over max qps for user ,the accessKey=alibaba-inc",
  "code": 4
}
```
如果想用淘宝的地址解析，我们需要写一套框架，能够适应他的频控，匀速的排队的能够重试的去慢慢异步解析我们的ip详情，我是怎么做的呢？
### 异步解析淘宝ip接口框架实现
用户的登录请求，对于后台来说都是并发的。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/a1f53d60cbd79763ea75d255a406b934.png)
每个用户的请求线程都会阻塞等待ip解析完，才响应结果。并且并发的情况下解析ip容易出现淘宝ip的限流。
根据淘宝ip的限流，我们会想到框架需要的几个重点
1.`排队`，把ip解析当成一个任务，存进队列，一个个排队解析，不要太快。
2.`重试`，针对某个任务解析失败，需要能重试，但是重试也要有最大次数。
3.`异步`，淘宝解析接口很慢，异步解析，不要影响主任务。
综上，用`coreSize`为1的线程池，就可以比较方便的实现这一点。线程池的队列用来排队。
同时还能异步执行。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/2eff6acbda0228639197ee88947da3f7.png)
我们创建了一个这样的线程池
```java
private static ExecutorService executor = new ThreadPoolExecutor(1, 1,
            0L, TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>(500), new NamedThreadFactory("refresh-ipDetail", false));
```
同时写了个ip解析的接口
![image.png](https://raw.githubusercontent.com/danmuking/image/main/d30f7ce3614e29499f9da57d0e67c98a.png)
需要刷新ip详情的用户，把uid扔进来就好了，最小的耦合业务。
里面具体的实现逻辑
com.abin.mallchat.common.user.service.impl.IpServiceImpl#refreshIpDetailAsync
```java
private static IpDetail TryGetIpDetailOrNullTreeTimes(String ip) {
    for (int i = 0; i < 3; i++) {
        IpDetail ipDetail = getIpDetailOrNull(ip);
        if (Objects.nonNull(ipDetail)) {
            return ipDetail;
        }
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    return null;
}
```
每个任务最多重试三次，被频控后，就休眠一会儿再重试
```java
public static IpDetail getIpDetailOrNull(String ip) {
    String body = HttpUtil.get("https://ip.taobao.com/outGetIpInfo?ip=" + ip + "&accessKey=alibaba-inc");
    try {
        IpResult<IpDetail> result = JSONUtil.toBean(body, new TypeReference<IpResult<IpDetail>>() {
        }, false);
        if (result.isSuccess()) {
            return result.getData();
        }
    } catch (Exception ignored) {
    }
    return null;
}
```
这个线程是并不是spring管理的线程池，所以需要我们自己来实现优雅停机。
```java
@Override
public void destroy() throws InterruptedException {
    executor.shutdown();
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {//最多等30秒，处理不完就拉倒
        if (log.isErrorEnabled()) {
            log.error("Timed out while waiting for executor [{}] to terminate", executor);
        }
    }
}
```
> 线程池的任务总是不安全的，会有解析失败的小概率事件，但是失败又如何，下一次登录重新解析就好了嘛

### 基于mq实现ip解析框架
上面提到的`排队`，`重试`，`异步`。我们也很容易想到mq。当然可以用mq来实现。用mq的话并发就需要进行一定的控制，不然会造成大量的竞争和失败。所以要限制mq的消费并发度`consumeThreadNumber`设置为1。
