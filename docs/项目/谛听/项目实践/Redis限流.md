## 理论基础
面对越来越多的高并发场景，限流显示的尤为重要。当然，限流有许多种实现的方式，Redis具有很强大的功能，我用Redis实践了三种的实现方式，可以较为简单的实现其方式。Redis不仅仅是可以做限流，还可以做数据统计，附近的人等功能，这些可能会后续写到。
### 第一种：基于Redis的setnx的操作
我们在使用Redis的分布式锁的时候，大家都知道是依靠了setnx的指令，在CAS（Compare and swap）的操作的时候，同时给指定的key设置了过期实践（expire），我们在限流的主要目的就是为了在单位时间内，有且仅有N数量的请求能够访问我的代码程序。所以依靠setnx可以很轻松的做到这方面的功能。比如我们需要在10秒内限定20个请求，那么我们在setnx的时候可以设置过期时间10，当请求的setnx数量达到20时候即达到了限流效果。代码比较简单就不做展示了。当然这种做法的弊端是很多的，比如当统计1-10秒的时候，无法统计2-11秒之内，如果需要统计N秒内的M个请求，那么我们的Redis中需要保持N个key等等问题
### 第二种：基于Redis的数据结构zset
其实限流涉及的最主要的就是滑动窗口，上面也提到1-10怎么变成2-11。其实也就是起始值和末端值都各+1即可。
而我们如果用Redis的list数据结构可以轻而易举的实现该功能
我们可以将请求打造成一个zset数组，当每一次请求进来的时候，value保持唯一，可以用UUID生成，而score可以用当前时间戳表示，因为score我们可以用来计算当前时间戳之内有多少的请求数量。而zset数据结构也提供了range方法让我们可以很轻易的获取到2个时间戳内有多少请求
代码如下
```java
public Response limitFlow(){
    Long currentTime = new Date().getTime();
    System.out.println(currentTime);
    if(redisTemplate.hasKey("limit")) {
        Integer count = redisTemplate.opsForZSet().rangeByScore("limit", currentTime -  intervalTime, currentTime).size();        // intervalTime是限流的时间 
        System.out.println(count);
        if (count != null && count > 5) {
            return Response.ok("每分钟最多只能访问5次");
        }
    }
    redisTemplate.opsForZSet().add("limit",UUID.randomUUID().toString(),currentTime);
    return Response.ok("访问成功");
}
```
通过上述代码可以做到滑动窗口的效果，并且能保证每N秒内至多M个请求，缺点就是zset的数据结构会越来越大。实现方式相对也是比较简单的。
### 第三种：基于Redis的令牌桶算法
提到限流就不得不提到令牌桶算法了。
令牌桶算法提及到输入速率和输出速率，当输出速率大于输入速率，那么就是超出流量限制了。
也就是说我们每访问一次请求的时候，可以从Redis中获取一个令牌，如果拿到令牌了，那就说明没超出限制，而如果拿不到，则结果相反。
依靠上述的思想，我们可以结合Redis的List数据结构很轻易的做到这样的代码，只是简单实现
依靠List的leftPop来获取令牌
```
// 输出令牌
public Response limitFlow2(Long id){
        Object result = redisTemplate.opsForList().leftPop("limit_list");
        if(result == null){
            return Response.ok("当前令牌桶中无令牌");
        }
        return Response.ok(articleDescription2);
    }
```
再依靠Java的定时任务，定时往List中rightPush令牌，当然令牌也需要唯一性，所以我这里还是用UUID进行了生成
```
// 10S的速率往令牌桶中添加UUID，只为保证唯一性
@Scheduled(fixedDelay = 10_000,initialDelay = 0)
public void setIntervalTimeTask(){
    redisTemplate.opsForList().rightPush("limit_list",UUID.randomUUID().toString());
}
```

