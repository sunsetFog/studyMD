# 50 · 微服务设计模式

## 1. 按层次分类

| 类别 | 模式 |
|---|---|
| 拆分 | 按业务能力拆分、按子域（DDD）拆分、绞杀者模式（Strangler） |
| 通信 | API 网关、BFF、客户端发现/服务端发现、远程调用、消息队列 |
| 数据 | Database per Service、Saga、CQRS、事件溯源、共享数据库（反模式） |
| 可靠性 | 熔断、隔离（舱壁）、限流、重试、超时、降级、健康检查 |
| 可观测 | 日志聚合、分布式追踪、指标监控、审计日志 |
| 安全 | 访问令牌（JWT/OAuth2）、API 网关统一鉴权 |
| 部署 | 单服务单主机、服务实例弹性、蓝绿部署、金丝雀发布、Sidecar |

## 2. API 网关模式

统一入口，处理：路由、鉴权、限流、日志、跨域、协议转换、缓存。

> 反模式：把业务逻辑写进网关 → 网关变成新的单体。

## 3. BFF（Backend for Frontend）

```
Web App  → BFF-Web   → 各微服务
Mobile   → BFF-Mobile → 各微服务
```

- 为每种客户端定制聚合接口，减少前端多次请求。
- 好处：前端体验好、各端独立演进。
- 坏处：多一层维护成本；不要做成「万能聚合层」。

```java
@RestController
public class OrderBffController {
    // 一次返回订单 + 商品 + 用户，避免前端三次请求
    @GetMapping("/bff/order/{id}")
    public OrderDetailVO detail(@PathVariable Long id) {
        OrderVO o = orderClient.get(id);
        List<ItemVO> items = itemClient.listByOrder(id);
        UserVO u = userClient.get(o.getUserId());
        return new OrderDetailVO(o, items, u);   // 并行调用用 CompletableFuture
    }
}
```

## 4. Database per Service

每个服务**独立数据库/独立 schema**，其它服务只能通过 API 访问。

- 好处：解耦、独立演进、独立伸缩。
- 代价：无法 JOIN、跨服务查询难、分布式事务。

## 5. CQRS（命令查询职责分离）

```
写侧（Command）→ 写库（规范化）→ 事件 → 读侧（Query）→ 读库/物化视图（反规范化）
```

- 写模型和读模型分离，各自优化。
- 常配 **事件驱动**同步读模型。
- 适用：读写比极高、查询复杂（如报表、搜索）。
- 代价：最终一致、复杂度上升。

```java
// Command
@Transactional public void createOrder(CreateOrderCommand c) { ... 写主库，发事件 }
// Query
public OrderDetailVO query(Long id) { return readRepository.findDetail(id); }  // 读宽表/ES
```

## 6. 事件溯源（Event Sourcing）

不存当前状态，只存**事件序列**，状态由事件重放（replay）得到。

```
Event Store: [OrderCreated, ItemAdded, Paid, Shipped]
Aggregate 状态 = fold(events)
```

- 好处：完整审计轨迹、可回溯、天然与 CQRS 搭配。
- 代价：学习成本、查询需投影、事件版本演进。
- 适用：金融、审计要求高的领域。

## 7. Saga（长事务）

把一个跨服务事务拆成一串本地事务 + 补偿操作。

| 编排方式 | 说明 |
|---|---|
| **编排（Orchestration）** | 中心协调器按顺序调用各服务，失败则调用补偿（Seata Saga、Camunda） |
| **协同（Choreography）** | 各服务通过事件自主推进（无中心，耦合低但难追踪） |

```
创建订单 → 扣库存 → 扣余额 → 发券
                ↓失败
          退库存 ← 退余额 ← 收券
```

> 补偿不是「回滚」，是**业务上的反向操作**（如：退款而非删除流水）。

## 8. 事件驱动架构（EDA）

```
Producer → Event Bus(Kafka/RMQ) → Consumer1 / Consumer2 / Consumer3
```

- 松耦合、可扩展、易加新消费者。
- 模式：**发布/订阅**、**事件通知**、**事件携带状态转移**、**事件溯源**。
- 注意：事件要**幂等消费**、要有**版本演进**策略、要有**schema 规范**（Avro/Protobuf/JSON Schema）。

## 9. 服务发现与调用模式

| 模式 | 说明 |
|---|---|
| 客户端发现 | 客户端查注册中心后直连（Spring Cloud） |
| 服务端发现 | 通过负载均衡器/Service 转发（K8s、Mesh） |
| 注册表 | 自注册 / 第三方注册 |
| 自注册 vs 外部注册 | Spring Cloud 自注册；K8s 由平台注册 |

## 10. 部署与发布模式

| 模式 | 说明 |
|---|---|
| 蓝绿部署 | 两套环境切换，回滚快，资源翻倍 |
| 金丝雀发布 | 小比例流量验证，逐步放量 |
| 滚动更新 | 逐批替换（K8s 默认） |
| 影子流量 | 复制生产流量到新版本验证（不影响用户） |
| 特性开关 | 代码里加开关，随时开关功能 |
| 暗发布 | 功能已上线但不可见 |

## 11. 反模式（要避免）

| 反模式 | 说明 |
|---|---|
| 分布式单体 | 拆了服务但必须一起发布 |
| 共享数据库 | 服务间通过 DB 耦合 |
| 微型单体 | 服务拆得过细，一次请求跨 10 个服务 |
| 无熔断/超时 | 雪崩 |
| 智能管道哑端点 | 把业务塞进中间件 |
| 无版本管理 | 接口改动导致上游全挂 |
| 无幂等 | 重试导致重复数据 |

## 12. 面试速答

- **BFF 是什么**：为不同客户端提供定制后端聚合层，减少前端请求次数。
- **CQRS**：读写模型分离，写侧负责业务规则，读侧为查询优化（宽表/ES），通过事件同步，最终一致。
- **Saga vs 2PC**：Saga 由一串本地事务 + 补偿组成，适合长流程、高并发；2PC 强一致但锁资源、性能差。
- **事件驱动的价值**：解耦、削峰、易扩展；代价是一致性与可观测性挑战。
- **绞杀者模式**：在单体外围逐步建立新服务，用网关把流量逐步迁移，最终淘汰单体。
