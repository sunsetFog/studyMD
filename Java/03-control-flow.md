# 03 · 流程控制

## 一、if / if-else / if-else if

```java
int score = 85;

if (score >= 90) {
    System.out.println("优秀");
} else if (score >= 80) {
    System.out.println("良好");
} else if (score >= 60) {
    System.out.println("及格");
} else {
    System.out.println("不及格");
}
```

**要点**：
- 单行 if 也写 `{}`（代码规范，防「悬挂 else」与后续加行事故）
- 条件必须是 boolean，`if (n)` 不合法（与 C 不同）

## 二、switch

### 传统 switch（fall-through 陷阱）

```java
int day = 3;
switch (day) {
    case 1:
    case 7:
        System.out.println("周末");   // 多 case 共享逻辑
        break;                        // 不写 break 会贯穿！
    case 2:
    case 3:
    case 4:
    case 5:
    case 6:
        System.out.println("工作日");
        break;
    default:
        System.out.println("无效");
}
```

支持类型：int/short/byte/char、String（Java 7+）、枚举。**不支持 long/浮点**。

### 新式 switch 表达式（Java 14+，推荐）

```java
String type = switch (day) {
    case 1, 7 -> "周末";
    case 2, 3, 4, 5, 6 -> "工作日";
    default -> "无效";
};   // 表达式：有返回值、无 fall-through、箭头语法

// 复杂逻辑用 yield
int days = switch (month) {
    case 1, 3, 5, 7, 8, 10, 12 -> 31;
    case 2 -> isLeap ? 29 : 28;
    default -> {
        int d = 30;
        yield d;
    }
};

// Java 21 模式匹配
Object obj = ...;
String desc = switch (obj) {
    case Integer i when i > 0 -> "正整数 " + i;
    case Integer i -> "非正整数";
    case String s -> "字符串 " + s;
    case null -> "空值";               // 21 支持匹配 null
    default -> "其他";
};
```

## 三、for / 增强 for

```java
// 标准 for：需要索引/自定义步长
for (int i = 0; i < 10; i++) { }

// 增强 for（for-each）：遍历数组/Collection，只读场景首选
int[] arr = {1, 2, 3};
for (int x : arr) { }

for (Map.Entry<String, Integer> e : map.entrySet()) {
    System.out.println(e.getKey() + "=" + e.getValue());
}

// 嵌套：break 只跳内层，用标签跳外层（慎用，可读性差）
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (i * j > 4) break outer;
    }
}
```

## 四、while / do-while

```java
int i = 0;
while (i < 10) { i++; }          // 先判后执行

do { i--; } while (i > 0);       // 至少执行一次（菜单循环常用）
```

**死循环**：`while (true)` + 内部 break/return，或轮询场景。

## 五、break / continue / return

```java
for (...) {
    if (invalid) continue;   // 跳过本次，继续下一次
    if (found) break;        // 跳出整个循环
}
```

## 六、选型经验

| 场景 | 用 |
|---|---|
| 确定次数 | for |
| 遍历集合/数组 | 增强 for |
| 条件循环（次数未知） | while |
| 至少执行一次 | do-while |
| 离散值分支 ≤ 5 且枚举/常量 | switch 表达式 |
| 集合转换/过滤 | Stream（18 篇），替代命令式循环 |

## 七、控制流反模式

```java
// 反模式 1：深层嵌套 → 用卫语句提前返回
// 坏
if (user != null) {
    if (user.isActive()) {
        if (user.hasPermission()) { doWork(); }
    }
}
// 好（卫语句）
if (user == null) return;
if (!user.isActive()) return;
if (!user.hasPermission()) return;
doWork();

// 反模式 2：用异常控制流程
try { Integer.parseInt(s); } catch (...) { /* 不是数字 */ }
// 好：能预判的条件用判断，不用异常当 goto
```

## 常见坑

- **坑 1**：switch 忘写 break → 逻辑贯穿到下一个 case
- **坑 2**：for-each 中修改集合 → ConcurrentModificationException；要删除用 Iterator.remove 或 removeIf
- **坑 3**：for-each 中给数组元素赋值 OK，但无法改遍历结构
- **坑 4**：`==` 比较字符串 → 恒比较引用（String 字面量池的巧合命中不可依赖），用 equals
- **坑 5**：浮点作循环条件累计 → 精度误差死循环或早停

## Checklist

- [ ] 新式 switch 表达式能熟练使用
- [ ] 卫语句消除深层嵌套
- [ ] for-each 与索引 for 按场景选择
- [ ] 不用异常控制业务流程
