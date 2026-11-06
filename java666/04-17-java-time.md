# 17 · java.time 日期时间 API

## 1. 为什么废弃 Date/Calendar

| 问题 | 旧 API | 新 API |
|---|---|---|
| 可变性 | `Date.setTime()` 可变 | 全部不可变、线程安全 |
| 月份从 0 开始 | `Calendar.JANUARY = 0` | `Month.JANUARY`（1-12） |
| 格式化线程安全 | `SimpleDateFormat` 非线程安全 | `DateTimeFormatter` 不可变 |
| 时区处理 | 混乱 | `ZonedDateTime` + `ZoneId` |
| 运算 | 手写 Calendar | `plusDays` 等链式 API |

> 新代码**禁止**使用 `Date`/`Calendar`/`SimpleDateFormat`（老接口交互除外）。

## 2. 核心类型全景

| 类型 | 含义 | 示例 |
|---|---|---|
| `Instant` | 时间戳（UTC，纳秒精度） | `2026-01-01T00:00:00Z` |
| `LocalDate` | 日期（无时间无时区） | `2026-01-01` |
| `LocalTime` | 时间（无日期无时区） | `14:30:00` |
| `LocalDateTime` | 日期 + 时间（无时区） | `2026-01-01T14:30` |
| `ZonedDateTime` | 带时区的完整时间 | `2026-01-01T14:30+08:00[Asia/Shanghai]` |
| `OffsetDateTime` | 带偏移量 | `2026-01-01T14:30+08:00` |
| `Duration` | 时间量（秒/纳秒） | `PT2H30M` |
| `Period` | 日期量（年月日） | `P1Y2M3D` |
| `ZoneId` | 时区 | `Asia/Shanghai` |
| `YearMonth` / `MonthDay` | 年月 / 月日 | `2026-01` |

## 3. 创建

```java
LocalDate today = LocalDate.now();
LocalDate date = LocalDate.of(2026, 1, 1);
LocalDate parse = LocalDate.parse("2026-01-01");

LocalTime time = LocalTime.of(14, 30, 0);
LocalDateTime dt = LocalDateTime.of(date, time);
LocalDateTime dt2 = LocalDateTime.parse("2026-01-01T14:30:00");

Instant now = Instant.now();                       // UTC 时间戳
Instant fromMillis = Instant.ofEpochMilli(1767225600000L);

ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
ZonedDateTime utc = ZonedDateTime.now(ZoneOffset.UTC);
```

## 4. 运算（不可变，返回新对象）

```java
LocalDate d = LocalDate.of(2026, 1, 31);
d.plusDays(1);          // 2026-02-01
d.plusMonths(1);        // 2026-02-28（自动处理月末）
d.minusYears(1);
d.withDayOfMonth(15);
d.with(TemporalAdjusters.firstDayOfNextMonth());

// 常用调整器
LocalDate firstDay = d.with(TemporalAdjusters.firstDayOfMonth());
LocalDate lastDay = d.with(TemporalAdjusters.lastDayOfMonth());
LocalDate nextMonday = d.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
LocalDate workday = d.with(TemporalAdjusters.nextOrSame(DayOfWeek.MONDAY));
```

判断：

```java
d.isBefore(other); d.isAfter(other); d.isEqual(other);
d.isLeapYear();
d.lengthOfMonth();
d.getDayOfWeek();       // DayOfWeek.MONDAY
d.getDayOfYear();
```

## 5. 时间差

```java
Duration dur = Duration.between(start, end);
dur.toMillis(); dur.toSeconds(); dur.toMinutes(); dur.toHours();

Period p = Period.between(LocalDate.of(2020,1,1), LocalDate.of(2026,3,15));
p.getYears(); p.getMonths(); p.getDays();     // 6, 2, 14

// 总天数差（最常用）
long days = ChronoUnit.DAYS.between(d1, d2);
long months = ChronoUnit.MONTHS.between(d1, d2);
long hours = ChronoUnit.HOURS.between(t1, t2);
```

> `Period.getDays()` 是「余数」不是总天数，跨月计算要用 `ChronoUnit.DAYS.between`。

## 6. 与旧 API 互转

```java
// Date <-> Instant
Date date = Date.from(instant);
Instant instant = date.toInstant();

// Calendar <-> ZonedDateTime
ZonedDateTime zdt = calendar.toZonedDateTime();

// java.sql.Date/Time/Timestamp
java.sql.Date sqlDate = java.sql.Date.valueOf(localDate);
LocalDate ld = sqlDate.toLocalDate();
java.sql.Timestamp ts = java.sql.Timestamp.valueOf(localDateTime);

// 时间戳
long millis = instant.toEpochMilli();
Instant i = Instant.ofEpochMilli(millis);
long seconds = instant.getEpochSecond();
```

## 7. 格式化与解析

