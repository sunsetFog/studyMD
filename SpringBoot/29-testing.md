# 29 · 测试体系

## 一、测试金字塔

```
        /\   集成测试（少量，关键链路 E2E）
       /--\  切片测试（Controller/数据层）
      /----\ 单元测试（大量：Service 逻辑）
```

依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>   <!-- JUnit5 + Mockito + AssertJ + Spring Test -->
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
```

## 二、单元测试（Service 层：Mockito）

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock OrderMapper orderMapper;
    @Mock StockMapper stockMapper;
    @InjectMocks OrderService orderService;

    @Test
    void createOrder_shouldThrow_whenStockInsufficient() {
        when(stockMapper.deduct(1L, 10)).thenReturn(0);

        OrderCreateDTO dto = new OrderCreateDTO(1L, 1L, 10);

        assertThatThrownBy(() -> orderService.createOrder(dto))
                .isInstanceOf(BusinessException.class)
                .hasMessageContaining("库存不足");

        verify(orderMapper, never()).insert(any());     // 验证未落单
    }

    @Test
    void createOrder_success() {
        when(stockMapper.deduct(1L, 10)).thenReturn(1);
        when(orderMapper.insert(any())).thenReturn(1);

        orderService.createOrder(new OrderCreateDTO(1L, 1L, 10));

        verify(orderMapper).insert(argThat(o -> o.getStatus().equals("CREATED")));
    }
}
```

核心 API：`when(...).thenReturn/thenThrow/thenAnswer`、`verify(mock, times(1)/never())`、`argThat` 自定义断言、`@Captor` 捕获参数。

## 三、Web 层切片测试（@WebMvcTest）

```java
@WebMvcTest(UserController.class)
@Import(SecurityTestConfig.class)     // 跳过真实安全链
class UserControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean UserService userService;   // Boot 3.4+ 改用 @MockitoBean

    @Test
    void getUser_shouldReturn200() throws Exception {
        when(userService.getById(1L)).thenReturn(new UserDTO(1L, "tom"));

        mockMvc.perform(get("/api/v1/users/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(0))
                .andExpect(jsonPath("$.data.username").value("tom"));
    }

    @Test
    void createUser_invalidBody_400() throws Exception {
        mockMvc.perform(post("/api/v1/users")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"username\":\"\"}"))
                .andExpect(status().isOk())                     // 统一 200 + code=400
                .andExpect(jsonPath("$.code").value(400));
    }
}
```

## 四、集成测试（Testcontainers：真实数据库）

```java
@SpringBootTest
@Testcontainers
class OrderIntegrationTest {

    @Container
    @ServiceConnection                                   // 自动替换数据源配置
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

    @Container
    @ServiceConnection
    static GenericContainer<?> redis = new GenericContainer<>("redis:7").withExposedPorts(6379);

    @Autowired OrderService orderService;

    @Test
    @Transactional    // 测试结束自动回滚
    void fullOrderFlow() {
        orderService.createOrder(dto);
        assertThat(orderMapper.selectByUserId(1L)).isNotEmpty();
    }
}
```

> Testcontainers 优于 H2：真实 MySQL 行为（方言/锁/事务），CI 里 Docker 可用即跑。

## 五、测试原则

| 原则 | 说明 |
|---|---|
| AAA 结构 | Arrange 准备 → Act 执行 → Assert 断言 |
| 一个测试一个关注点 | 失败即定位 |
| 不测私有方法 | 测行为不测实现 |
| Mock 边界 | 只 mock 外部依赖（DB/MQ/三方），不 mock 被测类内部 |
| 数据自建自清理 | @Transactional 回滚或 Testcontainers 新实例 |
| 确定性 | 无时间依赖（注入 Clock）、无随机（固定种子） |

## 六、时间与静态依赖

```java
// Service 注入 Clock
private final Clock clock = Clock.systemDefaultZone();
LocalDateTime.now(clock);

// 测试中
when(...) ; service.clock = Clock.fixed(Instant.parse("2026-01-01T00:00:00Z"), ZoneId.of("UTC"));
```

## 七、CI 中的测试

```yaml
# Maven
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <configuration><rules>...80% 覆盖率门槛...</rules></configuration>
</plugin>
# mvn verify  → 失败即阻断合并
```

## 常见坑

- **坑 1**：@SpringBootTest 全量启动太慢 → 单测用 @ExtendWith(Mockito)，集成测试才起容器
- **坑 2**：@MockBean 在共享 Context 造成污染 → 3.4+ 用 @MockitoBean，且按配置类分组
- **坑 3**：测试依赖真实数据库/环境 → Testcontainers/H2 隔离
- **坑 4**：@Transactional 测试后数据残留认知错误 → 测试类的事务只回滚测试内操作，MQ/异步副作用不会回滚
- **坑 5**：断言只验证不抛异常 → 必须验证状态与副作用

## Checklist

- [ ] 核心业务 Service 有单元测试（正常 + 边界 + 异常三分支）
- [ ] Controller 切片测试覆盖校验与鉴权
- [ ] 关键链路有 Testcontainers 集成测试
- [ ] JaCoCo 覆盖率门槛接入 CI
