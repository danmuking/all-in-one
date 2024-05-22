![image.png](https://raw.githubusercontent.com/danmuking/image/main/43bf8a17e302143d7a6d0b1e7be887e5.png)
url能够识别高亮，并且还解析出对应网页的标题。这样点击就能跳转，不点击也能知道这个网站是干嘛用的，是不是方便很多。这样一个功能是怎么做出来的，又有哪些细节呢？本文会详细的从调研到方案选型，再到技术实现，一一道来。
## 背景
为啥要做这个url跳转的功能呢？首先是因为我们的项目有点赞的功能。大家为了能够获赞，就会发一些很好的博客链接。
如果这个链接能够被识别出来，点击就能跳转，是不是对大家都会很方便？甚至做的再好些，直接把url对应的标题解析出来，不用点击就知道这个链接是啥内容，我感不感兴趣。
## 调研
url的识别其实很简单，一般就是正则匹配。url的标题解析也很简单，就是提前访问一下，然后获取对应网站的title标签内容。
那难点在哪里呢？
> 难点在于，你什么时候去解析或者匹配url。是后端做还是前端做，是入库时做，还是列表查询做？
> 和前端的消息体是怎么样的？

于是我们先调研了一下市面上已经成熟的产品，相信大家并不陌生，就是我们的知识星球。
[https://t.zsxq.com/12KvLsk3f](https://t.zsxq.com/12KvLsk3f)
![image.png](https://raw.githubusercontent.com/danmuking/image/main/c7dc7a14cbe3804ac1d573d37de920ba.png)
熟悉星球的小伙伴应该都知道，星球会解析url的标题。我们通过抓包就可以了解他们实现的思路。
打开F12，抓到了这么多包，不知道哪个是我们想要的内容。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/5eeb33d4ae71fb3a3a10ac1a7378ca8c.png)
这时候交大家一个小技巧，通过内容搜索请求，ctrl+f出现搜索框搜索jianshu
![image.png](https://raw.githubusercontent.com/danmuking/image/main/a906f18d064a9b5a1e643f9f5b8b2e40.png)
点击info下的消息体，复制出来发现被unicode编码了，进行解码一下。
[http://www.kuquidc.com/convert/Unicode.php](http://www.kuquidc.com/convert/Unicode.php)
![image.png](https://raw.githubusercontent.com/danmuking/image/main/216877106a0866c95792b643e66096ba.png)
后端是直接在原文用一种特殊格式传给前端，里面指定了title文本内容和跳转链接。
我们要不要学它这么去做呢？
至少确定了，这件事是后端来做。
## 方案选型
首先我们思考下url什么时候去解析。url的正则匹配不怎么耗时，主要是请求外部网站解析标题的时候比较耗时。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/55304acee9baca1ddb6bdfacc8fe46c0.png)
你可以选择：

1. 发送消息，入库前去解析
2. 用户访问消息列表的时候，查出来在后端解析
3. 直接原文扔给用户，前端自己去解析。

首先排除2，总共才一份消息。不同的人请求，都要去重复解析，拉高的整体的接口响应，占用了后端的资源。
3和2的区别在于，3是用户端自己解析，不会占用服务的资源。这样消息阅读者的加载会慢一些。但是它有个致命的缺点，如果你在千人在线的群发一条链接。这一千个前端都会去请求那个网站，解析url，对别人的网站负担比较大。我们还是不要给别人的网站造成这样的困扰比较好。
思来想去，就剩最后一个选择了。那就是让消息的发送者稍微委屈下（谁让你要发链接呢！）。
那有没有可能异步呢？也不要去阻塞发送者。先入库，再异步解析，再推送给其他用户。
答案是否定的，消息发送者其实也需要实时看见自己的链接被解析了。这个时间与其都要等，不如就直接等着吧。
于是最终我们决定让消息发送者承担下所有，但是别太悲观，该有的优化，我们也会尽力去优化的。
## 技术实现
在技术实现之前，我们需要先进行一个最小粒度的验证。就是先验证我们能够识别url和标题解析，再去进行代码更优雅的编写。
### 尝试识别url
```java
public static void main(String[] args) {
    String content = "这是一个很长的字符串再来 www.github.com，其中包含一个URL www.baidu.com,, 一个带有端口号的URL http://www.jd.com:80, 一个带有路径的URL http://mallchat.cn, 还有美团技术文章https://mp.weixin.qq.com/s/hwTf4bDck9_tlFpgVDeIKg";
    Pattern pattern = Pattern.compile("((http|https)://)?(www.)?([\\w_-]+(?:(?:\\.[\\w_-]+)+))([\\w.,@?^=%&:/~+#-]*[\\w@?^=%&/~+#-])?");
    List<String> matchList = ReUtil.findAll(pattern, content, 0);//hutool工具类
    System.out.println(matchList);
}
```
输出的结果
```java
[www.github.com, www.baidu.com, http://www.jd.com:80, http://mallchat.cn, https://mp.weixin.qq.com/s/hwTf4bDck9_tlFpgVDeIKg]
```
大部分的case都匹配上了，没有问题。其实想要找一个完美的链接匹配正则是有难度的。这个正则还是一个粉丝用gpt4生成的。
### 尝试获取标题
引入jsoup依赖
```bash
<dependency>
    <groupId>org.jsoup</groupId>
    <artifactId>jsoup</artifactId>
    <version>1.15.3</version>
</dependency>
```
获取标题，我们可用java常用的爬虫工具Jsoup来进行内容的解析。一般浏览器展示的标签页的文本，就是html里面的`<title></title>`标签
```java
public static void main(String[] args) throws IOException {
    Connection connect = Jsoup.connect("http://www.baidu.com");
    Document document = connect.get();
    String title = document.title();
    System.out.println(title);
}
```
输出结果
```java
百度一下，你就知道
```
为了让你能更加清楚的了解`jsoup`爬取到的`document`是啥样，我们debug看看
![image.png](https://raw.githubusercontent.com/danmuking/image/main/57a885cf230645c3fb8ff7cb72d4e446.png)
其实就是一个html页面。我们通过`document.title()`去匹配标题的标签内容。
所有网页的标题都是这个title标签吗？也不全是。比如微信文章的网页。
[https://mp.weixin.qq.com/s/O4Ts0UnnDlYB5OQyCxO0Og](https://mp.weixin.qq.com/s/O4Ts0UnnDlYB5OQyCxO0Og)，你去访问这个title是拿不到任何东西的
![image.png](https://raw.githubusercontent.com/danmuking/image/main/c0f362acbfc2603beb253f1b23deacf3.png)
因为它压根就没有title标签。通过深入分析它的html文档，你会发现，它的标题是在<meta>标签里，也有一定的规律。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/245e2b68fa943f779fdf0b2b4cad6318.png)
找到了规律，就能匹配出我们的标题
```java
public static void main(String[] args) throws IOException {
    Connection connect = Jsoup.connect("https://mp.weixin.qq.com/s/O4Ts0UnnDlYB5OQyCxO0Og");
    Document document = connect.get();
    String title = document.getElementsByAttributeValue("property", "og:title").attr("content");
    System.out.println(title);
}
```
这样就能解析出我们的标题了。
你发现没，不同的页面，标题在不同的标签内，需要不同的解析方式。我们也不知道哪些标题需要用到什么解析方式，只能把解析不出来的标题打个日志，然后再去慢慢的添加更多的解析方式，那未来肯定会扩展很多的解析方式。
把标题的解析方式做成解析器。每个解析器串成一个链条。通用的解析器优先级更高。链条中直到其中某个解析器解析出标题，就返回。
解析器串起的链条就是责任链模式，创建责任链的地方就是工厂模式。而不同的类实现不同的url解析方法，这就是策略模式，而抽象类里面的逻辑，又像是模板方法模式，一口气就能实现四种模式。
### 搭建url解析框架
正好想到之前学spring源码的时候，看见spring的参数解析器（就是识别方法的参数名）和我们的这个需求类似，我们可以参考spring去进行框架的搭建。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/59be32e647d6f98903ef56f695ae61cd.png)
定义接口，核心的接口是`getContentTitleMap`，其他的方法只是细分的获取步骤。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/cc7349e030f1018cfe516e5e445381ef.png)
公共的逻辑放在抽象类。子类`commonUrl`和`WxUrl`就是不同的标题解析策略。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/43206646d7e1e88653e64a0e964f8b72.png)
![image.png](https://raw.githubusercontent.com/danmuking/image/main/0bd61ad25b355e1eb18b3a7f00a75a24.png)
![image.png](https://raw.githubusercontent.com/danmuking/image/main/d34a0ba55dd065761ebe493658bd80e5.png)
`prioritizedUrl`是我们的策略类，同时也是组装责任链的工厂。如果你调用它，它会按顺序执行责任链，直到解析出url标题。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/5990ff1b3157f2666d2e5d32a6d04825.png)
> 看源码能让我们更加深入的理解设计模式的运用，更能为我们平时的开发提供思路灵感

这样一个url解析框架基本完成，发送消息的时候只需要匹配出所有url，然后通过责任链去解析标题就ok了。
真的ok了吗？
### 并行解析
如果一个用户发了一大段话，里面有非常多个url，我们要串行的去一步步解析吗。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/329fc7fa33ad061253b75d021e2389db.png)
耗费的时间是每一个网址解析时间的总和。每个网址的解析其实互不相关，完全可以独立进行。
![image.png](https://raw.githubusercontent.com/danmuking/image/main/a5d535a4e7afbb1b57dd661eaee21769.png)
这样立马就省下了大多数的时间，好歹消息的发送者是要阻塞在这里的，也要考虑下它的体验吧。
这样还不够，针对github这种网站，请求时间可能会很长，它决定了用户的等待上限。我们要对它们做一个熔断，也就是超过1s，没拿到网站，就算了。对于代码就是这样。
```java
Connection connect = Jsoup.connect(matchUrl);
connect.timeout(1000);
return connect.get();
```
![image.png](https://raw.githubusercontent.com/danmuking/image/main/f737cf4f8664992cbd6560ff3db8d62f.png)
github解析超时直接丢弃。这样的一个并行框架要怎么去写呢？可以用`CompletableFuture`，不了解的要去先学学[JUC](https://www.yuque.com/snab/java/tbma9x40r1nv40to#iyJF9)，然后再看看美团的一篇技术文章：[外卖商家端API的异步化](https://mp.weixin.qq.com/s/GQGidprakfticYnbVYVYGQ)。
美团提供了一个异步的工具类，对`CompletableFuture`进行了一层封装，用起来更加简洁优雅。
com.abin.mallchat.common.common.utils.discover.AbstractUrlTitleDiscover#getContentTitleMap
![image.png](https://raw.githubusercontent.com/danmuking/image/main/61867e93eaba1392488b02886e7d3751.png)
## 总结
看完本篇文章，你能了解到一个简单的url解析框架的从0到1，为什么要做这个功能？调研了哪些方案？方案选型的思考？最小粒度的技术尝试？以及为了支持灵活的扩展与性能做出的框架的搭建过程。
当你明白这些之后，就像是自己从0到1的搭建了这个框架，还有谁会质疑？
完成该功能可体现出：

1. 爬虫框架jsoup的灵活运用，与chatGPT生成正则对工作的加成（解决问题的能力）
2. 参考Spring框架，写出一套适合自己场景的URL解析框架。体现你对源码的熟悉程度。与对大量设计模式的熟练度。
3. 对并发框架的熟悉程度，以及平常阅读技术博客的习惯，并能灵活运用。

你需要准备：

1. 多了解设计模式，以及它们可能会对应被提问的问题。
2. 提前把JUC学好，对你的工作，源码阅读，面试，都有很高性价比
3. 深入阅读Spring源码，或者其他框架代码，比如dubbo，netty

对应的学习路线与资源，可参考《阿斌Java之路》的知识库
