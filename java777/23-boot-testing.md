# 23 · 测试：单元测试、MockMvc 与 Testcontainers

## 1. 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<!-- 内含 JUnit5、Spring Test、AssertJ、Hamcrest、Mockito、JSONassert、JsonPath、MockMvc -->
```

Boot 3 起默认 JUnit 5（JUnit 4 需手动加 vintage 引擎）。

## 2. 测试金字塔

```
      / E2E \        少量：全流程、Testcontainers
     / 集成  \       中量：@SpringBootTest + MockMvc
    /  单元   \      大量：纯 JUnit + Mockito，不启容器
```

> 单元测试**不要**动不动就 `@SpringBootTest`（启动一次好几秒），业务类应能 `new` 出来用 mock 测。

## 3. 纯单元测试（快）

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderMapper orderMapper;
    @Mock InventoryClient inventoryClient;
    @InjectMocks OrderService orderService;      // 自动构造器注入 mock

    @Test
    void create_shouldFail_whenStockNotEnough() {
        when(inventoryClient.deduct(anyLong(), anyInt())).thenReturn(false);
        assertThatThrownBy(() -> orderService.create(new OrderDTO(1L, 2)))
                .isInstanceOf(BizException.class)
                .hasMessageContaining("库存不足");
        verify(orderMapper, never()).insert(any());
    }

    @Test
    void create_shouldInsert() {
        when(inventoryClient.deduct(1L, 2)).thenReturn(true);
        orderService.create(new OrderDTO(1L, 2));
        verify(orderMapper).insert(argThat(o -> o.getSkuId().equals(1L)));
    }
}
```

Mockito 常用：`when/thenReturn`、`thenThrow`、`verify(.., times(1))`、`any()`、`argThat`、`doThrow`（void 方法）、`@Spy`、`ArgumentCaptor`。

## 4. @SpringBootTest（集成测试）

```java
@SpringBootTest                                   // 启动完整容器
@ActiveProfiles("test")
class OrderIntegrationTest {

    @Autowired OrderService orderService;
    @MockitoBean OrderMapper mapper;              // Boot 3.4+（旧版用 @MockBean）

    @Test void contextLoads() { assertThat(orderService).isNotNull(); }
}
```

### 三种 webEnvironment

| 值 | 说明 |
|---|---|
| `MOCK`（默认） | 模拟 Servlet 环境，可用 MockMvc |
| `RANDOM_PORT` | 启动真实服务器 + 随机端口，可用 `TestRestTemplate` |
| `DEFINED_PORT` | 指定端口 |
| `NONE` | 无 Web 环境 |

## 5. MockMvc（测 Controller）

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    @Autowired MockMvc mvc;
    @Autowired ObjectMapper om;

    @Test
    void getById() throws Exception {
        mvc.perform(get("/api/users/{id}", 1L)
                .accept(MediaType.APPLICATION_JSON))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.code").value(0))
           .andExpect(jsonPath("$.data.name").value("张三"))
           .andDo(print());                       // 打印请求/响应，调试神器
    }

    @Test
    void create_validation() throws Exception {
        UserDTO dto = new UserDTO();              // name 为空
        mvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(om.writeValueAsString(dto)))
           .andExpect(status().isBadRequest());
    }

    @Test
    void create_unauthorized() throws Exception {
        mvc.perform(post("/api/users").content("{}").contentType(MediaType.APPLICATION_JSON))
           .andExpect(status().isUnauthorized());
    }
}
```

独立测试（只加载 Web 层，更快）：

```java
@WebMvcTest(UserController.class)      // 只加载 Controller + MVC 基础设施
class UserControllerSliceTest {
    @Autowired MockMvc mvc;
    @MockitoBean UserService userService;       // 依赖需 mock
}
```

其他切片测试：
| 注解 | 加载范围 |
|---|---|
| `@WebMvcTest` | Controller、Filter、Advice |
| `@DataJpaTest` | JPA（默认 H2/嵌入式，带事务回滚） |
| `@MybatisTest` | MyBatis |
| `@JsonTest` | Jackson 序列化 |
| `@WebFluxTest` | WebFlux |
| `@RestClientTest` | RestTemplate/WebClient |

## 6. 数据库测试

### H2（快，SQL 兼容性有限）

```xml
<dependency><groupId>com.h2database</groupId><artifactId>h2</artifactId><scope>test</scope></dependency>
```
```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:test;MODE=MySQL;DATABASE_TO_LOWER=TRUE
    driver-class-name: org.h2.Driver
  sql:
    init:
      schema-locations: classpath:sql/schema.sql
      data-locations: classpath:sql/data.sql
