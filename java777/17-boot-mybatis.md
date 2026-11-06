# 17 · MyBatis 集成与动态 SQL

## 1. 依赖与配置

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>     <!-- 适配 Boot 3 -->
</dependency>
```

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
mybatis:
  mapper-locations: classpath*:mapper/**/*.xml
  type-aliases-package: com.example.domain.entity
  configuration:
    map-underscore-to-camel-case: true     # user_id → userId
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
    default-enum-type-handler: org.apache.ibatis.type.EnumTypeHandler
    cache-enabled: false                    # ★ 二级缓存建议关闭
    jdbc-type-for-null: null
  type-handlers-package: com.example.type
```

```java
@SpringBootApplication
@MapperScan("com.example.**.mapper")     // 或每个 mapper 加 @Mapper
public class App {}
```

## 2. 三种使用方式

| 方式 | 说明 |
|---|---|
| XML（`mapper/UserMapper.xml`） | 复杂 SQL 首选，可读性好 |
| 注解（`@Select/@Insert...`） | 简单 SQL |
| 注解 + `@SelectProvider` | 动态拼装 |

```java
public interface UserMapper {
    User selectById(Long id);

    @Select("select * from t_user where id = #{id}")
    User getById(@Param("id") Long id);

    int insert(User u);

    @Update("update t_user set name=#{name} where id=#{id}")
    int updateName(@Param("id") Long id, @Param("name") String name);

    List<User> selectByCondition(UserQuery q);
}
```

```xml
<!-- resources/mapper/UserMapper.xml -->
<mapper namespace="com.example.mapper.UserMapper">
    <resultMap id="BaseMap" type="User">
        <id column="id" property="id"/>
        <result column="user_name" property="userName"/>
    </resultMap>

    <sql id="cols">id,user_name,age,status,create_time</sql>

    <select id="selectById" resultMap="BaseMap">
        select <include refid="cols"/> from t_user where id = #{id}
    </select>

    <insert id="insert" useGeneratedKeys="true" keyProperty="id">
        insert into t_user(user_name, age) values(#{userName}, #{age})
    </insert>
</mapper>
```

## 3. 动态 SQL 标签

| 标签 | 作用 |
|---|---|
| `<if>` | 条件判断 |
| `<where>` | 智能去掉首个 `and/or` |
| `<set>` | 智能去掉末尾逗号 |
| `<choose>/<when>/<otherwise>` | switch |
| `<foreach>` | 循环（IN、批量插入） |
| `<trim>` | 自定义前缀后缀 |
| `<bind>` | 创建变量（如模糊查询拼接 %） |
| `<include>` + `<sql>` | 片段复用 |

```xml
<select id="selectByCondition" resultMap="BaseMap">
    select <include refid="cols"/> from t_user
    <where>
        <if test="name != null and name != ''">
            and user_name like concat('%', #{name}, '%')
        </if>
        <if test="age != null"> and age = #{age} </if>
        <if test="status != null"> and status = #{status} </if>
        <if test="ids != null and ids.size() > 0">
            and id in
            <foreach collection="ids" item="i" open="(" separator="," close=")">#{i}</foreach>
        </if>
        <choose>
            <when test="orderBy == 'age'"> order by age desc </when>
            <otherwise> order by id desc </otherwise>
        </choose>
    </where>
</select>

<update id="updateSelective">
    update t_user
    <set>
        <if test="userName != null">user_name = #{userName},</if>
        <if test="age != null">age = #{age},</if>
    </set>
    where id = #{id}
</update>

<insert id="batchInsert">
    insert into t_user(user_name, age) values
    <foreach collection="list" item="u" separator=",">
        (#{u.userName}, #{u.age})
    </foreach>
</insert>
```

## 4. `#{}` vs `${}`（必考）

| 对比 | `#{}` | `${}` |
|---|---|---|
| 处理 | `PreparedStatement` 占位符 `?` | 字符串直接拼接 |
| SQL 注入 | **安全** | **有注入风险** |
| 用途 | 参数值 | 动态表名/列名/order by（必须白名单校验） |

```xml
order by ${orderColumn} ${orderDirection}   <!-- 必须校验枚举白名单 -->
```

## 5. 结果映射

```xml
<!-- 一对一 -->
<resultMap id="OrderMap" type="Order">
    <id property="id" column="id"/>
    <association property="user" javaType="User">
        <id property="id" column="u_id"/><result property="name" column="u_name"/>
    </association>
    <!-- 一对多 -->
    <collection property="items" ofType="OrderItem">
        <id property="id" column="i_id"/><result property="skuName" column="sku_name"/>
    </collection>
</resultMap>
```

