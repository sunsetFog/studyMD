# 28 · Spring Data MongoDB

## 1. 定位与依赖

MongoDB：文档型 NoSQL，BSON 存储，**schema-free**、支持二级索引、聚合管道、副本集高可用、分片。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://user:pass@host1:27017,host2:27017/dbname?replicaSet=rs0&authSource=admin
      # 或 host/port/database/username/password
      auto-index-creation: false     # 生产建议关闭，手工建索引
```

## 2. 文档建模

```java
@Document(collection = "orders")        // 对应集合
@Data
public class Order {
    @Id private String id;              // ObjectId，String 即可
    @Indexed(unique = true) private String orderNo;
    @Field("amt") private BigDecimal amount;
    @Indexed private Long userId;
    @Indexed(expireAfterSeconds = 3600) private Date expireAt;   // TTL 索引
    @CreatedDate private LocalDateTime createTime;
    @Version private Long version;      // 乐观锁

    @DocumentReference(lazy = true) private User user;           // 引用（4.x）
    private List<OrderItem> items;      // ★ 内嵌（反范式，MongoDB 推荐）
}
```

### 建模原则：内嵌 vs 引用

| 场景 | 选择 |
|---|---|
| 一对一 / 一对少、常一起读、子文档不独立查询 | **内嵌**（Embed） |
| 一对多且子文档独立访问、数据量无上限 | **引用**（Reference） |
| 需要多文档事务 | MongoDB 4.0+ 支持副本集多文档事务 |

> MongoDB 的核心思想：**按查询方式设计文档**，而不是按关系范式拆分。

## 3. 两种访问方式

### MongoTemplate（灵活）

```java
@Repository
@RequiredArgsConstructor
public class OrderDao {
    private final MongoTemplate mongo;

    public Order save(Order o) { return mongo.save(o); }

    public List<Order> findByUser(Long userId) {
        Query q = Query.query(Criteria.where("userId").is(userId))
                       .with(Sort.by(Sort.Direction.DESC, "createTime"))
                       .limit(20);
        return mongo.find(q, Order.class);
    }

    public void updateAmount(String id, BigDecimal amt) {
        mongo.updateFirst(Query.query(Criteria.where("_id").is(id)),
                          Update.update("amount", amt).currentDate("updateTime"), Order.class);
    }

    // 聚合
    public List<StatVO> statByDay() {
        Aggregation agg = Aggregation.newAggregation(
            Aggregation.match(Criteria.where("status").is("PAID")),
            Aggregation.project("amount").andExpression("{$dateToString:{format:'%Y-%m-%d',date:'$createTime'}}").as("day"),
            Aggregation.group("day").sum("amount").as("total").count().as("cnt"),
            Aggregation.sort(Sort.Direction.DESC, "day"),
            Aggregation.limit(30));
        return mongo.aggregate(agg, "orders", StatVO.class).getMappedResults();
    }

    // 原子自增
    public long nextSeq(String name) {
        Query q = Query.query(Criteria.where("_id").is(name));
        Update u = new Update().inc("seq", 1);
        return mongo.findAndModify(q, u, FindAndModifyOptions.options().returnNew(true).upsert(true),
                                   Seq.class).getSeq();
    }
}
```

### Repository（简单）

```java
public interface OrderRepository extends MongoRepository<Order, String> {
    List<Order> findByUserIdAndStatus(Long userId, String status);
    @Query("{ 'amount': { $gt: ?0 } }")
    List<Order> findBig(BigDecimal amt);
    Page<Order> findByUserId(Long userId, Pageable pageable);
}
```

## 4. 查询条件构造

```java
Criteria.where("age").gt(18).lte(60)
        .and("status").in("A","B")
        .and("name").regex("^张")          // 正则（注意索引失效）
        .andOperator(...)                   // $and
        .orOperator(...)                    // $or

Query q = new Query(c)
    .with(Sort.by("createTime").descending())
    .skip(20).limit(10)
    .withHint("userId_1")                   // 强制索引
    .setReadPreference(ReadPreference.secondaryPreferred());   // 读从库

