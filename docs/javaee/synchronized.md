
[来源：juejin.im/post/5d31928c51882564c966a71c](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)

- [1.什么是幂等](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
- [2.产生原因](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
- [3.解决方案](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
- [①配置注解](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
- [②实例化锁](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
- [③AOP 切面](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)
- [④注解使用案例![](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcibqe6CcryvpHnjXIcz4LCmXVvjIwNLiaiaL775sIhTiagcxbIjx3wXOJBsgSOq14N4buC2KNUzyvuMQ/640?wx_fmt=jpeg)](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21&token=899450012&lang=zh_CN#wechat_redirect)

---

## [1.什么是幂等](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfcibqe6CcryvpHnjXIcz4LCmz6CvtrtN25evvKGdKudTe2jF6Om2ljMLWbseq2N3jtVo59mBTobiasQ/640?wx_fmt=png)

在我们编程中常见幂等

- select查询天然幂等
- delete删除也是幂等,删除同一个多次效果一样
- update直接更新某个值的,幂等
- update更新累加操作的,非幂等
- insert非幂等操作,每次新增一条

## [2.产生原因](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

由于重复点击或者网络重发 eg:

- 点击提交按钮两次;
- 点击刷新按钮;
- 使用浏览器后退按钮重复之前的操作，导致重复提交表单;
- 使用浏览器历史记录重复提交表单;
- 浏览器重复的HTTP请;
- nginx重发等情况;
- 分布式RPC的try重发等;

## [3.解决方案](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

**1\)前端js提交禁止按钮可以用一些js组件****2\)使用Post/Redirect/Get模式**在提交后执行页面重定向，这就是所谓的Post-Redirect-Get \(PRG\)模式。简言之，当用户提交了表单后，你去执行一个客户端的重定向，转到提交成功信息页面。这能避免用户按F5导致的重复提交，而其也不会出现浏览器表单重复提交的警告，也能消除按浏览器前进和后退按导致的同样问题。**3\)在session中存放一个特殊标志**在服务器端，生成一个唯一的标识符，将它存入session，同时将它写入表单的隐藏字段中，然后将表单页面发给浏览器，用户录入信息后点击提交，在服务器端，获取表单中隐藏字段的值，与session中的唯一标识符比较，相等说明是首次提交，就处理本次请求，然后将session中的唯一标识符移除；不相等说明是重复提交，就不再处理。**4\)其他借助使用header头设置缓存控制头Cache-control等方式**比较复杂 不适合移动端APP的应用 这里不详解**5\)借助数据库**insert使用唯一索引 update使用 乐观锁 version版本法这种在大数据量和高并发下效率依赖数据库硬件能力,可针对非核心业务**6\)借助悲观锁**使用select … for update ,这种和 synchronized 锁住先查再insert or update一样,但要避免死锁,效率也较差针对单体 请求并发不大 可以推荐使用**7\)借助本地锁\(本文重点\)**原理:使用了 ConcurrentHashMap 并发容器 putIfAbsent 方法,和 ScheduledThreadPoolExecutor 定时任务,也可以使用guava cache的机制, gauva中有配有缓存的有效时间也是可以的key的生成Content-MD5Content-MD5 是指 Body 的 MD5 值，只有当 Body 非Form表单时才计算MD5，计算方式直接将参数和参数名称统一加密MD5MD5在一定范围类认为是唯一的 近似唯一 当然在低并发的情况下足够了本地锁只适用于单机部署的应用.

## [①配置注解](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

`import java.lang.annotation.*;  
  
@Target(ElementType.METHOD)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Resubmit {  
  
    /**  
     * 延时时间 在延时多久后可以再次提交  
     *  
     * @return Time unit is one second  
     */  
    int delaySeconds() default 20;  
}  
`

## [②实例化锁](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

