

## Sentinel简介

### 背景分析

在我们日常生活中，经常会在淘宝、天猫、京东、拼多多等平台上参与商品的秒杀、抢购以及一些优惠活动，也会在节假日使用12306 手机APP抢火车票、高铁票，甚至有时候还要帮助同事、朋友为他们家小孩拉投票、刷票，这些场景都无一例外的会引起服务器流量的暴涨，导致网页无法显示、APP反应慢、功能无法正常运转，甚至会引起整个网站的崩溃。我们如何在这些业务流量变化无常的情况下，保证各种业务安全运营，系统在任何情况下都不会崩溃呢？我们可以在系统负载过高时，采用限流、降级和熔断，三种措施来保护系统，由此一些流量控制中间件诞生。例如Sentinel。

### Sentinel概述

Sentinel \(分布式系统的流量防卫兵\) 是阿里开源的一套用于服务容错的综合性解决方案。它以流量为切入点, 从流量控制、熔断降级、系统负载保护等多个维度来保护服务的稳定性。Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景, 例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。Sentinel核心分为两个部分:

- 核心库（Java 客户端）：能够运行于所有 Java 运行时环境，同时对Dubbo /Spring Cloud 等框架也有较好的支持。
- 控制台（Dashboard）：基于 Spring Boot 开发，打包后可以直接运行。

### 安装Sentinel服务

Sentinel 提供一个轻量级的控制台, 它提供机器发现、单机资源实时监控以及规则管理等功能，其控制台安装步骤如下：第一步：打开sentinel下载网址

> https://github.com/alibaba/Sentinel/releases

第二步：下载Jar包（可以存储到一个sentinel目录），如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZBorJjl2yibDt0HwIIhBicjwsH7eGb5dtxibpTN2fPagm4N4zB5lpqa4oA/640?wx_fmt=png)

第三步：在sentinel对应目录，打开命令行\(cmd\)，启动运行sentinel

```
java -Dserver.port=8180 -Dcsp.sentinel.dashboard.server=localhost:8180 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.1.jar
```

检测启动过程，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZSP23DdzOKalRLRc18VlBL3O8KTZme6L6NCxAOgVsCLXrhqI3sQtAOA/640?wx_fmt=png)

### 访问Sentinal服务

第一步：假如Sentinal启动ok，通过浏览器进行访问测试，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZC79Bq572jvr9ZKBE3icS678EeTSnT1gKuBFOpc2XlQy8PV4PFVohS4g/640?wx_fmt=png)

第二步：登陆sentinel,默认用户和密码都是sentinel,登陆成功以后的界面如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZdrIsAqYhiciaV2FaoQEyhrZ1AgVVYdNOwvL8kSaiba76lnpjJZ1yiaDTZw/640?wx_fmt=png)

## Sentinel限流入门

### 概述

