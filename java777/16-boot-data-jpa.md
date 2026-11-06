# 16 · Spring Data JPA

## 1. 定位与依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency><groupId>com.mysql</groupId><artifactId>mysql-connector-j</artifactId><scope>runtime</scope></dependency>
```

JPA 是规范，**Hibernate** 是默认实现。Spring Data JPA 在 JPA 之上又封装了 Repository（方法名即查询）。

> 选型：简单 CRUD / 领域模型复杂 → JPA；复杂 SQL / 团队 SQL 能力强 / 需要精细优化 → MyBatis。

## 2. 配置

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update        # none(生产) / validate / update / create / create-drop
    show-sql: false           # 生产用 logging 控制
    open-in-view: false       # ★ 必须关！否则懒加载在视图层开事务，连接被长时间占用
    properties:
      hibernate:
        format_sql: true
        jdbc:
          batch_size: 100
          time_zone: Asia/Shanghai
    database-platform: org.hibernate.dialect.MySQLDialect
logging:
  level:
    org.hibernate.SQL: debug
    org.hibernate.orm.jdbc.bind: trace    # 打印参数
```

## 3. 实体映射

```java
@Entity
@Table(name = "t_order", indexes = @Index(columnList = "user_id"))
@Getter @Setter @NoArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // MySQL 自增
    private Long id;

    @Column(nullable = false, length = 64, unique = true)
    private String orderNo;

    @Column(precision = 12, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)      // ★ 千万别用 ORDINAL
    private OrderStatus status;

    @Temporal(TemporalType.TIMESTAMP) // 或直接用 LocalDateTime（推荐）
    private LocalDateTime payTime;

    @Version                           // 乐观锁
    private Integer version;

    @CreationTimestamp @Column(updatable = false)
    private LocalDateTime createTime;
    @UpdateTimestamp
    private LocalDateTime updateTime;

    @CreatedBy private String creator;   // 需开启 @EnableJpaAuditing
    @LastModifiedBy private String updater;

    @Transient private String temp;      // 非持久化字段

    // 关联
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}
```

### 主键策略对比

| 策略 | 说明 | 适用 |
|---|---|---|
| `IDENTITY` | 数据库自增 | MySQL，插入后才能拿 ID（无法批量） |
| `SEQUENCE` | 序列 | Oracle/PG，可批量 |
| `TABLE` | 表模拟序列 | 跨库，性能差 |
| `AUTO` | 自动选择 | 默认 |
| UUID | 应用层生成 | 分布式 |

## 4. Repository 三种用法

```java
public interface OrderRepository extends JpaRepository<Order, Long>,
                                        JpaSpecificationExecutor<Order> {

    // ① 方法名派生查询
    Optional<Order> findByOrderNo(String orderNo);
    List<Order> findByUserIdAndStatusOrderByCreateTimeDesc(Long userId, OrderStatus status);
    boolean existsByOrderNo(String orderNo);
    long countByStatus(OrderStatus status);
    void deleteByStatusAndCreateTimeBefore(OrderStatus s, LocalDateTime t);

    // ② @Query
    @Query("select o from Order o join fetch o.items where o.id = :id")
    Optional<Order> findWithItems(@Param("id") Long id);

    @Modifying
    @Transactional
    @Query("update Order o set o.status = :s where o.id = :id")
    int updateStatus(@Param("id") Long id, @Param("s") OrderStatus s);

    // 原生 SQL
    @Query(value = "select * from t_order where amount > :amt", nativeQuery = true)
    List<Order> findBig(@Param("amt") BigDecimal amt);

    // 投影
    @Query("select new com.example.vo.OrderSummary(o.id, o.orderNo, o.amount) from Order o where o.userId=:uid")
    List<OrderSummary> summary(@Param("uid") Long uid);
}
```

### 方法名关键字

`And/Or`、`Between`、`LessThan/GreaterThan`、`Like/NotLike`、`StartingWith/EndingWith/Containing`、`In/NotIn`、`IsNull/NotNull`、`OrderBy`、`Desc/Asc`、`Top3/First`、`Distinct`、`IgnoreCase`。

