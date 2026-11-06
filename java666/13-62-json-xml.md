# 62 · JSON 与 XML 处理

## 1. JSON 库对比

| 库 | 特点 | 场景 |
|---|---|---|
| **Jackson** | 功能全、Spring 默认、流式 + 树模型 | 通用首选 |
| **Gson** | 简单、Google 出品 | Android/轻量 |
| **Fastjson2** | 极快、国内常用 | 高性能，注意历史安全漏洞 |
| **JSON-P / JSON-B** | Jakarta 标准 | Java EE 环境 |

## 2. Jackson 核心 API

### ObjectMapper

```java
ObjectMapper mapper = new ObjectMapper();

// 序列化
String json = mapper.writeValueAsString(user);
byte[] bytes = mapper.writeValueAsBytes(user);
mapper.writeValue(new File("user.json"), user);

// 反序列化
User u = mapper.readValue(json, User.class);
List<User> list = mapper.readValue(json, new TypeReference<List<User>>() { });
Map<String, Object> map = mapper.readValue(json, Map.class);

// 树模型
JsonNode node = mapper.readTree(json);
node.get("name").asText();
node.get("age").asInt();
node.path("address").path("city").asText("unknown");
node.isArray(); node.size();

// 流式（大文件）
JsonParser parser = mapper.createParser(inputStream);
while (parser.nextToken() != null) { ... }
```

> `ObjectMapper` **线程安全**（配置完成后），应作为单例复用。

### 常用配置

```java
ObjectMapper mapper = JsonMapper.builder()
        .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)  // 忽略未知字段
        .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false)     // 日期用 ISO
        .configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false)
        .serializationInclusion(JsonInclude.Include.NON_NULL)                 // 忽略 null
        .addModule(new JavaTimeModule())                                      // java.time 支持
        .addModule(new ParameterNamesModule())                                // record/构造器
        .build();
```

## 3. 注解

```java
public class User {
    @JsonProperty("user_id")            // 重命名
    private Long id;

    @JsonIgnore                          // 忽略
    private String password;

    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "Asia/Shanghai")
    private LocalDateTime createdAt;

    @JsonInclude(JsonInclude.Include.NON_NULL)
    private String nickname;

    @JsonProperty(access = JsonProperty.Access.WRITE_ONLY)  // 只反序列化
    private String secret;
}
```

### 枚举

```java
public enum Status {
    @JsonProperty("created") CREATED,
    @JsonProperty("paid") PAID;

    @JsonValue                        // 序列化为该值
    public String code() { return name().toLowerCase(); }

    @JsonCreator                      // 反序列化
    public static Status from(String code) {
        return valueOf(code.toUpperCase());
    }
}
```

### 多态

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = Alipay.class, name = "alipay"),
    @JsonSubTypes.Type(value = Wechat.class, name = "wechat")
})
public sealed interface Payment permits Alipay, Wechat { }
```

## 4. record 与不可变对象

```java
public record UserDto(Long id, String name, LocalDate birthday) { }

ObjectMapper mapper = new ObjectMapper()
        .registerModule(new ParameterNamesModule())
        .registerModule(new JavaTimeModule());

UserDto dto = mapper.readValue(json, UserDto.class);   // Jackson 2.12+ 支持 record
```

## 5. 泛型与 TypeReference

```java
// 泛型擦除 → 必须用 TypeReference 保留类型
List<User> users = mapper.readValue(json, new TypeReference<List<User>>() { });
Map<String, List<User>> map = mapper.readValue(json,
        new TypeReference<Map<String, List<User>>>() { });

// JavaType（避免匿名类开销）
JavaType type = mapper.getTypeFactory().constructCollectionType(List.class, User.class);
List<User> list = mapper.readValue(json, type);
```

## 6. Gson

```java
Gson gson = new GsonBuilder()
        .setPrettyPrinting()
        .serializeNulls()
        .setDateFormat("yyyy-MM-dd HH:mm:ss")
        .create();

String json = gson.toJson(user);
User u = gson.fromJson(json, User.class);

Type type = new TypeToken<List<User>>(){}.getType();
List<User> list = gson.fromJson(json, type);

// 注解
@SerializedName("user_id") private Long id;
@Expose private String name;     // 需 excludeFieldsWithoutExposeAnnotation
```

限制：不支持 record 的规范构造器（需 Gson 2.10+）、无内置 java.time 支持（需适配器）。

## 7. JSON 安全与校验

```java
// 1. 限制深度/大小（防 DoS）
mapper.configure(StreamReadConstraints.builder()
        .maxNestingDepth(100)
        .maxStringLength(1_000_000)
        .build());

// 2. 忽略未知字段（避免前端加字段导致 500）
FAIL_ON_UNKNOWN_PROPERTIES = false;

// 3. 校验必填
if (dto.name() == null) throw new BizException("name required");
// 或 @NotBlank + @Valid

// 4. 防止多态反序列化漏洞（Jackson 的 enableDefaultTyping 已废弃，禁用）
```

## 8. XML 处理

### DOM（全量加载，可修改）

```java
DocumentBuilderFactory f = DocumentBuilderFactory.newInstance();
f.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);  // 防 XXE
DocumentBuilder b = f.newDocumentBuilder();
Document doc = b.parse(new File("a.xml"));

