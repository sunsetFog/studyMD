# 63 · 设计模式（Java 落地）

## 1. 六大原则（SOLID + 迪米特）

| 原则 | 含义 | 一句话 |
|---|---|---|
| **S** 单一职责 | 一个类只做一件事 | 类超过 500 行就该拆 |
| **O** 开闭 | 对扩展开放、对修改关闭 | 新增策略不改调用方 |
| **L** 里氏替换 | 子类可替换父类 | 别让 Square 继承 Rectangle |
| **I** 接口隔离 | 小而专的接口 | 别做胖接口 |
| **D** 依赖倒置 | 依赖抽象 | 面向接口编程 |
| 迪米特法则 | 最少知识 | 别 A.getB().getC().doX() |

## 2. 创建型

### 单例（Singleton）

```java
// 1. 枚举（推荐）
public enum Config {
    INSTANCE;
    private final Map<String, String> props = new HashMap<>();
    public String get(String k) { return props.get(k); }
}

// 2. 静态内部类（懒加载 + 线程安全）
public class Config {
    private Config() { }
    private static class Holder { static final Config INSTANCE = new Config(); }
    public static Config getInstance() { return Holder.INSTANCE; }
}

// 3. 双重检查锁
public class Config {
    private static volatile Config instance;
    public static Config getInstance() {
        if (instance == null) {
            synchronized (Config.class) {
                if (instance == null) instance = new Config();
            }
        }
        return instance;
    }
}
```

**破坏单例的方式**：反射、序列化、克隆。枚举天然防前两种。

### 工厂方法（Factory Method）

```java
public interface PaymentFactory {
    Payment create();
}

public class AlipayFactory implements PaymentFactory {
    @Override public Payment create() { return new AlipayPayment(); }
}

// 简单工厂（静态方法）
public final class PaymentFactory {
    public static Payment create(String type) {
        return switch (type) {
            case "alipay" -> new AlipayPayment();
            case "wechat" -> new WechatPayment();
            default -> throw new IllegalArgumentException(type);
        };
    }
}
```

### 抽象工厂（Abstract Factory）

```java
public interface UiFactory {
    Button createButton();
    Checkbox createCheckbox();
}

public class MacFactory implements UiFactory { ... }
public class WinFactory implements UiFactory { ... }
```

用于「一族相关产品」的创建。

### 建造者（Builder）

```java
public final class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final Duration timeout;

    private HttpRequest(Builder b) {
        this.url = b.url; this.method = b.method;
        this.headers = Map.copyOf(b.headers); this.timeout = b.timeout;
    }

    public static Builder builder(String url) { return new Builder(url); }

    public static final class Builder {
        private final String url;
        private String method = "GET";
        private final Map<String, String> headers = new HashMap<>();
        private Duration timeout = Duration.ofSeconds(10);

        private Builder(String url) { this.url = Objects.requireNonNull(url); }

        public Builder method(String m) { this.method = m; return this; }
        public Builder header(String k, String v) { headers.put(k, v); return this; }
        public Builder timeout(Duration d) { this.timeout = d; return this; }
        public HttpRequest build() { return new HttpRequest(this); }
    }
}
```

### 原型（Prototype）

```java
public record Point(int x, int y) {
    public Point withX(int nx) { return new Point(nx, y); }
}

// 或实现 Cloneable（不推荐，见 03-13）
```

## 3. 结构型

### 适配器（Adapter）

```java
// 旧接口
public interface LegacyLogger { void log(String msg); }

// 适配到新接口
public class Slf4jAdapter implements LegacyLogger {
    private final Logger log = LoggerFactory.getLogger(Slf4jAdapter.class);
    @Override public void log(String msg) { log.info(msg); }
}
```

### 装饰器（Decorator）

