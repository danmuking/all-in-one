## 分布式系统面临的问题
复杂分布式体系结构中的应用程序有数十个依赖关系，每个依赖关系在某些时候将不可避免地失败。![download.png](https://raw.githubusercontent.com/danmuking/image/main/4d46b31596b1eabc60f9fcd035b298c3.png)
### 服务雪崩
多个微服务之间调用的时候，假设微服务A调用微服务B和微服务C，微服务B和微服务C又调用其它的微服务，这就是所谓的“扇出”。如果扇出的链路上某个微服务的调用响应时间过长或者不可用，对微服务A的调用就会占用越来越多的系统资源，进而引起系统崩溃，所谓的“雪崩效应”.
对于高流量的应用来说，单一的后端依赖可能会导致所有服务器上的所有资源都在几秒钟内饱和。比失败更糟糕的是，这些应用程序还可能导致服务之间的延迟增加，备份队列，线程和其他系统资源紧张，导致整个系统发生更多的级联故障。这些都表示需要对故障和延迟进行隔离和管理，以便单个依赖关系的失败，不能取消整个应用程序或系统。
所以，通常当你发现一个模块下的某个实例失败后，这时候这个模块依然还会接收流量，然后这个有问题的模块还调用了其他的模块，这样就会发生级联故障，或者叫雪崩。
## 我们的诉求
问题：
禁止服务雪崩故障
解决： 
- 有问题的节点，快速熔断（快速返回失败处理或者返回默认兜底数据【服务降级】）。
“断路器”本身是一种开关装置，当某个服务单元发生故障之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个符合预期的、可处理的备选响应(FallBack)，而不是长时间的等待或者抛出调用方无法处理的异常，这样就保证了服务调用方的线程不会被长时间、不必要地占用，从而避免了故障在分布式系统中的蔓延，乃至雪崩。
一句话，出故障了“保险丝”跳闸，别把整个家给烧了，😄
## 解决方案

1. 服务熔断
2. 服务降级
3. 服务限流
4. 服务限时
5. 服务预热
## CircuitBeaker是什么
circuit breaker是一套规范和接口，落地实现者是Resilience4J![download.png](https://raw.githubusercontent.com/danmuking/image/main/56c47830487d5fa3514e379332eebeb1.png)
### 实现原理
CircuitBreaker的目的是保护分布式系统免受故障和异常，提高系统的可用性和健壮性。
当一个组件或服务出现故障时，CircuitBreaker会迅速切换到开放OPEN状态(保险丝跳闸断电)，阻止请求发送到该组件或服务从而避免更多的请求发送到该组件或服务。这可以减少对该组件或服务的负载，防止该组件或服务进一步崩溃，并使整个系统能够继续正常运行。同时，CircuitBreaker还可以提高系统的可用性和健壮性，因为它可以在分布式系统的各个组件之间自动切换，从而避免单点故障的问题。
## ![download.png](https://raw.githubusercontent.com/danmuking/image/main/a03f39cd7b0a360397ec201d23fb1ed0.png)Resilience4J
### 是什么
![download.png](https://raw.githubusercontent.com/danmuking/image/main/9f008554baefa3b8df2c9e8740ec57a4.png)
### 能做什么
![download.png](https://raw.githubusercontent.com/danmuking/image/main/4f362eef96ec311b86c328743f00716c.png)
### 使用教程
#### 熔断
##### 状态之间的转换![download.png](https://raw.githubusercontent.com/danmuking/image/main/830f5d966017076d39d594eab43d89aa.png)
##### 配置餐数![download.png](https://raw.githubusercontent.com/danmuking/image/main/2dc73673897fff5f2a51389b8d9405c2.png)![download.png](https://raw.githubusercontent.com/danmuking/image/main/95c5595b43c7b42a14d57a2c22d4073f.png)![download.png](https://raw.githubusercontent.com/danmuking/image/main/6d3010f4916fb0944cb59b17261472f7.png)![download.png](https://raw.githubusercontent.com/danmuking/image/main/eca312642cc1e6f4abfd12874ba6acdc.png)![download.png](https://raw.githubusercontent.com/danmuking/image/main/6d4b55541041ee0f2ec0d264b57a0c60.png)
##### COUNT_BASED
###### controller
```java
package com.atguigu.cloud.controller;

import cn.hutool.core.util.IdUtil;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.TimeUnit;

/**
 * @auther zzyy
 * @create 2023-11-13 14:55
 */
@RestController
public class PayCircuitController
{
    //=========Resilience4j CircuitBreaker 的例子
    @GetMapping(value = "/pay/circuit/{id}")
    public String myCircuit(@PathVariable("id") Integer id)
    {
        if(id == -4) throw new RuntimeException("----circuit id 不能负数");
        if(id == 9999){
            try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
        }
        return "Hello, circuit! inputId:  "+id+" \t " + IdUtil.simpleUUID();
    }
}
```
###### FeignApi
```java
package com.atguigu.cloud.apis;

import com.atguigu.cloud.entities.PayDTO;
import com.atguigu.cloud.resp.ResultData;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;

/**
 * @auther zzyy
 * @create 2023-11-09 15:29
 */
@FeignClient(value = "cloud-payment-service")
public interface PayFeignApi
{
    /**
     * 新增一条支付相关流水记录
     * @param payDTO
     * @return
     */
    @PostMapping("/pay/add")
    public ResultData addPay(@RequestBody PayDTO payDTO);

    /**
     * 按照主键记录查询支付流水信息
     * @param id
     * @return
     */
    @GetMapping("/pay/get/{id}")
    public ResultData getPayInfo(@PathVariable("id") Integer id);

    /**
     * openfeign天然支持负载均衡演示
     * @return
     */
    @GetMapping(value = "/pay/get/info")
    public String mylb();

    /**
     * Resilience4j CircuitBreaker 的例子
     * @param id
     * @return
     */
    @GetMapping(value = "/pay/circuit/{id}")
    public String myCircuit(@PathVariable("id") Integer id);
}
```
###### POM
```java
<!--resilience4j-circuitbreaker-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
<!-- 由于断路保护等需要AOP实现，所以必须导入AOP包 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
###### YML
```java
server:
  port: 80

spring:
  application:
    name: cloud-consumer-openfeign-order
  ####Spring Cloud Consul for Service Discovery
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        prefer-ip-address: true #优先使用服务ip进行注册
        service-name: ${spring.application.name}
    openfeign:
      client:
        config:
          default:
            #cloud-payment-service:
            #连接超时时间，为避免演示出错，讲解完本次内容后设置为20秒
            connectTimeout: 20000
            #读取超时时间，为避免演示出错，讲解完本次内容后设置为20秒
            readTimeout: 20000
            #开启httpclient5
      httpclient:
        hc5:
          enabled: true
          #开启压缩特性
            compression:
        request:
          enabled: true
          min-request-size: 2048
          mime-types: text/xml,application/xml,application/json
        response:
          enabled: true
       # 开启circuitbreaker和分组激活 spring.cloud.openfeign.circuitbreaker.enabled
            circuitbreaker:
        enabled: true
        group:
          enabled: true #没开分组永远不用分组的配置。精确优先、分组次之(开了分组)、默认最后


# feign日志以什么级别监控哪个接口
logging:
  level:
    com:
      atguigu:
        cloud:
          apis:
            PayFeignApi: debug

# Resilience4j CircuitBreaker 按照次数：COUNT_BASED 的例子
#  6次访问中当执行方法的失败率达到50%时CircuitBreaker将进入开启OPEN状态(保险丝跳闸断电)拒绝所有请求。
#  等待5秒后，CircuitBreaker 将自动从开启OPEN状态过渡到半开HALF_OPEN状态，允许一些请求通过以测试服务是否恢复正常。
#  如还是异常CircuitBreaker 将重新进入开启OPEN状态；如正常将进入关闭CLOSE闭合状态恢复正常处理请求。
resilience4j:
  circuitbreaker:
    configs:
      default:
        failureRateThreshold: 50 #设置50%的调用失败时打开断路器，超过失败请求百分⽐CircuitBreaker变为OPEN状态。
                slidingWindowType: COUNT_BASED # 滑动窗口的类型
                slidingWindowSize: 6 #滑动窗⼝的⼤⼩配置COUNT_BASED表示6个请求，配置TIME_BASED表示6秒
                minimumNumberOfCalls: 6 #断路器计算失败率或慢调用率之前所需的最小样本(每个滑动窗口周期)。如果minimumNumberOfCalls为10，则必须最少记录10个样本，然后才能计算失败率。如果只记录了9次调用，即使所有9次调用都失败，断路器也不会开启。
                automaticTransitionFromOpenToHalfOpenEnabled: true # 是否启用自动从开启状态过渡到半开状态，默认值为true。如果启用，CircuitBreaker将自动从开启状态过渡到半开状态，并允许一些请求通过以测试服务是否恢复正常
                waitDurationInOpenState: 5s #从OPEN到HALF_OPEN状态需要等待的时间
                permittedNumberOfCallsInHalfOpenState: 2 #半开状态允许的最大请求数，默认值为10。在半开状态下，CircuitBreaker将允许最多permittedNumberOfCallsInHalfOpenState个请求通过，如果其中有任何一个请求失败，CircuitBreaker将重新进入开启状态。
                recordExceptions:
          - java.lang.Exception
    instances:
      cloud-payment-service:
        baseConfig: default
```
###### Controller
```java
package com.atguigu.cloud.controller;

import com.atguigu.cloud.apis.PayFeignApi;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import jakarta.annotation.Resource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

/**
 * @auther zzyy
 * @create 2023-11-13 14:54
 * Resilience4j CircuitBreaker 的例子
 */
@RestController
public class OrderCircuitController
{
    @Resource
    private PayFeignApi payFeignApi;

    @GetMapping(value = "/feign/pay/circuit/{id}")
    @CircuitBreaker(name = "cloud-payment-service", fallbackMethod = "myCircuitFallback")
    public String myCircuitBreaker(@PathVariable("id") Integer id)
    {
        return payFeignApi.myCircuit(id);
    }
    //myCircuitFallback就是服务降级后的兜底处理方法
        public String myCircuitFallback(Integer id,Throwable t) {
        // 这里是容错处理逻辑，返回备用结果
        return "myCircuitFallback，系统繁忙，请稍后再试-----/(ㄒoㄒ)/~~";
    }
}
```
##### TIME_BASED
![download.png](https://raw.githubusercontent.com/danmuking/image/main/ee1c170544f7fe73991fe7c4b1ea12aa.png)
###### YML
```java
server:
  port: 80

spring:
  application:
    name: cloud-consumer-openfeign-order
  ####Spring Cloud Consul for Service Discovery
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        prefer-ip-address: true #优先使用服务ip进行注册
        service-name: ${spring.application.name}
    openfeign:
      client:
        config:
          default:
            #cloud-payment-service:
            #连接超时时间，为避免演示出错，讲解完本次内容后设置为20秒
            connectTimeout: 20000
            #读取超时时间，为避免演示出错，讲解完本次内容后设置为20秒
            readTimeout: 20000
            #开启httpclient5
      httpclient:
        hc5:
          enabled: true
          #开启压缩特性
      compression:
        request:
          enabled: true
          min-request-size: 2048
          mime-types: text/xml,application/xml,application/json
        response:
          enabled: true
      #开启circuitbreaker和分组激活
      circuitbreaker:
        enabled: true
        group:
          enabled: true #没开分组永远不用分组的配置。精确优先、分组次之(开了分组)、默认最后


# feign日志以什么级别监控哪个接口
logging:
  level:
    com:
      atguigu:
        cloud:
          apis:
            PayFeignApi: debug

# Resilience4j CircuitBreaker 按照时间：TIME_BASED 的例子
resilience4j:
  timelimiter:
    configs:
      default:
        timeout-duration: 10s #神坑的位置，timelimiter 默认限制远程1s，超于1s就超时异常，配置了降级，就走降级逻辑
  circuitbreaker:
    configs:
      default:
        failureRateThreshold: 50 #设置50%的调用失败时打开断路器，超过失败请求百分⽐CircuitBreaker变为OPEN状态。
        slowCallDurationThreshold: 2s #慢调用时间阈值，高于这个阈值的视为慢调用并增加慢调用比例。
        slowCallRateThreshold: 30 #慢调用百分比峰值，断路器把调用时间⼤于slowCallDurationThreshold，视为慢调用，当慢调用比例高于阈值，断路器打开，并开启服务降级
        slidingWindowType: TIME_BASED # 滑动窗口的类型
        slidingWindowSize: 2 #滑动窗口的大小配置，配置TIME_BASED表示2秒
        minimumNumberOfCalls: 2 #断路器计算失败率或慢调用率之前所需的最小样本(每个滑动窗口周期)。
        permittedNumberOfCallsInHalfOpenState: 2 #半开状态允许的最大请求数，默认值为10。
        waitDurationInOpenState: 5s #从OPEN到HALF_OPEN状态需要等待的时间
        recordExceptions:
          - java.lang.Exception
    instances:
      cloud-payment-service:
        baseConfig: default 
```
#### 隔离
##### 是什么
bulkhead(船的)舱壁/(飞机的)隔板
隔板来自造船行业，床仓内部一般会分成很多小隔舱，一旦一个隔舱漏水因为隔板的存在而不至于影响其它隔舱和整体船。
![download.png](https://raw.githubusercontent.com/danmuking/image/main/da089dc90a7bca6772df8ac0f2dad5cc.png)
##### 作用
限制并发
##### SemaphoreBulkHead
###### 概述
基本上就是我们JUC信号灯内容的同样思想![download.png](https://raw.githubusercontent.com/danmuking/image/main/4791681056180827a1904f1d5a09e989.png)
信号量舱壁（SemaphoreBulkhead）原理
当信号量有空闲时，进入系统的请求会直接获取信号量并开始业务处理。
当信号量全被占用时，接下来的请求将会进入阻塞状态，SemaphoreBulkhead提供了一个阻塞计时器，
如果阻塞状态的请求在阻塞计时内无法获取到信号量则系统会拒绝这些请求。
若请求在阻塞计时内获取到了信号量，那将直接获取信号量并执行相应的业务处理。
###### Controller
```java
//=========Resilience4j bulkhead 的例子
@GetMapping(value = "/pay/bulkhead/{id}")
public String myBulkhead(@PathVariable("id") Integer id)
{
    if(id == -4) throw new RuntimeException("----bulkhead id 不能-4");

    if(id == 9999)
    {
        try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
    }

    return "Hello, bulkhead! inputId:  "+id+" \t " + IdUtil.simpleUUID();
```
###### FeignApi
```java
/**
 * Resilience4j Bulkhead 的例子
 * @param id
 * @return
 */
@GetMapping(value = "/pay/bulkhead/{id}")
public String myBulkhead(@PathVariable("id") Integer id);
```
###### POM
```java
<!--resilience4j-bulkhead-->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-bulkhead</artifactId>
</dependency>
```
###### YML
```java
server:
  port: 80

spring:
  application:
    name: cloud-consumer-openfeign-order
  ####Spring Cloud Consul for Service Discovery
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        prefer-ip-address: true #优先使用服务ip进行注册
        service-name: ${spring.application.name}
    openfeign:
      client:
        config:
          default:
          #cloud-payment-service:
            #连接超时时间，为避免演示出错，讲解完本次内容后设置为20秒
            connectTimeout: 20000
            #读取超时时间，为避免演示出错，讲解完本次内容后设置为20秒
            readTimeout: 20000
            #开启httpclient5
      httpclient:
        hc5:
          enabled: true
          #开启压缩特性
      compression:
        request:
          enabled: true
          min-request-size: 2048
          mime-types: text/xml,application/xml,application/json
        response:
          enabled: true
      #开启circuitbreaker和分组激活
      circuitbreaker:
        enabled: true
        group:
          enabled: true #没开分组永远不用分组的配置。精确优先、分组次之(开了分组)、默认最后


# feign日志以什么级别监控哪个接口
logging:
  level:
    com:
      atguigu:
        cloud:
          apis:
            PayFeignApi: debug

####resilience4j bulkhead 的例子
resilience4j:
  bulkhead:
    configs:
      default:
        maxConcurrentCalls: 2 # 隔离允许并发线程执行的最大数量
                maxWaitDuration: 1s # 当达到并发调用数量时，新的线程的阻塞时间，我只愿意等待1秒，过时不候进舱壁兜底fallback
    instances:
      cloud-payment-service:
        baseConfig: default
  timelimiter:
    configs:
      default:
        timeout-duration: 20s
```
###### Controller
```java
/**
 *(船的)舱壁,隔离
 * @param id
 * @return
 */
@GetMapping(value = "/feign/pay/bulkhead/{id}")
@Bulkhead(name = "cloud-payment-service",fallbackMethod = "myBulkheadFallback",type = Bulkhead.Type.SEMAPHORE)
public String myBulkhead(@PathVariable("id") Integer id)
{
    return payFeignApi.myBulkhead(id);
}
public String myBulkheadFallback(Throwable t)
{
    return "myBulkheadFallback，隔板超出最大数量限制，系统繁忙，请稍后再试-----/(ㄒoㄒ)/~~";
}

 
```
##### FixedThreadPoolBulkhead
###### 概述
基本上就是我们JUC-线程池内容的同样思想
![download.png](https://raw.githubusercontent.com/danmuking/image/main/068a8030ae7e3024a287aace5d23e888.png)
固定线程池舱壁（FixedThreadPoolBulkhead）
FixedThreadPoolBulkhead的功能与SemaphoreBulkhead一样也是用于限制并发执行的次数的，但是二者的实现原理存在差别而且表现效果也存在细微的差别。FixedThreadPoolBulkhead使用一个固定线程池和一个等待队列来实现舱壁。
当线程池中存在空闲时，则此时进入系统的请求将直接进入线程池开启新线程或使用空闲线程来处理请求。
当线程池中无空闲时时，接下来的请求将进入等待队列，
若等待队列仍然无剩余空间时接下来的请求将直接被拒绝，
在队列中的请求等待线程池出现空闲时，将进入线程池进行业务处理。
另外：ThreadPoolBulkhead只对CompletableFuture方法有效，所以我们必创建返回CompletableFuture类型的方法
###### YML
```java
server:
  port: 80

spring:
  application:
    name: cloud-consumer-openfeign-order
  ####Spring Cloud Consul for Service Discovery
  cloud:
    consul:
      host: localhost
      port: 8500
      discovery:
        prefer-ip-address: true #优先使用服务ip进行注册
        service-name: ${spring.application.name}
    openfeign:
      client:
        config:
          default:
            #cloud-payment-service:
            #连接超时时间，为避免演示出错，讲解完本次内容后设置为20秒
            connectTimeout: 20000
            #读取超时时间，为避免演示出错，讲解完本次内容后设置为20秒
            readTimeout: 20000
            #开启httpclient5
      httpclient:
        hc5:
          enabled: true
          #开启压缩特性
      compression:
        request:
          enabled: true
          min-request-size: 2048
          mime-types: text/xml,application/xml,application/json
        response:
          enabled: true
      #开启circuitbreaker和分组激活
      circuitbreaker:
        enabled: true
#        group:
#          enabled: true # 演示Bulkhead.Type.THREADPOOL时spring.cloud.openfeign.circuitbreaker.group.enabled

设为false新启线程和原来主线程脱离了。



# feign日志以什么级别监控哪个接口
logging:
  level:
    com:
      atguigu:
        cloud:
          apis:
            PayFeignApi: debug

####resilience4j bulkhead -THREADPOOL的例子
resilience4j:
  timelimiter:
    configs:
      default:
        timeout-duration: 10s #timelimiter默认限制远程1s，超过报错不好演示效果所以加上10秒
  thread-pool-bulkhead:
    configs:
      default:
        core-thread-pool-size: 1
        max-thread-pool-size: 1
        queue-capacity: 1
    instances:
      cloud-payment-service:
        baseConfig: default
# spring.cloud.openfeign.circuitbreaker.group.enabled 请设置为false 新启线程和原来主线程脱离
```