`import com.google.common.cache.Cache;  
import com.google.common.cache.CacheBuilder;  
import lombok.extern.slf4j.Slf4j;  
import org.apache.commons.codec.digest.DigestUtils;  
  
import java.util.Objects;  
import java.util.concurrent.ConcurrentHashMap;  
import java.util.concurrent.ScheduledThreadPoolExecutor;  
import java.util.concurrent.ThreadPoolExecutor;  
import java.util.concurrent.TimeUnit;  
  
/**  
 * @author lijing  
 * 重复提交锁  
 */  
@Slf4j  
public final class ResubmitLock {  
  
  
    private static final ConcurrentHashMap<String, Object> LOCK_CACHE = new ConcurrentHashMap<>(200);  
    private static final ScheduledThreadPoolExecutor EXECUTOR = new ScheduledThreadPoolExecutor(5, new ThreadPoolExecutor.DiscardPolicy());  
  
  
   // private static final Cache<String, Object> CACHES = CacheBuilder.newBuilder()  
            // 最大缓存 100 个  
   //          .maximumSize(1000)  
            // 设置写缓存后 5 秒钟过期  
   //         .expireAfterWrite(5, TimeUnit.SECONDS)  
   //         .build();  
  
  
    private ResubmitLock() {  
    }  
  
    /**  
     * 静态内部类 单例模式  
     *  
     * @return  
     */  
    private static class SingletonInstance {  
        private static final ResubmitLock INSTANCE = new ResubmitLock();  
    }  
  
    public static ResubmitLock getInstance() {  
        return SingletonInstance.INSTANCE;  
    }  
  
  
    public static String handleKey(String param) {  
        return DigestUtils.md5Hex(param == null ? "" : param);  
    }  
  
    /**  
     * 加锁 putIfAbsent 是原子操作保证线程安全  
     *  
     * @param key   对应的key  
     * @param value  
     * @return  
     */  
    public boolean lock(final String key, Object value) {  
        return Objects.isNull(LOCK_CACHE.putIfAbsent(key, value));  
    }  
  
    /**  
     * 延时释放锁 用以控制短时间内的重复提交  
     *  
     * @param lock         是否需要解锁  
     * @param key          对应的key  
     * @param delaySeconds 延时时间  
     */  
    public void unLock(final boolean lock, final String key, final int delaySeconds) {  
        if (lock) {  
            EXECUTOR.schedule(() -> {  
                LOCK_CACHE.remove(key);  
            }, delaySeconds, TimeUnit.SECONDS);  
        }  
    }  
}  
`

## [③AOP 切面](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

`import com.alibaba.fastjson.JSONObject;  
import com.cn.xxx.common.annotation.Resubmit;  
import com.cn.xxx.common.annotation.impl.ResubmitLock;  
import com.cn.xxx.common.dto.RequestDTO;  
import com.cn.xxx.common.dto.ResponseDTO;  
import com.cn.xxx.common.enums.ResponseCode;  
import lombok.extern.log4j.Log4j;  
import org.aspectj.lang.ProceedingJoinPoint;  
import org.aspectj.lang.annotation.Around;  
import org.aspectj.lang.annotation.Aspect;  
import org.aspectj.lang.reflect.MethodSignature;  
import org.springframework.stereotype.Component;  
  
import java.lang.reflect.Method;  
  
/**  
 * @ClassName RequestDataAspect  
 * @Description 数据重复提交校验  
 * @Author lijing  
 * @Date 2019/05/16 17:05  
 **/  
@Log4j  
@Aspect  
@Component  
public class ResubmitDataAspect {  
  
    private final static String DATA = "data";  
    private final static Object PRESENT = new Object();  
  
    @Around("@annotation(com.cn.xxx.common.annotation.Resubmit)")  
    public Object handleResubmit(ProceedingJoinPoint joinPoint) throws Throwable {  
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();  
        //获取注解信息  
        Resubmit annotation = method.getAnnotation(Resubmit.class);  
        int delaySeconds = annotation.delaySeconds();  
        Object[] pointArgs = joinPoint.getArgs();  
        String key = "";  
        //获取第一个参数  
        Object firstParam = pointArgs[0];  
        if (firstParam instanceof RequestDTO) {  
            //解析参数  
            JSONObject requestDTO = JSONObject.parseObject(firstParam.toString());  
            JSONObject data = JSONObject.parseObject(requestDTO.getString(DATA));  
            if (data != null) {  
                StringBuffer sb = new StringBuffer();  
                data.forEach((k, v) -> {  
                    sb.append(v);  
                });  
                //生成加密参数 使用了content_MD5的加密方式  
                key = ResubmitLock.handleKey(sb.toString());  
            }  
        }  
        //执行锁  
        boolean lock = false;  
        try {  
            //设置解锁key  
            lock = ResubmitLock.getInstance().lock(key, PRESENT);  
            if (lock) {  
                //放行  
                return joinPoint.proceed();  
            } else {  
                //响应重复提交异常  
                return new ResponseDTO<>(ResponseCode.REPEAT_SUBMIT_OPERATION_EXCEPTION);  
            }  
        } finally {  
            //设置解锁key和解锁时间  
            ResubmitLock.getInstance().unLock(lock, key, delaySeconds);  
        }  
    }  
}  
`