```java
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String s = LocalDateTime.now().format(fmt);
LocalDateTime dt = LocalDateTime.parse("2026-01-01 12:00:00", fmt);

// 预定义常量（推荐，线程安全且标准）
DateTimeFormatter.ISO_LOCAL_DATE
DateTimeFormatter.ISO_DATE_TIME
DateTimeFormatter.BASIC_ISO_DATE       // 20260101
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)

// 带时区/语言
DateTimeFormatter f = DateTimeFormatter.ofPattern("yyyy年MM月dd日 EEEE", Locale.CHINA);
```

### 模式字母

| 字母 | 含义 | 示例 |
|---|---|---|
| `yyyy` | 年 | 2026 |
| `MM` | 月 | 01 |
| `dd` | 日 | 05 |
| `HH` | 时（0-23） | 14 |
| `hh` | 时（1-12） | 02 |
| `mm` | 分 | 30 |
| `ss` | 秒 | 45 |
| `SSS` | 毫秒 | 123 |
| `EEEE` | 星期全称 | 星期一 |
| `a` | 上午/下午 | 下午 |
| `XXX` | 时区偏移 | +08:00 |
| `z` | 时区名 | CST |

**注意**：`YYYY`（week-based year）与 `yyyy` 不同，跨年周会出错；`DD`（年中的天）与 `dd` 不同。用错是经典 bug。

## 8. 时区处理

```java
ZoneId shanghai = ZoneId.of("Asia/Shanghai");
ZoneId utc = ZoneOffset.UTC;

// 同一时刻在不同时区的表示
Instant now = Instant.now();
ZonedDateTime cn = now.atZone(shanghai);
ZonedDateTime us = now.atZone(ZoneId.of("America/New_York"));

// 转换
ZonedDateTime converted = cn.withZoneSameInstant(utc);   // 同一时刻
LocalDateTime localShifted = cn.withZoneSameLocal(utc); // 同一本地时间（语义变了）

// 所有可用时区
ZoneId.getAvailableZoneIds();
```

规范：

- **存储用 UTC 时间戳/`Instant`**，展示时转本地时区。
- 数据库用 `timestamp with time zone` 或统一存 UTC。
- 跨时区业务（航班、跨国订单）必须显式带时区。

## 9. 夏令时（DST）陷阱

```java
ZoneId ny = ZoneId.of("America/New_York");
LocalDate day = LocalDate.of(2026, 3, 8);
ZonedDateTime zdt = day.atTime(2, 30).atZone(ny);
// 该时刻不存在（跳过 2:00-3:00），Java 会自动调整到 3:30

// 使用 resolve 策略
zdt.withEarlierOffsetAtOverlap();
```

中国无夏令时，但做国际化产品必须考虑。

## 10. 与数据库映射

| SQL 类型 | Java 类型 |
|---|---|
| `DATE` | `LocalDate` |
| `TIME` | `LocalTime` |
| `TIMESTAMP` | `LocalDateTime` |
| `TIMESTAMP WITH TIME ZONE` | `OffsetDateTime` / `Instant` |
| `BIGINT`（时间戳） | `Instant` / `long` |

MyBatis 3.5+/JPA 2.2+ 原生支持 JSR-310。

```yaml
# Jackson 配置
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    serialization:
      write-dates-as-timestamps: false
```

```java
// 局部定制
public record Order(@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss") LocalDateTime createdAt) { }
```

## 11. 常用片段

```java
// 今天开始/结束
LocalDateTime startOfDay = LocalDate.now().atStartOfDay();
LocalDateTime endOfDay = LocalDate.now().atTime(LocalTime.MAX);

// 本月第一天
LocalDate firstDay = LocalDate.now().withDayOfMonth(1);

// 上周一
LocalDate lastMonday = LocalDate.now().minusWeeks(1)
        .with(TemporalAdjusters.previousOrSame(DayOfWeek.MONDAY));

// 年龄
int age = Period.between(birthday, LocalDate.now()).getYears();

// 两个日期之间的所有日期
List<LocalDate> range = start.datesUntil(end).toList();   // Java 9+

// 秒级时间戳
long ts = Instant.now().getEpochSecond();

// 兼容前端毫秒时间戳
LocalDateTime t = LocalDateTime.ofInstant(Instant.ofEpochMilli(ms), ZoneId.systemDefault());

// 耗时统计
long start = System.nanoTime();
// ...
long costMs = (System.nanoTime() - start) / 1_000_000;
```

## 12. 常见坑

1. 用 `Period.getDays()` 当总天数。
2. 用 `YYYY` 格式化年份（跨年周错位）。
3. `SimpleDateFormat` 作为静态字段共享（线程不安全，结果错乱）。
4. `LocalDateTime` 直接当时间戳跨时区传输（丢失时区信息）。
5. 数据库时区与 JVM 时区不一致（`serverTimezone` 未配置）。
6. `Duration.between` 参数顺序反了（负数）。
7. 比较用 `equals` 而 `LocalDateTime` 还带纳秒——`truncatedTo(ChronoUnit.SECONDS)` 再比。
8. 月份从 1 开始（新 API）vs 从 0 开始（Calendar），迁移时注意。