```java
public interface Coffee { BigDecimal cost(); String desc(); }

public class Espresso implements Coffee {
    public BigDecimal cost() { return new BigDecimal("18"); }
    public String desc() { return "Espresso"; }
}

public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee inner;
    protected CoffeeDecorator(Coffee inner) { this.inner = inner; }
}

public class Milk extends CoffeeDecorator {
    public Milk(Coffee inner) { super(inner); }
    @Override public BigDecimal cost() { return inner.cost().add(new BigDecimal("3")); }
    @Override public String desc() { return inner.desc() + " + 牛奶"; }
}

// 使用：层层包装
Coffee c = new Milk(new Milk(new Espresso()));
```

**JDK 中的装饰器**：`BufferedReader(new FileReader(...))`、`Collections.unmodifiableList`。

### 代理（Proxy）

见 11-53 动态代理。

### 外观（Facade）

```java
public class OrderFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;

    public void placeOrder(Order order) {
        inventory.reserve(order);      // 对外只暴露一个方法
        payment.charge(order);
        shipping.schedule(order);
    }
}
```

### 桥接（Bridge）

```java
// 抽象：形状
public abstract class Shape {
    protected final Color color;              // 桥接维度
    protected Shape(Color c) { this.color = c; }
    public abstract void draw();
}

// 实现维度：颜色
public interface Color { String fill(); }

// 两个维度独立扩展：RedCircle、BlueSquare...
```

### 组合（Composite）

```java
public interface Node {
    long size();
}

public class File implements Node {
    private final long bytes;
    public long size() { return bytes; }
}

public class Dir implements Node {
    private final List<Node> children = new ArrayList<>();
    public long size() { return children.stream().mapToLong(Node::size).sum(); }
}
```

### 享元（Flyweight）

```java
// 复用不可变对象（如棋盘上的棋子、字符串常量池）
public class ChessPieceFactory {
    private static final Map<String, ChessPiece> CACHE = new ConcurrentHashMap<>();

    public static ChessPiece of(String type, String color) {
        return CACHE.computeIfAbsent(color + ":" + type,
                k -> new ChessPiece(type, color));
    }
}
```

## 4. 行为型

### 策略（Strategy）

```java
public interface DiscountStrategy {
    BigDecimal apply(BigDecimal price);
}

public class VipDiscount implements DiscountStrategy {
    public BigDecimal apply(BigDecimal p) { return p.multiply(new BigDecimal("0.9")); }
}

public class PriceService {
    private final Map<String, DiscountStrategy> strategies;

    public PriceService(List<DiscountStrategy> list) { ... }

    public BigDecimal price(String type, BigDecimal p) {
        return strategies.getOrDefault(type, x -> x).apply(p);
    }
}
```

### 观察者（Observer）

```java
public interface Listener { void onEvent(Event e); }

public class EventBus {
    private final List<Listener> listeners = new CopyOnWriteArrayList<>();

    public void register(Listener l) { listeners.add(l); }

    public void publish(Event e) {
        for (Listener l : listeners) {
            try { l.onEvent(e); }
            catch (Exception ex) { log.error("listener failed", ex); }
        }
    }
}
```

### 模板方法（Template Method）

```java
public abstract class DataImporter {
    public final void importData(Path p) {       // 固定流程
        validate(p);
        List<String> lines = read(p);
        List<Row> rows = parse(lines);
        save(rows);
    }
    protected abstract List<String> read(Path p);
    protected abstract List<Row> parse(List<String> lines);
    protected abstract void save(List<Row> rows);
    protected void validate(Path p) { }
}
```

### 责任链（Chain of Responsibility）

```java
public abstract class Handler {
    private Handler next;
    public Handler setNext(Handler h) { this.next = h; return h; }

    public final void handle(Request req) {
        if (canHandle(req)) doHandle(req);
        else if (next != null) next.handle(req);
    }

    protected abstract boolean canHandle(Request req);
    protected abstract void doHandle(Request req);
}

// 链：AuthHandler → RateLimitHandler → BusinessHandler
```

Servlet Filter、Spring Interceptor、Netty Pipeline 都是责任链。

### 状态（State）