## [④注解使用案例](https://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&scene=21#wechat_redirect)

`@ApiOperation(value = "保存我的帖子接口", notes = "保存我的帖子接口")  
    @PostMapping("/posts/save")  
    @Resubmit(delaySeconds = 10)  
    public ResponseDTO<BaseResponseDataDTO> saveBbsPosts(@RequestBody @Validated RequestDTO<BbsPostsRequestDTO> requestDto) {  
        return bbsPostsBizService.saveBbsPosts(requestDto);  
    }  
`

以上就是本地锁的方式进行的幂等提交 使用了Content-MD5 进行加密 只要参数不变,参数加密 密值不变,key存在就阻止提交当然也可以使用 一些其他签名校验 在某一次提交时先 生成固定签名 提交到后端 根据后端解析统一的签名作为 每次提交的验证token 去缓存中处理即可.**8\)借助分布式redis锁 （参考其他）**在 pom.xml 中添加上 starter-web、starter-aop、starter-data-redis 的依赖即可

`<dependencies>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-web</artifactId>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-aop</artifactId>  
    </dependency>  
    <dependency>  
        <groupId>org.springframework.boot</groupId>  
        <artifactId>spring-boot-starter-data-redis</artifactId>  
    </dependency>  
</dependencies>  
`

属性配置 在 application.properites 资源文件中添加 redis 相关的配置项

`spring.redis.host=localhost  
spring.redis.port=6379  
spring.redis.password=123456  
`

**主要实现方式:**熟悉 Redis 的朋友都知道它是线程安全的，我们利用它的特性可以很轻松的实现一个分布式锁，如 opsForValue\(\).setIfAbsent\(key,value\)它的作用就是如果缓存中没有当前 Key 则进行缓存同时返回 true 反之亦然；当缓存后给 key 在设置个过期时间，防止因为系统崩溃而导致锁迟迟不释放形成死锁；那么我们是不是可以这样认为当返回 true 我们认为它获取到锁了，在锁未释放的时候我们进行异常的抛出…

`package com.battcn.interceptor;  
  
import com.battcn.annotation.CacheLock;  
import com.battcn.utils.RedisLockHelper;  
import org.aspectj.lang.ProceedingJoinPoint;  
import org.aspectj.lang.annotation.Around;  
import org.aspectj.lang.annotation.Aspect;  
import org.aspectj.lang.reflect.MethodSignature;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.util.StringUtils;  
  
import java.lang.reflect.Method;  
import java.util.UUID;  
  
/**  
 * redis 方案  
 *  
 * @author Levin  
 * @since 2018/6/12 0012  
 */  
@Aspect  
@Configuration  
public class LockMethodInterceptor {  
  
    @Autowired  
    public LockMethodInterceptor(RedisLockHelper redisLockHelper, CacheKeyGenerator cacheKeyGenerator) {  
        this.redisLockHelper = redisLockHelper;  
        this.cacheKeyGenerator = cacheKeyGenerator;  
    }  
  
    private final RedisLockHelper redisLockHelper;  
    private final CacheKeyGenerator cacheKeyGenerator;  
  
  
    @Around("execution(public * *(..)) && @annotation(com.battcn.annotation.CacheLock)")  
    public Object interceptor(ProceedingJoinPoint pjp) {  
        MethodSignature signature = (MethodSignature) pjp.getSignature();  
        Method method = signature.getMethod();  
        CacheLock lock = method.getAnnotation(CacheLock.class);  
        if (StringUtils.isEmpty(lock.prefix())) {  
            throw new RuntimeException("lock key don't null...");  
        }  
        final String lockKey = cacheKeyGenerator.getLockKey(pjp);  
        String value = UUID.randomUUID().toString();  
        try {  
            // 假设上锁成功，但是设置过期时间失效，以后拿到的都是 false  
            final boolean success = redisLockHelper.lock(lockKey, value, lock.expire(), lock.timeUnit());  
            if (!success) {  
                throw new RuntimeException("重复提交");  
            }  
            try {  
                return pjp.proceed();  
            } catch (Throwable throwable) {  
                throw new RuntimeException("系统异常");  
            }  
        } finally {  
            // TODO 如果演示的话需要注释该代码;实际应该放开  
            redisLockHelper.unlock(lockKey, value);  
        }  
    }  
}  
`

