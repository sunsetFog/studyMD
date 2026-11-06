# 10 · MyBatis

## 一、集成配置

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>3.0.3</version>
</dependency>
```

```yaml
mybatis:
  mapper-locations: classpath:mapper/*.xml
  type-aliases-package: com.example.demo.entity
  configuration:
    map-underscore-to-camel-case: true   # user_name → userName（必开）
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
```

```java
@Mapper                     // 或启动类 @MapperScan("com.example.demo.mapper")
public interface UserMapper {
    User selectById(Long id);
    List<User> selectByCondition(@Param("query") UserQueryDTO query);
    int insert(User user);
    int update(User user);
}
```

## 二、XML Mapper 规范

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.demo.mapper.UserMapper">

    <!-- resultMap：字段映射 -->
    <resultMap id="BaseMap" type="com.example.demo.entity.User">
        <id column="id" property="id"/>
        <result column="username" property="username"/>
        <result column="created_at" property="createdAt"/>
    </resultMap>

    <sql id="Base_Columns">id, username, age, created_at</sql>

    <select id="selectById" resultMap="BaseMap">
        SELECT <include refid="Base_Columns"/> FROM users WHERE id = #{id}
    </select>
</mapper>
```

## 三、#{} 与 ${} —— SQL 注入分水岭

| 写法 | 机制 | 风险 |
|---|---|---|
| `#{}` | PreparedStatement 占位符，预编译 | **安全** |
| `${}` | 字符串拼接进 SQL | **注入风险，仅用于白名单动态列名/排序** |

```xml
<!-- 正确：排序字段走白名单 -->
<choose>
    <when test="orderBy == 'created_at'">ORDER BY created_at DESC</when>
    <otherwise>ORDER BY id DESC</otherwise>
</choose>

<!-- LIKE 模糊查询的正确写法 -->
WHERE username LIKE CONCAT('%', #{keyword}, '%')
```

## 四、动态 SQL

```xml
<select id="selectByCondition" resultMap="BaseMap">
    SELECT * FROM users
    <where>
        <if test="query.username != null and query.username != ''">
            AND username LIKE CONCAT('%', #{query.username}, '%')
        </if>
        <if test="query.ageStart != null">
            AND age &gt;= #{query.ageStart}
        </if>
        <choose>
            <when test="query.status != null">AND status = #{query.status}</when>
            <otherwise>AND status = 1</otherwise>
        </choose>
    </where>
</select>
```

批量操作：

```xml
<insert id="batchInsert">
    INSERT INTO users(username, age) VALUES
    <foreach collection="list" item="u" separator=",">
        (#{u.username}, #{u.age})
    </foreach>
</insert>

<update id="batchUpdate">
    <foreach collection="list" item="u" separator=";">
        UPDATE users SET age = #{u.age} WHERE id = #{u.id}
    </foreach>
</update>
<!-- batchUpdate 需在 JDBC url 加 allowMultiQueries=true；更优方案：批量走 MyBatis ExecutorType.BATCH -->
```

## 五、一对多 / 多对一

```xml
<resultMap id="OrderMap" type="Order">
    <id column="id" property="id"/>
    <collection property="items" ofType="OrderItem">
        <id column="item_id" property="id"/>
        <result column="product_name" property="productName"/>
    </collection>
</resultMap>

<select id="selectWithItems" resultMap="OrderMap">
    SELECT o.*, i.id AS item_id, i.product_name
    FROM orders o LEFT JOIN order_items i ON o.id = i.order_id
    WHERE o.id = #{id}
</select>
```

> 注意 LEFT JOIN 一对多分页会出现「分页数量错乱」（一条订单多行被算作多条）——分页时先查主表 id 分页，再 IN 查明细。

## 六、分页

PageHelper（通用方案）：

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
```

```java
PageHelper.startPage(pageNum, pageSize);   // 紧跟的第一条查询生效
PageInfo<User> info = new PageInfo<>(userMapper.selectByCondition(query));
```

## 七、一级/二级缓存

- **一级缓存**：SqlSession 级，默认开启。Spring 集成下每次 Mapper 方法后 session 关闭，基本无感
- **二级缓存**：namespace 级，`<cache/>` 开启。**生产慎用**——多表关联更新会导致脏读，分布式下更不可控，用 Redis 缓存替代（24 篇）

## 常见坑

- **坑 1**：忘了 `map-underscore-to-camel-case: true` → 查询结果字段全 null
- **坑 2**：`${}` 拼接用户输入 → SQL 注入
- **坑 3**：insert 后拿不到自增 id → XML 加 `useGeneratedKeys="true" keyProperty="id"`
- **坑 4**：`if test` 判断字符串用 `!= ""` 但字段是数字类型 → 类型转换异常
- **坑 5**：XML 中文注释含 `--` 或不匹配的 `>` `<` → 解析失败，比较符用 `&gt;` `&lt;` 或 `<![CDATA[ ]]>`
- **坑 6**：一对多 + PageHelper 分页 → 总条数错乱

## Checklist

- [ ] 所有用户输入使用 #{}，${} 仅限白名单场景
- [ ] 开启驼峰映射与 SQL 日志（dev）
- [ ] 批量插入用 foreach 或 BATCH 执行器
- [ ] 分页统一 PageHelper，一对多分页先分主表
