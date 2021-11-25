##  redis分布式锁注解实现

- 需求

  - 支持spel表达式指定参数做key，参考@cacheable
  - 当执行业务时长大于锁时长能自动续期
  - 支持自定义重试次数，重试时间
  - 使用注解方式实现

- 实现思路

  - 采用jedis封装setnx，setpx上锁保证原子性
  - 将获取分布式锁成功的线程等相关信息存储放入队列中(先进先出)
  - 定义一个固定大小的线程池执行定时检测队列，当剩余三分之一有效期时进行续期操作

- 流程图

  

 **talk is cheap，show the code**

### 注解类

```java

/**
 * redis分布式锁注解
 * @author void
 * @create 2021-01-15
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface RedisLock {

    /**
     * 特定参数名 支持SPEL表达式
     */
    String key() default "business";
    /**
     * 失败重试次数
     */
    int tryCount() default 3;
    /**
     * 占用锁时长，单位ms
     */
    long lockTime() default 1000;

    /**
     * 重试等待时间 单位ms
     */
    int retryWait() default 200;
}

```

### 切面类

```java

/**
 * redis分布式锁切面
 * @author void
 * @create 2021-01-15
 */
@Slf4j
@Aspect
@Component
public class RedisLockAspect {

    private final ExpressionParser parser = new SpelExpressionParser();

    private final LocalVariableTableParameterNameDiscoverer discoverer = new LocalVariableTableParameterNameDiscoverer();

    /**
     * 线程池，维护keyAliveTime
     */
    private static final ScheduledExecutorService SCHEDULER = new ScheduledThreadPoolExecutor(1,
            new BasicThreadFactory.Builder().namingPattern("redisLock-schedule-pool").daemon(true).build());
    /**
     * 存储目前有效的key定义
     */
    private static final ConcurrentLinkedQueue<RedisLockHolder> HOLDER_LIST = new ConcurrentLinkedQueue();

    @Resource
    private RedisLockUtil redisLockUtil;

    {
        // 两秒执行一次「续时」操作
        SCHEDULER.scheduleAtFixedRate(() -> {
            Iterator<RedisLockHolder> iterator = HOLDER_LIST.iterator();
            while (iterator.hasNext()) {
                RedisLockHolder holder = iterator.next();
                log.debug("执行定时任务，持锁对象为:{}",holder);
                // 判空
                if (holder == null) {
                    iterator.remove();
                    continue;
                }
                // 判断 key 是否还有效，无效的话进行移除
                if (redisLockUtil.tryAcquire(holder.getBusinessKey()) == null) {
                    iterator.remove();
                    continue;
                }
                // 判断是否进入最后三分之一时间
                long curTime = System.currentTimeMillis();
                boolean shouldExtend = (holder.getLastModifyTime() + holder.getModifyPeriod()) <= curTime;
                if (shouldExtend) {
                    log.debug("开始为:{}锁续期",holder.getBusinessKey());
                    holder.setLastModifyTime(curTime);
                    redisLockUtil.tryRenew(holder.getBusinessKey(), holder.getLockTime(), TimeUnit.MILLISECONDS);
                }
            }
        }, 0, 2, TimeUnit.SECONDS);
    }

    @Pointcut("@annotation(redisLock)")
    public void aspect(RedisLock redisLock){}

    @Around(value = "aspect(redisLock)", argNames = "pjp,redisLock")
    public Object execute(ProceedingJoinPoint pjp, RedisLock redisLock) throws Throwable {
        MethodSignature methodSignature = (MethodSignature) pjp.getSignature();
        Method targetMethod = methodSignature.getMethod();
        //获取方法的参数值
        String className = methodSignature.getDeclaringType().getName();
        String keyName = redisLock.key();
        String businessKey;
        if (keyName.startsWith("#")){
            //根据spel表达式获取值
            EvaluationContext context = this.bindParam(targetMethod, pjp.getArgs());
            Expression expression = parser.parseExpression(keyName);
            Object value = expression.getValue(context);
            // key = 类名::参数名::参数值 
            businessKey = className + "::" + keyName + "::" + value;
        }else {
            // 非spel表达式 key = 类名::key
            businessKey = className + "::" + keyName;
        }
        String uniqueValue = UUID.randomUUID().toString();
        Object result;
        try {
            boolean isSuccess;
            for (int i = 0; i < redisLock.tryCount(); i++) {
              // 加锁
                isSuccess = redisLockUtil.tryLock(businessKey,uniqueValue,redisLock.lockTime());
                if (isSuccess){
                    break;
                }
                Thread.sleep(redisLock.retryWait());
                log.info("获取分布式锁:{}失败,开始重试，当前重试次数为:{},最大重试次数为:{}",businessKey,i+1,redisLock.tryCount());
                if (i == redisLock.tryCount()-1){
                    return ResponseResult.failed("当前系统繁忙，请稍后再试");
                }
            }
            HOLDER_LIST.add(new RedisLockHolder(businessKey, redisLock.lockTime(), System.currentTimeMillis(), Thread.currentThread()));
            result = pjp.proceed();
            // 线程被中断，抛出异常，中断此次请求
            if (Thread.currentThread().isInterrupted()) {
                throw new InterruptedException("You had been interrupted =-=");
            }
        } catch (Exception e) {
            log.error("has some error, please check again", e);
            return ResponseResult.failed("系统内部错误，请稍后再试");
        } finally {
            // 请求结束后，释放锁
            redisLockUtil.unLock(businessKey,uniqueValue);
        }
        return result;
    }

    /**
     * 将方法的参数名和参数值绑定
     *
     * @param method 方法，根据方法获取参数名
     * @param args   方法的参数值
     * @return
     */
    private EvaluationContext bindParam(Method method, Object[] args) {
        //获取方法的参数名
        String[] params = discoverer.getParameterNames(method);
        //将参数名与参数值对应起来
        EvaluationContext context = new StandardEvaluationContext();
        for (int len = 0; len < (params != null ? params.length : 0); len++) {
            context.setVariable(params[len], args[len]);
        }
        return context;
    }

}

```

