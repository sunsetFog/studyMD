# 35 · 序列化

## 1. 什么是序列化

把对象转换成字节流（可存储/传输），反序列化是逆过程。

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;

    private Long id;
    private String name;
    private transient String password;      // 不参与序列化
}
```

```java
// 序列化
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.bin"))) {
    oos.writeObject(user);
}

// 反序列化
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.bin"))) {
    User u = (User) ois.readObject();
}
```

## 2. 核心规则

| 规则 | 说明 |
|---|---|
| 必须实现 `Serializable` | 标记接口，无方法 |
| 所有非 transient 字段必须可序列化 | 否则 `NotSerializableException` |
| `static` 字段不序列化 | 属于类不属于对象 |
| `transient` 字段不序列化 | 反序列化后为默认值 |
| 父类未实现 Serializable | 父类无参构造器会被调用，字段为默认值 |
| 反序列化不走构造器 | 直接由 JVM 分配并填充字段 |

```java
// 父类无参构造器必须可访问
class Parent {             // 未实现 Serializable
    Parent() { System.out.println("parent ctor"); }
}
class Child extends Parent implements Serializable { }

new ObjectInputStream(in).readObject();   // 会打印 "parent ctor"
```

## 3. serialVersionUID

```java
private static final long serialVersionUID = 1L;
```

作用：版本一致性校验。反序列化时比较流中的 UID 与类中的 UID，不一致抛 `InvalidClassException`。

不显式声明时，编译器根据类结构（字段、方法、接口）自动生成——**任何改动都会变**，导致旧数据无法反序列化。

规范：**显式声明 UID**，修改类时保持不变（只要兼容）。

兼容的修改：

- 新增字段（旧数据反序列化后新字段为默认值）。
- 删除字段（旧数据中的该字段被忽略）。
- 修改 `static`/`transient` 修饰。

不兼容：

- 修改字段类型。
- 修改类名/包名。
- 改变继承关系。
- 从 Serializable 改为不可序列化。

## 4. transient

```java
public class Session implements Serializable {
    private String token;
    private transient Instant createdAt;       // 不序列化
    private transient Thread worker;           // 线程不可序列化，必须 transient
}
```

适合：

- 敏感信息（密码）。
- 不可序列化的对象（Thread、Stream、Connection）。
- 可重新计算的派生字段（缓存）。

注意：`transient` 对 `static` 无意义（static 本来就不序列化）。

## 5. 自定义序列化

### writeObject / readObject

```java
public class Account implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password;

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();                        // 先序列化非 transient
        out.writeObject(encrypt(password));              // 自定义字段
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        this.password = decrypt((String) in.readObject());
    }

    private void readObjectNoData() throws ObjectStreamException {
        // 流中没有该类数据时调用
    }
}
```

方法签名必须精确（private 也会被反射调用）。

### Externalizable

```java
public class Point implements Externalizable {
    private int x, y;

    public Point() { }                     // 必须有无参构造器

    @Override public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(x);
        out.writeInt(y);
    }

    @Override public void readExternal(ObjectInput in) throws IOException {
        x = in.readInt();
        y = in.readInt();
    }
}
```

- 完全控制序列化格式，性能更好。
- 必须公开无参构造器（反序列化时调用）。

### readResolve（单例保护）

```java
public class Singleton implements Serializable {
    public static final Singleton INSTANCE = new Singleton();
    private Singleton() { }

