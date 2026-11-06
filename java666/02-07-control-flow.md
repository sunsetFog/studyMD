# 07 · 流程控制

## 1. if / else if / else

```java
if (score >= 90) {
    grade = 'A';
} else if (score >= 80) {
    grade = 'B';
} else if (score >= 60) {
    grade = 'C';
} else {
    grade = 'D';
}
```

要点：

- 条件必须是 `boolean`，**不能写 `if (1)`**（与 C 不同）。
- 单行也必须用大括号（避免后续添加语句时的经典 bug）。
- 卫语句（guard clause）优先，减少嵌套：

```java
// 差
if (user != null) {
    if (user.isActive()) {
        if (user.hasPermission()) {
            doWork(user);
        }
    }
}

// 好
if (user == null) return;
if (!user.isActive()) return;
if (!user.hasPermission()) return;
doWork(user);
```

## 2. switch

### 2.1 传统 switch

```java
switch (day) {
    case 1:
    case 2:
        type = "工作日";
        break;                // 忘记 break 会「穿透」
    case 6:
    case 7:
        type = "周末";
        break;
    default:
        type = "未知";
}
```

支持类型：`byte/short/char/int`、对应包装类、`String`（Java 7+）、`enum`。**不支持** `long/float/double/boolean`。

### 2.2 箭头 switch（Java 14+）

```java
switch (day) {
    case 1, 2, 3, 4, 5 -> type = "工作日";
    case 6, 7 -> type = "周末";
    default -> type = "未知";
}
```

- 无穿透，无需 break。
- 多标签用逗号。
- `default` 仍可省略（但不建议）。

### 2.3 switch 表达式

```java
String type = switch (day) {
    case 1, 2, 3, 4, 5 -> "工作日";
    case 6, 7 -> "周末";
    default -> throw new IllegalArgumentException("bad day: " + day);
};
```

用代码块时需要 `yield` 返回：

```java
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY -> 7;
    default -> {
        int len = day.name().length();
        yield len;                 // 不是 return
    }
};
```

### 2.4 模式匹配 switch（Java 21 正式）

```java
String describe(Object o) {
    return switch (o) {
        case null -> "null";
        case Integer i when i > 0 -> "正整数 " + i;
        case Integer i -> "整数 " + i;
        case String s -> "字符串长度 " + s.length();
        case int[] arr -> "数组长度 " + arr.length;
        default -> "其他";
    };
}
```

- 类型模式 + `when` 守卫。
- 编译器检查**穷尽性**：sealed 类型需覆盖所有子类型。
- `case null` 显式处理 null（否则抛 NPE）。

### 2.5 enum + switch 最佳实践

```java
switch (order.getStatus()) {
    case CREATED -> handleCreated(order);
    case PAID -> handlePaid(order);
    case SHIPPED -> handleShipped(order);
    default -> throw new IllegalStateException("未处理状态: " + order.getStatus());
}
```

> 若逻辑复杂，考虑用**枚举抽象方法**替代 switch（见 03-14 枚举篇），更符合开闭原则。

## 3. 循环

### 3.1 for

```java
for (int i = 0; i < 10; i++) { }

// 多变量
for (int i = 0, j = 10; i < j; i++, j--) { }

// 死循环
for (;;) { break; }
```

### 3.2 增强 for（for-each）

```java
for (String s : list) { }
for (Map.Entry<String, Integer> e : map.entrySet()) { }
for (int v : arr) { }
```

限制：

- 不能修改集合结构（会 ConcurrentModificationException）。
- 无法获取索引，需要索引时用普通 for 或 `IntStream.range`。
- 遍历数组有边界检查，性能与普通 for 接近。

### 3.3 while / do-while

```java
while (condition) { }

do {
    // 至少执行一次
} while (condition);
```

### 3.4 迭代安全删除

```java
// 正确：Iterator.remove
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().isEmpty()) it.remove();
}

// 正确：removeIf
list.removeIf(String::isEmpty);

// 错误：增强 for 中 list.remove -> ConcurrentModificationException
```

## 4. break / continue / label

```java
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 2) continue outer;   // 跳到外层下一次迭代
        if (i == 2) break outer;      // 跳出外层循环
    }
}
```

- `break` 跳出最近一层循环/switch。
- `continue` 跳过本次迭代。
- 带标签可跳出多层，但会降低可读性；能用方法 + `return` 解决就别用 label。

```java
// 更清晰：抽方法
private boolean findTarget(int[][] matrix) {
    for (int[] row : matrix)
        for (int v : row)
            if (v == target) return true;
    return false;
}
```

## 5. 循环性能与习惯

| 写法 | 建议 |
|---|---|
| `for (int i = 0; i < list.size(); i++)` | 每次调用 size()，ArrayList 无碍；LinkedList 是 O(n)，必须避免 |
| 增强 for 遍历 LinkedList | 内部用迭代器，O(n) 正确 |
| 循环内字符串拼接 | 用 StringBuilder |
| 循环内 `list.contains` | O(n²)，改 Set |
| 循环内创建对象 | 关注 GC 压力，必要时复用或提出循环 |
| 循环内远程调用 | 批量接口替代（N+1 问题） |

```java
// 典型 N+1
for (Long id : ids) {
    User u = userMapper.selectById(id);   // 100 次查询
}
// 改：selectByIds(ids) 一次查完
```

## 6. 常见陷阱

1. **浮点循环变量**：`for (double d = 0; d != 1; d += 0.1)` 可能死循环。
2. **修改循环变量**：增强 for 中修改引用不影响集合。
3. **switch 忘 break**：传统写法特有。
4. **循环条件里做重计算**：`for (int i = 0; i < expensive(); i++)`。
5. **do-while 少执行**：条件为 false 时也执行一次。
6. **break 只跳一层**：嵌套循环需 label 或抽方法。
7. **循环内抛异常中断**：批量处理应 try-catch 单条，避免整批失败。

```java
// 批量处理：单条失败不影响其余
for (Item item : items) {
    try {
        process(item);
    } catch (Exception e) {
        log.warn("处理失败, id={}", item.getId(), e);
    }
}
```

## 7. 控制流的可读性设计

- **早返回**优于深层嵌套。
- **策略表**优于长 if-else 链：

```java
Map<Type, Handler> handlers = Map.of(
    Type.A, this::handleA,
    Type.B, this::handleB
);
handlers.getOrDefault(type, this::handleDefault).handle(input);
```

- **状态机**优于嵌套 switch：订单/工单流转用枚举状态 + 转换表。
- **异常替代错误码**处理非正常路径，但不要用异常做流程控制（性能与语义都差）。
