# 37 · 常用设计模式（Java 视角）

> 原则基础见 12 篇。本文只讲**实战高频**模式 + JDK 中的真实案例。

## 一、创建型

### 1. 单例（Singleton）

```java
// 推荐 1：枚举（防反射/反序列化破坏）
enum Config { INSTANCE; }

// 推荐 2：静态内部类（惰性 + 线程安全，31 篇原理）
class ConnectionManager {
    private ConnectionManager() { }
    private static class Holder {
        static final ConnectionManager INSTANCE = new ConnectionManager();
    }
    public static ConnectionManager getInstance() { return Holder.INSTANCE; }
}

// 3：双重检查锁（面试必会手写，21 篇 volatile 原理）
class Singleton {
    private static volatile Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}
```

### 2. 工厂方法 / 简单工厂

```java
// 简单工厂：静态方法 + switch/map 分发（小规模够用）
static PayChannel of(String type) {
    return switch (type) {
        case "alipay" -> new Alipay();
        case "wechat" -> new WechatPay();
        default -> throw new IllegalArgumentException(type);
    };
}

// JDK 案例：Integer.valueOf（缓存）、Calendar.getInstance、LoggerFactory
```

### 3. 建造者（Builder）

```java
// 链式构建多参数对象，消灭望远镜构造器
User user = User.builder()
        .name("tom").age(18).email("t@x.com")
        .build();

// 手写核心（Lombok @Builder 一键生成）
class User {
    private final String name;
    private final Integer age;
    private User(Builder b) { this.name = b.name; this.age = b.age; }
    static Builder builder() { return new Builder(); }
    static class Builder {
        private String name; private Integer age;
        Builder name(String v) { this.name = v; return this; }
        Builder age(Integer v) { this.age = v; return this; }
        User build() { return new User(this); }
    }
}
// JDK 案例：StringBuilder、Stream.Builder、HttpRequest.newBuilder
```

## 二、结构型

### 4. 适配器（Adapter）

```java
// 让不兼容接口协同：包装旧接口成新接口
class LegacyPaymentAdapter implements PayChannel {
    private final LegacyPay legacy;
    public PayResult pay(Order o) {
        return map(legacy.doPayLegacy(o.toLegacyForm()));   // 协议转换
    }
}
// JDK 案例：InputStreamReader（字节流适配字符流）、Arrays.asList
```

### 5. 装饰器（Decorator）

```java
// 不改原类，动态叠加能力：IO 体系核心（26 篇）
InputStream in = new BufferedInputStream(new FileInputStream(f));
// BufferedInputStream 装饰 FileInputStream，接口一致可层层叠加

// JDK 案例：Collections.unmodifiableList、synchronizedList
```

### 6. 代理（Proxy）

```java
// 控制访问：懒加载、权限、日志、RPC stub
// JDK 动态代理 / CGLIB（29 篇完整示例）—— Spring AOP 的实现基础
```

### 7. 外观（Facade）

```java
// 子系统复杂门面简化：一个入口编排多个服务
class CheckoutFacade {
    public Receipt checkout(Order o) {
        cartService.validate(o);
        inventoryService.deduct(o);
        paymentService.charge(o);
        return logisticsService.ship(o);
    }
}
```

## 三、行为型

### 8. 策略（Strategy）—— 业务开发第一高频

```java
interface DiscountPolicy { BigDecimal apply(Order o); }

// Map 注册表替代 if-else 链（衔接 12 篇 OCP）
class DiscountRegistry {
    private final Map<MemberLevel, DiscountPolicy> policies;
    BigDecimal settle(Order o, Member m) {
        return policies.get(m.level()).apply(o);
    }
}
// JDK 案例：Comparator、ThreadPoolExecutor 的拒绝策略
```

### 9. 模板方法（Template Method）

```java
// 父类定骨架，子类填细节（08 篇抽象类）
abstract class AbstractBatchJob {
    public final void run() {          // final 骨架
        validate();
        List<Item> items = load();     // 钩子
        process(items);                // 钩子
        report();
    }
    protected abstract List<Item> load();
    protected abstract void process(List<Item> items);
    private void validate() { }
    private void report() { }
}
// JDK 案例：AbstractList、JdbcTemplate、AbstractApplicationContext.refresh
```

### 10. 观察者（Observer）

```java
// 事件广播解耦：Spring Event（SpringBoot 系列 18 篇）就是它的框架化
interface Listener<T> { void on(T event); }
class EventBus {
    private final List<Listener<Event>> listeners = new CopyOnWriteArrayList<>();
    void publish(Event e) { listeners.forEach(l -> l.on(e)); }
}
// JDK 案例：PropertyChangeListener、GUI 事件
```

### 11. 责任链（Chain of Responsibility）

```java
// 请求沿链传递：过滤器/拦截器/审批流
interface Filter { void doFilter(Request req, Chain next); }
// Servlet Filter 链、Netty Pipeline、Spring Interceptor
```

### 12. 状态（State）

```java
// 状态机替代 if-else 状态判断
enum OrderStatus {
    CREATED { OrderStatus pay() { return PAID; } },
    PAID { OrderStatus ship() { return SHIPPED; } },
    SHIPPED { OrderStatus done() { return COMPLETED; } };
    OrderStatus pay() { throw new IllegalStateException(); }   // 非法转移默认拒绝
}
```

## 四、模式选择速查

| 问题 | 模式 |
|---|---|
| 全局唯一资源 | 单例（枚举/Holder） |
| 对象创建逻辑复杂/参数多 | Builder |
| 类型分发消除 if-else | 策略 + 注册表 |
| 流程固定步骤可变 | 模板方法 |
| 无侵入增强能力 | 装饰器 / 代理 |
| 事件通知解耦 | 观察者（直接用 Spring Event） |
| 多层校验/处理 | 责任链 |
| 状态流转 | 状态机 |
| 接口不兼容整合 | 适配器 |

## 五、反模式警示

```
- 过度设计：三行代码上五层抽象 → YAGNI（12 篇）
- 模式堆砌：为用而用 → 模式服务于变化点
- 单例滥用：全局可变状态 → 并发灾难
- 继承复用替代组合 → 12 篇
```

## Checklist

- [ ] 单例三种写法与破坏防御
- [ ] 策略+注册表消灭业务 if-else
- [ ] 能说出每个模式在 JDK/Spring 中的案例
- [ ] 抽象时机：第二变化点出现