    private Object readResolve() {
        return INSTANCE;                  // 反序列化返回已有实例
    }
}
```

### writeReplace

```java
private Object writeReplace() {
    return new SerializedForm(this);      // 序列化时替换为另一个对象
}
```

## 6. 安全性问题（重要）

Java 原生反序列化是**高危功能**：

1. **反序列化漏洞**：恶意构造的字节流可触发任意代码执行（gadget chain，如 CommonsCollections）。
2. **数据篡改**：无完整性校验。
3. **拒绝服务**：超大对象/深递归导致 OOM。
4. **绕过构造器**：不变量与校验被跳过。

防护：

- **不要反序列化不可信数据**（尤其是 HTTP 请求、MQ 消息）。
- 使用 `ObjectInputFilter`（JDK 9+）白名单：

```java
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.acme.dto.*;java.base/*;!*");
ObjectInputStream ois = new ObjectInputStream(in);
ois.setObjectInputFilter(filter);
```

- 改用 JSON/Protobuf（结构化、无代码执行风险）。
- 网络传输加签名/HMAC。

## 7. 替代方案

| 方案 | 优点 | 缺点 |
|---|---|---|
| JSON（Jackson） | 可读、跨语言、安全 | 体积大、慢 |
| Protobuf | 紧凑、快、强 schema | 需要 .proto、不可读 |
| Avro | schema 演进 | 生态略小 |
| MessagePack | 紧凑 | 可读性差 |
| Java 序列化 | 内置、简单 | 不安全、体积大、不跨语言 |

生产建议：**对外接口用 JSON，内部高性能 RPC 用 Protobuf，禁止 Java 原生序列化跨进程传输**。

## 8. 深拷贝实现

```java
@SuppressWarnings("unchecked")
public static <T extends Serializable> T deepCopy(T obj) {
    try (ByteArrayOutputStream bos = new ByteArrayOutputStream();
         ObjectOutputStream oos = new ObjectOutputStream(bos)) {
        oos.writeObject(obj);
        try (ObjectInputStream ois = new ObjectInputStream(
                new ByteArrayInputStream(bos.toByteArray()))) {
            return (T) ois.readObject();
        }
    } catch (IOException | ClassNotFoundException e) {
        throw new IllegalStateException("deep copy failed", e);
    }
}
```

问题：慢、要求全部可序列化。**推荐**：拷贝构造器、JSON 序列化往返、或明确的手写深拷贝。

## 9. 常见不可序列化对象

| 对象 | 原因 | 处理 |
|---|---|---|
| `Thread` | 平台相关状态 | `transient` |
| `InputStream/OutputStream` | 文件句柄 | `transient` |
| `Connection`/`Socket` | 网络资源 | `transient` |
| `Lambda` | 只有捕获 Serializable 目标才可 | 用具体类或 `transient` |
| 非静态内部类 | 隐式持有外部引用，外部必须可序列化 | 改静态嵌套类 |
| `Optional` | 未实现 Serializable | 用 null 或自定义 DTO |

## 10. 序列化与框架

- **JPA 实体**：不要实现 Serializable 用于传输（用 DTO）。Hibernate 要求实体可序列化仅用于二级缓存/分布式。
- **Redis 存储**：用 JSON 或自定义序列化器（`GenericJackson2JsonRedisSerializer`），不要 JDK 序列化（体积大、不兼容）。
- **Spring Session**：默认 JDK 序列化，建议改 JSON。
- **RPC（Dubbo）**：需实现 Serializable（或使用 hessian2/protobuf）。

```java
// Spring Data Redis 配置 JSON 序列化
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory f) {
    RedisTemplate<String, Object> t = new RedisTemplate<>();
    t.setConnectionFactory(f);
    t.setKeySerializer(new StringRedisSerializer());
    t.setValueSerializer(new GenericJackson2JsonRedisSerializer());
    t.setHashKeySerializer(new StringRedisSerializer());
    t.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
    return t;
}
```

## 11. 常见坑

1. 忘记 `serialVersionUID` → 类一改就反序列化失败。
2. 字段未序列化却为 null（transient 或 static）。
3. 父类未实现 Serializable，期望字段被保存。
4. 集合元素未实现 Serializable → `NotSerializableException`。
5. 非静态内部类序列化连带外部对象。
6. 序列化后修改类结构（增删字段类型）导致不兼容。
7. 用 JDK 序列化存 Redis（升级类后老数据读不出）。
8. 反序列化不可信数据（RCE 风险）。
9. `readObject` 中未做防御性校验（不变量被破坏）。
10. 深拷贝用序列化实现（性能差）。

## 12. 安全 readObject 模板

```java
private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
    in.defaultReadObject();

    // 重新校验不变量
    if (amount == null || amount.signum() < 0) {
        throw new InvalidObjectException("amount must be non-negative");
    }
    if (items == null) items = new ArrayList<>();
}
```

## 13. 速查

| 需求 | 做法 |
|---|---|
| 让类可序列化 | `implements Serializable` + `serialVersionUID` |
| 排除字段 | `transient` |
| 自定义格式 | `writeObject/readObject` |
| 完全控制 | `Externalizable` |
| 单例保护 | `readResolve` |
| 安全过滤 | `ObjectInputFilter` |
| 替代 | JSON / Protobuf |
| 深拷贝 | 拷贝构造器 / JSON 往返 |
