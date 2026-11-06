# 12 · Spring Data JPA 与 Hibernate

## 一、定位与集成

JPA 是「接口驱动的 ORM」：方法名即查询、Repository 即 DAO。适合领域模型清晰的中小项目与海外风格代码库。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate     # 生产必为 validate/none；开发可用 update
    show-sql: true           # dev
    open-in-view: false      # 推荐：关闭 OSIV（见坑 5）
    properties:
      hibernate:
        format_sql: true
```

## 二、实体

```java
@Entity
@Table(name = "users")
@Data
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 20)
    private String username;

    private Integer age;

    @Enumerated(EnumType.STRING)          // 必须显式 STRING，ORDINAL 是坑
    private UserStatus status;

    @ManyToOne(fetch = FetchType.LAZY)    // 关联默认 LAZY（@ManyToOne 默认 EAGER，要改）
    @JoinColumn(name = "dept_id")
    private Dept dept;

    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

## 三、Repository

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // 1. 方法名推导查询
    List<User> findByUsernameLikeAndAgeGreaterThanEqual(String username, Integer age);

    Optional<User> findByUsername(String username);

    // 2. @Query（JPQL）
    @Query("SELECT u FROM User u WHERE u.status = :status")
    List<User> findByStatus(@Param("status") UserStatus status);

    // 3. 原生 SQL
    @Query(value = "SELECT * FROM users WHERE age > :age", nativeQuery = true)
    List<User> findOlderThan(@Param("age") int age);

    // 4. 更新需 @Modifying + 事务
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") UserStatus status);
}
```

## 四、Specification 动态查询

```java
public Specification<User> buildSpec(UserQueryDTO q) {
    return (root, query, cb) -> {
        List<Predicate> ps = new ArrayList<>();
        if (q.getUsername() != null) {
            ps.add(cb.like(root.get("username"), "%" + q.getUsername() + "%"));
        }
        if (q.getAge() != null) {
            ps.add(cb.ge(root.get("age"), q.getAge()));
        }
        return cb.and(ps.toArray(new Predicate[0]));
    };
}

Page<User> page = userRepository.findAll(buildSpec(query),
        PageRequest.of(pageNum - 1, pageSize, Sort.by("createdAt").descending()));
```

## 五、持久化上下文核心机制

```
save()       → 新实体 INSERT；已有 ID 的实体先 SELECT 再比较快照决定 UPDATE
脏检查       → 事务内查出实体并改字段，不调 save 也会 UPDATE（易惊吓新人）
getOne/getReferenceById → 懒代理，不查库；只能用于设置关联外键
findById     → 真实查询
```

## 六、N+1 问题与解决方案

```java
// 场景：查 100 个用户再访问 dept → 1 + 100 条 SQL

// 方案 1：JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.dept WHERE u.age >= :age")
List<User> findWithDept(@Param("age") int age);

// 方案 2：@EntityGraph
@EntityGraph(attributePaths = "dept")
List<User> findByAgeGreaterThanEqual(int age);
```

## 七、事务边界

Repository 方法自带事务；组合操作在 **Service 层加 @Transactional**（见 14 篇）。

```java
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    from.setBalance(from.getBalance().subtract(amount));  // 脏检查自动 UPDATE
    Account to = accountRepository.findById(toId).orElseThrow();
    to.setBalance(to.getBalance().add(amount));
}
```

## 八、与 MyBatis 对比决策（复述 09 篇）

- 团队 Java 面向对象思维强、SQL 简单 → JPA 效率高
- 报表/复杂查询多、国内团队习惯 → MyBatis(-Plus)
- 二者可共存于同一项目（不同模块），注意事务管理器统一为 JpaTransactionManager 或 DataSourceTransactionManager 的一致性

## 常见坑

- **坑 1**：@ManyToOne 默认 EAGER → 无脑 join 查询爆炸；显式设 LAZY
- **坑 2**：LazyInitializationException → 事务外访问懒属性；用 JOIN FETCH 或 DTO 投影
- **坑 3**：open-in-view 默认 true → 懒加载逃逸到 Controller 层，连接被占用；显式关闭
- **坑 4**：ddl-auto=update 在生产 → 结构失控，一律 validate
- **坑 5**：@Enumerated 忘写 → 默认 ORDINAL，枚举顺序变更后数据错乱
- **坑 6**：大数据量 saveAll → 逐条 flush 极慢；需配置 batch_size 与 IDENTITY 限制

## Checklist

- [ ] ddl-auto=validate，show-sql 仅 dev
- [ ] 所有关联显式 LAZY，列表查询用 DTO 投影或 JOIN FETCH
- [ ] 枚举 @Enumerated(EnumType.STRING)
- [ ] 更新语句 @Modifying + 事务
