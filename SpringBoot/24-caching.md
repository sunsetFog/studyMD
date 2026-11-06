# 24 · 缓存抽象与多级缓存设计

## 一、Spring Cache 注解

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(30))                  // 默认 TTL
                .serializeValuesWith(RedisSerializationContext.SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .computePrefixWith(name -> "cache:" + name + ":");

        Map<String, RedisCacheConfiguration> configs = new HashMap<>();
        configs.put("dict", config.entryTtl(Duration.ofHours(24)));     // 按缓存名单独 TTL
        configs.put("user", config.entryTtl(Duration.ofMinutes(10)));

        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .withInitialCacheConfigurations(configs)
                .transactionAware()                  // 缓存操作在事务提交后执行
                .build();
    }
}
```

```java
@Service
public class DictService {

    @Cacheable(cacheNames = "dict", key = "#type", unless = "#result == null")
    public List<DictDTO> listByType(String type) {
        return dictMapper.selectByType(type);
    }

    @CachePut(cacheNames = "dict", key = "#dto.type")     // 更新缓存（返回新值）
    public DictDTO update(DictDTO dto) { ... }

    @CacheEvict(cacheNames = "dict", key = "#type")
    public void delete(String type) { }

    @CacheEvict(cacheNames = "dict", allEntries = true)   // 清空整个缓存名
    public void reload() { }
}
```

注解语义：
- `@Cacheable`：先查缓存，miss 才执行方法并写缓存
- `@CachePut`：总是执行方法并更新缓存
- `@CacheEvict`：删除缓存（beforeInvocation 控制方法前/后）
- `unless`：返回值不缓存的条件（如 null 不缓存防穿透需另行处理）

## 二、注解失效场景（同 AOP/事务）

- 自调用（同类内部方法调用注解不生效）
- 方法非 public
- 缓存 key 用了复杂对象且未实现 hashCode/equals

## 三、多级缓存架构

```
浏览器/CDN → Nginx 缓存 → JVM 本地缓存（Caffeine） → Redis → DB
```

### Caffeine 本地缓存

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```java
Cache<String, List<DictDTO>> localCache = Caffeine.newBuilder()
        .maximumSize(10_000)                       // 最大条数（防 OOM）
        .expireAfterWrite(Duration.ofMinutes(5))   // 写后过期
        .recordStats()
        .build();

List<DictDTO> data = localCache.get(type,
        k -> dictMapper.selectByType(k));          // miss 时加载
```

### 二级缓存流程（以「查字典」为例）

```java
public List<DictDTO> getDict(String type) {
    // L1: 本地缓存（纳秒级，容量小，5 分钟）
    List<DictDTO> local = localCache.getIfPresent(type);
    if (local != null) return local;

    // L2: Redis（毫秒级，容量大，30 分钟 + 随机抖动）
    List<DictDTO> redis = redisUtil.get("dict:" + type, List.class);
    if (redis != null) {
        localCache.put(type, redis);
        return redis;
    }

    // L3: DB + 回填 + 防穿透（缓存空值）
    List<DictDTO> db = dictMapper.selectByType(type);
    long ttl = 30 + ThreadLocalRandom.current().nextLong(0, 5);   // 防雪崩抖动
    redisUtil.set("dict:" + type, db, Duration.ofMinutes(ttl));
    localCache.put(type, db);
    return db;
}
```

**本地缓存失效广播**（多实例一致性）：

```java
// 更新时：更新 DB → 删 Redis → 广播 MQ/Redis pub-sub → 各实例清本地缓存
redisTemplate.convertAndSend("cache:evict", "dict:" + type);

@Component
public class CacheEvictListener {
    @RedisListener(channel = "cache:evict")   // 或用 Spring Data Redis MessageListenerContainer
    public void onEvict(String key) { localCache.invalidate(key); }
}
```

## 四、缓存三大问题对策

| 问题 | 场景 | 对策 |
|---|---|---|
| 穿透 | 查询不存在的数据，绕过缓存打 DB | 缓存空值（短 TTL）/ 布隆过滤器 / 参数校验 |
| 雪崩 | 大量 key 同时过期 / Redis 宕机 | TTL 随机抖动 / 多级缓存 / 限流降级 |
| 击穿 | 热点 key 过期瞬间并发打 DB | 互斥锁重建 / 逻辑过期 / 热点永不过期 |

## 五、一致性策略

| 方案 | 说明 |
|---|---|
| Cache-Aside（主流） | 更新 DB → 删缓存。先库后缓 |
| 延迟双删 | 删缓存 → 更新库 → 延迟 500ms 再删一次 |
| binlog 订阅 | Canal 监听 MySQL binlog 删缓存，最终一致 |

**选型口诀**：一般业务「先库后删缓存」；强一致要求场景根本不该用缓存。

## 常见坑

- **坑 1**：@Cacheable 缓存 null → 需防穿透时反而要缓存空对象（包装类型判断）
- **坑 2**：缓存 key 冲突 → 规范 key 前缀 `业务:对象:id`
- **坑 3**：transactionAware 未开 → 事务未提交缓存已写入新值，读线程拿到「未来值」
- **坑 4**：Caffeine 不设 maximumSize → OOM
- **坑 5**：多实例本地缓存不一致 → 广播失效机制

## Checklist

- [ ] 统一 CacheManager 配置（序列化/前缀/TTL 分级）
- [ ] 热点小数据加 Caffeine 一级缓存
- [ ] 穿透/雪崩/击穿三问题各有对策
- [ ] 缓存更新策略统一「先库后删」
