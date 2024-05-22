## 介绍
随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Sentinel 是面向分布式、多语言异构化服务架构的流量治理组件，主要以流量为切入点，从流量路由、流量控制、流量整形、熔断降级、系统自适应过载保护、热点流量防护等多个维度来帮助开发者保障微服务的稳定性。
## 整合
### POM
```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.atguigu.cloud</groupId>
        <artifactId>mscloudV5</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>cloudalibaba-sentinel-service8401</artifactId>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>


    <dependencies>
        <!--SpringCloud alibaba sentinel -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
        <!--nacos-discovery-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <!-- 引入自己定义的api通用包 -->
        <dependency>
            <groupId>com.atguigu.cloud</groupId>
            <artifactId>cloud-api-commons</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <!--SpringBoot通用依赖模块-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!--hutool-->
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
        </dependency>
        <!--lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.28</version>
            <scope>provided</scope>
        </dependency>
        <!--test-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>


```
### YML
```java
server:
  port: 8401

spring:
  application:
    name: cloudalibaba-sentinel-service
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848         #Nacos服务注册中心地址
    sentinel:
      transport:
        dashboard: localhost:8080 #配置Sentinel dashboard控制台服务地址
        port: 8719 #默认8719端口，假如被占用会自动从8719开始依次+1扫描,直至找到未被占用的端口
```
### Controller
```java
package com.atguigu.cloud;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

/**
 * @auther zzyy
 * @create 2023-11-27 18:17
 */
@EnableDiscoveryClient
@SpringBootApplication
public class Main8401
{
    public static void main(String[] args)
    {
        SpringApplication.run(Main8401.class,args);
    }
}
```
### 业务类
```java
package com.atguigu.cloudalibaba.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @auther zzyy
 * @create 2023-05-24 15:35
 */
@RestController
public class FlowLimitController
{

    @GetMapping("/testA")
    public String testA()
    {
        return "------testA";
    }

    @GetMapping("/testB")
    public String testB()
    {
        return "------testB";
    }
}
```
## 流量控制
[cluster-flow-control | Sentinel](https://sentinelguard.io/zh-cn/docs/cluster-flow-control.html)
## 熔断规则
[cluster-flow-control | Sentinel](https://sentinelguard.io/zh-cn/docs/cluster-flow-control.html)
## @SentinelResource 
### 注解说明
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface SentinelResource {

    //资源名称
    String value() default "";

    //entry类型，标记流量的方向，取值IN/OUT，默认是OUT
    EntryType entryType() default EntryType.OUT;
    //资源分类
    int resourceType() default 0;

    //处理BlockException的函数名称,函数要求：
    //1. 必须是 public
    //2.返回类型 参数与原方法一致
    //3. 默认需和原方法在同一个类中。若希望使用其他类的函数，可配置blockHandlerClass ，并指定blockHandlerClass里面的方法。
    String blockHandler() default "";

    //存放blockHandler的类,对应的处理函数必须static修饰。
    Class<?>[] blockHandlerClass() default {};

    //用于在抛出异常的时候提供fallback处理逻辑。 fallback函数可以针对所
    //有类型的异常（除了 exceptionsToIgnore 里面排除掉的异常类型）进行处理。函数要求：
    //1. 返回类型与原方法一致
    //2. 参数类型需要和原方法相匹配
    //3. 默认需和原方法在同一个类中。若希望使用其他类的函数，可配置fallbackClass ，并指定fallbackClass里面的方法。
    String fallback() default "";

    //存放fallback的类。对应的处理函数必须static修饰。
    String defaultFallback() default "";

    //用于通用的 fallback 逻辑。默认fallback函数可以针对所有类型的异常进
    //行处理。若同时配置了 fallback 和 defaultFallback，以fallback为准。函数要求：
    //1. 返回类型与原方法一致
    //2. 方法参数列表为空，或者有一个 Throwable 类型的参数。
    //3. 默认需要和原方法在同一个类中。若希望使用其他类的函数，可配置fallbackClass ，并指定 fallbackClass 里面的方法。
    Class<?>[] fallbackClass() default {};
 

    //需要trace的异常
    Class<? extends Throwable>[] exceptionsToTrace() default {Throwable.class};

    //指定排除忽略掉哪些异常。排除的异常不会计入异常统计，也不会进入fallback逻辑，而是原样抛出。
    Class<? extends Throwable>[] exceptionsToIgnore() default {};
}
```
### 按照Rest地址限流+默认返回
#### 业务类
```java
package com.atguigu.cloudalibaba.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @auther zzyy
 * @create 2023-05-30 16:13
 */
@RestController
@Slf4j
public class RateLimitController
{
    @GetMapping("/rateLimit/byUrl")
    public String byUrl()
    {
        return "按rest地址限流测试OK";
    }
}

```
#### 配置![download.png](https://raw.githubusercontent.com/danmuking/image/main/a476340b5d738984568660805e0112ac.png)![download.png](https://raw.githubusercontent.com/danmuking/image/main/2d7773ece5c111018833268c8b6e025c.png)![download.png](https://raw.githubusercontent.com/danmuking/image/main/5fee5e25e3c17b6e7322e6cdf3839d69.png)
### 资源名称限流+自定义返回
#### 业务类
```java
package com.atguigu.cloudalibaba.controller;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @auther zzyy
 * @create 2023-05-30 16:13
 */
@RestController
@Slf4j
public class RateLimitController
{
    @GetMapping("/rateLimit/byUrl")
    public String byUrl()
    {
        return "按rest地址限流测试OK";
    }

    @GetMapping("/rateLimit/byResource")
    @SentinelResource(value = "byResourceSentinelResource",blockHandler = "handleException")
    public String byResource()
    {
        return "按资源名称SentinelResource限流测试OK";
    }
    public String handleException(BlockException exception)
    {
        return "服务不可用@SentinelResource启动"+"\t"+"o(╥﹏╥)o";
    }
}
 

 
```
#### 配置
![download.png](https://raw.githubusercontent.com/danmuking/image/main/84ce138137a6cedf7afbb2041124c14b.png)![download.png](https://raw.githubusercontent.com/danmuking/image/main/12fb7445ed7a8c46bc3f286c8b2239bb.png)
### 资源名称限流+自定义限流返回+服务降级
#### 业务类
```java
package com.atguigu.cloudalibaba.controller;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

/**
 * @auther zzyy
 * @create 2023-05-30 16:13
 */
@RestController
@Slf4j
public class RateLimitController
{
    @GetMapping("/rateLimit/byUrl")
    public String byUrl()
    {
        return "按rest地址限流测试OK";
    }

    @GetMapping("/rateLimit/byResource")
    @SentinelResource(value = "byResourceSentinelResource",blockHandler = "handleException")
    public String byResource()
    {
        return "按资源名称SentinelResource限流测试OK";
    }
    public String handleException(BlockException exception)
    {
        return "服务不可用@SentinelResource启动"+"\t"+"o(╥﹏╥)o";
    }

    @GetMapping("/rateLimit/doAction/{p1}")
    @SentinelResource(value = "doActionSentinelResource",
            blockHandler = "doActionBlockHandler", fallback = "doActionFallback")
    public String doAction(@PathVariable("p1") Integer p1) {
        if (p1 == 0){
            throw new RuntimeException("p1等于零直接异常");
        }
        return "doAction";
    }

    public String doActionBlockHandler(@PathVariable("p1") Integer p1,BlockException e){
        log.error("sentinel配置自定义限流了:{}", e);
        return "sentinel配置自定义限流了";
    }

    public String doActionFallback(@PathVariable("p1") Integer p1,Throwable e){
        log.error("程序逻辑异常了:{}", e);
        return "程序逻辑异常了"+"\t"+e.getMessage();
    }

}
 
```
#### 配置
![download.png](https://raw.githubusercontent.com/danmuking/image/main/6e90b632c2bb46a2915e6426383d834b.png)
## 热点规则
何为热点
热点即经常访问的数据，很多时候我们希望统计或者限制某个热点数据中访问频次最高的TopN数据，并对其访问进行限流或者其它操作
![download.png](https://raw.githubusercontent.com/danmuking/image/main/8dc26dabe7d27f33204ff6055b7946e1.png)
### 业务类
```java
@GetMapping("/testHotKey")
@SentinelResource(value = "testHotKey",blockHandler = "dealHandler_testHotKey")
public String testHotKey(@RequestParam(value = "p1",required = false) String p1, 

                         @RequestParam(value = "p2",required = false) String p2){
    return "------testHotKey";
}
public String dealHandler_testHotKey(String p1,String p2,BlockException exception)
{
    return "-----dealHandler_testHotKey";
}

 

 

 sentinel系统默认的提示：Blocked by Sentinel (flow limiting)

 
```
### 配置![download.png](https://raw.githubusercontent.com/danmuking/image/main/0dfcc1c70afd9fcca61895aee84c7547.png)
限流模式只支持QPS模式，固定写死了。（这才叫热点）
@SentinelResource注解的方法参数索引，0代表第一个参数，1代表第二个参数，以此类推
单机阀值以及统计窗口时长表示在此窗口时间超过阀值就限流。
上面的抓图就是第一个参数有值的话，1秒的QPS为1，超过就限流，限流后调用dealHandler_testHotKey支持方法。
## 授权规则
### 业务类
```java
package com.atguigu.cloud.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @auther zzyy
 * @create 2023-11-30 19:30
 */
@RestController
@Slf4j
public class EmpowerController //Empower授权规则，用来处理请求的来源
{
    @GetMapping(value = "/empower")
    public String requestSentinel4(){
        log.info("测试Sentinel授权规则empower");
        return "Sentinel授权规则";
    }
}
```
```java
package com.atguigu.cloud.handler;

import com.alibaba.csp.sentinel.adapter.spring.webmvc.callback.RequestOriginParser;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Component;

/**
 * @auther zzyy
 * @create 2023-11-30 19:33
 */
@Component
public class MyRequestOriginParser implements RequestOriginParser
{
    @Override
    public String parseOrigin(HttpServletRequest httpServletRequest) {
        return httpServletRequest.getParameter("serverName");
    }
}
 
```
### 配置
![download.png](https://raw.githubusercontent.com/danmuking/image/main/b69832f39ec9c37c34b8ebee173c5377.png)
## 持久化
### POM
```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.atguigu.cloud</groupId>
        <artifactId>mscloudV5</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <artifactId>cloudalibaba-sentinel-service8401</artifactId>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>


    <dependencies>
        <!--SpringCloud ailibaba sentinel-datasource-nacos -->
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
        <!--SpringCloud alibaba sentinel -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
        <!--nacos-discovery-->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <!-- 引入自己定义的api通用包 -->
        <dependency>
            <groupId>com.atguigu.cloud</groupId>
            <artifactId>cloud-api-commons</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <!--SpringBoot通用依赖模块-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!--hutool-->
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
        </dependency>
        <!--lombok-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.28</version>
            <scope>provided</scope>
        </dependency>
        <!--test-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```
### YML
```java
server:
  port: 8401

spring:
  application:
    name: cloudalibaba-sentinel-service #8401微服务提供者后续将会被纳入阿里巴巴sentinel监管
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848         #Nacos服务注册中心地址
    sentinel:
      transport:
        dashboard: localhost:8080 #配置Sentinel dashboard控制台服务地址
                port: 8719 #默认8719端口，假如被占用会自动从8719开始依次+1扫描,直至找到未被占用的端口
            web-context-unify: false # controller层的方法对service层调用不认为是同一个根链路
            datasource:
         ds1:
           nacos:
             server-addr: localhost:8848
             dataId: ${spring.application.name}
             groupId: DEFAULT_GROUP
             data-type: json
             rule-type: flow # com.alibaba.cloud.sentinel.datasource.RuleType

 
```
### nacos配置
![download.png](https://raw.githubusercontent.com/danmuking/image/main/31033cfa7ffb8634f670561db79ab020.png)resource：资源名称；
limitApp：来源应用；
grade：阈值类型，0表示线程数，1表示QPS；
count：单机阈值；
strategy：流控模式，0表示直接，1表示关联，2表示链路；
controlBehavior：流控效果，0表示快速失败，1表示Warm Up，2表示排队等待；
clusterMode：是否集群。
## 集成gateway
### 新建项目
新建一个gateway-sentinel9026模块，添加如下依赖：
```
xml
复制代码<!--nacos注册中心-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!--spring cloud gateway-->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <!--    spring cloud gateway整合sentinel的依赖-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
    </dependency>

    <!--    sentinel的依赖-->
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
```
**注意**：这依然是一个网关服务，不要添加WEB的依赖
### 配置文件
配置文件中主要指定以下三种配置：

- nacos的地址
- sentinel控制台的地址
- 网关路由的配置

配置如下：
```
yaml
复制代码spring:
  cloud:
    ## 整合sentinel，配置sentinel控制台的地址
    sentinel:
      transport:
        ## 指定控制台的地址，默认端口8080
        dashboard: localhost:8080
    nacos:
      ## 注册中心配置
      discovery:
        # nacos的服务地址，nacos-server中IP地址:端口号
        server-addr: 127.0.0.1:8848
    gateway:
      ## 路由
      routes:
        ## id只要唯一即可，名称任意
        - id: gateway-provider
          uri: lb://gateway-provider
          ## 配置断言
          predicates:
            ## Path Route Predicate Factory断言，满足/gateway/provider/**这个请求路径的都会被路由到http://localhost:9024这个uri中
            - Path=/gateway/provider/**
```
上述配置中设置了一个路由gateway-provider，只要请求路径满足/gateway/provider/**都会被路由到gateway-provider这个服务中。
### 限流配置
经过上述两个步骤其实已经整合好了Sentinel，此时访问一下接口：[http://localhost:9026/gateway/provider/port](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A9026%2Fgateway%2Fprovider%2Fport)
然后在sentinel控制台可以看到已经被监控了，监控的路由是gateway-provider，如下图：
![](https://raw.githubusercontent.com/danmuking/image/main/f8cd4dd5f3dd52729b6e35def60c105f.webp)
此时我们可以为其新增一个route维度的限流，如下图：
![](https://raw.githubusercontent.com/danmuking/image/main/f1eef76b69add4fee32c3418b15a7382.webp)
上图中对gateway-provider这个路由做出了限流，QPS阈值为1。
此时快速访问：[http://localhost:9026/gateway/provider/port，看到已经被限流了，如下图：](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A9026%2Fgateway%2Fprovider%2Fport%25EF%25BC%258C%25E7%259C%258B%25E5%2588%25B0%25E5%25B7%25B2%25E7%25BB%258F%25E8%25A2%25AB%25E9%2599%2590%25E6%25B5%2581%25E4%25BA%2586%25EF%25BC%258C%25E5%25A6%2582%25E4%25B8%258B%25E5%259B%25BE%25EF%25BC%259A)
![](https://raw.githubusercontent.com/danmuking/image/main/6b2920b4d056203ced027ca475bc626a.webp)
以上route维度的限流已经配置成功，小伙伴可以自己照着上述步骤尝试一下。
API分组限流也很简单，首先需要定义一个分组，**API管理-> 新增API分组**，如下图：
![](https://raw.githubusercontent.com/danmuking/image/main/9a4af73f434772cbaa367c4097dbfa69.webp)
匹配模式选择了精确匹配（还有前缀匹配，正则匹配），因此只有这个uri：http://xxxx/gateway/provider/port会被限流。
第二步需要对这个分组添加流控规则，**流控规则->新增网关流控**，如下图：
![](https://raw.githubusercontent.com/danmuking/image/main/e432623845089b4e4dac00e000ca46fa.webp)
API名称那里选择对应的分组即可，新增之后，限流规则就生效了。
陈某不再测试了，小伙伴自己动手测试一下吧...............
陈某这里只是简单的配置一下，至于限流规则持久化一些内容请看陈某的Sentinel文章，这里就不再过多的介绍了。
### 如何自定义限流异常信息？
从上面的演示中可以看到默认的异常返回信息是："Block........."，这种肯定是客户端不能接受的，因此需要定制自己的异常返回信息。
下面介绍两种不同的方式定制异常返回信息，开发中自己选择其中一种。
#### 直接配置文件中定制
开发者可以直接在配置文件中直接修改返回信息，配置如下：
```
yaml
复制代码spring:
  cloud:
    ## 整合sentinel，配置sentinel控制台的地址
    sentinel:
      #配置限流之后，响应内容
      scg:
        fallback:
          ## 两种模式，一种是response返回文字提示信息，
          ## 一种是redirect，重定向跳转，需要同时配置redirect(跳转的uri)
          mode: response
          ## 响应的状态
          response-status: 200
          ## 响应体
          response-body: '{"code": 200,"message": "请求失败，稍后重试！"}'
```
上述配置中mode配置的是response，一旦被限流了，将会返回JSON串。
```
json
复制代码{
    "code": 200,
    "message": "请求失败，稍后重试！"
}
```
**重定向**的配置如下：
```
yaml
复制代码spring:
  cloud:
    ## 整合sentinel，配置sentinel控制台的地址
    sentinel:
      #配置限流之后，响应内容
      scg:
        fallback:
          ## 两种模式，一种是response返回文字提示信息，一种是redirect，重定向跳转，需要同时配置redirect(跳转的uri)
          mode: redirect
          ## 跳转的URL
          redirect: http://www.baidu.com
```
一旦被限流，将会直接跳转到：[www.baidu.com](https://link.juejin.cn?target=http%3A%2F%2Fwww.baidu.com)
#### 编码定制
这种就不太灵活了，通过硬编码的方式，完整代码如下：
```
java
复制代码@Configuration
public class GatewayConfig {
    /**
     * 自定义限流处理器
     */
    @PostConstruct
    public void initBlockHandlers() {
        BlockRequestHandler blockHandler = (serverWebExchange, throwable) -> {
            Map map = new HashMap();
            map.put("code",200);
            map.put("message","请求失败，稍后重试！");
            return ServerResponse.status(HttpStatus.OK)
                    .contentType(MediaType.APPLICATION_JSON_UTF8)
                    .body(BodyInserters.fromObject(map));
        };
        GatewayCallbackManager.setBlockHandler(blockHandler);
    }
}
```
两种方式介绍完了，根据业务需求自己选择适合的方式，当然陈某更喜欢第一种，理由：**约定>配置>编码**。
## 集成OpenFeign
### 配置文件
配置文件 application.yml ：
```
yaml
复制代码spring:
  application:
    name: open-feign-service

  cloud:
    nacos:
      discovery:
        server-addr: 192.168.242.112:81
    sentinel:
      transport:
        dashboard: localhost:8080
        port: 8719
      # https://github.com/alibaba/Sentinel/issues/1213
      web-context-unify: false
      # Sentinel 规则持久化到 Nacos
      datasource:
        rule1:
          nacos:
            serverAddr: 192.168.242.112:81
            groupId: DEFAULT_GROUP
            dataId: sentinelFlowRule.json
            ruleType: flow
            
feign:
  client:
    config:
      # 默认的超时时间设置
      default:
        connectTimeout: 5000
        readTimeout: 5000
      # 在指定的 FeignClient 设置超时时间，覆盖默认的设置
      nacos-provider:
        connectTimeout: 1000
        readTimeout: 1000
        loggerLevel: full
  # 激活 Sentinel
  sentinel:
    enabled: true
```
这里增加了 Sentinel 的数据持久化内容，以及激活 OpenFeign 与 Sentinel 联合使用的 feign.sentinel.enabled=true 配置。
### 全局统一异常处理
不管是 Sentinel 限流后返回，还是 OpenFeign 的 fallback 返回，本质上他们都是出现异常了，这里配置一下全局的统一异常处理。
首先，增加一个业务异常类：
```
public class BusinessException extends RuntimeException {

    private String code;

    private String message;

    public BusinessException(String code, String message) {
        this.code = code;
        this.message = message;
    }

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    @Override
    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```
然后使用 **Spring** 的 **@RestControllerAdvice** 注解进行全局的异常进行处理：
```
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger LOGGER = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    /**
     * 业务异常，统一处理
     * @param e 异常对象
     * @return ResponseResult 全局异常响应
     */
    @ExceptionHandler(BusinessException.class)
    public ResponseResult<String> businessException(BusinessException e) {
        LOGGER.info("code={}, message={}", e.getCode(), e.getMessage());
        return ResponseResult.fail(e.getCode(), e.getMessage());
    }

    // 其他异常...
}
```
这样，只要指定了抛出的异常类型，就会返回统一的响应格式。
