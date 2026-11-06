# 26 · Spring Data Redis 实战

## 1. 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency><groupId>org.apache.commons</groupId><artifactId>commons-pool2</artifactId></dependency>
```

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      password:
      database: 0
      timeout: 3s                 # 命令执行超时
      connect-timeout: 3s
      lettuce:                    # 默认客户端（Boot 2+）
        pool:
          enabled: true
          max-active: 16
          max-idle: 8
          min-idle: 2
          max-wait: 3s
```

> Boot 默认用 **Lettuce**（基于 Netty，线程安全，支持异步/响应式）；Jedis 需排除 lettuce 后引入。

## 2. 序列化：必须改的默认配置

**默认 `JdkSerializationRedisSerializer`** 导致：值带 `AC ED` 乱码、跨语言不可用、可读性差。

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> t = new RedisTemplate<>();
        t.setConnectionFactory(factory);

        ObjectMapper om = new ObjectMapper();
        om.registerModule(new JavaTimeModule());
        om.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.activateDefaultTyping(LaissezFaireSubTypeValidator.instance,
                ObjectMapper.DefaultTyping.NON_FINAL, JsonTypeInfo.As.PROPERTY);  // ★ 保留类型信息
        Jackson2JsonRedisSerializer<Object> json = new Jackson2JsonRedisSerializer<>(om, Object.class);

        StringRedisSerializer str = new StringRedisSerializer();
        t.setKeySerializer(str);
        t.setHashKeySerializer(str);
        t.setValueSerializer(json);
        t.setHashValueSerializer(json);
        t.afterPropertiesSet();
        return t;
    }

    // 纯字符串场景用 StringRedisTemplate（Boot 已自动配置）
}
```

| 序列化器 | 优点 | 缺点 |
|---|---|---|
| `StringRedisSerializer` | 可读、通用 | 只能存字符串 |
| `Jackson2JsonRedisSerializer` | 跨语言、可读 | 需类型信息（多态） |
| `GenericJackson2JsonRedisSerializer` | 自带 `@class` 类型 | 值变大、有反序列化安全风险 |
| JDK | 无需配置 | 乱码、有漏洞 |

> **反序列化安全**：`activateDefaultTyping` 存在 RCE 风险，务必限制可反序列化的基类白名单，或干脆不用默认类型（用具体类型的序列化器）。

## 3. 五种数据结构操作

```java
// String
redis.opsForValue().set("k", v, Duration.ofMinutes(10));
redis.opsForValue().setIfAbsent("lock", "1", Duration.ofSeconds(30));   // 分布式锁基础
redis.opsForValue().increment("counter", 1);                            // 计数

// Hash（适合对象、购物车）
redis.opsForHash().put("cart:1", "sku:100", 2);
redis.opsForHash().increment("cart:1", "sku:100", 1);
Map<Object,Object> all = redis.opsForHash().entries("cart:1");

// List（队列/栈）
redis.opsForList().rightPush("queue", "task1");
String t = redis.opsForList().leftPop("queue", Duration.ofSeconds(5));  // 阻塞

// Set（去重、标签、共同好友）
redis.opsForSet().add("tags:1", "a", "b");
redis.opsForSet().intersect("tags:1", "tags:2");

// ZSet（排行榜、延迟队列）
redis.opsForZSet().add("rank", "user1", 100);
redis.opsForZSet().reverseRange("rank", 0, 9);                          // Top10
redis.opsForZSet().incrementScore("rank", "user1", 5);

// 通用
redis.hasKey("k"); redis.delete("k");
redis.expire("k", 60, TimeUnit.SECONDS);
redis.getExpire("k");
redis.keys("prefix:*");      // ★ 生产禁用，用 scan
```

## 4. Spring Cache 抽象

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory f) {
        RedisCacheConfiguration cfg = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))
                .serializeKeysWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .disableCachingNullValues();        // ★ 不缓存 null，防穿透
        return RedisCacheManager.builder(f)
                .cacheDefaults(cfg)
                .withInitialCacheConfigurations(Map.of("user", cfg.entryTtl(Duration.ofHours(2))))
                .transactionAware()
                .build();
    }
}
```

```java
@Cacheable(cacheNames = "user", key = "#id", unless = "#result == null")
public User getById(Long id) { ... }

@CachePut(cacheNames = "user", key = "#user.id")
public User update(User user) { ... }

@CacheEvict(cacheNames = "user", key = "#id")
public void delete(Long id) { ... }

@CacheEvict(cacheNames = "user", allEntries = true)
public void clearAll() { ... }

@Caching(evict = {@CacheEvict(...), @CacheEvict(...)})
```