我们系统中的数据库连接池，线程池，nginx的瞬时并发等在使用时都会给定一个限定的值，这本身就是一种限流的设计。[限流](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247509214&idx=2&sn=59fab89e05bb33e5128b82a75e98f3d7&chksm=fc79c970cb0e406688196599d6f1e6995e983406e4f19dae5814c258de0abed5780a39cbd98c&scene=21#wechat_redirect)的目的防止恶意请求流量、恶意攻击，或者防止流量超过系统峰值。

### 准备工作

第一步：Sentinel 应用于服务提供方\(sca-provider\)，在消费方添加依赖如下：

```
<dependency>
  <groupId>com.alibaba.cloud</groupId>    
  <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

第二步：打开服务消费方配置文件bootstrap.yml，添加sentinel配置，代码如下：

```
spring:  
  cloud:    
    sentinel:      
      transport:         
        dashboard: localhost:8180 # 指定sentinel控制台地址。
```

第三步：创建一个用于演示限流操作的Controller对象，例如：

```
package com.jt.provider.controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/provider")
public class ProviderSentinelController {       

  @GetMapping("/sentinel01")       
  public String doSentinel01(){           
      return "sentinel 01 test  ...";       
  }

}
```

第四步：启动sca-provider服务，然后对指定服务进行访问，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZnTumVuDMGBvJDTkzdicZ4Z8ib7lSjmWVFQYdMjmxsqfQDZMrJPHgylhA/640?wx_fmt=png)

第五步：刷新sentinel 控制台，实时监控信息，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZyCZQTwpGuDVNa0YJmicdylepwOJ8ZoQAwibAfvmWA5ibGWAkSeNWAyqmA/640?wx_fmt=png)

Sentinel的控制台其实就是一个SpringBoot编写的程序，我们需要将我们的服务注册到控制台上，即在[微服务中](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247506367&idx=1&sn=01ae2f5b95c54e0a61a0041338fb6650&chksm=fc79b411cb0e3d07e9e53870606767752df8474ccd8ba4fe159c4d686af4eee00730170e0f6b&scene=21#wechat_redirect)指定控制台的地址，并且还要在消费端开启一个与sentinel控制台传递数据端的端口，控制台可以通过此端口调用微服务中的监控程序来获取各种信息。

### Sentinel限流入门实践

我们设置一下指定接口的流控\(流量控制\)，QPS（每秒请求次数）单机阈值为1，代表每秒请求不能超出1次，要不然就做限流处理，处理方式直接调用失败。第一步：选择要限流的链路，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZrWy7xNsNk5hXWYQepjUVmZrLNwTibHYkHHHyUGQFrmqyw7amkibiax8Tw/640?wx_fmt=png)

第二步：设置限流策略，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZjCcjo6MJJyNDX2U2KjLyJWZ4hwM4icJj3CsqjfROFALxDa8bqPT6yQQ/640?wx_fmt=png)

第三步：反复刷新访问消费端端服务，检测是否有限流信息输出，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6Z9JKtKZxKNkPCsoEj4mypptUChGa1iadktLA6tOIUuNdWCcb3YlcA6TA/640?wx_fmt=png)

### 小节面试分析

- Sentinel是什么？\(阿里推出一个流量控制平台，防卫兵\)
- 类似Sentinel的产品你知道有什么？\(hystrix-一代微服务产品\)
- Sentinel是如何对请求进行限流的\?\(基于sentinel依赖提供的拦截器\)
- 你了解哪些限流算法？\(计数器、令牌桶、漏斗算法，滑动窗口算法，…\)
- Sentinel 默认的限流算法是什么？\(滑动窗口算法\)

## Sentinel流控规则分析

### 阈值类型

- QPS\(Queries Per Second\)：当调用相关url对应的资源时，QPS达到单机阈值时，就会限流。
- 线程数：当调用相关url对应的资源时，线程数达到单机阈值时，就会限流。

### 设置限流模式

Sentinel的流控模式代表的流控的方式，默认【直接】，还有关联，链路。

> 直接模式

Sentinel默认的流控处理就是【直接->快速失败】。

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZfGe97ssbibEa9FwyEiaZsQCqmR9HtXq1GUawocA9DFDyLe4ycAxOicrvQ/640?wx_fmt=png)

> 关联模式

当关联的资源达到阈值，就限流自己。例如设置了关联资源为/ur2时，假如关联资源/url2的qps阀值超过1时，就限流/url1接口（是不是感觉很霸道，关联资源达到阀值，是本资源接口被限流了）。这种关联模式有什么应用场景呢？我们举个例子，订单服务中会有2个重要的接口，一个是读取订单信息接口，一个是写入订单信息接口。在高并发业务场景中，两个接口都会占用资源，如果读取接口访问过大，就会影响写入接口的性能。业务中如果我们希望写入[订单](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247502209&idx=3&sn=6070dc46d3c9797751ebeda584e76ae1&chksm=fc79a42fcb0e2d3905e3de30e819bd49ebc7e88379c3bee36fe9be47421e1478a1ed9ed92644&scene=21#wechat_redirect)比较重要，要优先考虑写入订单接口。那就可以利用关联模式；在关联资源上面设置写入接口，资源名设置读取接口就行了；这样就起到了优先写入，一旦写入请求多，就限制读的请求。例如第一步：在ProviderSentinelController中添加一个方法，例如：

```
   @GetMapping("/sentinel02")   public String doSentinel02(){     return "sentinel 02 test  ...";   }
```

第二步：在sentinel中做限流设计，例如

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZxVdIJYHwWsD2dB9gI41cp15qVUiahDNPNLjdXqGG5OMAsmmdDUmS2FQ/640?wx_fmt=png)

第三步：打开两个测试窗口，对/provider/sentinel02进行访问，检查/provider/sentinel01的状态,例如:

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZGpZs9Zvx7LY9LlHCYX0kiaHxuZpq4tJ1p1CjxSmloGyv5yMDEAWY7RQ/640?wx_fmt=png)

> [链路模式](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247495722&idx=3&sn=16041a82d9125053a6d6b50315037589&chksm=fc799d84cb0e14925c104e615ffe97f5370e164c9e5f88812bc59d526817c37df37eca2b58bd&scene=21#wechat_redirect)

链路模式只记录指定链路入口的流量。也就是当多个服务对指定资源调用时，假如流量超出了指定阈值，则进行限流。被调用的方法用\@SentinelResource进行注解，然后分别用不同业务方法对此业务进行调用，假如A业务设置了链路模式的限流,在B业务中是不受影响的。现在对链路模式做一个实践,例如:例如现在设计一个业务对象，代码如下\(为了简单,可以直接写在启动类内部\)：第一步:在指定包创建一个ResourceService类,代码如下:

```
package com.jt.provider.service;
@Service
public class ResourceService{ 

  @SentinelResource("doGetResource")    
  public String doGetResource(){        
     return "doGetResource";    
 }   
}
```

第二步:在ProviderSentinelController中添加一个方法,例如:

```
    @Autowired    
    private ResourceService resourceService;    
    @GetMapping("/sentinel03")    
    public String doSentinel03() throws InterruptedException {        
    resourceService.doGetResource();        return "sentinel 03 test";    
    }
```

第三步:在sentinel中配置限流规则,例如:

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6Zy7zbKicVoAadG5Uktqq4seN3j8VSfA9Gp6SOl76z9IIsQx9Z4Tia8G4A/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6Za7K5ia4ibL60wxMN15yq84TSv84OfxqBdqna0TJ2EsM4Mvn2N0BZ2j9g/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZaQB4RE7NJhlJ0chW2AmtCMT45NJwziaC8fUkIfY5eLorXUaMToLs61A/640?wx_fmt=png)

设置链路流控规则后，再频繁对限流链路进行访问，检测是否会出现500异常,例如

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZSiaDeLDpB6ERFD3Yv48Y0HQIP1I0LtvFlKLEniaWNp6GAHLm0b20ZAvQ/640?wx_fmt=png)

说明，流控模式为链路模式时，假如是sentinel 1.7.2以后版本，Sentinel Web过滤器默认会聚合所有URL的入口为sentinel\_spring\_web\_context，因此单独对指定链路限流会不生效，需要在application.yml添加如下语句来关闭URL PATH聚合,例如：

```
sentinel:     
  web-context-unify: false
```

### 小节面试分析

- 你了解sentinel中的阈值应用类型吗\?（两种-QPS,线程数）
- Sentinel的限流规则中默认有哪些限流模式\?\(直连，关联，链路\)
- Sentinel的限流效果有哪些？\(快速失败，预热，排队\)

## Sentinel降级应用实践

### 概述

除了流量控制以外，对调用链路中不稳定的资源进行熔断降级也是保障高可用的重要措施之一。由于调用关系的复杂性，如果调用链路中的某个资源不稳定，最终会导致请求发生堆积。Sentinel 熔断降级会在调用链路中某个资源出现不稳定状态时（例如调用超时或异常比例升高），对这个资源的调用进行限制，让请求快速失败，避免影响到其它的资源而导致级联错误。当资源被降级后，在接下来的降级时间窗口之内，对该资源的调用都自动熔断（默认行为是抛出 DegradeException）。

### 准备工作

在ProviderController 类中添加doSentinel04方法,基于此方法演示慢调用过程下的限流,代码如下:

```
     //AtomicLong 类支持线程安全的自增自减操作    
     private AtomicLong atomicLong=new AtomicLong(1);    
     @GetMapping("/sentinel04")    
     public  String doSentinel04() throws InterruptedException {        
     //获取自增对象的值,然后再加1        
     long num=atomicLong.getAndIncrement();        
     if(num%2==0){
     //模拟50%的慢调用比例          
     Thread.sleep(200);       
     }        
     return "sentinel 04 test";    
     }
```

说明,我们在此方法中设置休眠,目的是为了演示慢调用\(响应时间比较长\).

### Sentinel降级入门

接下来,我们基于一个请求链路,进行服务降级及应用实践,例如:第一步：服务启动后，选择要降级的链路，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6Z9LejsfsTLjwISibvn1BCibP66ZOV82LyCaA1AXWJvCbhLhZYUbibPa6og/640?wx_fmt=png)

第二步：选择要降级的链路，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZYic4CdWWB2e3j5O0ZATnstLYZ71EBdqAozg8kCNAP4DlVVON6DOVoHw/640?wx_fmt=png)

这里表示熔断策略选择"慢调用比例"，表示请求数超过3时，假如平均响应时间超过200毫秒的有30\%，则对请求进行熔断，熔断时长为10秒钟，10秒以后恢复正常。第三步：对指定链路进行刷新，多次访问测试，检测页面上是否会出现 Blocked By Sentinel \(flow Limiting\)内容.我们也可以进行断点调试，在DefaultBlockExceptionHandler中的handle方法内部加断点，分析异常类型，假如异常类型DegradeException则为降级熔断。

### Sentinel 异常处理

系统提供了默认的异常处理机制，假如默认处理机制不满足我们需求，我们可以自己进行定义。定义方式上可以直接或间接实现BlockExceptionHandler接口，并将对象交给spring管理。

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZZXGWiaJsDwibo9eeMHwSk2v8XhBLRDsnYlXsgd81bOIMIUL6ZMN392lg/640?wx_fmt=png)

```java
package com.jt.provider.controller;
import com.alibaba.csp.sentinel.adapter.spring.webmvc.callback.BlockExceptionHandler;
import com.alibaba.csp.sentinel.slots.block.BlockException;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.PrintWriter;
import java.util.HashMap;
import java.util.Map;
/** * 自定义限流,降级等异常处理对象 */
@Slf4j
@Component
public class ServiceBlockExceptionHandler     implements BlockExceptionHandler {    
  /**     * 用于处理BlockException类型以及子类类型异常     */   
  @Override   
  public void handle(HttpServletRequest request,                       HttpServletResponse response,                       BlockException e) throws Exception {        
  //设置响应数据编码        
  response.setCharacterEncoding("utf-8");        
  //告诉客户端响应数据的类型,以及客户端显示内容的编码        
  response.setContentType("text/html;charset=utf-8");       
  //向客户端响应一个json格式的字符串        
  //String str="{\"status\":429,\"message\":\"访问太频繁了\"}";       
  Map<String,Object> map=new HashMap<>();        
  map.put("status", 429);       
  map.put("message","访问太频繁了");       
  String jsonStr=new ObjectMapper().writeValueAsString(map);    
  PrintWriter out = response.getWriter();        
  out.print(jsonStr);    
  out.flush();    
  out.close();  
  }
}
```

### 小节面试分析

- 何为降级熔断？（让外部应用停止对服务的访问，生活中跳闸，路障设置-此路不通）
- 为什么要进行熔断呢？\(平均响应速度越来越慢或经常出现异常，这样可能会导致调用链堆积，最终系统崩溃\)
- Sentinel中限流，降级的异常父类是谁？\(BlockException\)
- Sentinel 出现降级熔断时，系统底层抛出的异常是谁？\(DegradeException\)
- Sentinel中异常处理接口是谁？（BlockExceptionHandler）
- Sentinel中异常处理接口下默认的实现类为\? \(DefaultBlockExceptionHandler\)
- 假如Sentinel中默认的异常处理规则不满足我们的需求怎么办\?\(自己定义\)
- 我们如何自己定义Sentinel中异常处理呢？（直接或间接实现BlockExceptionHandler ）
- Sentinel熔断降级策略有哪些\?\(慢调用比例、异常比例、异常数\)

## Sentinel热点规则分析（重点）

### 概述

何为热点？热点即经常访问的数据。比如：

- 商品 ID 为参数，统计一段时间内最常购买的商品 ID 并进行限制。
- 用户 ID 为参数，针对一段时间内频繁访问的用户 ID 进行限制。

热点参数限流会统计传入参数中的热点数据，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流。热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效。其中，Sentinel会利用 LRU 策略统计最近最常访问的热点参数，结合令牌桶算法来进行参数级别的流控。

### 快速入门

第一步：在sca-provider中添加如下方法,例如：

```
@GetMapping("/sentinel/findById")
@SentinelResource("resource")
public String doFindById(@RequestParam("id") Integer id){     
    return "resource id is "+id;
}
```

第二步：服务启动后，选择要限流的热点链路，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6Z5bXxN0SmzZ2ZJQ7pDTNHyv3rdibwE98YiaqvsNbOyomvqxd5Sa48Q0Bg/640?wx_fmt=png)

第三步：设置要限流的热点，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZNlhoaDCwaUOFrXW45ibichkhdHia3LBERlEhXeJO9XD55seg8lmyhNajQ/640?wx_fmt=png)

热点规则的限流模式只有QPS模式（这才叫热点）。参数索引为\@SentinelResource注解的方法参数下标，0代表第一个参数，1代表第二个参数。单机阈值以及统计窗口时长表示在此窗口时间超过阈值就限流。第四步：多次访问热点参数方法，前端会出现如下界面，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZRL5gRicAvocg3OPDzqZj7U6Qb8Kzae4tcEY8xaWPXAR9XicKRzJSZjIQ/640?wx_fmt=png)

然后，在后台出现如下异常表示限流成功。

```
com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowException: 2
```

其中，热点参数其实说白了就是特殊的流控，流控设置是针对整个请求的；但是热点参数他可以设置到具体哪个参数，甚至参数针对的值，这样更灵活的进行流控管理。一般应用在某些特殊资源的特殊处理，如：某些商品流量大，其他商品流量很正常，就可以利用热点参数限流的方案。

### 特定参数设计

配置参数例外项，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6Z5eGKYBwhFxuKgLeQazaSMYA9LywdsNPlFJGbVK3WjbicVZicnx5vWCBA/640?wx_fmt=png)

其中,这里表示参数值为5时阈值为100，其它参数值阈值为1.

### 小节面试分析

- 如何理解热点数据？\(访问频度比较高的数据，某些商品、谋篇文章、某个视频\)
- 热点数据的限流规则是怎样的\?\(主要是针对参数进行限流设计\)
- 热点数据中的特殊参数如何理解？\(热点限流中的某个参数值的阈值设计\)
- 对于热点数据的访问出现限流以后底层异常是什么？\(ParamFlowException\)

## Sentinel系统规则（了解）

### 概述

系统在生产环境运行过程中，我们经常需要监控服务器的状态，看[服务器CPU](http://mp.weixin.qq.com/s?__biz=MzU2MTI4MjI0MQ==&mid=2247502594&idx=1&sn=0fe668e7589abfe9d2295fe451feceb3&chksm=fc79a6accb0e2fba2d0c59986c489b9687594d990263b2d2e033478ac7f1cd44a5e79da22797&scene=21#wechat_redirect)、内存、IO等的使用率；主要目的就是保证服务器正常的运行，不能被某些应用搞崩溃了；而且在保证稳定的前提下，保持系统的最大吞吐量。

### 快速入门

Sentinel的系统保护规则是从应用级别的入口流量进行控制，从单台机器的总体 Load（负载）、RT（响应时间）、入口 QPS 、线程数和CPU使用率五个维度监控应用数据，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6Zfp2UcTtwbv7q4aIXIIwg4K0OXRBo3sm0mHg8WibSjj86DR59zR1icRdw/640?wx_fmt=png)

系统规则是一种全局设计规则，其中，

- Load（仅对 Linux/Unix-like 机器生效）：当系统 load1 超过阈值，且系统当前的并发线程数超过系统容量时才会触发系统保护。系统容量由系统的 maxQps \* minRt 计算得出。设定参考值一般是 CPU cores \* 2.5。
- CPU使用率：当系统 CPU 使用率超过阈值即触发系统保护（取值范围 0.0-1.0）。
- RT：当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护，单位是毫秒。
- 线程数：当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护。
- 入口 QPS：当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护。
- 说明，系统保护规则是应用整体维度的，而不是资源维度的，并且仅对入口流量生效。入口流量指的是进入应用的流量（EntryType.IN），比如 Web 服务。

### 小节面试分析

- 如何理解sentinel中的系统规则？\(是对所有链路的控制规则,是一种系统保护策略\)
- Sentinel的常用系统规则有哪些？\(RT,QPS,CPU,线程,Load-linux,unix\)
- Sentinel系统保护规则被触发以后底层会抛出什么异常？（SystemBlockException）

## Sentinel授权规则\(重要\)

### 概述

很多时候，我们需要根据调用方来限制资源是否通过，这时候可以使用 Sentinel 的黑白名单控制的功能。黑白名单根据资源的请求来源（origin）限制资源是否通过，若配置白名单则只有请求来源位于白名单内时才可通过；若配置黑名单则请求来源位于黑名单时不通过，其余的请求通过。例如微信中的黑名单。

### 快速入门

sentinel可以基于黑白名单方式进行授权规则设计，如图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZiaZdB19CWibiaPDITvatOPa2TXKYBfP5FiabdicD5OEFyLPNDnnWatS7gxQ/640?wx_fmt=png)

黑白名单规则（AuthorityRule）非常简单，主要有以下配置项：

- 资源名:即限流规则的作用对象
- 流控应用:对应的黑名单/白名单中设置的规则值,多个值用逗号隔开.
- 授权类型:白名单，黑名单\(不允许访问\).

**案例实现:**定义请求解析器,用于对请求进行解析,并返回解析结果,sentinel底层在拦截到用户请求以后,会对请求数据基于此对象进行解析,判定是否符合黑白名单规则,例如:第一步:定义RequestOriginParser接口的实现类，在接口方法中解析请求参数数据并返回,底层会基于此返回值进行授权规则应用。

```
@Component
public class DefaultRequestOriginParser implements RequestOriginParser {    
  @Override    
  public String parseOrigin(HttpServletRequest request) {       
  String origin = request.getParameter("origin");
  //这里的参数名会与请求中的参数名一致        
  return origin;    }
}
```

第二步:定义流控规则,如图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZiaZdB19CWibiaPDITvatOPa2TXKYBfP5FiabdicD5OEFyLPNDnnWatS7gxQ/640?wx_fmt=png)

第三步:执行资源访问,检测授权规则应用,当我们配置的流控应用值为app1时，假如规则为黑名单，则基于 http://ip:port/path\?origin=app1的请求不可以通过,其请求处理流程如图下:

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZqNPayZBL2L2R6pJ3t11En519VRJrMCD9hNnqftkvkDXD07me1l8fYg/640?wx_fmt=png)

> 拓展：尝试基于请求ip等方式进行黑白名单的规则设计,例如：

第一步: 修改请求解析器,获取请求ip并返回,例如:

```
@Component
public class DefaultRequestOriginParser  implements RequestOriginParser {    
  //解析请求源数据   
  @Override    
  public String parseOrigin(HttpServletRequest request) {    
  //获取访问请求中的ip地址,基于ip地址进行黑白名单设计（例如在流控应用栏写ip地址）        
  String ip= request.getRemoteAddr();        
  System.out.println("ip="+ip);        
  return ip;    
  }
  //授权规则中的黑白名单的值,来自此方法的返回值
}
```

第二步:在sentinel控制台定义授权规则,例如:

![](https://mmbiz.qpic.cn/mmbiz_png/JfTPiahTHJhqIic7GrEib0Rc6NWdCXebv6ZCEibicEGib8LSZVlogJp219YiaRs9KicodmoiaAmicx4YmJVSIGpcWNnSJneg/640?wx_fmt=png)

第三步:规则定义后以后,基于你的ip地址,进行访问测试,检测黑白名单效果.

### 小节面试分析

- 如何理解Sentinel中的授权规则？\(对指定资源的访问给出的一种简易的授权策略\)
- Sentinel的授权规则是如何设计的？\(白名单和黑名单\)
- 如何理解Sentinel中的白名单？（允许访问的资源名单）
- 如何理解Sentinel中的黑名单？（不允许访问的资源名单）、
- Sentinel如何识别白名单和黑名单？\(在拦截器中通过调用RequestOriginParser对象的方法检测具体的规则\)
- 授权规则中RequestOriginParser类的做用是什么？（对流控应用值进行解析，检查服务访问时传入的值是否与RequestOriginParser的parseOrigin方法返回值是否相同。）

## 总结\(Summary\)

总之，Sentinel可为秒杀、抢购、抢票、拉票等高并发应用，提供API接口层面的流量限制，让突然暴涨而来的流量用户访问受到统一的管控，使用合理的流量放行规则使得用户都能正常得到服务。

### 重难点分析

- Sentinel诞生的背景\?\(计算机的数量是否有限,处理能力是否有限,并发比较大或突发流量比较大\)
- 服务中Sentinel环境的集成,初始化\?\(添加依赖-两个,sentinel配置\)
- Sentinel 的限流规则\?\(阈值类型-QPS\&线程数,限流模式-直接,关联,链路\)
- Sentinel 的降级\(熔断\)策略\?\(慢调用,异常比例,异常数\)
- Sentinel 的热点规则设计\(掌握\)\?
- Sentinel 系统规则设计\?\(了解,全局规则定义,针对所有请求有效\)
- Sentinel 授权规则设计\?\(掌握,黑白名单\)

### FAQ分析

- 为什么要限流\?
- 你了解的那些限流框架\?\(sentinel\)
- 常用的限流算法有那些\?\(计数,令牌桶-电影票,漏桶-漏斗,滑动窗口\)
- Sentinel有哪些限流规则\?\(QPS,线程数\)
- Sentinel有哪些限流模式\?\(直接,关联-创建订单和查询订单,链路限流-北京六环外不限号,但是五环就限号\)
- Sentinel 的降级\(熔断\)策略有哪些\?\(慢调用-响应时长,异常比例-异常占比,异常数\)
- Sentinel 的热点规则中的热点数据\?\(热卖商品,微博大咖,新上映的电影\)
- 如何理解Sentinel 授权规则中的黑白名单\?

### Bug分析

依赖下载失败 \(maven-本地库,网络,镜像仓库\) 单词错误\(拼写错误\)

### _来源：blog.csdn.net/maitian\_2008/_
