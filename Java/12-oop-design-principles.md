# 12 · OOP 设计原则：SOLID 与组合优于继承

## 一、SOLID 五原则

### S — 单一职责（SRP）

一个类只有一个变化理由。

```java
// 坏：User 又管数据又管存储又管发邮件
class User {
    void save() { }
    void sendEmail() { }
}

// 好：拆分
class User { /* 纯数据+领域行为 */ }
class UserRepository { void save(User u) { } }
class EmailService { void sendWelcome(User u) { } }
```

### O — 开闭原则（OCP）

对扩展开放，对修改关闭——**靠多态实现**。

```java
// 坏：每加一种支付方式都要改这个 if
if (type.equals("alipay")) { ... }
else if (type.equals("wechat")) { ... }

// 好：策略接口，新增实现不改旧代码（07 篇多态实战）
interface PayChannel { PayResult pay(Order o); }
```

### L — 里氏替换（LSP）

子类必须能无感替换父类——**不能削弱父类契约**。

```java
// 经典反例：正方形继承长方形 → setWidth 破坏了长方形语义
// 判据：所有用父类的地方，换成子类后行为不应意外
```

### I — 接口隔离（ISP）

客户端不依赖用不到的方法。**大接口拆小**。

```java
// 坏：一个巨型 Worker 接口，机器人被迫实现 eat/sleep
// 好：
interface Workable { void work(); }
interface Eatable { void eat(); }        // 人类实现两者，机器人只实现 Workable
```

### D — 依赖倒置（DIP）

高层模块依赖抽象，不依赖具体。

```java
// 坏：OrderService 直接 new MySQLDatabase()
// 好：依赖 Repository 接口，具体实现由配置/容器注入（Spring 的基石）
public class OrderService {
    private final OrderRepository repo;      // 抽象
    public OrderService(OrderRepository repo) { this.repo = repo; }   // 构造注入
}
```

## 二、组合优于继承（合成复用原则）

**继承是白盒复用**（耦合父类实现），**组合是黑盒复用**（只依赖接口）。

```java
// 坏：为复用 PrintWriter 的能力而继承
class MyLogger extends PrintWriter { }     // 暴露了一堆用不到的方法

// 好：组合
class MyLogger {
    private final PrintWriter writer;
    MyLogger(PrintWriter w) { this.writer = w; }
    void log(String msg) { writer.println(msg); }
}
```

**继承只用于真正的 is-a + 需要多态**；否则组合。

```java
// 经典反例：栈继承向量
class MyStack<E> extends ArrayList<E> { }   // 暴露 add(0,..) 破坏栈语义
// JDK 教训：java.util.Stack extends Vector 就是被诟病的历史设计
// 正确做法：组合 + 委托
class MyStack<E> {
    private final List<E> items = new ArrayList<>();
    public void push(E e) { items.add(e); }
    public E pop() { return items.remove(items.size() - 1); }
}
```

## 三、常用设计原则补充

| 原则 | 一句话 |
|---|---|
| KISS | 保持简单，别过度设计 |
| YAGNI | 你不会需要它——不为想象中的需求写代码 |
| DRY | 不重复自己（但「错误的抽象比重复更贵」） |
| 最小惊讶 | API 行为符合直觉 |

## 四、原则如何落到日常代码

```
Controller 只做协议转换          → SRP
Service 依赖 Repository 接口     → DIP
新渠道新策略不改 switch          → OCP（策略/工厂）
DTO 用 record，领域对象用 class  → 封装
工具类 final + private 构造器    → 防误用
```

## 五、实战：一个遵循原则的小案例

```java
// 需求：订单按不同会员等级打折
// SRP：折扣计算独立成组件
// OCP：新等级加实现不改旧代码
// DIP：OrderService 依赖 DiscountPolicy 接口

public interface DiscountPolicy {
    BigDecimal apply(BigDecimal price, Member member);
}

public class VipDiscount implements DiscountPolicy {
    @Override
    public BigDecimal apply(BigDecimal price, Member m) {
        return price.multiply(new BigDecimal("0.9"));
    }
}

public class OrderService {
    private final DiscountPolicy policy;                  // DIP
    public OrderService(DiscountPolicy policy) { this.policy = policy; }

    public BigDecimal settle(Order o, Member m) {         // SRP：只管下单结算
        return policy.apply(o.getAmount(), m);
    }
}
```

## 常见坑

- **坑 1**：为「代码复用」滥用继承 → 耦合父类实现，父类改子类崩
- **坑 2**：过度设计（上来就抽象工厂三件套）→ YAGNI；先直写，出现第二/三个变化点再抽象
- **坑 3**：接口爆炸（几十个单方法接口无章法）→ ISP 是按客户端拆分，不是越碎越好
- **坑 4**：子类加强前置条件（LSP 违反）→ 调用方被埋雷

## Checklist

- [ ] 能对每条 SOLID 举出一个「违反 vs 遵守」的代码对
- [ ] 默认组合，继承需 is-a 论证
- [ ] 抽象时机：第二次变化点出现时
- [ ] 依赖注入风格贯穿设计
