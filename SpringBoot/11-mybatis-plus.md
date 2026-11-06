# 11 · MyBatis-Plus

> 定位：在 MyBatis 之上做增强，**不做改变**——简单 CRUD 零 SQL，复杂查询仍写 XML。

## 一、集成

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.7</version>
</dependency>
```

```yaml
mybatis-plus:
  mapper-locations: classpath:mapper/*.xml      # 自己写的 XML 照常生效
  global-config:
    db-config:
      id-type: assign_id          # 雪花 ID
      logic-delete-field: deleted # 逻辑删除字段
      logic-delete-value: 1
      logic-not-delete-value: 0
  configuration:
    map-underscore-to-camel-case: true
```

## 二、实体与 Mapper

```java
@Data
@TableName("users")
public class User implements Serializable {

    @TableId(type = IdType.ASSIGN_ID)       // 雪花算法，无需数据库自增
    private Long id;

    private String username;

    private Integer age;

    @TableField(fill = FieldFill.INSERT)    // 自动填充
    private LocalDateTime createdAt;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updatedAt;

    @TableLogic                              // 逻辑删除
    private Integer deleted;

    @Version                                 // 乐观锁
    private Integer version;
}

public interface UserMapper extends BaseMapper<User> {}
```

```java
// BaseMapper 已提供：insert / deleteById / updateById / selectById / selectPage ...
userMapper.selectById(1L);
userMapper.insert(user);
```

## 三、条件构造器（Wrapper）

```java
// QueryWrapper：查列表
List<User> list = userMapper.selectList(
        new QueryWrapper<User>()
                .like("username", "tom")
                .ge("age", 18)
                .orderByDesc("created_at"));

// LambdaQueryWrapper（推荐：字段名编译期检查，防写错列名）
List<User> list = userMapper.selectList(
        new LambdaQueryWrapper<User>()
                .like(User::getUsername, "tom")
                .between(User::getAge, 18, 60)
                .orderByDesc(User::getCreatedAt));

// 分页
Page<User> page = userMapper.selectPage(
        new Page<>(pageNum, pageSize),
        new LambdaQueryWrapper<User>().eq(User::getStatus, 1));

// 更新
userMapper.update(null, new LambdaUpdateWrapper<User>()
        .set(User::getStatus, 0)
        .eq(User::getId, 1L));
```

> Wrapper 复杂查询可读性急剧下降的经验法则：**wrapper 条件超过 3 层嵌套 / 需要 join，改写 XML**。

## 四、Service 层封装

```java
public interface UserService extends IService<User> {}

@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
    // 免费获得 save / saveBatch / getOne / lambdaQuery 等
}

// 调用侧
userService.lambdaQuery()
        .like(User::getUsername, "tom")
        .list();
userService.saveBatch(userList, 1000);
```

## 五、自动填充

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createdAt", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updatedAt", LocalDateTime.class, LocalDateTime.now());
    }
    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updatedAt", LocalDateTime.class, LocalDateTime.now());
    }
}
```

## 六、乐观锁插件

```java
@Configuration
public class MpConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

```java
User user = userMapper.selectById(1L);
user.setAge(20);
userMapper.updateById(user);   // 自动 WHERE version = 旧值，更新成功才 version+1
```

## 七、逻辑删除

```java
userMapper.deleteById(1L);        // 实际执行 UPDATE users SET deleted = 1 WHERE id = 1 AND deleted = 0
userMapper.selectById(1L);        // 自动追加 AND deleted = 0
```

> 逻辑删除后唯一索引冲突问题：唯一索引列改为 `username + deleted` 组合，或删除时把唯一列值改写（如 `name_del_{id}`）。

## 八、代码生成器

```java
FastAutoGenerator.create(url, username, password)
        .globalConfig(b -> b.author("dev").outputDir("src/main/java"))
        .packageConfig(b -> b.parent("com.example.demo"))
        .strategyConfig(b -> b.addInclude("users", "orders")
                .entityBuilder().enableLombok()
                .controllerBuilder().enableRestStyle())
        .execute();
```

## 常见坑

- **坑 1**：雪花 ID 用 Long 返回给前端 → JS 精度丢失；Jackson 配置 Long → String 序列化
- **坑 2**：实体新增字段但数据库没加列 → insert 报 SQL 错误（MP 按实体全字段生成 SQL）；不映射的加 `@TableField(exist = false)`
- **坑 3**：逻辑删除的表有唯一索引 → 重复插入报错（见上）
- **坑 4**：PaginationInnerInterceptor 没注册 → 分页不生效查全表
- **坑 5**：Service 继承 ServiceImpl 却泛型写错顺序 → 运行时 ClassCastException

## Checklist

- [ ] 统一 IdType、逻辑删除、自动填充策略写入全局配置
- [ ] Lambda Wrapper 优先，复杂查询落 XML
- [ ] 乐观锁/分页插件已注册
- [ ] Long ID 对前端序列化为字符串