```

### Testcontainers（真实数据库，推荐）

```xml
<dependency><groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-testcontainers</artifactId><scope>test</scope></dependency>
<dependency><groupId>org.testcontainers</groupId><artifactId>mysql</artifactId><scope>test</scope></dependency>
<dependency><groupId>org.testcontainers</groupId><artifactId>junit-jupiter</artifactId><scope>test</scope></dependency>
```

```java
@Testcontainers
@SpringBootTest
@TestcontainersConfiguration
class MysqlIT {

    @Container
    @ServiceConnection                            // Boot 3.1+ 自动注入数据源配置
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("test").withUsername("t").withPassword("t");

    @Autowired OrderMapper mapper;

    @Test void insert() { ... }
}
```

也支持 Redis、Kafka、MongoDB、Elasticsearch 等容器。

> 需要本机有 Docker。CI 上可用 `--privileged` 或 Docker-in-Docker。

## 7. 事务回滚与数据隔离

```java
@SpringBootTest
@Transactional                 // 测试结束自动回滚
class DaoTest { ... }

@Sql("/sql/order.sql")         // 执行脚本
@Sql(scripts = "/clean.sql", executionPhase = AFTER_TEST_METHOD)
```

> `@DataJpaTest` 默认带 `@Transactional` 回滚；用 `@Commit` 可强制提交。

## 8. 测试配置技巧

```java
@TestConfiguration          // 测试专用配置（不会被主扫描扫到）
static class TestConfig {
    @Bean Clock fixedClock() { return Clock.fixed(Instant.parse("2024-01-01T00:00:00Z"), ZoneOffset.UTC); }
}

@SpringBootTest(properties = {"app.feature.x=true", "spring.datasource.url=jdbc:h2:mem:t"})
@MockitoBean(answer = Answers.RETURNS_DEEP_STUBS) RemoteClient client;

@DynamicPropertySource              // 动态属性（常用于 Testcontainers 端口）
static void props(DynamicPropertyRegistry r) {
    r.add("redis.host", () => redis.getHost());
}
```

## 9. 测试异步与定时

```java
@SpringBootTest
@EnableScheduling
class AsyncTest {
    @Autowired AsyncService s;
    @Test void async() {
        CompletableFuture<String> f = s.doAsync();
        assertThat(f.get(3, TimeUnit.SECONDS)).isEqualTo("done");   // 必须等待
    }
}
// 或用 Awaitility
await().atMost(Duration.ofSeconds(5)).until(() -> repo.count() == 1);
```

## 10. 覆盖率与质量

```xml
<plugin><groupId>org.jacoco</groupId><artifactId>jacoco-maven-plugin</artifactId>
  <executions>
    <execution><goals><goal>prepare-agent</goal></goals></execution>
    <execution><id>report</id><phase>test</phase><goals><goal>report</goal></goals></execution>
  </executions>
</plugin>
```

- 目标：核心业务分支覆盖率 70%+，不追求 100%。
- 测**行为**不测实现：断言输出与副作用，而非内部调用次数（避免过度验证）。

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 测试启动慢 | 每个类都重启容器 | 同配置会复用上下文；避免滥用 `@MockitoBean`（会使上下文失效重新创建） |
| `@MockBean` 导致上下文缓存失效 | 慢 | 统一放抽象基类或改用 `@TestConfiguration` |
| 测试间数据污染 | 用例互相影响 | `@Transactional` 回滚 / `@Sql` 清理 / 每次重建容器数据 |
| 异步测试偶发失败 | 未等待完成 | `get(timeout)` / `Awaitility` |
| H2 与 MySQL 行为不一致 | 本地过 CI 挂 | 关键 SQL 用 Testcontainers |
| MockMvc 404 | 路径不对 | 注意 `context-path`；`mockMvc.perform(get("/api/..."))` |
| 时间依赖测试不稳定 | 偶发失败 | 注入 `Clock` 并 mock |

## 12. 面试速答

- **@SpringBootTest 与 @WebMvcTest**：前者启动完整容器，后者只加载 Web 层（更快，依赖需 mock）。
- **@MockBean 与 @Mock**：前者替换容器中的 Bean（会影响上下文缓存），后者是纯 Mockito 本地 mock。
- **Testcontainers 价值**：用真实中间件（MySQL/Redis/Kafka）做集成测试，避免 H2 兼容性差异。
- **为什么测试要快**：反馈周期决定测试是否被使用；用切片测试 + 复用 Spring 上下文。