```java
// 见 03-14 的枚举状态机
public sealed interface OrderState permits Created, Paid, Shipped {
    OrderState pay();
    OrderState ship();
}

record Created() implements OrderState {
    public OrderState pay() { return new Paid(); }
    public OrderState ship() { throw new IllegalStateException("未支付不能发货"); }
}
```

### 命令（Command）

```java
public interface Command { void execute(); }

public class OrderCommand implements Command {
    private final OrderService svc;
    private final Order order;
    public void execute() { svc.create(order); }
}

// 可用于撤销/重做、队列化、事务日志
```

### 迭代器（Iterator）

见 05-28。

### 中介者（Mediator）

```java
// 聊天室：用户不直接互相引用，通过 ChatRoom 转发
public class ChatRoom {
    public void send(User from, String msg) {
        users.stream().filter(u -> u != from).forEach(u -> u.receive(from, msg));
    }
}
```

### 备忘录（Memento）

```java
public record Memento(String state) { }

public class Editor {
    private String content;
    public Memento save() { return new Memento(content); }
    public void restore(Memento m) { this.content = m.state(); }
}
```

### 访问者（Visitor）

```java
public sealed interface Shape permits Circle, Square {
    <R> R accept(Visitor<R> v);
}

record Circle(double r) implements Shape {
    public <R> R accept(Visitor<R> v) { return v.visit(this); }
}

interface Visitor<R> {
    R visit(Circle c);
    R visit(Square s);
}
```

适合「数据结构稳定、操作多变」的场景（编译器 AST）。

### 解释器（Interpreter）

用于简单 DSL（如规则引擎、SQL 解析）。现代多用 ANTLR。

## 5. 模式选型速查

| 问题 | 模式 |
|---|---|
| 需要唯一实例 | 单例 |
| 创建逻辑复杂 | 工厂 / 建造者 |
| 接口不兼容 | 适配器 |
| 动态增加功能 | 装饰器 |
| 控制访问 | 代理 |
| 简化复杂子系统 | 外观 |
| 多维度变化 | 桥接 |
| 树形结构统一处理 | 组合 |
| 大量相似对象 | 享元 |
| 算法可替换 | 策略 |
| 状态流转 | 状态 |
| 事件通知 | 观察者 |
| 流程固定、步骤可变 | 模板方法 |
| 逐级处理请求 | 责任链 |
| 操作可撤销 | 命令 |
| 对象结构稳定、操作多变 | 访问者 |

## 6. Java/Spring 中的模式

| 模式 | 实例 |
|---|---|
| 单例 | Spring Bean（默认） |
| 工厂 | `BeanFactory`、`Calendar.getInstance()` |
| 建造者 | `StringBuilder`、`HttpRequest.newBuilder()` |
| 适配器 | `InputStreamReader`、`HandlerAdapter` |
| 装饰器 | `BufferedInputStream`、`Collections.unmodifiableList` |
| 代理 | Spring AOP、MyBatis Mapper |
| 外观 | `JdbcTemplate`、`RestTemplate` |
| 观察者 | Spring `ApplicationEvent` |
| 模板方法 | `JdbcTemplate`、`AbstractList` |
| 责任链 | `FilterChain`、`HandlerInterceptor` |
| 策略 | `Comparator`、`ResourceLoader` |
| 组合 | `CompositeCacheManager` |

## 7. 反模式提醒

1. **过度设计**：3 行代码引入 5 个接口。
2. **模式堆砌**：为了用模式而用模式。
3. **单例滥用**：全局状态、难测试。
4. **继承滥用**：应优先组合。
5. **上帝类**：什么都知道的 `Manager`/`Util`。
6. **贫血模型**：只有 getter/setter 的实体 + 一堆 Service（视团队约定）。
7. **万能 Util**：`CommonUtils` 几百个方法。

## 8. 使用建议

- 先写「能跑」的代码，重构时再引入模式。
- 模式是**沟通词汇**，不是目标。
- 优先用语言特性（Lambda 替代策略类、record 替代 DTO 样板）。
- 一个模式解决一个具体变化点，别指望它解决所有问题。
