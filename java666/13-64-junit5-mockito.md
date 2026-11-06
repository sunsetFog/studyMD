# 64 · 单元测试（JUnit 5 + Mockito）

## 1. JUnit 5 体系

```
JUnit Platform（运行引擎）
├── Jupiter（JUnit 5 API）
└── Vintage（兼容 JUnit 4）
```

依赖：

```xml
<dependency>
  <groupId>org.junit.jupiter</groupId>
  <artifactId>junit-jupiter</artifactId>
  <version>5.10.2</version>
  <scope>test</scope>
</dependency>
```

## 2. 基础注解

```java
class CalculatorTest {
    @BeforeAll  static void beforeAll() { }     // 类级，一次
    @AfterAll   static void afterAll() { }
    @BeforeEach void setUp() { }                // 每个测试前
    @AfterEach  void tearDown() { }

    @Test
    void should_add_two_numbers() {
        assertEquals(3, Calculator.add(1, 2));
    }

    @Disabled("暂不执行")
    @Test
    void skipped() { }

    @DisplayName("用户名不能为空")
    @Test
    void blank_username_should_fail() { }
}
```

## 3. 断言

```java
import static org.junit.jupiter.api.Assertions.*;

assertEquals(expected, actual);
assertEquals(expected, actual, "失败信息");
assertNotEquals(a, b);
assertTrue(cond);
assertFalse(cond);
assertNull(obj);
assertNotNull(obj);
assertArrayEquals(expectedArr, actualArr);
assertIterableEquals(expectedList, actualList);

// 异常
assertThrows(IllegalArgumentException.class, () -> service.create(null));
var ex = assertThrows(BizException.class, () -> service.pay(order));
assertEquals("余额不足", ex.getMessage());

// 超时
assertTimeout(Duration.ofSeconds(1), () -> service.process());

// 组合
assertAll("user",
    () -> assertEquals("Tom", u.getName()),
    () -> assertEquals(18, u.getAge()));

// 自定义失败信息（Supplier，惰性）
assertEquals(3, result, () -> "计算错误: " + result);
```

> 优先 `AssertJ`（流式、可读性强）：

```java
assertThat(user.getName()).isEqualTo("Tom");
assertThat(list).hasSize(3).contains("a").doesNotContain("z");
assertThatThrownBy(() -> service.pay(null))
    .isInstanceOf(BizException.class)
    .hasMessage("订单不存在");
```

## 4. 参数化测试

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3})
void positive(int n) { assertTrue(n > 0); }

@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "-1, 1, 0"
})
void add(int a, int b, int expected) {
    assertEquals(expected, Calculator.add(a, b));
}

@ParameterizedTest
@MethodSource("provideUsers")
void valid_users(User u) { assertTrue(validator.isValid(u)); }

static Stream<User> provideUsers() {
    return Stream.of(new User("Tom", 18), new User("Jerry", 20));
}

@ParameterizedTest
@EnumSource(Status.class)
void status_has_label(Status s) { assertNotNull(s.getLabel()); }

@ParameterizedTest
@CsvFileSource(resources = "/test-data.csv", numLinesToSkip = 1)
void from_csv(String name, int age) { }
```

## 5. 嵌套与生命周期

```java
class OrderServiceTest {
    OrderService service;

    @BeforeEach void init() { service = new OrderService(); }

    @Nested
    class WhenEmpty {
        @Test void should_throw() {
            assertThrows(IllegalStateException.class, () -> service.checkout());
        }
    }

    @Nested
    class WhenHasItems {
        @BeforeEach void addItem() { service.add(new Item("a", 1)); }
        @Test void should_pass() { assertDoesNotThrow(() -> service.checkout()); }
    }
}
```

## 6. Mockito

### 依赖

```xml
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-core</artifactId>
  <version>5.11.0</version>
  <scope>test</scope>
</dependency>
<dependency>
  <groupId>org.mockito</groupId>
  <artifactId>mockito-junit-jupiter</artifactId>
  <version>5.11.0</version>
  <scope>test</scope>
</dependency>
```

### 基本用法

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock UserRepository userRepo;
    @Mock PaymentGateway gateway;
    @InjectMocks OrderService service;      // 自动注入 mock

    @Test
    void create_order_success() {
        // given
        when(userRepo.findById(1L)).thenReturn(Optional.of(new User(1L, "Tom")));
        when(gateway.pay(any())).thenReturn(PaymentResult.success());

        // when
        Order order = service.create(1L, List.of(new Item("sku", 2)));

        // then
        assertNotNull(order);
        verify(userRepo).findById(1L);
        verify(gateway, times(1)).pay(any());
        verifyNoMoreInteractions(gateway);
    }
}
```

### 打桩（Stubbing）

```java
when(repo.findById(1L)).thenReturn(Optional.of(user));
when(repo.findById(anyLong())).thenThrow(new NotFoundException());
when(repo.save(any())).thenAnswer(inv -> {
    User u = inv.getArgument(0);
    u.setId(100L);
    return u;
});

// 连续返回
when(repo.next()).thenReturn(1).thenReturn(2).thenThrow(new RuntimeException());

// void 方法
doNothing().when(mailer).send(any());
doThrow(new MailException()).when(mailer).send(any());
doAnswer(inv -> null).when(mailer).send(any());

// 参数匹配器
when(repo.find(anyString(), eq(1), anyInt())).thenReturn(list);
// 注意：一旦使用 matcher，所有参数都必须用 matcher
```

