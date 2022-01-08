

### 一、常用的限流算法

##### 1.计数器方式（传统计数器缺点：临界问题 可能违背定义固定速率原则）

![](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAO7sFUpesfx3VImdcichfo3zPmuapiaL4BgPSV6oalterJc6BGcicfWIRPrbAoUv6FdQiajGFkMOaicZQ/640?wx_fmt=png)

##### 2.令牌桶方式

令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。从原理上看，令牌桶算法和漏桶算法是相反的，一个“进水”，一个是“漏水”。

![](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAO7sFUpesfx3VImdcichfo3APFicyRaiaWnwhVGdS3AqvAt9bhDb5FkL7ysWY5vBL7ecQFPQiaeNL3tA/640?wx_fmt=png)

RateLimiter是guava提供的基于令牌桶算法的实现类，可以非常简单的完成限流特技，并且根据系统的实际情况来调整生成token的速率。RateLimiter 是单机（单进程）的限流，是JVM级别的的限流，所有的令牌生成都是在内存中，在分布式环境下不能直接这么用。

##### 3、漏桶算法

![](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAO7sFUpesfx3VImdcichfo3TzCiaTxraMPHiaaoymwbywLoA5KpFc7NFsGgusSbE490IqcS9PPSLZyg/640?wx_fmt=png)

如上图所示，我们假设系统是一个漏桶，当请求到达时，就是往漏桶里“加水”，而当请求被处理掉，就是水从漏桶的底部漏出。水漏出的速度是固定的，当“加水”太快，桶就会溢出，也就是“拒绝请求”。从而使得桶里的水的体积不可能超出桶的容量。令牌桶算法与漏桶算法的区别：

> 令牌桶里面装载的是令牌，然后让令牌去关联到数据发送，常规漏桶里面装载的是数据，令牌桶允许用户的正常的持续突发量（Bc），就是一次就将桶里的令牌全部用尽的方式来支持续突发，而常规的漏桶则不允许用户任何突发行。

### 二、限流实现

##### 基于 redis 的分布式限流

单机版中我们了解到 AtomicInteger、RateLimiter、Semaphore 这几种解决方案，但它们也仅仅是单机的解决手段，在集群环境下就透心凉了，后面又讲述了 Nginx 的限流手段，可它又属于网关层面的策略之一，并不能解决所有问题。例如供短信接口，你无法保证消费方是否会做好限流控制，所以自己在应用层实现限流还是很有必要的。

##### 导入依赖

在 pom.xml 中添加上 starter-web、starter-aop、starter-data-redis 的依赖即可，习惯了使用 commons-lang3 和 guava 中的一些工具包…

```
<dependencies>
    <!-- 默认就内嵌了Tomcat 容器，如需要更换容器也极其简单-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>21.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
    </dependency>
</dependencies>
```

##### 属性配置

在 `application.properites` 资源文件中添加 redis 相关的配置项

```
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=battcn
```

##### Limit 注解

创建一个 Limit 注解，不多说注释都给各位写齐全了….

```
package com.johnfnash.learn.springboot.ratelimiter.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

// 限流
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Limit {

    /**
     * 资源的名称
     * @return
     */
    String name() default "";
    
    /**
     * 资源的key
     *
     * @return
     */
    String key() default "";

    /**
     * Key的prefix
     *
     * @return
     */
    String prefix() default "";
    
    /**
     * 给定的时间段
     * 单位秒
     *
     * @return
     */
    int period();

    /**
     * 最多的访问限制次数
     *
     * @return
     */
    int count();
    
    /**
     * 类型
     *
     * @return
     */
    LimitType limitType() default LimitType.CUSTOMER;
}
```

```

// 限制的类型
public enum LimitType {

    /**
     * 自定义key
     */
    CUSTOMER,
    /**
     * 根据请求者IP
     */
    IP;
    
}
```

##### RedisTemplate

默认情况下 spring-boot-data-redis 为我们提供了StringRedisTemplate 但是满足不了其它类型的转换，所以还是得自己去定义其它类型的模板….

```

import java.io.Serializable;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisLimiterHelper {

    @Bean
    public RedisTemplate<String, Serializable> limitRedisTemplate(LettuceConnectionFactory factory) {
        RedisTemplate<String, Serializable> template = new RedisTemplate<String, Serializable>();
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setConnectionFactory(factory);
        return template;
    }
    
}
```