Element root = doc.getDocumentElement();
NodeList items = doc.getElementsByTagName("item");
for (int i = 0; i < items.getLength(); i++) {
    Element e = (Element) items.item(i);
    String id = e.getAttribute("id");
    String text = e.getTextContent();
}
```

### SAX（流式，只读，内存小）

```java
SAXParserFactory f = SAXParserFactory.newInstance();
f.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
SAXParser parser = f.newSAXParser();

parser.parse(new File("a.xml"), new DefaultHandler() {
    @Override public void startElement(String uri, String local, String qName, Attributes attrs) {
        if ("item".equals(qName)) System.out.println(attrs.getValue("id"));
    }
});
```

### StAX（拉模式流式）

```java
XMLInputFactory f = XMLInputFactory.newInstance();
f.setProperty(XMLInputFactory.SUPPORT_DTD, false);
XMLStreamReader r = f.createXMLStreamReader(new FileInputStream("a.xml"));

while (r.hasNext()) {
    if (r.next() == XMLStreamConstants.START_ELEMENT && "item".equals(r.getLocalName())) {
        System.out.println(r.getAttributeValue(null, "id"));
    }
}
```

### 对比

| 方式 | 内存 | 可修改 | 速度 | 适用 |
|---|---|---|---|---|
| DOM | 高 | ✔ | 慢 | 小文件、需修改 |
| SAX | 低 | ✘ | 快 | 大文件只读 |
| StAX | 低 | ✘ | 快 | 大文件、拉式 |

## 9. JAXB（对象 ↔ XML）

```java
@XmlRootElement
@XmlAccessorType(XmlAccessType.FIELD)
public class User {
    @XmlAttribute private Long id;
    @XmlElement private String name;
    private List<Item> items;
}

// 序列化
JAXBContext ctx = JAXBContext.newInstance(User.class);
Marshaller m = ctx.createMarshaller();
m.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);
m.marshal(user, System.out);

// 反序列化
Unmarshaller um = ctx.createUnmarshaller();
User u = (User) um.unmarshal(new File("user.xml"));
```

> JDK 11 起 JAXB 移除，需引入 `jakarta.xml.bind-api` + 实现（如 `jaxb-runtime`）。

## 10. 配置文件格式

| 格式 | 优点 | 缺点 |
|---|---|---|
| Properties | 简单 | 只支持键值、无层次 |
| YAML | 可读、层次清晰 | 缩进敏感、规范复杂 |
| JSON | 通用 | 无注释 |
| TOML | 清晰、支持类型 | 生态较少 |
| XML | 强 schema | 冗长 |

```java
// Properties
Properties p = new Properties();
try (InputStream in = Files.newInputStream(Path.of("app.properties"))) {
    p.load(in);                       // 按 ISO-8859-1 解码
}
// JDK 9+ 支持 UTF-8
p.load(new InputStreamReader(in, StandardCharsets.UTF_8));

// SnakeYAML
Yaml yaml = new Yaml();
Map<String, Object> cfg = yaml.load(inputStream);
```

## 11. 常见坑

1. `ObjectMapper` 每次 new（性能差，应全局单例）。
2. 反序列化泛型 `List<User>` 用 `Class` 参数（得到 `List<LinkedHashMap>`）。
3. 日期未注册 `JavaTimeModule`（抛 `InvalidDefinitionException`）。
4. record 反序列化失败（缺 `ParameterNamesModule` 或未加 `-parameters` 编译参数）。
5. 忽略未知字段未配置（上游加字段导致 500）。
6. 循环引用（A→B→A）序列化栈溢出（用 `@JsonIgnore` 或 DTO）。
7. BigDecimal 序列化为科学计数法（配置 `WRITE_BIGDECIMAL_AS_PLAIN`）。
8. Long 超出 JS 精度（前端丢失精度，用 String 序列化）。
9. XML 未禁用 DTD（XXE 漏洞）。
10. 大 JSON 用 `readValue` 全量加载（应用流式）。
11. 未限制嵌套深度（DoS）。
12. Fastjson 老版本反序列化漏洞（升级 fastjson2 或换 Jackson）。

```java
// Long → String（防前端精度丢失）
@JsonSerialize(using = ToStringSerializer.class)
private Long id;
```

## 12. 速查

| 需求 | 做法 |
|---|---|
| 对象 → JSON | `mapper.writeValueAsString(obj)` |
| JSON → 对象 | `mapper.readValue(json, X.class)` |
| 泛型集合 | `new TypeReference<List<X>>(){}` |
| 树模型 | `mapper.readTree(json)` |
| 忽略未知字段 | `FAIL_ON_UNKNOWN_PROPERTIES=false` |
| 日期 | `JavaTimeModule` + `@JsonFormat` |
| record | Jackson 2.12+ + `ParameterNamesModule` |
| XML 解析 | DOM / SAX / StAX |
| 对象 ↔ XML | JAXB |
| YAML | SnakeYAML |
| 防 XXE | `disallow-doctype-decl=true` |