## 项目实战
### 限流策略
```java
@Slf4j
public abstract class AbstractFrequencyControlService<K extends FrequencyControlDTO> {

    @PostConstruct
    protected void registerMyselfToFactory() {
        FrequencyControlStrategyFactory.registerFrequencyController(getStrategyName(), this);
    }

    /**
     * @param frequencyControlMap 定义的注解频控 Map中的Key-对应redis的单个频控的Key Map中的Value-对应redis的单个频控的Key限制的Value
     * @param supplier            函数式入参-代表每个频控方法执行的不同的业务逻辑
     * @return 业务方法执行的返回值
     * @throws Throwable
     */
    private <T> T executeWithFrequencyControlMap(Map<String, K> frequencyControlMap, SupplierThrowWithoutParam<T> supplier) throws Throwable {
        if (reachRateLimit(frequencyControlMap)) {
            throw new FrequencyControlException(CommonErrorEnum.FREQUENCY_LIMIT);
        }
        try {
            return supplier.get();
        } finally {
            //不管成功还是失败，都增加次数
            addFrequencyControlStatisticsCount(frequencyControlMap);
        }
    }


    /**
     * 多限流策略的编程式调用方法 无参的调用方法
     *
     * @param frequencyControlList 频控列表 包含每一个频率控制的定义以及顺序
     * @param supplier             函数式入参-代表每个频控方法执行的不同的业务逻辑
     * @return 业务方法执行的返回值
     * @throws Throwable 被限流或者限流策略定义错误
     */
    @SuppressWarnings("unchecked")
    public <T> T executeWithFrequencyControlList(List<K> frequencyControlList, SupplierThrowWithoutParam<T> supplier) throws Throwable {
        boolean existsFrequencyControlHasNullKey = frequencyControlList.stream().anyMatch(frequencyControl -> ObjectUtils.isEmpty(frequencyControl.getKey()));
        AssertUtil.isFalse(existsFrequencyControlHasNullKey, "限流策略的Key字段不允许出现空值");
        Map<String, K> frequencyControlDTOMap = frequencyControlList.stream().collect(Collectors.groupingBy(K::getKey, Collectors.collectingAndThen(Collectors.toList(), list -> list.get(0))));
        return executeWithFrequencyControlMap(frequencyControlDTOMap, supplier);
    }

    /**
     * 单限流策略的调用方法-编程式调用
     *
     * @param frequencyControl 单个频控对象
     * @param supplier         服务提供着
     * @return 业务方法执行结果
     * @throws Throwable
     */
    public <T> T executeWithFrequencyControl(K frequencyControl, SupplierThrowWithoutParam<T> supplier) throws Throwable {
        return executeWithFrequencyControlList(Collections.singletonList(frequencyControl), supplier);
    }


    @FunctionalInterface
    public interface SupplierThrowWithoutParam<T> {

        /**
         * Gets a result.
         *
         * @return a result
         */
        T get() throws Throwable;
    }

    @FunctionalInterface
    public interface Executor {

        /**
         * Gets a result.
         *
         * @return a result
         */
        void execute() throws Throwable;
    }

    /**
     * 是否达到限流阈值 子类实现 每个子类都可以自定义自己的限流逻辑判断
     *
     * @param frequencyControlMap 定义的注解频控 Map中的Key-对应redis的单个频控的Key Map中的Value-对应redis的单个频控的Key限制的Value
     * @return true-方法被限流 false-方法没有被限流
     */
    protected abstract boolean reachRateLimit(Map<String, K> frequencyControlMap);

    /**
     * 增加限流统计次数 子类实现 每个子类都可以自定义自己的限流统计信息增加的逻辑
     *
     * @param frequencyControlMap 定义的注解频控 Map中的Key-对应redis的单个频控的Key Map中的Value-对应redis的单个频控的Key限制的Value
     */
    protected abstract void addFrequencyControlStatisticsCount(Map<String, K> frequencyControlMap);

    /**
     * 获取策略名称
     *
     * @return 策略名称
     */
    protected abstract String getStrategyName();

}
```
#### 计数法
```java
@Slf4j
@Service
public class TotalCountWithInFixTimeFrequencyController extends AbstractFrequencyControlService<FixedWindowDTO> {


    /**
     * 是否达到限流阈值 子类实现 每个子类都可以自定义自己的限流逻辑判断
     *
     * @param frequencyControlMap 定义的注解频控 Map中的Key-对应redis的单个频控的Key Map中的Value-对应redis的单个频控的Key限制的Value
     * @return true-方法被限流 false-方法没有被限流
     */
    @Override
    protected boolean reachRateLimit(Map<String, FixedWindowDTO> frequencyControlMap) {
        //批量获取redis统计的值
        List<String> frequencyKeys = new ArrayList<>(frequencyControlMap.keySet());
        List<Integer> countList = RedisUtils.mget(frequencyKeys, Integer.class);
        for (int i = 0; i < frequencyKeys.size(); i++) {
            String key = frequencyKeys.get(i);
            Integer count = countList.get(i);
            int frequencyControlCount = frequencyControlMap.get(key).getCount();
            if (Objects.nonNull(count) && count >= frequencyControlCount) {
                //频率超过了
                log.warn("frequencyControl limit key:{},count:{}", key, count);
                return true;
            }
        }
        return false;
    }

    /**
     * 增加限流统计次数 子类实现 每个子类都可以自定义自己的限流统计信息增加的逻辑
     *
     * @param frequencyControlMap 定义的注解频控 Map中的Key-对应redis的单个频控的Key Map中的Value-对应redis的单个频控的Key限制的Value
     */
    @Override
    protected void addFrequencyControlStatisticsCount(Map<String, FixedWindowDTO> frequencyControlMap) {
        frequencyControlMap.forEach((k, v) -> RedisUtils.inc(k, v.getTime(), v.getUnit()));
    }

    @Override
    protected String getStrategyName() {
        return FrequencyControlConstant.TOTAL_COUNT_WITH_IN_FIX_TIME;
    }
}
```
#### 滑动窗法
```java
@Slf4j
@Service
public class SlidingWindowFrequencyController extends AbstractFrequencyControlService<SlidingWindowDTO> {
    @Override
    protected boolean reachRateLimit(Map<String, SlidingWindowDTO> frequencyControlMap) {
        // 批量获取redis统计的值
        List<String> frequencyKeys = new ArrayList<>(frequencyControlMap.keySet());
        for (int i = 0; i < frequencyKeys.size(); i++) {
            String key = frequencyKeys.get(i);
            SlidingWindowDTO controlDTO = frequencyControlMap.get(key);
            // 获取窗口时间内计数
            Long count = RedisUtils.ZSetGet(key);
            int frequencyControlCount = controlDTO.getCount();
            if (Objects.nonNull(count) && count >= frequencyControlCount) {
                //频率超过了
                log.warn("frequencyControl limit key:{},count:{}", key, count);
                return true;
            }
        }
        return false;
    }

    @Override
    protected void addFrequencyControlStatisticsCount(Map<String, SlidingWindowDTO> frequencyControlMap) {
        List<String> frequencyKeys = new ArrayList<>(frequencyControlMap.keySet());
        for (int i = 0; i < frequencyKeys.size(); i++) {
            String key = frequencyKeys.get(i);
            SlidingWindowDTO controlDTO = frequencyControlMap.get(key);
            // 窗口最小周期转秒
            long period = controlDTO.getUnit().toMillis(controlDTO.getPeriod());
            long current = System.currentTimeMillis();
            // 窗口大小 单位 秒
            long length = period * controlDTO.getWindowSize();
            long start = current - length;
//            long expireTime = length + period;
            RedisUtils.ZSetAddAndExpire(key, start, length, current);
        }
    }

    @Override
    protected String getStrategyName() {
        return FrequencyControlConstant.SLIDING_WINDOW;

    }
}
```
#### 令牌桶
```java
@Slf4j
@Service
public class TokenBucketFrequencyController extends AbstractFrequencyControlService<TokenBucketDTO> {

    @Autowired
    private TokenBucketManager tokenBucketManager;

    @Override
    protected boolean reachRateLimit(Map<String, TokenBucketDTO> frequencyControlMap) {
        // 批量获取redis统计的值
        List<String> frequencyKeys = new ArrayList<>(frequencyControlMap.keySet());
        for (int i = 0; i < frequencyKeys.size(); i++) {
            String key = frequencyKeys.get(i);
            // 获取 1 个令牌
            return tokenBucketManager.tryAcquire(key, 1);
        }
        return false;
    }

    @Override
    protected void addFrequencyControlStatisticsCount(Map<String, TokenBucketDTO> frequencyControlMap) {
        List<String> frequencyKeys = new ArrayList<>(frequencyControlMap.keySet());
        for (int i = 0; i < frequencyKeys.size(); i++) {
            String key = frequencyKeys.get(i);
            TokenBucketDTO tokenBucketDTO = frequencyControlMap.get(key);
            tokenBucketManager.createTokenBucket(key, tokenBucketDTO.getCapacity(), tokenBucketDTO.getRefillRate());
            // 扣减 1 个令牌
            tokenBucketManager.deductionToken(key, 1);
        }
    }

    @Override
    protected String getStrategyName() {
        return FrequencyControlConstant.TOKEN_BUCKET;
    }
}

```
```java
// 令牌桶算法的本地实现
@Component
public class TokenBucketManager {

    private final Map<String, TokenBucketDTO> tokenBucketMap = new ConcurrentHashMap<>();
    private final ReentrantLock lock = new ReentrantLock();

    public void createTokenBucket(String key, long capacity, double refillRate) {
        lock.lock();
        try {
            if (!tokenBucketMap.containsKey(key)) {
                TokenBucketDTO tokenBucket = new TokenBucketDTO(capacity, refillRate);
                tokenBucketMap.put(key, tokenBucket);
            }
        } finally {
            lock.unlock();
        }
    }

    public void removeTokenBucket(String key) {
        lock.lock();
        try {
            tokenBucketMap.remove(key);
        } finally {
            lock.unlock();
        }
    }

    public boolean tryAcquire(String key, int permits) {
        TokenBucketDTO tokenBucket = tokenBucketMap.get(key);
        if (tokenBucket != null) {
            return tokenBucket.tryAcquire(permits);
        }
        return false;
    }

    public void deductionToken(String key, int permits) {
        TokenBucketDTO tokenBucket = tokenBucketMap.get(key);
        if (tokenBucket != null) {
            tokenBucket.deductionToken(permits);
        }
    }
}

```
### 基础注解
```java
@Repeatable(FrequencyControlContainer.class) // 可重复
@Retention(RetentionPolicy.RUNTIME)// 运行时生效
@Target(ElementType.METHOD)//作用在方法上
public @interface FrequencyControl {
    /**
     * 策略名
     */
    String strategy() default FrequencyControlConstant.TOTAL_COUNT_WITH_IN_FIX_TIME;

    /**
     * key的前缀，默认取方法全限定名，除非我们在不同方法上对同一个资源做频控，就自己指定
     *
     * @return key的前缀
     */
    String prefixKey() default "";

    /**
     * 频控对象，默认el表达指定具体的频控对象
     * 对于ip 和uid模式，需要是http入口的对象，保证RequestHolder里有值
     *
     * @return 对象
     */
    Target target() default Target.EL;

    /**
     * springEl 表达式，target=EL必填
     *
     * @return 表达式
     */
    String spEl() default "";

    /**
     * 频控时间范围，默认单位秒
     *
     * @return 时间范围
     */
    int time() default 10;

    /**
     * 频控时间单位，默认秒
     *
     * @return 单位
     */
    TimeUnit unit() default TimeUnit.SECONDS;

    /**
     * 单位时间内最大访问次数
     *
     * @return 次数
     */
    int count() default 1;

    /**
     * 窗口大小，默认 5 个 period
     */
    int windowSize() default 5;

    /**
     * 窗口最小周期 1s (窗口大小是 5s， 1s一个小格子，共10个格子)
     */
    int period() default 1;

    long capacity() default 3; // 令牌桶容量

    double refillRate() default 0.5; // 每秒补充的令牌数

    enum Target {
    UID,
    IP,
    EL
}
}

```
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)// 运行时生效
public @interface FrequencyControlContainer {
    FrequencyControl[] value();
}