##### Limit 拦截器（AOP）

熟悉 Redis 的朋友都知道它是线程安全的，我们利用它的特性可以实现分布式锁、分布式限流等组件。官方虽然没有提供相应的API，但却提供了支持 Lua 脚本的功能，我们可以通过编写 Lua 脚本实现自己的API，同时他是满足原子性的….下面核心就是调用 execute 方法传入我们的 Lua 脚本内容，然后通过返回值判断是否超出我们预期的范围，超出则给出错误提示。

```
import java.io.Serializable;
import java.lang.reflect.Method;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang3.StringUtils;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

import com.google.common.collect.ImmutableList;
import com.johnfnash.learn.springboot.ratelimiter.annotation.Limit;
import com.johnfnash.learn.springboot.ratelimiter.annotation.LimitType;

@Aspect
@Configuration
public class LimitInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(LimitInterceptor.class);;
    
    private final String REDIS_SCRIPT = buildLuaScript();
    
    @Autowired
    private RedisTemplate<String, Serializable> redisTemplate;
    
    @Around("execution(public * *(..)) && @annotation(com.johnfnash.learn.springboot.ratelimiter.annotation.Limit)")
    public Object interceptor(ProceedingJoinPoint pjp) {
        MethodSignature signature = (MethodSignature) pjp.getSignature();
        Method method = signature.getMethod();
        Limit limitAnno = method.getAnnotation(Limit.class);
        LimitType limitType = limitAnno.limitType();
        String name = limitAnno.name();
        
        String key = null;
        int limitPeriod = limitAnno.period();
        int limitCount = limitAnno.count();
        switch (limitType) {
        case IP:
            key = getIpAddress();
            break;
        case CUSTOMER:
            // TODO 如果此处想根据表达式或者一些规则生成 请看 一起来学Spring Boot | 第二十三篇：轻松搞定重复提交（分布式锁）
            key = limitAnno.key();
            break;
        default:
            break;
        }
        
        ImmutableList<String> keys = ImmutableList.of(StringUtils.join(limitAnno.prefix(), key));
        try {
            RedisScript<Number> redisScript = new DefaultRedisScript<Number>(REDIS_SCRIPT, Number.class);
            Number count = redisTemplate.execute(redisScript, keys, limitCount, limitPeriod);
            logger.info("Access try count is {} for name={} and key = {}", count, name, key);
            if(count != null && count.intValue() <= limitCount) {
                return pjp.proceed();
            } else {
                throw new RuntimeException("You have been dragged into the blacklist");
            }
        } catch (Throwable e) {
            if (e instanceof RuntimeException) {
                throw new RuntimeException(e.getLocalizedMessage());
            }
            throw new RuntimeException("server exception");
        }
    }
    
    /**
     * 限流 脚本
     *
     * @return lua脚本
     */
    private String buildLuaScript() {
        StringBuilder lua = new StringBuilder();
        lua.append("local c")
           .append("\nc = redis.call('get', KEYS[1])")
           // 调用不超过最大值，则直接返回
           .append("\nif c and tonumber(c) > tonumber(ARGV[1]) then")
           .append("\nreturn c;")
           .append("\nend")
           // 执行计算器自加
           .append("\nc = redis.call('incr', KEYS[1])")
           .append("\nif tonumber(c) == 1 then")
           // 从第一次调用开始限流，设置对应键值的过期
           .append("\nredis.call('expire', KEYS[1], ARGV[2])")
           .append("\nend")
           .append("\nreturn c;");
        return lua.toString();
    }
    
    private static final String UNKNOWN = "unknown";

    public String getIpAddress() {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
        String ip = request.getHeader("x-forwarded-for");
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || UNKNOWN.equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
    
}
```

##### 控制层

在接口上添加`@Limit()`注解，如下代码会在 Redis 中生成过期时间为 100s 的 `key = test` 的记录，特意定义了一个 AtomicInteger 用作测试…

```
@RestController
public class LimiterController {

    private static final AtomicInteger ATOMIC_INTEGER = new AtomicInteger();

    @Limit(key = "test", period = 100, count = 10)
    // 意味著 100S 内最多允許訪問10次
    @GetMapping("/test")
    public int testLimiter() {
        return ATOMIC_INTEGER.incrementAndGet();
    }
    
}
```

