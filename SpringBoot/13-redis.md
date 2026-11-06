# 13 · Redis 集成

## 一、依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<!-- 连接池需要 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password: 123456
      database: 0
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 2
          max-wait: 3000ms
```

## 二、RedisTemplate 配置（序列化是重点）

默认 JdkSerializationRedisSerializer 存出来的 key 是乱码且不可读——**必须自定义**：

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        GenericJackson2JsonRedisSerializer jsonSer =
                new GenericJackson2JsonRedisSerializer();
        StringRedisSerializer stringSer = new StringRedisSerializer();

        template.setKeySerializer(stringSer);        // key 用字符串
        template.setValueSerializer(jsonSer);        // value 用 JSON
        template.setHashKeySerializer(stringSer);
        template.setHashValueSerializer(jsonSer);
        template.afterPropertiesSet();
        return template;
    }
}
```

## 三、核心操作封装

```java
@Component
@RequiredArgsConstructor
public class RedisUtil {

    private final RedisTemplate<String, Object> redisTemplate;

    public void set(String key, Object value, Duration ttl) {
        redisTemplate.opsForValue().set(key, value, ttl);
    }

    @SuppressWarnings("unchecked")
    public <T> T get(String key, Class<T> clazz) {
        return (T) redisTemplate.opsForValue().get(key);
    }

    public boolean setIfAbsent(String key, Object value, Duration ttl) {  // 分布式锁基础
        return Boolean.TRUE.equals(redisTemplate.opsForValue().setIfAbsent(key, value, ttl));
    }

    public Long incr(String key, long delta) {
        return redisTemplate.opsForValue().increment(key, delta);
    }
}
```

常用结构对应：

| 方法族 | Redis 结构 |
|---|---|
| opsForValue | String |
| opsForHash | Hash |
| opsForList | List |
| opsForSet | Set |
| opsForZSet | ZSet（排行榜） |

## 四、典型场景

### 1. 缓存旁路（Cache-Aside）

```java
public UserDTO getById(Long id) {
    String key = "user:" + id;
    UserDTO cached = redisUtil.get(key, UserDTO.class);
    if (cached != null) return cached;

    UserDTO db = userService.getById(id);
    if (db != null) redisUtil.set(key, db, Duration.ofMinutes(30));   // 必设 TTL
    return db;
}

public void update(UserDTO dto) {
    userService.update(dto);
    redisTemplate.delete("user:" + dto.getId());   // 先更新库，再删缓存
}
```

### 2. 分布式锁（setnx + 原子续期）

```java
public boolean tryLock(String key, String requestId, Duration ttl) {
    return redisUtil.setIfAbsent(key, requestId, ttl);
}

public void unlock(String key, String requestId) {
    // Lua 保证"判断持有者+删除"原子性
    String script = """
        if redis.call('get', KEYS[1]) == ARGV[1] then
            return redis.call('del', KEYS[1])
        else
            return 0
        end
        """;
    redisTemplate.execute(new DefaultRedisScript<>(script, Long.class),
            List.of(key), requestId);
}
```

> 生产推荐直接用 **Redisson**：看门狗自动续期、可重入、RedLock。

### 3. 全局唯一 ID / 计数 / 限流

```java
// 限流：1 分钟内最多 100 次
String key = "rate:login:" + userId;
Long count = redisTemplate.opsForValue().increment(key);
if (count != null && count == 1) redisTemplate.expire(key, Duration.ofMinutes(1));
if (count != null && count > 100) throw new BusinessException(429, "请求过于频繁");
```

### 4. 排行榜（ZSet）

```java
redisTemplate.opsForZSet().incrementScore("rank:score", "user:1", 10);
Set<ZSetOperations.TypedTuple<Object>> top =
        redisTemplate.opsForZSet().reverseRangeWithScores("rank:score", 0, 9);
```

## 五、Spring Cache 注解（衔接 24 篇）

```java
@Cacheable(cacheNames = "user", key = "#id")
public UserDTO getById(Long id) { ... }

@CacheEvict(cacheNames = "user", key = "#dto.id")
public void update(UserDTO dto) { ... }
```

## 常见坑

- **坑 1**：不配序列化 → key 前缀乱码、跨服务不可读
- **坑 2**：缓存不设 TTL → 内存打满；确需永久数据单独评估
- **坑 3**：先删缓存再更新库 → 并发下旧值回填；采用「更新库 → 删缓存」+ 延迟双删/订阅 binlog
- **坑 4**：缓存穿透（查不存在的 id）→ 缓存空值或布隆过滤器；缓存雪崩 → TTL 加随机抖动
- **坑 5**：@Transactional 方法内写缓存 → 事务未提交缓存已被读，或回滚后缓存残留 → 事务提交后再删缓存（TransactionSynchronization）
- **坑 6**：keys * 生产使用 → 阻塞；用 scan

## Checklist

- [ ] RedisTemplate 统一序列化配置
- [ ] 所有缓存 key 有命名规范（`业务:对象:id`）与 TTL
- [ ] 分布式锁用 Redisson 或原子 Lua
- [ ] 穿透/雪崩/击穿三问题有对应策略