## 5. 分页与排序

```java
Page<Order> page = repo.findByUserId(userId,
        PageRequest.of(0, 20, Sort.by(Sort.Direction.DESC, "createTime")));
page.getTotalElements(); page.getTotalPages(); page.getContent();

// 游标分页（深分页优化）
Slice<Order> slice = repo.findByStatus(OrderStatus.PAID, PageRequest.of(0, 20));

// 动态条件
Specification<Order> spec = (root, query, cb) -> {
    List<Predicate> ps = new ArrayList<>();
    if (userId != null) ps.add(cb.equal(root.get("user"), userId));
    if (status != null) ps.add(cb.equal(root.get("status"), status));
    return cb.and(ps.toArray(new Predicate[0]));
};
repo.findAll(spec, pageable);
```

> 深分页（` LIMIT 100000,20`）很慢，用**上一页最大 ID 游标**方案。

## 6. 审计（自动填充创建人/时间）

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {
    @Bean
    public AuditorAware<String> auditorAware() {
        return () -> Optional.ofNullable(SecurityUtil.currentUserId());
    }
}
```

## 7. N+1 问题（JPA 最大坑）

```java
// 触发 1 次查订单 + N 次查用户
List<Order> orders = repo.findAll();
orders.forEach(o -> System.out.println(o.getUser().getName()));
```

解决方案：

```java
// ① join fetch
@Query("select o from Order o join fetch o.user") List<Order> findAllWithUser();
// ② @EntityGraph
@EntityGraph(attributePaths = {"user", "items"}) List<Order> findByUserId(Long userId);
// ③ 批量抓取
spring.jpa.properties.hibernate.default_batch_fetch_size: 100
// ④ 用 DTO 投影，不返回实体
```

检测：`spring.jpa.properties.hibernate.session.events.log` 或打开 SQL 日志观察。

## 8. 懒加载与 LazyInitializationException

```java
@Transactional(readOnly = true)      // ★ 在事务内访问懒加载属性
public OrderVO get(Long id) {
    Order o = repo.findById(id).orElseThrow();
    o.getItems().size();              // 触发加载，此时 session 还开着
    return convert(o);
}
```

`open-in-view: true`（默认开启）会为每个请求开一个 session，导致**连接占用 + 懒加载 SQL 在渲染时才执行**，强烈建议关闭。

## 9. 乐观锁与悲观锁

```java
@Version private Integer version;       // 更新时自动 +1，冲突抛 ObjectOptimisticLockingFailureException

@Lock(LockModeType.PESSIMISTIC_WRITE)   // select ... for update
@Query("select o from Order o where o.id=:id")
Optional<Order> lockById(@Param("id") Long id);
```

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| `open-in-view` 警告且连接耗尽 | 慢查询堆积 | `spring.jpa.open-in-view=false` |
| N+1 | 接口慢 | `join fetch` / `@EntityGraph` / 投影 |
| `LazyInitializationException` | no Session | 事务内访问或 `join fetch` |
| `ddl-auto: update` 生产事故 | 表结构被改 | 生产用 `none` / `validate`，用 Flyway/Liquibase 管迁移 |
| `save()` 每次都 insert | 主键非 null 但被当新对象 | 实体实现 `Persistable` 或用 `merge` |
| 批量插入慢 | 逐条 insert | `jdbc.batch_size` + `rewriteBatchedStatements=true` |
| 枚举存成数字 | 改枚举顺序数据错乱 | `EnumType.STRING` |

## 11. 面试速答

- **JPA 与 Hibernate**：JPA 是 ORM 规范，Hibernate 是实现；Spring Data JPA 又在其上提供 Repository 抽象。
- **方法名查询原理**：`PartTree` 解析方法名 → 生成 `CriteriaQuery` → JPQL → SQL。
- **`getOne` 与 `findById`**：前者返回懒加载代理（已废弃，用 `getReferenceById`），后者返回 `Optional` 真实查询。
- **一级/二级缓存**：一级是 `EntityManager` 级别（同一持久化上下文）；二级是 SessionFactory 级别（需额外配置，生产少用，一般用 Redis）。