##### 测试

完成准备事项后，启动 启动类 自行测试即可，测试手段相信大伙都不陌生了，如 浏览器、postman、junit、swagger，此处基于 postman。未达设定的阀值时，正常响应

![](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAO7sFUpesfx3VImdcichfo3lIBHfibCU41m6VvSSeVNnf1axaSlvjLcRfibKeP5yvLBicbtxmD8nYs1A/640?wx_fmt=png)

达到设置的阀值时,错误响应

![](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XAO7sFUpesfx3VImdcichfo35W7pnK8QxGHgj5nmcOTREFatxbNF2DCicEb22bmMEfnrUHKib5Ium0ibg/640?wx_fmt=png)

### 总结

目前很多大佬都写过关于 Spring Boot 的教程了，如有雷同，请多多包涵，本教程基于最新的 spring-boot-starter-parent：2.0.3.RELEASE编写…本篇文章核心的 Lua 脚本截取自军哥的 Aquarius 开源项目，有兴趣的朋友可以 fork star ，该项目干货满满…全文代码：

> https://github.com/battcn/spring-boot2-learning/tree/master/chapter27

注：上面的方式是使用计数器的限流方式，无法处理临界的时候，大量请求的的情况。要解决这个问题，可以使用redis中列表类型的键来记录最近N次访问的时间，一旦键中的元素超过N个，就判断时间最早的元素距现在的时间是否小于M秒。如果是则表示用户最近M秒的访问次数超过了N次；如果不是就将现在的时间加入列表中同时把最早的元素删除（可以通过脚本功能避免竞态条件）。由于需要记录每次访问的时间，所以当要限制“M时间最多访问N次”时，如果“N”的数值较大，此方法会占用较多的存储空间，实际使用时还需要开发者自己去权衡。下面的解决思路的实现如下：

```
/**
 * 限流 脚本（处理临界时间大量请求的情况）
 *
 * @return lua脚本
 */
private String buildLuaScript2() {
    StringBuilder lua = new StringBuilder();
    lua.append("local listLen, time")
       .append("\nlistLen = redis.call('LLEN', KEYS[1])")
       // 不超过最大值，则直接写入时间
       .append("\nif listLen and tonumber(listLen) < tonumber(ARGV[1]) then")
            .append("\nlocal a = redis.call('TIME');")
            .append("\nredis.call('LPUSH', KEYS[1], a[1]*1000000+a[2])")
       .append("\nelse")
           // 取出现存的最早的那个时间，和当前时间比较，看是小于时间间隔
           .append("\ntime = redis.call('LINDEX', KEYS[1], -1)")
           .append("\nlocal a = redis.call('TIME');")
           .append("\nif a[1]*1000000+a[2] - time < tonumber(ARGV[2])*1000000 then")
               // 访问频率超过了限制，返回0表示失败
               .append("\nreturn 0;")
           .append("\nelse")                   
               .append("\nredis.call('LPUSH', KEYS[1], a[1]*1000000+a[2])")
               .append("\nredis.call('LTRIM', KEYS[1], 0, tonumber(ARGV[1])-1)")
           .append("\nend")   
       .append("\nend")
       .append("\nreturn 1;");
    return lua.toString();
}
```

调用的地方的也相应修改如下：

```
if(count != null && count.intValue() == 1) {
    return pjp.proceed();
} else {
    throw new RuntimeException("You have been dragged into the blacklist");
}
```

补充，最近执行 `buildLuaScript2()` 中的lua脚本，报错`Write commands not allowed after non deterministic commands.`这个错误的原因大家可以参见这篇文章：

> https://yq.aliyun.com/articles/195914

大致原因跟redis集群的重放和备份策略有关，相当于我调用TIME操作，会在主从各执行一次，得到的结果肯定会存在差异，这个差异就给最终逻辑正确性带来了不确定性。在redis 4.0之后引入了`redis.replicate_commands()`来放开限制。于是，在 buildLuaScript2 的 lua 脚本最前面加上 “`redis.replicate_commands();`”，错误得以解决。

> _感谢阅读，希望对你有所帮助 :\) __来源：blog.csdn.net/johnf\_nash/article/details/89791808_

  