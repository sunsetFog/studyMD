# 15 - MySQL 与 MyBatis

## 1. 索引考点（最高频⭐⭐）

- **B+ 树**：矮胖（3 层 ~2000 万行）、叶子链表利于范围查询；为什么不用哈希（无范围）/红黑树（太高）
- 聚簇索引（主键树存整行）vs 二级索引（叶子存主键，需**回表**）
- **覆盖索引**：查询列都在索引里，免回表（Extra: Using index）
- **最左前缀**：联合索引 (a,b,c) 支持 a / a,b / a,b,c；范围列之后的失效
- 索引失效：列上函数/运算、隐式类型转换（字符串列 vs 数字）、左模糊 '%x'、OR 非索引列
- 为什么主键自增：避免页分裂

## 2. 事务与锁

- ACID 实现：undo（原子）/redo（持久）/MVCC+锁（隔离）
- 隔离级别：RU/RC/**RR（默认）**/Serializable；三并发问题
- **MVCC**：隐藏列 trx_id + roll_ptr → undo 版本链 + ReadView；RC 每次新 ReadView，RR 首次生成复用
- 锁：行锁加在索引上（无索引条件 = 锁全表）；Gap/Next-Key（RR 防幻读）；死锁排查 SHOW ENGINE INNODB STATUS，预防=统一加锁顺序
- 乐观锁（version 字段）vs 悲观锁（FOR UPDATE）

## 3. 实战 SQL 考点

```sql
-- 深分页优化
SELECT t.* FROM t JOIN (SELECT id FROM t ORDER BY id LIMIT 100000, 20) i USING (id);
-- 或游标分页：WHERE id > lastId ORDER BY id LIMIT 20

-- 找每组前 N
SELECT * FROM (SELECT t.*, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) rn FROM t) x WHERE rn <= 3;
```

- EXPLAIN 关键：type（ALL 全表要优化）、key、rows、Extra（Using filesort/temporary）
- 大表 DDL：pt-osc/gh-ost；5000 万行分库分表
- 互联网不用外键：锁传递/分库不支持

## 4. MyBatis 考点

### #{} vs ${}（必考）

```xml
WHERE name = #{name}    <!-- 预编译占位符，防注入 -->
ORDER BY ${column}      <!-- 字符串拼接，必须白名单校验 -->
```

### 缓存

- 一级缓存：SqlSession 级，默认开启（Spring 下每请求新 session，意义不大）
- 二级缓存：namespace 级，多表关联易脏，生产慎用 → 交给 Redis

### 其他

- 接口绑定原理：动态代理（MapperProxy）→ MapperMethod 解析 → SqlSession
- `resultMap` vs `resultType`：列名驼峰映射 `map-underscore-to-camel-case`
- 动态 SQL：`<if>/<where>/<foreach>`（批量插入、IN 查询）
- 批量插入：foreach VALUES 多值，大批分批（1000/批）
- 分页：PageHelper（ThreadLocal 拦截改写 LIMIT）vs MyBatis-Plus Page 插件
- `useGeneratedKeys="true" keyProperty="id"` 取回自增主键

## 常见追问

1. MyBatis 一级缓存在分布式下的问题？——本地缓存各节点不一致，Spring 每次 mapper 方法新 SqlSession
2. `SELECT FOR UPDATE` 和乐观锁选型？——冲突高用悲观（秒杀行锁队列），冲突低用乐观（无锁重试）
3. 如何排查慢 SQL？——slow log → EXPLAIN/EXPLAIN ANALYZE → 索引/改写

## Checklist

- [ ] MVCC/ReadView 能完整讲
- [ ] #{} ${} 安全边界清晰
- [ ] 深分页、Top-N 模板能默写
