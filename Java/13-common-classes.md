# 13 · 常用类：Math、BigDecimal、Random、System

## 一、Math

```java
Math.abs(-5);            // 5
Math.max(3, 7);          // 7  min 同理
Math.pow(2, 10);         // 1024.0
Math.sqrt(16);           // 4.0
Math.round(3.5);         // 4（四舍五入；-3.5 → -3，向正无穷取整）
Math.ceil(3.1);          // 4.0 向上取整
Math.floor(3.9);         // 3.0 向下取整
Math.floorDiv(-7, 2);    // -4（向下取整除法，负数语义清晰）
Math.floorMod(-7, 2);    // 1 （% 对负数结果带符号，floorMod 恒非负）
Math.random();           // [0,1) 随机小数

// 精确运算（防溢出）
Math.addExact(Integer.MAX_VALUE, 1);   // 抛 ArithmeticException
Math.multiplyExact(...);

// 三角/对数
Math.sin/cos/tan/log/log10/exp
```

**取模负数陷阱**：`-7 % 2 == -1`；需要循环/分桶索引用 `Math.floorMod(-7, 2) == 1`。

## 二、BigDecimal（货币计算必用）

### 为什么 double 不行

```java
System.out.println(0.1 + 0.2);   // 0.30000000000000004
// 二进制浮点无法精确表示十进制小数
```

### 正确用法

```java
// 1. 创建：必须用 String 构造器（double 构造器带入精度误差！）
BigDecimal a = new BigDecimal("0.1");          // 精确
BigDecimal bad = new BigDecimal(0.1);          // 0.100000000000000005551115...
BigDecimal ok = BigDecimal.valueOf(0.1);       // 等价先转字符串，推荐

// 2. 运算（不可变：返回新对象！）
BigDecimal price = new BigDecimal("19.99");
BigDecimal total = price.multiply(new BigDecimal("3"));   // 结果要接住
price.add(x) / subtract / divide / remainder

// 3. 除法必须指定舍入模式与精度（除不尽直接抛异常！）
BigDecimal avg = total.divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP);

// 4. 比较：用 compareTo（equals 连 scale 都比！）
new BigDecimal("1.0").equals(new BigDecimal("1.00"));       // false！
new BigDecimal("1.0").compareTo(new BigDecimal("1.00"));    // 0 ✓

// 5. 输出
total.stripTrailingZeros().toPlainString();   // 去尾零、不用科学计数法
```

### 舍入模式速查

| 模式 | 语义 |
|---|---|
| HALF_UP | 四舍五入（最常用） |
| HALF_DOWN | 五舍六入 |
| HALF_EVEN | 银行家舍入（5 前奇进偶舍，统计上更公平） |
| CEILING / FLOOR | 向上/向下 |
| UP / DOWN | 远离/靠近零截断 |

**金额规范**：以「分」为单位的 long 存库也很常见（彻底避开小数）；展示层再转元。

## 三、Random 与 ThreadLocalRandom

```java
Random r = new Random(42);        // 种子：可复现（测试用固定种子）
r.nextInt(100);                   // [0,100)
r.nextInt(5, 10);                 // [5,10)（Java 17+）
r.nextBoolean(); r.nextDouble();
r.ints(10, 0, 100).toArray();     // 批量生成

// 多线程环境：共享 Random 会 CAS 竞争
ThreadLocalRandom.current().nextInt(100);   // 线程独立实例，性能好
```

**安全场景禁用 Random**（token/密码盐）→ `SecureRandom`。

## 四、System 与 Runtime

```java
System.currentTimeMillis();       // 毫秒时间戳（计时用 System.nanoTime 更准）
System.nanoTime();                // 纳秒（仅用于测量耗时，非挂钟时间）
System.getProperty("user.dir");   // 系统属性
System.getenv("PATH");            // 环境变量
System.out/err/in;                // 标准流
System.exit(0);                   // 强退（跳过优雅停机，框架层别用）

Runtime.getRuntime().availableProcessors();   // 核数（线程池依据）
```

## 五、Objects 工具

```java
Objects.equals(a, b);                  // null 安全比较
Objects.requireNonNull(obj, "msg");    // 断言非空（快速失败）
Objects.requireNonNullElse(v, def);    // 默认值
Objects.hash(a, b);                    // hashCode 组合
Objects.isNull(o) / nonNull(o);        // 谓词（Stream filter 用）
```

## 六、UUID

```java
UUID.randomUUID().toString();      // "550e8400-e29b-41d4-a716-446655440000"
// 用途：请求 ID、幂等键、文件名随机段
// 不适合做数据库主键：无序、长，索引分裂 → 主键用自增/雪花 ID
```

## 常见坑

- **坑 1**：`new BigDecimal(0.1)` → 精度污染；String 构造或 valueOf
- **坑 2**：BigDecimal 忘接返回值（不可变）→ 计算无效
- **坑 3**：divide 不给 scale/rounding → 除不尽抛 ArithmeticException
- **坑 4**：BigDecimal.equals 比较 → 用 compareTo
- **坑 5**：计时用 currentTimeMillis 跨时钟回拨 → 测耗时用 nanoTime
- **坑 6**：负数取模预期错 → floorMod

## Checklist

- [ ] 金额计算全链路 BigDecimal/String 构造（或分单位 long）
- [ ] divide 指定 scale + RoundingMode
- [ ] 多线程随机用 ThreadLocalRandom，安全随机用 SecureRandom
- [ ] Objects 工具替代手写判空