### 验证

```java
verify(repo).save(user);                          // 恰好一次
verify(repo, times(2)).save(any());
verify(repo, never()).delete(any());
verify(repo, atLeastOnce()).find(any());
verify(repo, atMost(3)).find(any());
verify(repo, timeout(1000)).save(any());

InOrder inOrder = inOrder(repo, gateway);
inOrder.verify(repo).save(any());
inOrder.verify(gateway).pay(any());

verifyNoInteractions(unusedService);
verifyNoMoreInteractions(repo);
```

### Spy（部分 mock）

```java
List<String> list = spy(new ArrayList<>());
list.add("a");
verify(list).add("a");
doReturn(100).when(list).size();     // 打桩真实对象的方法
```

### 静态方法 / 构造器（Mockito 3.4+ inline）

```java
try (MockedStatic<UUID> mocked = mockStatic(UUID.class)) {
    mocked.when(UUID::randomUUID).thenReturn(fixedUuid);
    // 测试
}
```

## 7. 测试设计原则

### Given-When-Then

```java
@Test
void should_reject_when_balance_insufficient() {
    // given
    Account acc = new Account(new BigDecimal("10"));
    // when
    var ex = assertThrows(BizException.class, () -> acc.withdraw(new BigDecimal("20")));
    // then
    assertEquals("余额不足", ex.getMessage());
    assertEquals(new BigDecimal("10"), acc.getBalance());
}
```

### FIRST 原则

| 字母 | 含义 |
|---|---|
| Fast | 快（毫秒级） |
| Independent | 独立（无顺序依赖） |
| Repeatable | 可重复（不依赖时间/网络） |
| Self-validating | 自动断言 |
| Timely | 及时编写 |

### 测试金字塔

```
       /\
      /E2E\        少（关键流程）
     /─────\
    / 集成  \      中（DB、HTTP）
   /─────────\
  /  单元测试  \    多（快、稳定）
 /─────────────\
```

## 8. 测试替身的区别

| 类型 | 说明 |
|---|---|
| Dummy | 仅填充参数，不使用 |
| Stub | 返回预设值 |
| Spy | 记录调用，可部分真实 |
| Mock | 预设期望并验证交互 |
| Fake | 简化实现（内存数据库） |

## 9. 时间与随机数

```java
// 注入 Clock（推荐）
public class OrderService {
    private final Clock clock;
    public OrderService(Clock clock) { this.clock = clock; }

    public Order create() {
        return new Order(LocalDateTime.now(clock));
    }
}

// 测试
Clock fixed = Clock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneOffset.UTC);
new OrderService(fixed).create();
```

随机数同理：注入 `Random` 或 `Supplier`。

## 10. 集成测试

```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderControllerTest {

    @Autowired MockMvc mvc;

    @Test
    void create_order() throws Exception {
        mvc.perform(post("/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"userId\":1}"))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.code").value(0));
    }
}
```

Testcontainers（真实 DB/Redis）：

```java
@Testcontainers
class UserRepositoryTest {
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", mysql::getJdbcUrl);
    }
}
```

## 11. 常见坑

1. 测试依赖执行顺序（应相互独立）。
2. 断言太弱（只 `assertNotNull`）。
3. 一个测试验证太多东西（失败难定位）。
4. Mock 过度（测试实现细节，重构即失败）。
5. `when` 用在 `void` 方法上（应 `doThrow`）。
6. 参数匹配器混用（全部都要用 matcher）。
7. 测试中 `Thread.sleep`（不稳定，应用 Awaitility）。
8. 依赖真实时间/时区（用 `Clock`）。
9. 测试共享可变状态（静态字段）。
10. 忽略测试的执行时间（单测应在毫秒级）。
11. 未覆盖异常分支。
12. 只测 happy path。

## 12. 覆盖率

- JaCoCo 生成报告：

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.12</version>
  <executions>
    <execution><goals><goal>prepare-agent</goal></goals></execution>
    <execution><id>report</id><phase>test</phase><goals><goal>report</goal></goals></execution>
  </executions>
</plugin>
```

- 目标：核心业务 80%+ 行覆盖，不必追求 100%（getter/setter 无意义）。
- 覆盖率 ≠ 质量（断言为空的测试也能覆盖）。

## 13. 速查

| 需求 | API |
|---|---|
| 断言相等 | `assertEquals` |
| 断言异常 | `assertThrows` |
| 分组断言 | `assertAll` |
| 参数化 | `@ParameterizedTest` + `@CsvSource` |
| Mock | `@Mock` + `when().thenReturn()` |
| 验证 | `verify(x, times(n))` |
| 部分 mock | `spy()` |
| 静态 mock | `mockStatic()` |
| Spring 集成 | `@SpringBootTest` + `MockMvc` |
| 真实 DB | Testcontainers |
| 覆盖率 | JaCoCo |