### RedisLockHolder

```java

/**
 * @author void
 * @create 2021-01-15
 */
@Data
public class RedisLockHolder {

    /**
     * 业务唯一 key
     */
    private String businessKey;

    /**
     * 加锁时间 (秒 s)
     */
    private Long lockTime;

    /**
     * 上次更新时间（ms）
     */
    private Long lastModifyTime;

    /**
     * 保存当前线程
     */
    private Thread currentTread;

    /**
     * 更新的时间周期（毫秒）,公式 = 加锁时间（单位毫秒） / 3
     */
    private Long modifyPeriod;

    public RedisLockHolder(String businessKey, Long lockTime, Long lastModifyTime, Thread currentTread) {
        this.businessKey = businessKey;
        this.lockTime = lockTime;
        this.lastModifyTime = lastModifyTime;
        this.currentTread = currentTread;
        this.modifyPeriod = lockTime / 3;
    }
}

```

### redislockUtil

```java
/**
 * redis分布式锁
 * @author void
 * @create 2020-11-04
 */
@Component
public class RedisLockUtil {

    @Autowired
    private StringRedisTemplate redisTemplate;


    private static final Long RELEASE_SUCCESS = 1L;
    private static final String LOCK_SUCCESS = "OK";
    private static final String RELEASE_LOCK_SCRIPT = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";


    /**
     *加锁
     * @param key   加锁键
     * @param value  加锁客户端唯一标识(采用UUID)
     * @param expireTime   锁过期时间
     * @return
     */
    public boolean tryLock(String key, String value, long expireTime) {
        return redisTemplate.execute((RedisCallback<Boolean>) redisConnection -> {
            Jedis jedis = (Jedis) redisConnection.getNativeConnection();
            /**
             * key 锁，唯一
             * value 分布式锁要满足 解铃还须系铃人，通过给value赋值为requestId，我们就知道这把锁是哪个请求加的了，在解锁的时候就可以有依据。
             * SET_IF_NOT_EXIST  当key不存在时，我们进行set操作；若key已经存在，则不做任何操作；
             * SET_WITH_EXPIRE_TIME  过期时间单位, EX秒，PX毫秒
             * expireTime 过期时间
             * 设置key 和 有效期 是两步操作  放在这里能保证原子性 不会因为某一步失败发生死锁
             */
            SetParams params = new SetParams();
            params.px(expireTime);
            params.nx();
            String result = jedis.set(key, value, params);
            if (LOCK_SUCCESS.equals(result)) {
                return true;
            }
            return false;
        });
    }

    /**
     * 解锁
     * @param key
     * @param value
     * @return
     */
    public boolean unLock(String key, String value) {
        return redisTemplate.execute((RedisCallback<Boolean>) redisConnection -> {
            Jedis jedis = (Jedis) redisConnection.getNativeConnection();
            /**
             * RELEASE_LOCK_SCRIPT lua脚本 保证原子性 eval命令执行Lua代码的时候，Lua代码将被当成一个命令去执行，并且直到eval命令执行完成，Redis才会执行其他命令。
             */
            Object result = jedis.eval(RELEASE_LOCK_SCRIPT, Collections.singletonList(key),
                    Collections.singletonList(value));
            if (RELEASE_SUCCESS.equals(result)) {
                return true;
            }
            return false;
        });
    }

    /**
     * 尝试获取
     * @param key
     * @return
     */
    public Object tryAcquire(String key){
        return redisTemplate.opsForValue().get(key);
    }

    /**
     * 续期
     * @param key
     * @param time
     * @param timeUnit
     * @return
     */
    public boolean tryRenew(String key , long time, TimeUnit timeUnit){
        return redisTemplate.expire(key, time, timeUnit);
    }
```

### 总结

​	这里的分布式锁采用的是基于jedis自定义封装实现的，采用lua脚本语言来保证set和expire的原子性。更优的方案可以采用redisson分布式可重入锁。其包中的RLock实现了Lock接口，还提供了异步，反射性和RxJava2标准的接口。内部提供了一个监控锁的看门狗（watchdog）,默认的检查时间为30秒，可通过config.lockWatchdogTimeout来指定。加锁还提供了leaseTime来指定加锁时间。

[^**将来的你，一定会感激现在拼命的自己，愿自己与读者的开发之路无限美好。**]:

