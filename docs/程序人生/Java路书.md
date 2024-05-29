### Java 编程基础（必须掌握！！！）
需要掌握的知识包括：
#### Java基础

- Java 特点
- **Java 基础语法**
   - **数据类型**
   - **流程控制**
- **数组**
- **面向对象**
   - **方法**
   - **重载**
   - **封装**
   - **继承**
   - **多态**
- **抽象类**
- **接口**
- 枚举
- **常用类**
   - **String**
   - 日期时间
- **集合类**
- 泛型
- **注解**
- **异常处理**
- **多线程**
- IO 流
- **反射**
- Stream API
- Lambda 表达式
- 新日期时间 API
- 接口默认方法
##### 学习建议

1. **坚持学习**：如果完全是一个新手，初学一门语言时，必须持续学习，不能中断！如果你已经学习过其他语言，这一块可以适当放松，只需要了解基本语法，在实践中逐步深入
2. **多加练习**：要学好编程，就必须多敲代码！建议先跟着书上的例子敲一遍，然后尝试自主编写代码，并完成课后练习。
3. **理解代码**：即使暂时不理解代码也没关系，可以学会使用 Debug，一行一行地打断点执行，查看程序的执行过程。这会帮助你节省很多重复学习的时间。
4. **别再说Java8是新特性了**，求求了，4202年了，Java已经更新到22了，还有人在简历上写掌握Java8是新特性，面试官看了都不知道怎么吐槽。
##### 相关资源

- 视频
   - 韩顺平 - 零基础 30 天学会 Java：[https://www.bilibili.com/video/BV1fh411y7R8](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1fh411y7R8) 
- 文档
   - 菜鸟教程：[https://www.runoob.com/java/java-tutorial.html](https://gitee.com/link?target=https%3A%2F%2Fwww.runoob.com%2Fjava%2Fjava-tutorial.html)
   - 廖雪峰 Java 教程：[https://www.liaoxuefeng.com/wiki/1252599548343744](https://gitee.com/link?target=https%3A%2F%2Fwww.liaoxuefeng.com%2Fwiki%2F1252599548343744)
   - 国外的优质教程：[https://www.geeksforgeeks.org/java/](https://www.geeksforgeeks.org/java/)
#### JVM

- **JVM 内存结构**
- **JVM 生命周期**
- **主流虚拟机**
- Java 代码执行流程
- **类加载**
   - **类加载器**
   - **类加载过程**
   - **双亲委派机制**
- **垃圾回收**
   - **垃圾回收器**
   - **垃圾回收策略**
   - **垃圾回收算法**
   - **StopTheWorld**
- 字节码
- 内存分配和回收
- JVM 性能调优
   - 性能分析方法
   - 常用工具
   - 参数设置
- Java 探针
- 线上故障分析
##### 学习建议
这一块在面试中也是**考察的频率极高**，但是现实中**使用的频率极低**，所以我建议，这一块之间面向面试背诵即可，**不需要花费太多的尽力去实操。**
##### 资源

- 视频
   - 尚硅谷宋红康 - JVM 全套教程详解：[https://www.bilibili.com/video/BV1PJ411n7xZ](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1PJ411n7xZ) （讲得相当全面！附有实操）
   - 【狂神说Java】JVM快速入门篇：[https://www.bilibili.com/video/BV1iJ411d7jS](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1iJ411d7jS) （讲得有点浅，但都是面试重点，时间紧的小伙伴可以直接看这个）
- 博客：
   - Java全栈知识体系 [https://pdai.tech/md/java/jvm/java-jvm-struct.html](https://pdai.tech/md/java/jvm/java-jvm-struct.html)
   - All In One [https://github.com/danmuking/all-in-one](https://github.com/danmuking/all-in-one) （夹带一下私货）
#### 并发编程

- **线程和进程**
- **线程状态**
- **并行和并发**
- **同步和异步**
- **Synchronized**
- **Volatile 关键字**
- Lock 锁
- **死锁**
- **可重入锁**
- 线程安全
- **线程池**
- JUC 的使用
- **AQS**
- Fork Join
- **CAS**
##### 学习建议
并发编程入门不难，依然是 **先学会使用** 基础的 Java 并发包，这部分在实践中同样较少使用，主要以调用api为主，因此这块仍然以理论学习为主，需要注意的是，现在有些公司会在面试时手撕简单的多线程场景题，比如多个线程交替打印，这类题目通常也不会太难，大家只要有所防备即可。
##### 资源

- 视频
   - 【尚硅谷】大厂必备技术之JUC并发编程2021最新版：[https://www.bilibili.com/video/BV1Kw411Z7dF](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1Kw411Z7dF) 
   - 黑马程序员全面深入学习Java并发编程：[https://www.bilibili.com/video/BV16J411h7Rd](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV16J411h7Rd)
- 博客
   - Java并发知识体系详解 [https://pdai.tech/md/java/thread/java-thread-x-overview.html](https://pdai.tech/md/java/thread/java-thread-x-overview.html)
   - All In One [https://github.com/danmuking/all-in-one](https://github.com/danmuking/all-in-one)
### Java常见框架
##### Spring 5

- 描述：Java 轻量级应用框架
- **IOC**
- **AOP**
- **事务**
##### SpringMVC

- 描述：Java 轻量级 web 开发框架
- **什么是 MVC？**
- 请求与响应
- Restful API
- 拦截器
- 配置
- **执行过程**
##### MyBatis

- 描述：数据访问框架，操作数据库进行增删改查等操作
- 增删改查
- 全局配置
- 动态 SQL
- 缓存
- 和其他框架的整合
- 逆向工程
##### SpringBoot 2

- 描述：简化 Spring 应用的初始搭建以及开发过程，提高效率
- **常用注解**
- **自动装配**
- 资源整合
- 高级特性
- 本地热部署
### MySQL 数据库
详情请参考学习路线：[数据库学习路线](https://gitee.com/link?target=https%3A%2F%2Fbcdh.yuque.com%2Fstaff-wpxfif%2Fresource%2Fdpikl6npld34ydll%3Fview%3Ddoc_embed)
企业中大部分业务数据都是用关系型数据库存储的，因此数据库是后台开发同学的必备技能，其中 MySQL 数据库是目前的主流，也是面试时的重点。
#### 知识

- 基本概念
- MySQL 搭建
- SQL 语句编写
- 约束
- 索引
- 事务
- 锁机制
- 设计数据库表
- 性能优化
#### 学习建议
其中，**SQL 语句编写** 和 **设计数据库表** 这两个能力一定要有！
比如让你做一个学生管理系统，你要能想到需要哪些表，比如学生表、班级表；每个表需要哪些字段、字段类型。
这就要求大家多写 SQL、多根据实际的业务场景去练习设计能力。
#### 经典面试题

1. MySQL 索引的最左原则
2. InnoDB 和 MyIsam 引擎的区别？
3. 有哪些优化数据库性能的方法？
4. 如何定位慢查询？
5. MySQL 支持行锁还是表锁？分别有哪些优缺点？
#### 资源

- 视频
   - ⭐ 2022 黑马 MySQL 教程：[https://www.bilibili.com/video/BV1Kr4y1i7ru](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1Kr4y1i7ru)（倾向于速成，初学只看完 P57 节前的基础篇即可，后面可以再来补进阶知识）
   - 老杜 - mysql入门基础 + 数据库实战：[https://www.bilibili.com/video/BV1Vy4y1z7EX](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1Vy4y1z7EX) （内容相对精炼，有习题）
   - 尚硅谷 - MySQL基础教程：[https://www.bilibili.com/video/BV1xW411u7ax](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1xW411u7ax) （小姐姐讲课，但感觉音质一般）
- 在线练习
   - ⭐ 鱼皮的闯关式 SQL 自学网：[http://sqlmother.yupi.icu/](https://gitee.com/link?target=http%3A%2F%2Fsqlmother.yupi.icu%2F)
   - ⭐ SQL 在线运行：[https://www.bejson.com/runcode/sql/](https://gitee.com/link?target=https%3A%2F%2Fwww.bejson.com%2Fruncode%2Fsql%2F)
- 文档
   - SQL - 菜鸟教程：[https://www.runoob.com/sql/sql-tutorial.html](https://gitee.com/link?target=https%3A%2F%2Fwww.runoob.com%2Fsql%2Fsql-tutorial.html)
   - MySQL - 菜鸟教程：[https://www.runoob.com/mysql/mysql-tutorial.html](https://gitee.com/link?target=https%3A%2F%2Fwww.runoob.com%2Fmysql%2Fmysql-tutorial.html)
- 网站
   - [数据库大全](https://gitee.com/link?target=https%3A%2F%2Fwww.code-nav.cn%2Frd%2F%3Frid%3Db00064a76012546b016e274a3724c5f0)：果创云收录的各种数据库表设计
### Redis（14 天）
详情请参考学习路线：[Redis 学习路线](https://gitee.com/link?target=https%3A%2F%2Fbcdh.yuque.com%2Fbooks%2Fshare%2F2dd2567c-a826-4d9d-9303-bd288269e874%2Fqunv5d)
缓存是高并发系统不可或缺的技术，可以提高系统的性能和并发，而 Redis 是实现缓存的最主流技术，因此它是后台开发必学的知识点，也是面试重点。
#### 知识

- Redis 基础
- 什么是缓存？
- 本地缓存
   - Caffeine 库
- 多级缓存
- Redis 分布式缓存
   - 数据类型
   - 常用操作
   - Java 操作 Redis
      - Spring Boot Redis Template
      - Redisson
   - 主从模型搭建
   - 哨兵集群搭建
   - 日志持久化
- 缓存（Redis）应用场景
   - 数据共享
   - 单点登录
   - 计数器
   - 限流
   - 点赞
   - 实时排行榜
   - 分布式锁
- 缓存常见问题
   - 缓存雪崩
   - 缓存击穿
   - 缓存穿透
   - 缓存更新一致性
- 相关技术：Memcached、Ehcache
#### 学习建议
学会如何简单地使用缓存并不难，和数据库类似，无非就是调用 API 对数据进行增删改查。
因此，建议先能够独立使用它，了解缓存的应用场景；再学习如何在 Java 中操作缓存中间件，并尝试和项目相结合，提高系统的性能。
跟着视频教程实操一遍即可，可以等到面试前再去深入了解原理和高级特性。
#### 经典面试题

1. Redis 为什么快？
2. Redis 有哪些常用的数据结构？
3. Redis RDB 和 AOF 持久化的区别，如何选择？
4. 如何解决缓存击穿、缓存穿透、雪崩问题？
5. 如何用 Redis 实现点赞功能，怎么设计 Key / Value？
#### 资源

- 项目
   - [项目实战 - 鱼皮原创项目教程系列](https://gitee.com/link?target=https%3A%2F%2Fyuyuanweb.feishu.cn%2Fwiki%2FSePYwTc9tipQiCktw7Uc7kujnCd) 中的伙伴匹配系统、智能 BI 项目都运用了 Redis 解决实际问题，推荐学习
- 视频
   - ⭐ 2022 黑马 Redis 从基础到原理：[https://www.bilibili.com/video/BV1cr4y1671t](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1cr4y1671t)（结合项目去讲，强烈推荐）
   - 尚硅谷 - 2021 最新 Redis 6 入门到精通教程：[https://www.bilibili.com/video/BV1Rv41177Af](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1Rv41177Af) （基于 Redis 6 的，推荐）
- 文档
   - Redis 命令参考：[http://redisdoc.com/](https://gitee.com/link?target=http%3A%2F%2Fredisdoc.com%2F)
   - Redis 面试题整理：[https://github.com/lokles/Web-Development-Interview-With-Java/blob/main/Redis问题.md](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Flokles%2FWeb-Development-Interview-With-Java%2Fblob%2Fmain%2FRedis%25E9%2597%25AE%25E9%25A2%2598.md)
- 书籍
   - 《Redis 实战》（经典）
- 工具
   - ⭐ Redis 在线练习：[https://try.redis.io/](https://gitee.com/link?target=https%3A%2F%2Ftry.redis.io%2F) （强烈推荐）
### 消息队列（14 天）
消息队列是用于传输和保存消息的容器，也是大型分布式系统中常用的技术，主要解决应用耦合、异步消息、流量削锋等问题。后台开发必学，也是面试重点。
#### 知识

- 消息队列的作用
- RabbitMQ 消息队列
   - 生产消费模型
   - 交换机模型
   - 死信队列
   - 延迟队列
   - 消息持久化
   - Java 操作
   - 集群搭建
- 相关技术：Kafka、ActiveMQ、TubeMQ、RocketMQ
#### 学习建议
和缓存一样，学会如何使用消息队列并不难，无非就是调用 API 去生产、转发和消费消息。
因此，建议先能够独立使用它，了解消息队列的应用场景；再学习如何在 Java 中操作消息队列中间件，并尝试和项目相结合，感受消息队列带来的好处。
这里我建议初学者先学习 RabbitMQ，比 Kafka 要好理解一些。跟着视频教程实操一遍即可，可以等到面试前再去深入了解原理和高级特性。
#### 经典面试题

1. 使用消息队列有哪些优缺点？
2. 如何保证消息消费的幂等性？
3. 消息队列有哪些路由模型？
4. 你是否用过消息队列，解决过什么问题？
#### 资源

- 项目
   - [项目实战 - 鱼皮原创项目教程系列](https://gitee.com/link?target=https%3A%2F%2Fyuyuanweb.feishu.cn%2Fwiki%2FSePYwTc9tipQiCktw7Uc7kujnCd) 中的智能 BI 项目、在线判题系统都运用了消息队列解决实际问题，推荐学习
- 视频
   - ⭐️ 2023 黑马 RabbitMQ 消息队列教程：[https://www.bilibili.com/video/BV1Xm4y1i7HP](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1Xm4y1i7HP)（适合快速入门）
   - ⭐ 尚硅谷 - 2021 最新 RabbitMQ 教程：[https://www.bilibili.com/video/BV1cb4y1o7zz](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1cb4y1o7zz) （更加全面）
- 文档
   - RabbitMQ 中文文档：[http://rabbitmq.mr-ping.com/](https://gitee.com/link?target=http%3A%2F%2Frabbitmq.mr-ping.com%2F)
- 书籍
   - ⭐️ 编程导航原创 Rocket MQ 消息队列专栏：[https://yuyuanweb.feishu.cn/wiki/R5mbwIMwCi9xkmkrpyOcp1pzn9b](https://gitee.com/link?target=https%3A%2F%2Fyuyuanweb.feishu.cn%2Fwiki%2FR5mbwIMwCi9xkmkrpyOcp1pzn9b)
   - 《RabbitMQ 实战：高效部署分布式消息队列》（经典）
- 工具
   - ⭐ RabbitMQ 在线模拟器：[http://tryrabbitmq.com/](https://gitee.com/link?target=http%3A%2F%2Ftryrabbitmq.com%2F)
### 微服务（60 天）
随着互联网的发展，项目越来越复杂，单机且庞大的巨石项目已无法满足开发、运维、并发、可靠性等需求。
因此，后台架构不断演进，可以将庞大的项目拆分成一个个职责明确、功能独立的细小模块，模块可以部署在多台服务器上，相互配合协作，提供完整的系统能力。
换言之，想做大型项目，这块儿一定要好好学！
#### 知识
##### Dubbo

- 架构演进
- RPC
- Zookeeper
- 服务提供者
- 服务消费者
- 项目搭建
- 相关技术：DubboX（对 Dubbo 的扩展）
##### 🌖 微服务

- 微服务概念
- Spring Cloud 框架
   - 子父工程
   - 服务注册与发现
   - 注册中心 Eureka、Zookeeper、Consul
   - Ribbon 负载均衡
   - Feign 服务调用
   - Hystrix 服务限流、降级、熔断
   - Resilience4j 服务容错
   - Gateway（Zuul）微服务网关
   - Config 分布式配置中心
   - 分布式服务总线
   - Sleuth + Zipkin 分布式链路追踪
- Spring Cloud Alibaba
   - Nacos 注册、配置中心
   - OpenFeign 服务调用
   - Sentinel 流控
   - Seata 分布式事务
##### 接口管理

- Swagger 接口文档
- Postman 接口测试
- 相关技术：YApi、ShowDoc
#### 学习建议
时间不急的话，建议先从 Dubbo 学起，对分布式、RPC、微服务有些基本的了解，再去食用 Spring Cloud 全家桶会更香。学完 Spring Cloud 全家桶后，再去学 Spring Cloud Alibaba 就很简单了。
这部分内容的学习，原理 + 实践都很重要，也不要被各种高大上的词汇唬住了，都是上层（应用层）的东西，基本没有什么算法，跟着视频教程学，其实还是很好理解的。
分布式相关知识非常多，但这里不用刻意去背，先通过视频教程实战使用一些微服务框架，也能对其中的概念有基本的了解。
大厂面试的时候很少问 Spring Cloud 框架的细节，更多的是微服务以及各组件的一些思想，比如网关的好处、消息总线的好处等。
#### 经典面试题

1. 什么是微服务，有哪些优缺点？
2. 什么是注册中心，能解决什么问题？
#### 资源

- 项目实战
   - [项目实战 - 鱼皮原创项目教程系列](https://gitee.com/link?target=https%3A%2F%2Fyuyuanweb.feishu.cn%2Fwiki%2FSePYwTc9tipQiCktw7Uc7kujnCd) 中的 API 开放平台、在线判题系统都运用了微服务，推荐学习
- 视频
   - ⭐️ 黑马 Spring Cloud 视频教程：[https://www.bilibili.com/video/BV1kH4y1S7wz](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1kH4y1S7wz)（11 小时，非常凝练，适合快速入门）
   - ⭐️ 尚硅谷 Dubbo 教程：[https://www.bilibili.com/video/BV1ns411c7jV](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1ns411c7jV)
   - 尚硅谷 SpringCloud（H版&alibaba）框架开发教程（微服务分布式架构）：[https://www.bilibili.com/video/BV18E411x7eT](https://gitee.com/link?target=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV18E411x7eT) （把国外的 Spring Cloud 和国内的 Spring Cloud Alibaba 结合在一起对比着去讲，主流技术栈、知识点都讲到了，内容更全面）
- 文档
   - Apache Dubbo 官方文档：[https://dubbo.apache.org/zh/](https://gitee.com/link?target=https%3A%2F%2Fdubbo.apache.org%2Fzh%2F)
   - Spring Cloud Alibaba 官方文档：[https://github.com/alibaba/spring-cloud-alibaba/blob/master/README-zh.md](https://gitee.com/link?target=https%3A%2F%2Fgithub.com%2Falibaba%2Fspring-cloud-alibaba%2Fblob%2Fmaster%2FREADME-zh.md)
   - ⭐ Swagger 教学文档：[https://doc.xiaominfo.com/](https://gitee.com/link?target=https%3A%2F%2Fdoc.xiaominfo.com%2F) （跟着快速开始直接用就好了）