| 注解 | 时机 | 用途 |
|---|---|---|
| `@Cacheable` | 方法前先查缓存 | 查询 |
| `@CachePut` | 总是执行方法并写缓存 | 更新 |
| `@CacheEvict` | 删缓存 | 删除 |
| `@CacheConfig` | 类级公共配置 | |
| `@Caching` | 组合多个 | |

### key 生成与 SpEL

```java
key = "#user.id + ':' + #user.type"
keyGenerator = "myKeyGenerator"
#root.methodName、#root.target、#result、#p0、#a0
```

## 5. 缓存三大问题（详见 [27](27-redis-patterns.md)）

| 问题 | 描述 | 方案 |
|---|---|---|
| 穿透 | 查不存在的 key | 布隆过滤器 / 缓存空值（短 TTL） |
| 击穿 | 热点 key 过期瞬间 | 互斥锁 / 逻辑过期 |
| 雪崩 | 大量 key 同时过期 | TTL 加随机值 / 多级缓存 / 熔断 |

## 6. Redisson（分布式对象）

```xml
<dependency><groupId>org.redisson</groupId><artifactId>redisson-spring-boot-starter</artifactId>
           <version>3.32.0</version></dependency>
```

```java
RLock lock = redisson.getLock("order:" + id);
lock.tryLock(10, 30, TimeUnit.SECONDS);
// 其他：RMapCache、RDelayedQueue（延迟队列）、RRateLimiter（限流）、RBloomFilter、RSemaphore
```

## 7. Pipeline / 批量

```java
// 批量写（pipeline）
redis.executePipelined((RedisCallback<Object>) con -> {
    for (int i = 0; i < 1000; i++) con.stringCommands().set(("k"+i).getBytes(), ("v"+i).getBytes());
    return null;
});

// MSET / MGET
redis.opsForValue().multiGet(List.of("k1","k2"));
```

> 大批量（万级）用 pipeline，网络往返从 N 次降到 1 次。

## 8. Lua 脚本（保证原子性）

```java
String script = """
    if redis.call('get', KEYS[1]) == ARGV[1] then
        return redis.call('del', KEYS[1])
    else
        return 0
    end
    """;
DefaultRedisScript<Long> rs = new DefaultRedisScript<>(script, Long.class);
redis.execute(rs, List.of("lock:key"), "uuid-value");       // 解锁（只能解自己的锁）
```

## 9. 连接池与超时

| 参数 | 建议 |
|---|---|
| `max-active` | 8~32（过大反而增加 Redis 负担） |
| `max-wait` | 1~3s（快速失败优于无限等待） |
| `timeout` | 2~5s（命令超时） |
| `min-idle` | 2~4 |

> 连接数过大 → Redis 服务端 `maxclients` 打满。公式参考：`maxTotal ≈ QPS × RT(s)`。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 值乱码 `\xAC\xED` | JDK 序列化 | 改 JSON/String 序列化 |
| `keys *` 卡死 | 单线程阻塞 | 用 `SCAN` |
| 大 key | 阻塞、网络打满 | 拆分（hash 分片）、压缩、避免存大集合 |
| 热 key | 单分片被打爆 | 本地缓存 + 随机前缀多副本 |
| 缓存 null 未处理 | 穿透 | `@Cacheable(unless="#result==null")` 或缓存空值 |
| 事务回滚但缓存已改 | 不一致 | `@TransactionalEventListener(AFTER_COMMIT)` 清缓存；或 `transactionAware()` |
| Redis 超时导致接口雪崩 | 全部超时 | 熔断降级 + 超时调小 + 连接池上限 |
| `READONLY` 报错 | 主从切换 | 使用哨兵/集群模式自动感知 |
| 反序列化 `ClassCastException` | LinkedHashMap 强转失败 | 用具体类型序列化器或手动 convert |

## 11. 面试速答

- **Lettuce vs Jedis**：Lettuce 基于 Netty，连接可多线程共享、支持异步/响应式，是 Boot 默认；Jedis 直连、线程不安全需池化。
- **为什么改序列化**：默认 JDK 序列化产生二进制乱码、无法跨语言、可读性与兼容性差。
- **Spring Cache 的三个注解**：`@Cacheable`（查）、`@CachePut`（写）、`@CacheEvict`（删）；自调用同样失效。
- **缓存一致性方案**：`Cache Aside`（先更 DB 再删缓存）+ 延迟双删 + 事务提交后删缓存；强一致用 MQ 重试或 Canal 订阅 binlog。