```
### 切面
```java
@Slf4j
@Aspect
@Component
public class FrequencyControlAspect {
    @Around("@annotation(com.linyi.annotation.FrequencyControl)||@annotation(com.linyi.annotation.FrequencyControlContainer)")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        FrequencyControl[] annotationsByType = method.getAnnotationsByType(FrequencyControl.class);
        Map<String, FrequencyControl> keyMap = new HashMap<>();
        String strategy = FrequencyControlConstant.TOTAL_COUNT_WITH_IN_FIX_TIME;
        for (int i = 0; i < annotationsByType.length; i++) {
            // 获取频控注解
            FrequencyControl frequencyControl = annotationsByType[i];
            String prefix = StrUtil.isBlank(frequencyControl.prefixKey()) ? /* 默认方法限定名 + 注解排名（可能多个）*/method.toGenericString() + ":index:" + i : frequencyControl.prefixKey();
            String key = "";
            switch (frequencyControl.target()) {
                case EL:
                    key = SpElUtils.parseSpEl(method, joinPoint.getArgs(), frequencyControl.spEl());
                    break;
                case IP:
                    key = RequestHolder.get().getIp();
                    break;
                case UID:
                    key = RequestHolder.get().getUid().toString();
            }
            keyMap.put(prefix + ":" + key, frequencyControl);
            strategy = frequencyControl.strategy();
        }
        // 将注解的参数转换为编程式调用需要的参数
        if (FrequencyControlConstant.TOTAL_COUNT_WITH_IN_FIX_TIME.equals(strategy)) {
            // 调用编程式注解 固定窗口
            List<FrequencyControlDTO> frequencyControlDTOS = keyMap.entrySet().stream().map(entrySet -> buildFixedWindowDTO(entrySet.getKey(), entrySet.getValue())).collect(Collectors.toList());
            return FrequencyControlUtil.executeWithFrequencyControlList(strategy, frequencyControlDTOS, joinPoint::proceed);

        } else if (FrequencyControlConstant.TOKEN_BUCKET.equals(strategy)) {
            // 调用编程式注解 令牌桶
            List<TokenBucketDTO> frequencyControlDTOS = keyMap.entrySet().stream().map(entrySet -> buildTokenBucketDTO(entrySet.getKey(), entrySet.getValue())).collect(Collectors.toList());
            return FrequencyControlUtil.executeWithFrequencyControlList(strategy, frequencyControlDTOS, joinPoint::proceed);
        } else {
            // 调用编程式注解 滑动窗口
            List<SlidingWindowDTO> frequencyControlDTOS = keyMap.entrySet().stream().map(entrySet -> buildSlidingWindowFrequencyControlDTO(entrySet.getKey(), entrySet.getValue())).collect(Collectors.toList());
            return FrequencyControlUtil.executeWithFrequencyControlList(strategy, frequencyControlDTOS, joinPoint::proceed);
        }
    }

    /**
     * 将注解参数转换为编程式调用所需要的参数
     *
     * @param key              频率控制Key
     * @param frequencyControl 注解
     * @return 编程式调用所需要的参数-FrequencyControlDTO
     */
    private SlidingWindowDTO buildSlidingWindowFrequencyControlDTO(String key, FrequencyControl frequencyControl) {
        SlidingWindowDTO frequencyControlDTO = new SlidingWindowDTO();
        frequencyControlDTO.setWindowSize(frequencyControl.windowSize());
        frequencyControlDTO.setPeriod(frequencyControl.period());
        frequencyControlDTO.setCount(frequencyControl.count());
        frequencyControlDTO.setUnit(frequencyControl.unit());
        frequencyControlDTO.setKey(key);
        return frequencyControlDTO;
    }

    /**
     * 将注解参数转换为编程式调用所需要的参数
     *
     * @param key              频率控制Key
     * @param frequencyControl 注解
     * @return 编程式调用所需要的参数-FrequencyControlDTO
     */
    private TokenBucketDTO buildTokenBucketDTO(String key, FrequencyControl frequencyControl) {
        TokenBucketDTO tokenBucketDTO = new TokenBucketDTO(frequencyControl.capacity(), frequencyControl.refillRate());
        tokenBucketDTO.setKey(key);
        return tokenBucketDTO;
    }

    /**
     * 将注解参数转换为编程式调用所需要的参数
     *
     * @param key              频率控制Key
     * @param frequencyControl 注解
     * @return 编程式调用所需要的参数-FrequencyControlDTO
     */
    private FixedWindowDTO buildFixedWindowDTO(String key, FrequencyControl frequencyControl) {
        FixedWindowDTO fixedWindowDTO = new FixedWindowDTO();
        fixedWindowDTO.setCount(frequencyControl.count());
        fixedWindowDTO.setTime(frequencyControl.time());
        fixedWindowDTO.setUnit(frequencyControl.unit());
        fixedWindowDTO.setKey(key);
        return fixedWindowDTO;
    }
}

```
## 参考资料
[Redis 实现限流的三种方式 - 掘金](https://juejin.cn/post/7033646189845151757)