// 投影（只返回部分字段）
q.fields().include("orderNo").include("amount").exclude("_id");
```

## 5. 索引与性能

```java
@CompoundIndex(name = "uid_status", def = "{'userId': 1, 'status': -1}")
mongo.indexOps(Order.class).ensureIndex(new Index().on("userId", Direction.ASC).unique());
```

| 要点 | 说明 |
|---|---|
| ESR 规则 | 索引顺序：**E**quality → **S**ort → **R**ange |
| 覆盖查询 | 索引包含所有返回字段，不回表 |
| 慢查询 | `db.setProfilingLevel(1, 100)` 或 `explain("executionStats")` |
| 避免 | 无索引正则、全表 `skip` 深分页、`$where` |

深分页优化：`find({_id: {$gt: lastId}}).limit(20)`。

## 6. 事务（MongoDB 4.0+）

```java
@Transactional     // 需要 MongoTransactionManager
public void transfer() { ... }

// 或编程式
mongo.executeInSession(s -> { ... });
```

> 多文档事务**只在副本集/分片集群**可用，单机不支持。事务有性能开销，慎用。

## 7. Change Stream（变更流）

类似 binlog 订阅，用于同步到 ES / 触发业务：

```java
@Bean
public MessageListenerContainer container(MongoTemplate t, ChangeStreamListener l) {
    MessageListenerContainer c = new DefaultMessageListenerContainer(t.getMongoDatabaseFactory()) {
        @Override public boolean isAutoStartup() { return true; }
    };
    ChangeStreamRequest<Order> req = ChangeStreamRequest.builder(l)
            .collection("orders").filter(Aggregation.newAggregation(MatchOperation...))
            .fullDocumentLookup(FullDocument.UPDATE_LOOKUP)
            .publishTo(l).build();
    c.register(req, Order.class);
    return c;
}
```

> 生产常用 **Canal / Debezium** 订阅 MongoDB oplog 同步到 ES / 数据仓库。

## 8. 与 MySQL 的选型

| 维度 | MySQL | MongoDB |
|---|---|---|
| 事务 | 强 | 4.0+ 支持但弱 |
| Schema | 固定 | 灵活，适合快速迭代 |
| 关联查询 | 强 | 弱（$lookup 性能差） |
| 写入吞吐 | 中 | 高 |
| 场景 | 交易、账务 | 日志、内容、社交、IoT、画像 |

## 9. GridFS（存文件）

```java
GridFsTemplate gridFs;
ObjectId id = gridFs.store(inputStream, "a.pdf", "application/pdf");
GridFsResource r = gridFs.getResource(id);
```
> 大文件建议直接存对象存储（OSS/S3），GridFS 只适合小文件与统一管理。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 字段为 `_class_` | 多出一个字段 | 用 `MongoCustomConversions` 或 `@TypeAlias` 控制；可 `setWriteConcern`/禁用 `_class` |
| 时间差 8 小时 | 时区 | MongoDB 存 UTC，读取时转换；Jackson 配 `Asia/Shanghai` |
| 深分页慢 | skip 大 | 游标分页 |
| 无索引全表扫 | 慢 | 建索引 + `explain` |
| 事务不可用 | `Transaction numbers are only allowed on a replica set` | 部署副本集 |
| 大数组无限增长 | 单文档超 16MB | 改引用建模 |
| 聚合内存超限 | `Exceeded memory limit` | 加 `allowDiskUse(true)` |

## 11. 面试速答

- **MongoTemplate 与 Repository**：前者灵活（Criteria/Update/Aggregation），后者方法名派生，简单 CRUD 用 Repository。
- **内嵌 vs 引用**：读多写少、子文档随父一起访问 → 内嵌；子文档独立访问或数量无限 → 引用。
- **ESR 索引原则**：等值 → 排序 → 范围，按此顺序建复合索引。
