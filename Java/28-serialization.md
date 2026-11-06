# 28 · 序列化：原生 Serializable 与 JSON

## 一、Java 原生序列化

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;    // 显式版本号！
    private String name;
    private transient String password;      // transient：不序列化（敏感/可重建字段）
}

// 序列化
try (var out = new ObjectOutputStream(new FileOutputStream("user.bin"))) {
    out.writeObject(user);
}
// 反序列化
try (var in = new ObjectInputStream(new FileInputStream("user.bin"))) {
    User u = (User) in.readObject();
}
```

**规则与坑**：
- serialVersionUID 不写 → 类一改就 InvalidClassException
- 反序列化构造器不执行（直接分配内存恢复字段）→ 单例被破坏（readResolve 防御）
- 深拷贝副作用（嵌套对象整个图被序列化）
- **安全红线**：Java 反序列化是历史漏洞重灾区（反序列化 RCE 攻击链）——**绝不对不可信数据用 ObjectInputStream**

## 二、JSON：现代首选（Jackson）

### 基本用法

```java
ObjectMapper mapper = new ObjectMapper();      // 全局复用，线程安全！

// 序列化
String json = mapper.writeValueAsString(user);
// {"name":"tom","age":18,"password":null}

// 反序列化（简单类型）
User u = mapper.readValue(json, User.class);

// 泛型集合（TypeReference，17 篇）
List<User> users = mapper.readValue(json, new TypeReference<List<User>>() { });
Map<String, List<User>> nested = mapper.readValue(json,
        new TypeReference<Map<String, List<User>>>() { });

// 树模型（结构未知时）
JsonNode node = mapper.readTree(json);
node.get("data").get("list").get(0).path("name").asText("默认");
// path() 不存在返回 MissingNode 而非 null → 防 NPE
```

### 常用注解

```java
public class UserDTO {
    @JsonProperty("user_name")              // 字段重命名
    private String userName;

    @JsonIgnore                             // 不序列化（密码）
    private String password;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
    private LocalDateTime createdAt;

    @JsonInclude(JsonInclude.Include.NON_NULL)   // null 不输出
    private String nickname;

    @JsonAlias({"id", "userId"})            // 反序列化多个别名
    private Long userId;

    @JsonCreator                            // record 兼容：构造反序列化
}
```

### 全局配置

```java
ObjectMapper mapper = JsonMapper.builder()
        .serializationInclusion(JsonInclude.Include.NON_NULL)
        .addModule(new JavaTimeModule())                    // java.time 支持
        .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)  // 日期输出字符串
        .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES) // 未知字段不报错
        .build();
```

## 三、其他格式速览

| 格式/库 | 场景 |
|---|---|
| Jackson | JSON 事实标准（Spring 默认） |
| Gson | 轻量 JSON（Android 生态多） |
| Fastjson2 | 国内高性能 JSON（注意历史漏洞版本） |
| Protobuf | gRPC/跨语言强类型，二进制紧凑 |
| Hessian/Kryo | Java RPC 内部序列化（Kryo 高性能） |
| Avro/Parquet | 大数据文件格式 |

## 四、序列化与系统设计

```
跨服务传输（HTTP API）     → JSON（人可读、调试友好）
内部高性能 RPC             → Protobuf/Kryo（体积小、快）
消息队列消息体             → JSON（可读可运维）或框架默认
缓存存取                   → JSON（跨版本兼容）
持久化快照                 → 数据库/文件格式，别用 Java 原生
```

### 版本兼容策略（演进的关键）

```java
// 加字段：旧读者忽略新字段 → disable FAIL_ON_UNKNOWN_PROPERTIES
// 删字段：写方停发，读方容忍缺失 → path("x").asText(default)
// 改名：@JsonAlias 双向过渡期
// 原则：JSON 消息向前兼容（加而不删，删前双写）
```

## 五、安全清单

- 不反序列化不可信来源的原生 Java 对象
- Jackson 白名单多态：`@JsonTypeInfo` + activateDefaultTyping 慎用（多态反序列化漏洞）
- 敏感字段 @JsonIgnore + 出参 DTO 化（不序列化实体）
- 反序列化目标类最小暴露（防止 gadget 类在 classpath）

## 常见坑

- **坑 1**：每次请求 new ObjectMapper → 性能损耗；单例复用
- **坑 2**：LocalDateTime 序列化成数组/时间戳 → 注册 JavaTimeModule
- **坑 3**：toMap 场景 json 值 null → Jackson 报错；改收集结构或用 readTree
- **坑 4**：循环引用对象序列化 → StackOverflow；@JsonManagedReference/@BackReference 或 DTO
- **坑 5**：实体直接作为 API 出参 → 懒加载字段/敏感字段泄露；DTO 隔离
- **坑 6**：原生序列化传给不可信端点 → RCE 风险

## Checklist

- [ ] ObjectMapper 单例 + JavaTimeModule + NON_NULL
- [ ] 泛型反序列化用 TypeReference
- [ ] API 出入参 DTO，禁用实体
- [ ] 消息格式演进遵守兼容策略
- [ ] 禁止不可信数据原生反序列化