RedisLockHelper 通过封装成 API 方式调用，灵活度更加高

`package com.battcn.utils;  
  
import org.springframework.boot.autoconfigure.AutoConfigureAfter;  
import org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.data.redis.connection.RedisStringCommands;  
import org.springframework.data.redis.core.RedisCallback;  
import org.springframework.data.redis.core.StringRedisTemplate;  
import org.springframework.data.redis.core.types.Expiration;  
import org.springframework.util.StringUtils;  
  
import java.util.concurrent.Executors;  
import java.util.concurrent.ScheduledExecutorService;  
import java.util.concurrent.TimeUnit;  
import java.util.regex.Pattern;  
  
/**  
 * 需要定义成 Bean  
 *  
 * @author Levin  
 * @since 2018/6/15 0015  
 */  
@Configuration  
@AutoConfigureAfter(RedisAutoConfiguration.class)  
public class RedisLockHelper {  
  
  
    private static final String DELIMITER = "|";  
  
    /**  
     * 如果要求比较高可以通过注入的方式分配  
     */  
    private static final ScheduledExecutorService EXECUTOR_SERVICE = Executors.newScheduledThreadPool(10);  
  
    private final StringRedisTemplate stringRedisTemplate;  
  
    public RedisLockHelper(StringRedisTemplate stringRedisTemplate) {  
        this.stringRedisTemplate = stringRedisTemplate;  
    }  
  
    /**  
     * 获取锁（存在死锁风险）  
     *  
     * @param lockKey lockKey  
     * @param value   value  
     * @param time    超时时间  
     * @param unit    过期单位  
     * @return true or false  
     */  
    public boolean tryLock(final String lockKey, final String value, final long time, final TimeUnit unit) {  
        return stringRedisTemplate.execute((RedisCallback<Boolean>) connection -> connection.set(lockKey.getBytes(), value.getBytes(), Expiration.from(time, unit), RedisStringCommands.SetOption.SET_IF_ABSENT));  
    }  
  
    /**  
     * 获取锁  
     *  
     * @param lockKey lockKey  
     * @param uuid    UUID  
     * @param timeout 超时时间  
     * @param unit    过期单位  
     * @return true or false  
     */  
    public boolean lock(String lockKey, final String uuid, long timeout, final TimeUnit unit) {  
        final long milliseconds = Expiration.from(timeout, unit).getExpirationTimeInMilliseconds();  
        boolean success = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, (System.currentTimeMillis() + milliseconds) + DELIMITER + uuid);  
        if (success) {  
            stringRedisTemplate.expire(lockKey, timeout, TimeUnit.SECONDS);  
        } else {  
            String oldVal = stringRedisTemplate.opsForValue().getAndSet(lockKey, (System.currentTimeMillis() + milliseconds) + DELIMITER + uuid);  
            final String[] oldValues = oldVal.split(Pattern.quote(DELIMITER));  
            if (Long.parseLong(oldValues[0]) + 1 <= System.currentTimeMillis()) {  
                return true;  
            }  
        }  
        return success;  
    }  
  
  
    /**  
     * @see <a href="http://redis.io/commands/set">Redis Documentation: SET</a>  
     */  
    public void unlock(String lockKey, String value) {  
        unlock(lockKey, value, 0, TimeUnit.MILLISECONDS);  
    }  
  
    /**  
     * 延迟unlock  
     *  
     * @param lockKey   key  
     * @param uuid      client(最好是唯一键的)  
     * @param delayTime 延迟时间  
     * @param unit      时间单位  
     */  
    public void unlock(final String lockKey, final String uuid, long delayTime, TimeUnit unit) {  
        if (StringUtils.isEmpty(lockKey)) {  
            return;  
        }  
        if (delayTime <= 0) {  
            doUnlock(lockKey, uuid);  
        } else {  
            EXECUTOR_SERVICE.schedule(() -> doUnlock(lockKey, uuid), delayTime, unit);  
        }  
    }  
  
    /**  
     * @param lockKey key  
     * @param uuid    client(最好是唯一键的)  
     */  
    private void doUnlock(final String lockKey, final String uuid) {  
        String val = stringRedisTemplate.opsForValue().get(lockKey);  
        final String[] values = val.split(Pattern.quote(DELIMITER));  
        if (values.length <= 0) {  
            return;  
        }  
        if (uuid.equals(values[1])) {  
            stringRedisTemplate.delete(lockKey);  
        }  
    }  
  
}  
`

