# MySQL 面试题深度清单

> 一份覆盖从基础原理到高级架构的 MySQL 面试题全集，每题附详细答案要点、原理图解与生产实战经验。

---

## 📋 目录

- [一、SQL 基础（必问）](#一sql-基础必问)
- [二、索引（高频核心）](#二索引高频核心)
- [三、存储引擎（高频）](#三存储引擎高频)
- [四、事务与锁（高频）](#四事务与锁高频)
- [五、MVCC 与隔离级别（高级必问）](#五mvcc-与隔离级别高级必问)
- [六、SQL 优化（高频实战）](#六sql-优化高频实战)
- [七、Explain 执行计划（高频实战）](#七explain-执行计划高频实战)
- [八、日志系统（高级必问）](#八日志系统高级必问)
- [九、架构与高可用（高级必问）](#九架构与高可用高级必问)
- [十、分库分表（高级必问）](#十分库分表高级必问)
- [十一、锁机制深度](#十一锁机制深度)
- [十二、Buffer Pool 与内存管理](#十二buffer-pool-与内存管理)
- [十三、SQL 练习题](#十三sql-练习题)
- [十四、场景设计题](#十四场景设计题)
- [十五、生产故障排查](#十五生产故障排查)
- [十六、面试准备清单](#十六面试准备清单)

---

## 一、SQL 基础（必问）

**Q1：SQL 的几种 JOIN 有什么区别？画图说明。**

> 考察意图：基础必须扎实，JOIN 是日常使用最频繁的操作之一。

**答案要点**：

```
INNER JOIN —— 取交集
  ┌──────┐
  │ A ∩ B│
  └──────┘

LEFT JOIN —— 左表全保留，右表匹配不上为 NULL
  ┌───────────┐
  │ A全部  │  B匹配 │
  └───────────┘

RIGHT JOIN —— 右表全保留，左表匹配不上为 NULL
  ┌───────────┐
  │ A匹配 │  B全部  │
  └───────────┘

FULL OUTER JOIN —— 两表全部（MySQL 不支持，用 UNION 模拟）
  ┌───────────────────┐
  │ A全部   │  B全部   │
  └───────────────────┘

CROSS JOIN —— 笛卡尔积（A 每行 × B 每行），无 ON 条件
```

```sql
-- LEFT JOIN 示例
SELECT u.name, o.order_no
FROM user u
LEFT JOIN orders o ON u.id = o.user_id;
-- 结果：小张有订单显示订单号，小李明没有订单则 order_no 为 NULL
```

---

**Q2：WHERE 和 HAVING 的区别？执行顺序是什么？**

**答案要点**：

| 维度 | WHERE | HAVING |
|------|-------|--------|
| 作用对象 | 原始数据行的过滤 | 分组聚合后的结果过滤 |
| 聚合函数 | **不能**使用聚合函数 | **可以**使用聚合函数（SUM/COUNT/AVG） |
| 执行顺序 | GROUP BY 之前 | GROUP BY 之后 |
| 能否用别名 | 不能（SELECT 中的别名还未定义） | 可以（MySQL 扩展） |

```sql
-- SQL 完整执行顺序：
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT

-- 例子：查询下单超过 3 次的用户
SELECT user_id, COUNT(*) as cnt
FROM orders
WHERE status = 'paid'      -- 先筛选已支付订单
GROUP BY user_id
HAVING COUNT(*) > 3         -- 再筛选超过 3 次的
ORDER BY cnt DESC;
```

---

**Q3：UNION 和 UNION ALL 的区别？**

**答案要点**：

| 维度 | UNION | UNION ALL |
|------|-------|-----------|
| 去重 | 自动去重（DISTINCT） | 不去重 |
| 排序 | 默认排序（可能导致性能问题） | 不排序 |
| 性能 | **慢**（需要临时表 + 排序去重） | **快**（直接合并） |
| 使用场景 | 需要去重的结果集 | 明确知道无重复或允许重复 |

> 面试建议：如果没有去重需求，**一定用 UNION ALL**，性能差距可达数倍。

---

**Q4：MySQL 中有哪些常见的数据类型？VARCHAR 和 CHAR 区别？**

**答案要点**：

| 维度 | CHAR(N) | VARCHAR(N) |
|------|---------|------------|
| 存储方式 | 定长，不足补空格 | 变长，额外 1-2 字节存长度 |
| 最大长度 | 255 字符 | 65535 字节（实际受行大小限制 ~16383 字符） |
| 空间效率 | N 小时浪费空间 | 按实际使用 |
| 性能 | **读取更快**（定长直接计算偏移量） | 需要读长度字段后定位 |
| 适用场景 | 固定长度字段（MD5、手机号、身份证号、状态码） | 变长字段（姓名、地址、描述） |

> 关键点：CHAR 适合频繁更新的字段（定长不会产生页分裂）。

**其他常用类型**：

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| TINYINT(1) | 等同于 Boolean | 状态标识 |
| INT / BIGINT | 整数 | 主键 ID |
| DECIMAL(M,D) | 精确小数（如 DECIMAL(10,2)） | **金额（严禁用 FLOAT/DOUBLE）** |
| TEXT | 大文本 | 文章内容、JSON 字符串 |
| BLOB | 二进制 | 图片、文件（通常存 OSS 链接而非 BLOB） |
| DATETIME | 日期时间（1000-9999） | 创建时间、更新时间 |
| TIMESTAMP | 时间戳（1970-2038） | 自动更新 `ON UPDATE CURRENT_TIMESTAMP` |

---

**Q5：COUNT(\*)、COUNT(1)、COUNT(列名) 的区别？哪个更快？**

> 考察意图：高频题，考察对 COUNT 底层行为的理解。

**答案要点**：

| 写法 | 含义 | 是否统计 NULL |
|------|------|--------------|
| `COUNT(*)` | 统计所有行（包含 NULL） | 包含 |
| `COUNT(1)` | 统计所有行（等价于 COUNT(*)） | 包含 |
| `COUNT(列名)` | 统计该列非 NULL 的行 | **不包含 NULL** |
| `COUNT(DISTINCT 列名)` | 去重计数 | 不包含 NULL |

**性能（MySQL 8.0+）**：
- `COUNT(*)` = `COUNT(1)`：MySQL 优化器会将它们同等处理，速度一样
- `COUNT(列名)`：如果列有索引则走索引，但需要检查 NULL，**略慢**
- 有非空索引列时，`COUNT(*)` 会自动选择最短的二级索引来统计

> 记忆结论：**性能上 COUNT(\*) = COUNT(1) ≥ COUNT(列名)**，语义上按需选用。

---

**Q6：SQL 注入是什么？MyBatis 中 #{} 和 ${} 如何防止 SQL 注入？**

**答案要点**：

SQL 注入：攻击者通过构造恶意输入，拼接 SQL 语句，执行非预期操作。

```sql
-- 恶意输入： ' OR '1'='1' -- 
SELECT * FROM user WHERE name = '' OR '1'='1' --';
-- 等价于：
SELECT * FROM user WHERE name = '' OR '1'='1';  -- 返回全部用户！
```

**MyBatis 中的防护**：
- `#{}`：使用预编译（PreparedStatement），参数作为 `?` 占位符传入，**安全**
- `${}`：直接字符串拼接，**不安全**，仅在动态表名/列名/ORDER BY 时使用 + 代码白名单校验

---

## 二、索引（高频核心）

**Q7：MySQL 索引的底层数据结构是什么？为什么用 B+ 树而不是 B 树或二叉树？**

> 考察意图：索引原理的灵魂之问，几乎每轮必有一道。

**答案要点**：

**B+ 树结构特点**：
```
                  [20 | 50]                     ← 非叶子节点（只存索引，不存数据）
                 /    |    \
        [5|10|15]  [25|35|45]  [60|80]          ← 非叶子节点
        / /  \  \   ...         /    \
       ○ ○   ○  ○              ○     ○          ← 叶子节点（双向链表，存完整数据）
      ↓ ↓   ↓  ↓              ↓     ↓             ← page 指针串联
     [实际数据行]
```

| 特性 | B+ 树 | B 树 | 二叉树/红黑树 |
|------|-------|------|-------------|
| 数据存储 | 仅叶子节点存数据 | 所有节点都存数据 | 所有节点都存数据 |
| 叶子节点连接 | 双向链表（范围查询友好） | 无链表 | 无链表 |
| 树高度 | **矮**（每页存储更多索引，扇出大） | 较高 | **高**（可能退化为链表） |
| 范围查询 | O(logN + M) **极快** | 需要中序遍历 | 需要中序遍历 |
| 磁盘 IO | 非叶子节点小，内存常驻 | 节点较大 | 每次比较可能一次 IO |

**B+ 树优势总结**：
1. **矮胖**：每个节点存储更多键值 → 树高度低 → 磁盘 IO 少
2. **叶子节点链表**：范围查询直接遍历链表，无需回溯
3. **非叶子不存数据**：同样大小的页可以放更多索引，减少磁盘 IO
4. **查询稳定**：任何查询都要走到叶子节点，复杂度稳定 O(logN)

> 加分：补充 InnoDB 中 B+ 树的一个节点大小 = 一个页（16KB）。

---

**Q8：什么是聚簇索引和非聚簇索引？区别是什么？**

> 考察意图：关键概念，决定了你对 MySQL 存储结构的理解深度。

**答案要点**：

```
聚簇索引（Clustered Index）：
  ┌─────────────┐
  │ 主键 B+ 树   │
  │ 叶子节点 =   │
  │ 完整数据行   │
  └─────────────┘

二级索引（Secondary Index）：
  ┌─────────────┐
  │ 索引 B+ 树   │
  │ 叶子节点 =   │
  │ 主键 ID     │──→ 回表查聚簇索引
  └─────────────┘
```

| 维度 | 聚簇索引 | 非聚簇索引（二级索引） |
|------|---------|---------------------|
| 叶子节点内容 | **完整行数据** | 索引列 + **主键值** |
| 数量限制 | 每表**仅一个** | 可以有多个 |
| 默认创建 | InnoDB 自动用主键创建 | 手动创建 |
| 查询过程 | 直接定位数据 | 需**回表**（根据主键再查聚簇索引） |
| 物理顺序 | 数据按索引顺序存储 | 独立结构 |

**无主键时 InnoDB 如何选择聚簇索引**？
1. 有主键 → 用主键
2. 无主键有唯一非空索引 → 用第一个唯一索引
3. 都没有 → 自动生成隐藏的 `row_id`（6 字节）作为聚簇索引

---

**Q9：什么是回表？什么是覆盖索引？如何避免回表？**

> 考察意图：SQL 优化的核心概念，高频题。

**答案要点**：

**回表**：通过二级索引查到主键 ID → 再到聚簇索引中查完整数据，多一次 B+ 树查找。

```
SELECT name, age FROM user WHERE email = 'abc@test.com';

1. 走 email 索引 → 叶子节点拿到 id=100
2. 用 id=100 走聚簇索引 → 拿到 name, age  （这就是回表）
```

**覆盖索引**：查询的列恰好都在索引中包含，无需回表。

```sql
-- 创建覆盖复合索引
CREATE INDEX idx_email_name_age ON user(email, name, age);

-- 查询的列全部在索引中 → 覆盖索引，不回表！
SELECT name, age FROM user WHERE email = 'abc@test.com';
```

在 Explain 中，`Extra` 列显示 `Using index` 即表示使用了覆盖索引。

**如何避免回表**：
- 合理设计复合索引，让查询列都在索引中
- 使用覆盖索引（高频查询场景）
- 减少 `SELECT *`，只查需要的列
- 分页查询先用覆盖索引定位 ID，再 JOIN 原表获取数据（延迟关联）

```sql
-- 延迟关联优化大分页
SELECT * FROM user u
INNER JOIN (
    SELECT id FROM user WHERE status = 1 ORDER BY create_time 
    LIMIT 100000, 20
) t ON u.id = t.id;
-- 子查询走覆盖索引（只需 id），回表只有 20 次
```

---

**Q10：什么是联合索引？最左前缀原则是什么？**

> 考察意图：高频题，索引失效的最常见原因之一。

**答案要点**：

联合索引：多个列组成一个索引，遵循**最左前缀原则**。

```sql
CREATE INDEX idx_a_b_c ON table1(a, b, c);
-- 相当于创建了三个索引：
-- 1. a
-- 2. a, b  
-- 3. a, b, c
```

**最左前缀原则**：查询条件必须从索引的最左列开始匹配，且**不能跳过中间列**。

```sql
-- ✅ 走索引
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3
WHERE a = 1 AND c = 3        -- 部分走索引（a 走索引，c 不走）
WHERE a > 1 AND b = 2        -- a 范围查询，b 不走（范围查询后的列失效）

-- ❌ 不走索引
WHERE b = 2                  -- 缺少最左列 a
WHERE c = 3                  -- 缺少最左列 a
WHERE b = 2 AND c = 3        -- 缺少最左列 a
```

> 追问：`WHERE a = 1 ORDER BY b` 会用到索引吗？  
> 答：会。a = 1 走索引定位，b 在索引中有序，直接使用索引排序（Extra 显示 `Using index condition`），无需额外排序。

> 追问：`WHERE a = 1 ORDER BY c` 呢？  
> 答：a = 1 走索引定位，但 c 索引无序（因为跳过了 b），需要 filesort。

---

**Q11：什么情况下索引会失效？列举常见场景。**

> 考察意图：必问题，日常开发和面试都是重点。

**答案要点**（10 大失效场景）：

| 场景 | 示例 | 原因 |
|------|------|------|
| 1. 违背最左前缀 | `WHERE b = 1`（索引是 a,b,c） | 缺少最左列 |
| 2. 对索引列使用函数 | `WHERE DATE(create_time) = '2024-01-01'` | 函数改变值，无法匹配索引 |
| 3. 对索引列做运算 | `WHERE id + 1 = 10` → 改为 `WHERE id = 9` | 运算后值变了 |
| 4. 隐式类型转换 | `WHERE phone = 13800138000`（phone 是 VARCHAR） | 字符串列与数字比较，MySQL 会 CAST 字符串为数字 |
| 5. LIKE 前导模糊 | `WHERE name LIKE '%张三'` | B+ 树从左到右匹配，前导 `%` 无法定位 |
| 6. OR 连接非索引列 | `WHERE a = 1 OR b = 2`（b 无索引） | OR 的条件要全部走索引才行 |
| 7. 不等于/不等于 | `WHERE status != 1` 或 `WHERE id NOT IN (1,2)` | 走全表扫描（数据量大时） |
| 8. IS NULL / IS NOT NULL | `WHERE name IS NULL` | 视数据分布，可能不走索引 |
| 9. 联合索引范围查询后的列 | `WHERE a = 1 AND b > 2 AND c = 3`（索引 a,b,c） | c 不走索引 |
| 10. 优化器认为全表扫描更快 | 表数据少或返回比例高 | 优化器基于成本估算 |

**口诀**：`模型数空，运函隐，OR常不`（模-模糊查询，型-类型转换，数-不等于，空-IS NULL，运-运算，函-函数，隐-隐式转换，OR常不-OR 和常量比较）

---

**Q12：什么是索引下推（ICP，Index Condition Pushdown）？**

> 考察意图：MySQL 5.6+ 的重要优化，面试加分项。

**答案要点**：

**没有 ICP 时**（MySQL 5.6 之前）：
```
存储引擎 → 按索引查 → 返回全部匹配行给 Server 层 → Server 层过滤
（回表后过滤，大量无效回表）
```

**有 ICP 时**（MySQL 5.6+）：
```
存储引擎 → 按索引查 → 在索引层面先过滤 → 只返回符合条件的行 → Server 层
（索引过滤后再回表，减少回表次数）
```

```sql
-- 索引：idx_name_age(name, age)
SELECT * FROM user WHERE name LIKE '张%' AND age = 25;

-- 无 ICP：所有 name LIKE '张%' 的行都回表，Server 层再筛 age=25
-- 有 ICP：存储引擎在索引中直接用 age=25 过滤，只回表符合条件的行
```

在 Explain 中，`Extra` 列显示 `Using index condition` 表示使用了 ICP。

---

**Q13：怎么查看 SQL 是否使用了索引？Explain 各字段含义？**

> 考察意图：面试中常直接让分析一条 Explain 结果。

**答案要点**（详见[第七章](#七explain-执行计划高频实战)）：

```sql
EXPLAIN SELECT * FROM user WHERE name = '张三' AND age > 20;
```

| 字段 | 含义 | 关注要点 |
|------|------|----------|
| **id** | 查询序号 | 越大越先执行，相同则从上到下 |
| **select_type** | 查询类型 | SIMPLE（简单查询）、PRIMARY（主查询）、SUBQUERY（子查询）、DERIVED（派生表） |
| **table** | 访问的表 | |
| **type** | ★ **访问类型** | system > const > eq_ref > ref > range > index > ALL（越靠前越好） |
| **possible_keys** | 可能用到的索引 | |
| **key** | ★ **实际使用的索引** | NULL 表示没走索引 |
| **key_len** | 索引使用长度 | 越短越好，可判断联合索引用了几个字段 |
| **ref** | 与索引比较的列/常量 | const 表示常量比较 |
| **rows** | ★ **预估扫描行数** | 越小越好 |
| **filtered** | 按条件过滤后剩余百分比 | 越大越好 |
| **Extra** | ★ **额外信息** | Using index（覆盖索引）/ Using where / Using filesort / Using temporary |

**type 从好到差排序**：
```
system > const > eq_ref > ref > range > index > ALL
（唯一   常量   唯一关联  非唯一  范围   全索引  全表）
```

| type | 说明 | 举例 |
|------|------|------|
| **const** | 主键或唯一索引等值匹配，最多一条 | `WHERE id = 1` |
| **eq_ref** | 联表时，被驱动表用主键/唯一索引 | JOIN 的 ON 条件用主键 |
| **ref** | 非唯一索引等值匹配 | `WHERE name = '张三'` |
| **range** | 索引范围扫描 | `WHERE id > 10` / `BETWEEN` / `IN` |
| **index** | 全索引扫描（比 ALL 好点，但也不是好事） | 覆盖索引但没 WHERE |
| **ALL** | **全表扫描（必须优化）** | 没走索引 |

---

**Q14：索引建立有哪些原则和注意事项？**

> 考察意图：考察索引设计经验。

**答案要点**：

**建立索引的原则**：
1. **WHERE、JOIN、ORDER BY 涉及的列建索引**
2. **选择性高的列建索引**（区分度大，如用户 ID >> 性别）
3. **联合索引优于多个单列索引**（最左前缀可覆盖更多查询）
4. **前缀索引**：长字符串列取前缀（`INDEX idx_name(title(20))`）
5. **覆盖索引**：高频查询把 SELECT 的列也加入索引

**不建索引的情况**：
1. 表数据很少（< 1000 行）
2. 频繁更新的列（索引维护成本高）
3. 区分度很低的列（如性别，只有男女）
4. WHERE 条件中用不到的列
5. 长文本字段（考虑前缀索引或全文索引）

**联合索引列的顺序选择**：
- **等值条件放前面，范围条件放后面**
- **区分度高的列放前面**
- 考虑业务查询频率，优先满足高频 SQL

---

**Q15：什么是索引的优点和缺点？**

**答案要点**：

**优点**：
- 大大减少数据扫描量，提高查询速度
- 加速 ORDER BY 和 GROUP BY（利用索引有序性避免 filesort）
- 覆盖索引避免回表，直接返回数据
- 唯一索引保证数据唯一性

**缺点**：
- **占用磁盘空间**（索引也是数据）
- **降低写操作性能**：INSERT/UPDATE/DELETE 需要同时维护索引
- 过多索引增加优化器选择成本

---

## 三、存储引擎（高频）

**Q16：InnoDB 和 MyISAM 的区别？为什么 InnoDB 是默认引擎？**

> 考察意图：经典高频题。

**答案要点**：

| 维度 | InnoDB | MyISAM |
|------|--------|--------|
| **事务支持** | ✅ 支持 ACID | ❌ 不支持 |
| **行级锁** | ✅ 支持行锁（并发高） | ❌ 仅表锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **MVCC** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ redo log 保证 | ❌ 崩溃后数据可能丢失 |
| **聚簇索引** | ✅ 数据按主键组织 | ❌ 数据和索引分离 |
| **全文索引** | MySQL 5.6+ 支持 | ✅ 原生支持 |
| **COUNT(\*)** | 需要遍历 | 单独存储计数，O(1) |
| **压缩** | 支持页压缩 | 支持表压缩 |
| **适用场景** | 高并发 OLTP（在线事务） | 读多写少、日志分析 |

**为什么 InnoDB 是默认引擎**：
1. 支持事务（ACID）—— 现代应用的必备
2. 行级锁 —— 高并发性能好
3. 崩溃恢复 —— redo log 保证数据安全
4. MySQL 8.0 开始，系统表也全部使用 InnoDB

---

**Q17：InnoDB 的数据存储结构是怎样的？Page（页）是什么？**

> 考察意图：深入理解 InnoDB 存储层。

**答案要点**：

```
InnoDB 存储结构层级：
  Tablespace（表空间）
    └─ Segment（段：数据段、索引段、回滚段）
        └─ Extent（区：1MB = 64 个 Page）
            └─ Page（页：16KB）← InnoDB 最小 IO 单位
                └─ Row（行）

Page 内部结构（16KB）：
┌─────────────────────────────┐
│  File Header       (38B)    │
│  Page Header       (56B)    │
│  Infimum + Supremum(26B)    │ ← 伪记录（最小/最大）
│  User Records      (变长)   │ ← 实际数据行（按主键顺序，单向链表）
│  Free Space        (变长)   │ ← 空闲空间
│  Page Directory    (变长)   │ ← 页目录（槽位 + 二分查找）
│  File Trailer      (8B)     │ ← 校验和
└─────────────────────────────┘
```

- 每条记录有 `next_record` 指针，形成单向链表
- Page Directory 将记录分组，每组一个槽，支持页内二分查找
- B+ 树的一个节点 = 一个 Page

---

## 四、事务与锁（高频）

**Q18：什么是事务的 ACID？MySQL 是如何实现的？**

> 考察意图：必问题，要能对应到具体实现机制。

**答案要点**：

| 特性 | 含义 | MySQL InnoDB 实现 |
|------|------|-------------------|
| **A 原子性** | 要么全成功，要么全失败 | **undo log**（回滚日志），记录修改前的值 |
| **C 一致性** | 事务前后数据满足约束 | 由其他三个特性共同保证 + 数据库约束 |
| **I 隔离性** | 并发事务互不干扰 | **MVCC + 锁**（ReadView + 行锁/间隙锁） |
| **D 持久性** | 已提交数据永久保存 | **redo log**（重做日志）+ **doublewrite buffer** |

```
原子性实现——undo log：
  UPDATE account SET balance = 200 WHERE id = 1;  -- balance 原来是 100
  undo log 记录：balance = 100（修改前的值）
  如果事务回滚 → 用 undo log 恢复为 100

持久性实现——redo log：
  WAL（Write-Ahead Logging）：
  1. 修改 Buffer Pool 中的数据页
  2. 写 redo log（顺序写，很快）
  3. 事务提交 → redo log 持久化
  4. 后台线程异步将脏页刷回磁盘
  5. 崩溃恢复 → 重放 redo log
```

---

**Q19：事务的隔离级别有哪些？各级别解决了什么问题？**

> 考察意图：高频必问题。

**答案要点**：

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | MySQL 如何解决 |
|----------|------|-----------|------|---------------|
| **READ UNCOMMITTED** | ✓ 存在 | ✓ 存在 | ✓ 存在 | 几乎不用 |
| **READ COMMITTED** | ✗ | ✓ 存在 | ✓ 存在 | 每次读生成新 ReadView |
| **REPEATABLE READ**（默认） | ✗ | ✗ | ✗（InnoDB 解决） | 事务开始时生成 ReadView，保持整个事务 |
| **SERIALIZABLE** | ✗ | ✗ | ✗ | 所有 SELECT 隐式加共享锁 |

**三个问题的定义**：

```
脏读（Dirty Read）：
  事务 A 读到事务 B 未提交的修改 → B 回滚 → A 读到了不存在的数据

不可重复读（Non-Repeatable Read）：
  事务 A 内两次读取同一行 → 期间事务 B 修改并提交 → 两次结果不同

幻读（Phantom Read）：
  事务 A 内两次范围查询 → 期间事务 B 插入了符合条件的数据并提交 → 第二次多出"幻影行"
```

> 追问：MySQL REPEATABLE READ 如何解决幻读？  
> 答：通过**间隙锁（Gap Lock）**+ **临键锁（Next-Key Lock）** 锁定索引间隙，防止其他事务插入。但对于 `SELECT ... FOR UPDATE` 类型的当前读有效；快照读（普通 SELECT）通过 MVCC 避免幻读。

---

**Q20：MVCC 是什么？工作原理是怎样的？**

> 考察意图：高级面试核心题，对 InnoDB 并发控制机制的理解。

**答案要点**：

MVCC（Multi-Version Concurrency Control）多版本并发控制：通过维护数据的多个版本，实现读写不冲突、提高并发性能。

**核心组件**：

```
1. 隐藏列（每行数据都有）：
   ┌──────┬──────┬─────────┐
   │ DB_TRX_ID │ DB_ROLL_PTR │ DB_ROW_ID │ ... 数据列 ...
   │ (6字节)   │ (7字节)     │ (6字节)   │
   │ 最近修改  │ 回滚指针    │ 隐藏主键   │
   │ 的事务 ID │            │           │
   └──────┴──────┴─────────┘

2. undo log 版本链：
   DB_ROLL_PTR 指向 undo log，形成版本链
   
   当前值 (trx_id=100) → undo: (trx_id=90) → undo: (trx_id=80) → ...

3. ReadView（读视图）：
   - m_ids：当前活跃事务 ID 列表
   - min_trx_id：m_ids 中最小的事务 ID
   - max_trx_id：下一个将分配的事务 ID
   - creator_trx_id：创建 ReadView 的事务 ID
```

**可见性判断规则**：
```
对于 undo log 中的某个版本，其 trx_id：
1. trx_id == creator_trx_id → 可见（自己改的）
2. trx_id < min_trx_id      → 可见（已提交）
3. trx_id >= max_trx_id     → 不可见（未来的事务）
4. min_trx_id <= trx_id < max_trx_id
   - trx_id 在 m_ids 中 → 不可见（活跃未提交）
   - trx_id 不在 m_ids 中 → 可见（已提交）
```

**RC 和 RR 的区别**：
- **READ COMMITTED**：每次 SELECT 都生成新的 ReadView
- **REPEATABLE READ**：第一次 SELECT 时生成 ReadView，之后都用同一个

---

**Q21：什么是当前读和快照读？**

**答案要点**：

| 类型 | 说明 | 示例 |
|------|------|------|
| **快照读** | 读 MVCC 快照版本，不加锁 | 普通 `SELECT` |
| **当前读** | 读最新版本，加锁 | `SELECT ... FOR UPDATE`、`SELECT ... LOCK IN SHARE MODE`、`UPDATE`、`DELETE`、`INSERT` |

```sql
-- 快照读（不加锁，基于 ReadView 读历史版本）
SELECT * FROM user WHERE id = 1;

-- 当前读（加锁，读最新提交版本）
SELECT * FROM user WHERE id = 1 FOR UPDATE;  -- 排他锁
SELECT * FROM user WHERE id = 1 LOCK IN SHARE MODE;  -- 共享锁
UPDATE user SET age = 18 WHERE id = 1;  -- 隐式加排他锁
```

---

## 五、MVCC 与隔离级别（高级必问）

> 本章与第四章紧密关联，高级面试常深入追问 MVCC 相关的隔离级别细节。

**Q22：READ COMMITTED 和 REPEATABLE READ 在 MVCC 实现上的核心区别是什么？**

**答案要点**：

| 维度 | READ COMMITTED | REPEATABLE READ |
|------|---------------|-----------------|
| ReadView 生成时机 | **每次 SELECT 都生成新 ReadView** | **事务第一次 SELECT 时生成，后续复用** |
| 可见性 | 能看到其他事务已提交的修改 | 只看事务开始时的快照 |
| 不可重复读 | 存在（同一事务两次读取结果不同） | 不存在 |

```
时间线          事务 A                   事务 B（trx_id=200）
─────────────────────────────────────────────────────
T1:         BEGIN;  ReadView(before T1)
T2:                                   BEGIN; UPDATE age=20; COMMIT;
T3:         SELECT age → RC: 20（新 ReadView 看到 B 的提交）
                         RR: 18（用 T1 时的 ReadView，看不到 B 的修改）
```

---

**Q23：undo log 是什么？什么时候产生？什么时候删除？**

**答案要点**：

- **产生时机**：数据被修改前，记录修改前的值到 undo log
- **作用**：
  - 事务回滚：将数据恢复到修改前的状态
  - MVCC：提供历史版本供其他事务的快照读
- **删除时机**：当没有任何事务需要这个版本时，由后台 purge 线程清理
- **存储位置**：undo log 存在共享表空间（`ibdata1`）或独立的 undo 表空间（MySQL 8.0）

---

## 六、SQL 优化（高频实战）

**Q24：你在项目中做过哪些 SQL 优化？具体思路是什么？**

> 考察意图：考察实战优化能力和方法论。

**答案要点**（参考答题框架）：

```
优化流程：
1. 发现慢 SQL
   - 慢查询日志（long_query_time=1s）
   - Druid 监控面板
   - 业务反馈页面慢

2. 定位问题
   - EXPLAIN 分析执行计划
   - 看 type 是否为 ALL（全表扫描）
   - 看 rows（预估扫描行数）
   - 看 Extra（Using filesort / Using temporary 是坏信号）

3. 具体优化案例
   案例 1：未走索引 → 添加合适的索引
   案例 2：索引失效（函数/NULL/隐式转换）→ 改写 SQL
   案例 3：N+1 循环查询 → 批量查询
   案例 4：大分页 OFFSET 过大 → 游标分页/延迟关联
   案例 5：SELECT * → 只查需要的列 + 覆盖索引
   案例 6：大表 JOIN → 先缩小结果集再 JOIN

4. 验证效果
   - 再次 EXPLAIN 确认
   - 压测对比优化前后 QPS/响应时间
```

**大分页优化示例**：
```sql
-- ❌ 传统分页：OFFSET 100000 需要扫描并丢弃前 10 万行
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- ✅ 游标分页（记住上次 ID）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;

-- ✅ 延迟关联（先走覆盖索引定位 id，再回表取数据）
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 100000, 20
) t ON o.id = t.id;
```

---

**Q25：一条 SQL 语句在 MySQL 中是如何执行的？**

> 考察意图：考察对 MySQL 整体架构的理解。

**答案要点**：

```
客户端
  ↓
连接器 — 建立连接、权限验证
  ↓
查询缓存（MySQL 8.0 已废弃）— 命中直接返回（8.0 前）
  ↓
分析器 — 词法分析、语法分析
  ↓
优化器 — 选择索引、决定 JOIN 顺序、生成执行计划
  ↓
执行器 — 调用存储引擎接口，逐行读取数据
  ↓
存储引擎（InnoDB）— 从 Buffer Pool / 磁盘读取数据
  ↓
返回结果
```

**各组件职责**：
- **连接器**：TCP 握手、用户认证、权限读取、连接管理
- **分析器**：词法分析（识别关键词）→ 语法分析（判断语法正确性）
- **优化器**：RBO（基于规则）+ CBO（基于成本），选择最优执行计划
- **执行器**：调用引擎接口，逐行读取 → 判断是否满足条件 → 返回

---

**Q26：EXPLAIN 中 filesort 和 temporary 是什么？如何优化？**

**答案要点**：

| Extra 值 | 含义 | 严重程度 | 优化方法 |
|----------|------|----------|----------|
| **Using filesort** | 需要额外排序（非索引排序） | 中 | 给 ORDER BY 列加索引、利用索引有序性 |
| **Using temporary** | 使用临时表（GROUP BY/DISTINCT/UNION） | **高** | 加索引避免创建临时表 |
| **Using index** | 覆盖索引 | ✅ 好 | |
| **Using where** | Server 层过滤 | 视情况 | |
| **Using index condition** | 索引下推 | ✅ 好 | |
| **Using join buffer** | JOIN 用了连接缓冲 | 中 | 给被驱动表加索引 |
| **Select tables optimized away** | 直接返回聚合结果 | ✅ 最好 | |

```sql
-- ❌ Using filesort（name 没有索引）
SELECT * FROM user ORDER BY name;

-- ✅ 给 ORDER BY 列加索引
CREATE INDEX idx_name ON user(name);
-- 或让查询利用已有复合索引的顺序
```

---

**Q27：in 和 exists 有什么区别？什么时候用哪个？**

**答案要点**：

```sql
-- IN：先执行子查询，主查询拿结果去匹配
SELECT * FROM user WHERE id IN (SELECT user_id FROM orders);

-- EXISTS：主查询逐行带入子查询判断，子查询有结果就返回 true
SELECT * FROM user u WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

| 维度 | IN | EXISTS |
|------|-----|--------|
| 执行逻辑 | 子查询结果集驱动 | 主查询（外表）驱动 |
| 适用场景 | 子查询结果集小 | 外表数据少 |
| NULL 处理 | 子查询有 NULL 时整个 IN 可能返回空 | 不受 NULL 影响 |
| 优化器 | MySQL 可能转为 semi-join 优化 | 不会自动转换 |

**选择口诀**：**外层大用 EXISTS，内层大用 IN**。外层（外表）数据多 → EXISTS 效率高；内层（子查询）数据多 → IN 效率高。

---

**Q28：COUNT(*) 和 LIMIT 1 谁更快？什么时候用？**

**答案要点**：

```sql
-- 判断是否存在
SELECT COUNT(*) FROM user WHERE email = 'test@test.com';  -- 要扫完所有匹配行
SELECT 1 FROM user WHERE email = 'test@test.com' LIMIT 1;  -- 找到一条就停

-- 判断是否存在，推荐写法：
SELECT EXISTS(SELECT 1 FROM user WHERE email = 'test@test.com');
-- 返回 1 或 0，也是找到一条就停
```

**区别**：
- `COUNT(*)`：扫描所有匹配行并计数
- `EXISTS` / `LIMIT 1`：找到第一条就返回，性能更好
- **判断是否存在用 EXISTS**，不要用 COUNT(*)

---

## 七、Explain 执行计划（高频实战）

**Q29：EXPLAIN 的 type 字段有哪些值？从好到差排序。**

**答案要点**：

```
system > const > eq_ref > ref > fulltext > ref_or_null > index_merge
> unique_subquery > index_subquery > range > index > ALL
```

| type | 含义 | 示例 |
|------|------|------|
| **system** | 表只有一行（系统表） | 极少见 |
| **const** | 通过主键或唯一索引等值匹配，最多返回 1 行 | `WHERE id = 1` |
| **eq_ref** | JOIN 时被驱动表用主键/唯一索引匹配，每次只返回 1 行 | `JOIN ON a.id = b.id` |
| **ref** | 非唯一索引等值匹配 | `WHERE name = '张三'`（name 有索引） |
| **range** | 索引范围扫描 | `WHERE id > 10`、`BETWEEN`、`IN` |
| **index** | 全索引扫描 | `SELECT name FROM user`（覆盖索引但无 WHERE） |
| **ALL** | **全表扫描（必须优化）** | 没走索引 |

---

**Q30：下面这条 EXPLAIN 结果说明了什么？如何优化？**

```sql
EXPLAIN SELECT * FROM orders 
WHERE user_id = 100 AND status = 'paid' 
ORDER BY create_time DESC LIMIT 10;
```

| id | select_type | table | type | key | rows | Extra |
|----|-------------|-------|------|-----|------|-------|
| 1 | SIMPLE | orders | ref | idx_user_id | 5000 | Using where; Using filesort |

**答案要点**：

**问题诊断**：
1. `type=ref`：走了 `idx_user_id` 索引 ✅
2. `rows=5000`：预估扫描 5000 行，数据量较大 ⚠️
3. `Using where`：`status='paid'` 在 Server 层过滤 ⚠️
4. `Using filesort`：`ORDER BY create_time` 需要额外排序 ❌

**优化方案**：建立复合索引
```sql
CREATE INDEX idx_user_status_time ON orders(user_id, status, create_time);
```

优化后预期：
- `key`: idx_user_status_time
- `Extra`: `Using index condition`（索引下推）或 `Using index`（如果覆盖索引）
- rows 大幅减少
- `Using filesort` 消失（create_time 在索引中有序）

---

## 八、日志系统（高级必问）

**Q31：redo log（重做日志）是什么？作用和工作原理？**

> 考察意图：高级面试高频题，InnoDB 持久性的基础。

**答案要点**：

**redo log**：InnoDB 特有的物理日志，记录"在某个数据页上做了什么修改"。

```
为什么需要 redo log？
  MySQL 内存（Buffer Pool）中的脏页不是立即刷盘的
  → 如果此时宕机，内存中已提交的修改会丢失
  → redo log 在修改内存后立即写入（顺序写，很快）
  → 宕机恢复时，重放 redo log 恢复数据

WAL（Write-Ahead Logging）：
  1. 修改 Buffer Pool 中的数据页
  2. 写 redo log buffer
  3. 事务提交 → redo log buffer 刷到磁盘（fsync）
  4. 后台线程异步将脏页刷到数据文件
  5. checkpoint：脏页刷盘后，对应的 redo log 可以覆盖
```

**redo log 的配置**：
```sql
-- redo log 文件大小（每组）
SHOW VARIABLES LIKE 'innodb_log_file_size';   -- 默认 48MB
SHOW VARIABLES LIKE 'innodb_log_files_in_group';  -- 默认 2 个

-- 刷盘策略（innodb_flush_log_at_trx_commit）
-- 0：每秒刷一次 redo log（性能最好，但可能丢 1 秒数据）
-- 1：每次提交都刷盘（最安全，默认值）
-- 2：每次提交写 OS cache，每秒刷盘（折中）
```

**redo log 的特点**：
- **顺序写**（环形日志文件 `ib_logfile0`、`ib_logfile1`），比随机写快得多
- **循环使用**：有 `write pos`（写入位置）和 `checkpoint`（刷盘位置）
- 数据页最终由 `checkpoint` 机制刷到磁盘，redo log 空间得以释放

---

**Q32：binlog（归档日志）是什么？和 redo log 的区别？**

> 考察意图：高频对比题。

**答案要点**：

| 维度 | redo log | binlog |
|------|----------|--------|
| **所属层** | InnoDB 引擎层特有 | MySQL Server 层（所有引擎共用） |
| **日志类型** | 物理日志（记录页的修改） | 逻辑日志（记录 SQL 语句或行变更） |
| **写入方式** | 循环写（固定大小，覆盖旧数据） | 追加写（文件写满切换新的） |
| **用途** | **崩溃恢复**（crash recovery） | **主从复制**、数据恢复（基于时间点） |
| **大小** | 默认每组 48MB × 2 | 可配置，无上限 |
| **刷盘时机** | 事务执行中持续写 | 事务提交时一次性写入 |

**两阶段提交**（保证 redo log 和 binlog 一致）：
```
1. Prepare 阶段：写 redo log，置为 prepare 状态
2. Commit 阶段：写 binlog
3. 把 redo log 置为 commit 状态
```

---

**Q33：undo log 是什么？与 redo log 的区别？**

> 考察意图：高级面试常见辨析题。

**答案要点**：

| 维度 | undo log | redo log |
|------|----------|----------|
| **记录内容** | 修改**前**的数据（逻辑日志） | 修改**后**的物理变更（物理日志） |
| **作用** | 事务回滚 + MVCC 历史版本 | 崩溃恢复 |
| **存储结构** | undo 表空间/共享表空间 | redo log 文件 |
| **生命周期** | 事务提交后不立即删除，等 purge | 不用的空间可被覆盖 |
| **读写方式** | 随机读写 | 顺序写 |

---

## 九、架构与高可用（高级必问）

**Q34：MySQL 主从复制的原理是什么？**

> 考察意图：高级面试常问，考察架构能力。

**答案要点**：

```
主从复制流程（异步复制）：

Master（主库）                     Slave（从库）
    │                                  │
    ├─ 1. 数据变更                      │
    │     ↓                             │
    ├─ 2. 写 binlog                     │
    │     ↓                             │
    ├─ 3. Binlog Dump 线程             │
    │     → 推送 binlog 事件 →          │
    │                                  ├─ 4. IO 线程接收 → 写入 relay log
    │                                  │     ↓
    │                                  ├─ 5. SQL 线程读取 relay log
    │                                  │     ↓
    │                                  └─ 6. 重放 SQL → 数据同步
```

**三种复制模式**：

| 模式 | 说明 | 优缺点 |
|------|------|--------|
| **STATEMENT** | 记录 SQL 语句 | 日志量小，但 `NOW()`/`UUID()` 等函数可能不一致 |
| **ROW**（默认，5.7.7+） | 记录每行数据的变更 | 最精确，但日志量大 |
| **MIXED** | STATEMENT + ROW 混合 | 一般情况下用 STATEMENT，不确定时用 ROW |

---

**Q35：主从延迟是什么？如何解决？**

**答案要点**：

**原因**：
1. 从库机器性能差
2. 从库单线程执行 relay log（MySQL 5.6 之前）
3. 主库大事务（从库重放耗时长）
4. 网络延迟
5. 从库上有慢查询，拖慢 SQL 线程

**解决方案**：
1. **并行复制**（MySQL 5.7+）：`slave_parallel_workers > 1`，多线程回放
2. **半同步复制**：主库等至少一个从库确认收到 binlog 后才返回（`AFTER_SYNC`）
3. 从库使用更高配置的机器
4. 避免大事务，拆分为小事务
5. 一主多从，读请求轮询/权重分发
6. 强制走主库读（关键业务场景）

---

**Q36：什么是分库分表？什么时候需要分库分表？**

> 考察意图：高级面试，考察大规模数据架构能力。

**答案要点**：

**分库分表时机**：
- 单表数据量 > 2000 万（或单表 > 2GB）
- 单库 QPS 超过单机上限
- 磁盘容量不够

**分库 vs 分表**：

| 方式 | 解决什么 | 方案 |
|------|---------|------|
| **垂直分表** | 单表列过多（宽表）→ 拆列 | 热点列 + 冷列分表 |
| **垂直分库** | 业务耦合 → 拆库 | 按业务模块（用户库、订单库、商品库） |
| **水平分表** | 单表数据量大 → 分行 | 按 ID 取模 / 按时间范围 |
| **水平分库分表** | 数据量巨大 → 分库 + 分表 | 取模 / 范围 + 多库 |

**核心问题与解决**：

| 问题 | 解决方案 |
|------|----------|
| 全局唯一 ID | Snowflake 雪花算法、美团 Leaf、数据库号段模式 |
| 跨分片查询 | 中间件（ShardingSphere）、应用层聚合 |
| 跨分片事务 | Seata 分布式事务、或尽量单分片内事务 |
| 分片键选择 | 选查询最频繁的 WHERE 条件作为分片键 |
| 扩容迁移 | 一致性 Hash、双写 + 灰度迁移 |

**ShardingSphere 配置示例**：
```yaml
# 分片规则：user 表按 id % 4 分 4 个库
rules:
  - !SHARDING
    tables:
      user:
        actualDataNodes: ds${0..3}.user
        databaseStrategy:
          standard:
            shardingColumn: id
            shardingAlgorithmName: mod4
    shardingAlgorithms:
      mod4:
        type: MOD
        props:
          sharding-count: 4
```

---

**Q37：MySQL 高可用架构有哪些？各有什么特点？**

**答案要点**：

| 方案 | 原理 | RPO | RTO | 复杂度 |
|------|------|-----|-----|--------|
| **主从 + 手动切换** | 主库故障后手动改从库为主 | 可能有丢失 | 分钟级 | 低 |
| **MHA** | 自动故障检测 + 切换 + 数据补偿 | 接近 0 | < 30 秒 | 中 |
| **MGR（组复制）** | 多主/Paxos 协议，自动故障转移 | 0 | 秒级 | 高 |
| **Orchestrator + VIP** | 自动拓扑发现 + 自动切换 | 接近 0 | 秒级 | 中 |
| **云数据库**（RDS/PolarDB） | 托管服务，自动 HA | 0 | 秒级 | 低 |

**MHA 工作原理简述**：
1. Manager 定期探测 Master 是否存活
2. Master 故障 → 选最新数据的 Slave 为新 Master
3. 从原 Master 的 binlog 中补偿差异数据到新 Master
4. 其他 Slave 切换到新 Master

---

## 十、锁机制深度

**Q38：MySQL 有哪些锁？分类说一下。**

> 考察意图：锁机制是并发控制的核心，高级岗位常深入追问。

**答案要点**：

```
锁的分类（多维度）：

按粒度分：
├── 全局锁：FLUSH TABLES WITH READ LOCK（全库只读）
├── 表级锁：意向锁、MDL（元数据锁）、AUTO-INC 锁
└── 行级锁：Record Lock、Gap Lock、Next-Key Lock

按模式分：
├── 共享锁（S 锁，读锁）：允许其他事务读，不允许写
└── 排他锁（X 锁，写锁）：不允许其他事务读和写

按算法分（InnoDB 行锁）：
├── Record Lock（记录锁）：锁定索引记录本身
├── Gap Lock（间隙锁）：锁定索引记录之间的间隙（防幻读）
└── Next-Key Lock（临键锁）：Record + Gap，锁定记录和前方间隙

按加锁方式分：
├── 隐式锁：由事务自动加锁
└── 显式锁：SELECT ... FOR UPDATE / LOCK IN SHARE MODE
```

**行锁的三种算法图示**：
```
索引记录：  5    10    15    20    25
           │ │  │  │  │  │  │  │  │
Record:     ●    ●     ●     ●     ●
Gap:       ├────┤├────┤├────┤├────┤├────┤
Next-Key:  ├────●├────●├────●├────●├────●
           (最左)  (5,10] (10,15](15,20](20,25]
```

---

**Q39：什么是死锁？如何排查和避免？**

> 考察意图：相当高频，是实际开发中的常见问题。

**答案要点**：

**死锁**：两个或多个事务相互持有对方需要的锁，形成循环等待。

```
事务 A：UPDATE user SET name='a' WHERE id=1;  → 持有 id=1 的 X 锁
         UPDATE user SET name='a' WHERE id=2;  → 等待 id=2 的 X 锁（B 持有）
事务 B：UPDATE user SET name='b' WHERE id=2;  → 持有 id=2 的 X 锁
         UPDATE user SET name='b' WHERE id=1;  → 等待 id=1 的 X 锁（A 持有）
→ 死锁！
```

**排查方法**：
```sql
-- 1. 查看当前锁等待
SELECT * FROM information_schema.INNODB_TRX;
SELECT * FROM information_schema.INNODB_LOCKS;  -- 8.0 前
SELECT * FROM performance_schema.data_locks;     -- 8.0+

-- 2. 查看最近死锁信息
SHOW ENGINE INNODB STATUS\G
-- 找到 LATEST DETECTED DEADLOCK 段落

-- 3. 开启死锁日志
SET GLOBAL innodb_print_all_deadlocks = ON;
```

**避免死锁的方法**：
1. **固定加锁顺序**：不同事务按相同顺序访问资源（如都按 id 升序加锁）
2. **减少事务时间**：缩小事务范围，把非必要的操作移出事务
3. **使用低隔离级别**：RC 级别没有 Gap Lock，死锁概率降低
4. **尽量用主键/唯一索引更新**：减少锁范围
5. **优化索引**：全表扫描会锁更多行
6. **加锁前排序**：`SELECT ... FOR UPDATE ORDER BY id`，保证加锁顺序一致

---

**Q40：乐观锁和悲观锁的区别？怎么实现？**

**答案要点**：

| 维度 | 乐观锁 | 悲观锁 |
|------|--------|--------|
| **思想** | 先操作，提交时检查冲突 | 先锁住，再操作 |
| **实现方式** | 版本号/时间戳/CAS | `SELECT ... FOR UPDATE` / 行锁 |
| **适用场景** | **读多写少**、冲突概率低 | **写多**、冲突概率高 |
| **性能** | 无锁等待，性能好 | 可能导致锁等待和死锁 |
| **是否阻塞** | 不阻塞 | 阻塞等待 |

```sql
-- 乐观锁（版本号实现）
-- 先查出当前版本
SELECT stock, version FROM product WHERE id = 1;  -- version=3
-- 更新时带上版本号
UPDATE product SET stock = stock - 1, version = version + 1 
WHERE id = 1 AND version = 3;
-- 如果 affected_rows = 0 → 版本号已变 → 重试或提示用户

-- 悲观锁
START TRANSACTION;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;  -- 加排他锁
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

---

## 十一、Buffer Pool 与内存管理

**Q41：Buffer Pool 是什么？工作流程是怎样的？**

> 考察意图：高级面试常问，考察对 InnoDB 内存结构的理解。

**答案要点**：

Buffer Pool 是 InnoDB 的内存缓存区，缓存数据页和索引页，减少磁盘 IO。

```
Buffer Pool 结构：
┌──────────────────────────────────┐
│  Buffer Pool（默认 128MB）       │
│  ┌─────────────────────────┐    │
│  │    链表结构管理 Page     │    │
│  │    ├─ Free List（空闲页）│    │
│  │    ├─ LRU List（缓存页） │    │
│  │    └─ Flush List（脏页） │    │
│  └─────────────────────────┘    │
│  ┌─────────────────────────┐    │
│  │    Change Buffer        │    │  ← 缓存非唯一二级索引的变更
│  │    Adaptive Hash Index  │    │  ← 自适应哈希索引
│  │    Lock Info            │    │  ← 锁信息
│  │    Data Dictionary      │    │  ← 数据字典
│  └─────────────────────────┘    │
└──────────────────────────────────┘
```

**LRU 算法（冷热分离）**：
```
LRU List：
  ┌──────────────┬──────────────┐
  │ 热端 (5/8)   │ 冷端 (3/8)   │
  │ Old Sublist  │ New Sublist  │（MySQL 实际叫法）
  │  热数据      │ 新读入的数据  │
  └──────────────┴──────────────┘

- 新页先放冷端头部（不是热端）
- 在 cold 区域停留 > 1 秒后，再被访问才移到 hot 端
- 防止全表扫描的数据污染整个 Buffer Pool
```

**相关参数**：
```sql
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';     -- 建议设为物理内存的 60%-80%
SHOW VARIABLES LIKE 'innodb_old_blocks_pct';        -- 冷端比例（默认 37%）
SHOW VARIABLES LIKE 'innodb_old_blocks_time';        -- 冷端停留时间（默认 1000ms）
```

---

**Q42：Change Buffer 是什么？什么情况下有效？**

**答案要点**：

Change Buffer 是对**非唯一二级索引**页的修改缓存。

```
当更新一个不在 Buffer Pool 中的二级索引页时：
  → 不立即读磁盘加载该页
  → 而是把修改记录在 Change Buffer 中
  → 等该页被读入 Buffer Pool 时再合并（merge）

适用条件：
  ✅ 非唯一二级索引（唯一索引需要磁盘读取来判断唯一性）
  ✅ 写多读少的场景（大量写入，后续较少读取）
  ❌ 唯一索引
  ❌ 读多写少（立即读取的场景，Change Buffer 无意义）
```

---

## 十二、SQL 练习题

**Q43：高频 SQL 手写题**

### 题 1：查询每个部门工资最高的员工

```sql
-- 解法 1：子查询
SELECT d.name AS dept, e.name, e.salary
FROM employee e
JOIN department d ON e.dept_id = d.id
WHERE (e.dept_id, e.salary) IN (
    SELECT dept_id, MAX(salary) FROM employee GROUP BY dept_id
);

-- 解法 2：窗口函数（MySQL 8.0+，推荐）
SELECT dept, name, salary FROM (
    SELECT d.name AS dept, e.name, e.salary,
           RANK() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC) AS rk
    FROM employee e JOIN department d ON e.dept_id = d.id
) t WHERE rk = 1;
-- 注意：RANK() 并列第一会都返回；ROW_NUMBER() 只返回一个
```

### 题 2：查询连续 3 天登录的用户

```sql
-- 窗口函数解法
SELECT DISTINCT user_id FROM (
    SELECT user_id, login_date,
           LEAD(login_date, 1) OVER (PARTITION BY user_id ORDER BY login_date) AS next1,
           LEAD(login_date, 2) OVER (PARTITION BY user_id ORDER BY login_date) AS next2
    FROM user_login
) t
WHERE DATEDIFF(next1, login_date) = 1 
  AND DATEDIFF(next2, next1) = 1;
```

### 题 3：行转列（每月销售额）

```sql
-- 原始表 sales(user, month, amount)
-- → 列：user, Jan, Feb, Mar, ...

SELECT user,
    SUM(CASE WHEN month = 'Jan' THEN amount ELSE 0 END) AS Jan,
    SUM(CASE WHEN month = 'Feb' THEN amount ELSE 0 END) AS Feb,
    SUM(CASE WHEN month = 'Mar' THEN amount ELSE 0 END) AS Mar
FROM sales
GROUP BY user;
```

### 题 4：留存率计算

```sql
-- 次日留存率 = 第二天还登录的用户数 / 第一天新增用户数
SELECT 
    a.first_date,
    COUNT(DISTINCT a.user_id) AS new_users,
    COUNT(DISTINCT b.user_id) AS retain_users,
    ROUND(COUNT(DISTINCT b.user_id) / COUNT(DISTINCT a.user_id) * 100, 2) AS retention_rate
FROM (
    -- 每个用户的首次登录日期
    SELECT user_id, MIN(login_date) AS first_date FROM user_login GROUP BY user_id
) a
LEFT JOIN user_login b 
    ON a.user_id = b.user_id AND b.login_date = DATE_ADD(a.first_date, INTERVAL 1 DAY)
GROUP BY a.first_date;
```

### 题 5：删除重复数据，保留 id 最小的一条

```sql
DELETE t1 FROM user t1
INNER JOIN user t2 
WHERE t1.id > t2.id AND t1.email = t2.email;
-- 保留 id 最小的，删除其他重复 email 的记录
```

---

## 十三、场景设计题

**Q44：如何设计一个数据库能扛住双十一的流量？**

> 考察意图：综合架构设计题。

**答案要点**：

```
全链路设计：
1. 客户端层：
   - CDN 静态资源分离
   - 前端限流（按钮防抖、验证码）

2. 网关层：
   - Nginx 限流限并发
   - 请求过滤（非法请求拦截）

3. 应用层：
   - Redis 预减库存（不直接打到 DB）
   - MQ 削峰（下单请求先入 MQ，后端慢慢消费）
   - 本地缓存（Caffeine）+ 分布式缓存（Redis）

4. 数据库层：
   - 读写分离（一主多从，读走从库）
   - 分库分表（订单表按 user_id 分片）
   - 热点数据隔离（秒杀商品单独库）
   - 连接池配置调优

5. 数据一致性：
   - 库存扣减：Redis Lua 脚本原子操作
   - 订单生成：MQ 异步 + 数据库唯一索引防重
   - 最终一致性：binlog 同步 + 对账

6. 监控告警：
   - 慢 SQL 监控（> 100ms 告警）
   - QPS/RT/错误率大盘
   - 数据库连接数告警
```

---

**Q45：一张百万级的用户表，如何高效地给邮箱字段加唯一索引？**

**答案要点**：

直接 `ALTER TABLE ADD UNIQUE INDEX` 会锁表（Online DDL 需要时间），对百万级表不可取。

**方案**：
```
1. 先在从库上试验耗时（评估影响）
2. 低峰期操作
3. 使用 pt-online-schema-change（Percona 工具）：
   - 创建新表（带索引）
   - 在原表上创建触发器，增量同步
   - 分批复制数据到新表
   - 原子切换表名
   - 全程不锁表，业务无感知
```

```bash
pt-online-schema-change \
  --alter "ADD UNIQUE INDEX idx_email (email)" \
  D=dbname,t=user \
  --execute
```

---

## 十四、生产故障排查

**Q46：MySQL 的 CPU 突然飙升到 100%，你怎么排查？**

**答案要点**：

```sql
-- 第一步：查看正在执行的 SQL
SHOW FULL PROCESSLIST;
-- 或
SELECT * FROM information_schema.PROCESSLIST WHERE command != 'Sleep';

-- 第二步：找到执行时间长的 SQL
SELECT id, user, host, db, time, state, info 
FROM information_schema.PROCESSLIST 
WHERE time > 5 AND command != 'Sleep'
ORDER BY time DESC;

-- 第三步：分析慢 SQL
-- 找到问题 SQL 后，EXPLAIN 分析
EXPLAIN SELECT ...;

-- 第四步：Kill 问题 SQL（紧急止损）
KILL <id>;

-- 第五步：事后优化
-- - 加索引
-- - 改写 SQL
-- - 限制并发（连接池限制）
```

**常见原因**：
1. 慢 SQL 大量堆积（最常见）
2. 索引失效导致全表扫描
3. 锁等待/死锁导致大量连接堆积
4. 大事务长时间未提交
5. Buffer Pool 太小，频繁磁盘 IO

---

**Q47：MySQL 连接数过多（Too many connections）怎么办？**

**答案要点**：

```sql
-- 1. 查看当前连接数
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';

-- 2. 紧急处理：调大上限（临时的）
SET GLOBAL max_connections = 500;

-- 3. 找到长时间未释放的连接
SELECT id, user, host, db, time, state 
FROM information_schema.PROCESSLIST 
WHERE command = 'Sleep' AND time > 60
ORDER BY time DESC;

-- 4. Kill 空闲连接
KILL <id>;

-- 5. 根治方案
-- - 连接池配置 review（最大连接数、超时时间）
-- - wait_timeout 调整（默认 28800 秒太长）
-- - 应用侧确保连接释放（try-finally 关闭）
-- - 检查是否有连接泄漏
```

---

## 十五、面试准备清单

### 面试前一天
- [ ] 回顾 B+ 树结构与索引原理（Q7-Q9）
- [ ] 回顾 ACID 与隔离级别（Q18-Q22）
- [ ] 回顾 MVCC 原理（Q20-Q22）
- [ ] 回顾 EXPLAIN 各字段含义（Q13、Q29-Q30）
- [ ] 回顾 redo log 与 binlog 区别（Q31-Q32）
- [ ] 回顾锁分类与死锁排查（Q38-Q39）
- [ ] 准备 2 个自己经历的 SQL 优化案例
- [ ] 准备 1 个生产故障排查案例

### 面试前 1 小时
- [ ] 过一遍索引失效的 10 个场景
- [ ] 默写一遍 EXPLAIN type 从好到差排序
- [ ] 默写一遍 SQL 执行顺序（FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT）

### 面试中注意事项
- [ ] SQL 手写题：先想清楚逻辑再动笔，注意边界条件（NULL、重复）
- [ ] 场景设计题：先问清楚需求，画架构图，再详细展开
- [ ] 索引问题：分析时务必提到 EXPLAIN 和覆盖索引
- [ ] 不会的问题：不要硬编，说明你目前的认知边界

### 面试后复盘
- [ ] 记录没答上来的题目，补充学习
- [ ] 总结面试官的出题风格（重原理/重实战/重SQL手写）
- [ ] 更新知识体系

---

### 📊 难度分级速查

| 级别 | 题号 | 适合年限 |
|------|------|----------|
| **初级** | Q1-Q6, Q16, Q24, Q29, Q43 | 1-3 年 |
| **中级** | Q7-Q15, Q17, Q25-Q28, Q30, Q43 窗口函数题 | 3-5 年 |
| **高级** | Q18-Q23, Q31-Q42, Q44-Q47 | 5-8 年 |
| **架构** | Q34-Q37, Q44 | 8+ 年 |

---

> **最后提醒**：MySQL 面试最看重的不是你背了多少八股文，而是你能否把原理和实际场景结合起来。每道题都试着关联到你的项目经验——"这个索引优化在我们的订单表上让查询从 2 秒降到了 50 毫秒"——这样的回答比纯理论更有说服力。

---

*最后更新：2026年7月19日*
