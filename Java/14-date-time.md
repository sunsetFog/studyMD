# 14 · 日期时间：java.time 全面替代 Date

> JSR-310（Java 8+）。老 API（Date/Calendar/SimpleDateFormat）**只读不写**——非线程安全 + 可变 + API 反直觉。

## 一、核心类型一览

| 类型 | 含义 | 不可变 |
|---|---|---|
| **LocalDate** | 日期（2026-11-06） | ✓ |
| **LocalTime** | 时间（09:30:00） | ✓ |
| **LocalDateTime** | 日期+时间，无时区 | ✓ |
| ZonedDateTime / OffsetDateTime | 带时区 | ✓ |
| Instant | UTC 时间戳点 | ✓ |
| Duration | 时间段（秒/纳秒） | ✓ |
| Period | 日期段（年月日） | ✓ |
| Year/Month/YearMonth | 局部概念 | ✓ |

**全家族不可变**——任何「修改」都返回新对象。

## 二、创建与获取

```java
LocalDate today = LocalDate.now();
LocalDate d = LocalDate.of(2026, 11, 6);
LocalDate parsed = LocalDate.parse("2026-11-06");    // ISO 格式直接解析
LocalTime t = LocalTime.of(9, 30);
LocalDateTime dt = LocalDateTime.of(2026, 11, 6, 9, 30);
LocalDateTime now = LocalDateTime.now();

Instant ts = Instant.now();                          // UTC 时间戳（存库推荐）
Instant fromEpoch = Instant.ofEpochMilli(1730000000000L);

today.getYear(); today.getMonthValue(); today.getDayOfMonth();
today.getDayOfWeek();                                // FRIDAY
today.lengthOfMonth(); today.isLeapYear();
```

## 三、运算（链式，返回新对象）

```java
LocalDate nextWeek = today.plusWeeks(1);
LocalDateTime later = now.plusHours(2).plusMinutes(30);
LocalDate lastMonth = today.minusMonths(1);
LocalDate first = today.withDayOfMonth(1);           // with 修改某分量
LocalDate nextSun = today.with(TemporalAdjusters.next(DayOfWeek.SUNDAY));
LocalDate endOfMonth = today.with(TemporalAdjusters.lastDayOfMonth());

// 间隔
Period age = Period.between(birthday, today);        // 年月日
Duration dur = Duration.between(t1, t2);             // 时长
long days = ChronoUnit.DAYS.between(d1, d2);         // 直接算天数差
long minutes = ChronoUnit.MINUTES.between(t1, t2);

// 比较
d1.isBefore(d2); d1.isAfter(d2); d1.equals(d2); d1.isEqual(d2);
```

## 四、格式化（线程安全）

```java
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
DateTimeFormatter cn = DateTimeFormatter.ofPattern("yyyy年MM月dd日", Locale.CHINA);

String s = now.format(fmt);                          // "2026-11-06 09:30:00"
LocalDateTime back = LocalDateTime.parse("2026-11-06 09:30:00", fmt);

// ISO 标准格式常量
DateTimeFormatter.ISO_LOCAL_DATE / ISO_DATE_TIME
```

**DateTimeFormatter 是线程安全的**（SimpleDateFormat 不是）——可定义为 static final 常量复用。

## 五、时区处理（坑最多的地方）

```java
// 1. 存储基准：全后端统一 UTC 或固定时区，DB 用 DATETIME/TIMESTAMP 存 LocalDateTime/Instant

// 2. 转换
ZonedDateTime shanghai = Instant.now().atZone(ZoneId.of("Asia/Shanghai"));
ZonedDateTime utc = shanghai.withZoneSameInstant(ZoneOffset.UTC);
LocalDateTime localView = shanghai.toLocalDateTime();   // 同一时刻的本地视图

// 3. UTC 时间戳与展示
long epoch = Instant.now().getEpochSecond();            // 秒
Instant.ofEpochSecond(epoch).atZone(ZONE_CN).toLocalDateTime();

// 4. JDBC 映射（JDBC 4.2+）
// TIMESTAMP ↔ Instant / OffsetDateTime
// DATETIME ↔ LocalDateTime
```

**铁律**：
- 计算机内部传递/存储：UTC（Instant）或明确时区（ZonedDateTime）
- 仅「用户视角展示」才用 LocalDateTime
- 不要混用 `Date.getTime()` 与 `Instant.toEpochMilli()` 语义（其实等价，但新旧 API 别混）

## 六、旧 API 互转

```java
Date old = Date.from(instant);                 // Instant → Date
Instant i = old.toInstant();
LocalDateTime ldt = LocalDateTime.ofInstant(old.toInstant(), ZoneId.systemDefault());
Date d2 = Date.from(ldt.atZone(ZoneId.systemDefault()).toInstant());
```

## 七、实战规范

```java
// 1. 实体字段
private LocalDate birthday;          // 纯日期
private LocalDateTime createdAt;     // 本地时间（配合统一时区约定）
private Instant updatedAt;           // UTC 精确点

// 2. 时间注入（可测试性）
private final Clock clock;
public boolean isExpired() {
    return Instant.now(clock).isAfter(expireAt);    // 测试注入 Clock.fixed(...)
}

// 3. 计算耗时（27 篇 System.nanoTime 对比）
long start = System.nanoTime();
...
long ms = (System.nanoTime() - start) / 1_000_000;
```

## 常见坑

- **坑 1**：忘接返回值（不可变）→ `today.plusDays(1)` 结果丢弃
- **坑 2**：SimpleDateFormat 多线程共享 → 数据错乱；换 DateTimeFormatter
- **坑 3**：`parse` 无 formatter 且格式非 ISO → DateTimeParseException
- **坑 4**：时区不一致（应用 GMT+8、DB UTC）→ 时间偏 8 小时；全链路约定统一
- **坑 5**：`plusMonths(1)` 跨月末（1/31 + 1 月 → 2/28）→ 需要月内逻辑时用 TemporalAdjusters 明确
- **坑 6**：Duration 用于 LocalDate（无时间部分）→ UnsupportedTemporalTypeException，日期差用 Period/ChronoUnit

## Checklist

- [ ] 新代码零 Date/Calendar，全部 java.time
- [ ] DateTimeFormatter 定义为常量复用
- [ ] 时区策略成文：存 UTC/Instant，展示本地化
- [ ] 时间获取走 Clock 注入（可测试）