redis的提交参照

> https://www.iocoder.cn/Spring-Boot/battcn/v2-cache-redislock/

<span style="font-size: 15px;display: block;text-align: center;margin-top: 50px;color: #999;border-bottom: 1px solid #eee;">- END -</span>

  
---

---
欢迎加入我的知识星球，一起探讨架构，交流源码。加入方式，**长按下方二维码噢**：  

![](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZffVTY1cbQPIhsMGJcNWFwIDn1HomDs3KMQdVdSpmRkicibAFj9FzVtNtQxAtyAhyqmo1N8aibviaU4Qxg/640?wx_fmt=jpeg)已在知识星球更新源码解析如下：  

[![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdCa89KZ4Ls04tTqXvgxWVian1HZ76BOz52l4pkqX0IMicM14rRFyiaO0vQENMOufUhDVVtPiadDdoKjQ/640?wx_fmt=jpeg)](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)[![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdCa89KZ4Ls04tTqXvgxWViaExAeJx1CZeSaJ9qxh0X70s4JGjIVVlT5ZqBGu51YTedMNfO49DKb6g/640?wx_fmt=jpeg)](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)[![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdCa89KZ4Ls04tTqXvgxWViaCibrYIXNgebWPd5g7Or9dcToN660aLAEJEhz4wLpBBiaFhejsaGDGd2g/640?wx_fmt=jpeg)](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)

[![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfdCa89KZ4Ls04tTqXvgxWVia4qmv743xvlia1HYmqCDPBLpo3HXtw8Hmo76GkGK5wCqvicAKxd9ET3ow/640?wx_fmt=jpeg)](http://mp.weixin.qq.com/s?__biz=MzUzMTA2NTU2Ng==&mid=2247487551&idx=1&sn=18f64ba49f3f0f9d8be9d1fdef8857d9&chksm=fa496f8ecd3ee698f4954c00efb80fe955ec9198fff3ef4011e331aa37f55a6a17bc8c0335a8&scene=21#wechat_redirect)最近更新《芋道 SpringBoot 2.X 入门》系列，已经 20 余篇，覆盖了 MyBatis、Redis、MongoDB、ES、分库分表、读写分离、SpringMVC、Webflux、权限、WebSocket、Dubbo、RabbitMQ、RocketMQ、Kafka、性能测试等等内容。  
提供近 3W 行代码的 SpringBoot 示例，以及超 4W 行代码的电商微服务项目。获取方式：点“**在看**”，关注公众号并回复 **666** 领取，更多内容陆续奉上。

**文章有帮助的话，在看，转发吧。****谢谢支持哟 \(\*\^\_\_\^\*）**