> 一对多用 `collection` 时分页会不准（结果集行数≠主表行数），应**先分页查主表 ID，再查子表**。

## 6. 分页

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.4.7</version>
</dependency>
```

```java
PageHelper.startPage(1, 10);          // ★ 紧跟第一个查询方法，之后必须紧跟 mapper 调用
List<User> list = userMapper.selectAll();
PageInfo<User> page = new PageInfo<>(list);
page.getTotal(); page.getPages();
```

或 MyBatis-Plus 的 `Page<T>` + 分页插件。

## 7. MyBatis-Plus（国内主流增强）

```xml
<dependency><groupId>com.baomidou</groupId>
           <artifactId>mybatis-plus-spring-boot3-starter</artifactId><version>3.5.7</version></dependency>
```

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {}

// 无需 XML 的 CRUD
userMapper.selectById(1L);
userMapper.selectList(Wrappers.<User>lambdaQuery()
        .eq(User::getStatus, 1).like(User::getUserName, "张").orderByDesc(User::getCreateTime));
userMapper.update(null, Wrappers.<User>lambdaUpdate().set(User::getAge, 20).eq(User::getId, 1));

// 分页插件
@Bean
public MybatisPlusInterceptor mpi() {
    MybatisPlusInterceptor i = new MybatisPlusInterceptor();
    i.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
    i.addInnerInterceptor(new OptimisticLockerInnerInterceptor());   // 乐观锁 @Version
    i.addInnerInterceptor(new BlockAttackInnerInterceptor());        // 防全表更新
    return i;
}
```

其他能力：逻辑删除 `@TableLogic`、自动填充 `@TableField(fill=...)`、代码生成器、多租户 `TenantLineInnerInterceptor`。

## 8. 类型处理器（枚举、JSON）

```java
@MappedTypes(OrderStatus.class) @MappedJdbcTypes(JdbcType.TINYINT)
public class StatusTypeHandler extends BaseTypeHandler<OrderStatus> {
    public void setNonNullParameter(PreparedStatement ps, int i, OrderStatus p, JdbcType t) throws SQLException {
        ps.setInt(i, p.getCode());
    }
    public OrderStatus getNullableResult(ResultSet rs, String col) throws SQLException {
        return OrderStatus.of(rs.getInt(col));
    }
    // ...
}
```

JSON 字段：`mybatis.type-handlers-package` 扫描自定义 `JacksonTypeHandler`。

## 9. 多数据源（简单方案）

```java
@Configuration
public class DataSourceConfig {
    @Bean @ConfigurationProperties("spring.datasource.master") @Primary
    public DataSource master() { return DataSourceBuilder.create().build(); }

    @Bean @ConfigurationProperties("spring.datasource.slave")
    public DataSource slave() { return DataSourceBuilder.create().build(); }
}
// 主从路由见 31 篇；动态数据源用 AbstractRoutingDataSource + AOP
```

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| `Invalid bound statement (not found)` | 方法找不到 | namespace/方法名与接口一致；`mapper-locations` 路径；`classpath*:` |
| 字段为 null | 下划线未转驼峰 | `map-underscore-to-camel-case: true` 或写 resultMap |
| `$` 注入 | 安全漏洞 | 只用 `#{}`；动态列名白名单 |
| 分页总数不准 | 一对多导致 | 子查询分页 / 先查 ID |
| 一级缓存脏读 | 同一 SqlSession 内两次查询不一致 | 理解 Session 生命周期；更新操作会清空 |
| 二级缓存脏数据 | 分布式下数据不一致 | 关闭二级缓存，用 Redis |
| 批量插入慢 | 逐条 insert | `rewriteBatchedStatements=true` + foreach |
| `@Transactional` 与 `PageHelper` | 分页失效 | `startPage` 后紧跟查询 |
| Mapper 注入失败 | `NoSuchBeanDefinition` | `@MapperScan` 或 `@Mapper` |

## 11. 面试速答

- **`#{}` 与 `${}`**：前者预编译防注入，后者字符串拼接有风险，仅用于动态表名/排序且需白名单。
- **MyBatis 与 JPA 选型**：SQL 复杂、需要精细优化、团队熟悉 SQL → MyBatis；快速 CRUD、领域建模 → JPA。
- **一级缓存**：SqlSession 级别，默认开启，任何 `insert/update/delete` 都会清空；分布式下建议关闭二级缓存。
