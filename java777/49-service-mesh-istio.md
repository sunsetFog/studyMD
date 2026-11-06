# 49 · 服务网格与 Istio

## 1. 什么是 Service Mesh

**基础设施层**，用于处理服务间通信，以 **Sidecar 代理**形式部署，对应用透明。

```
┌─────────────┐      ┌─────────────┐
│  App        │      │  App        │
│  ┌───────┐  │      │  ┌───────┐  │
│  │ Envoy │◄─┼─────►│  │ Envoy │  │   ← 数据平面（Data Plane）
│  └───────┘  │      │  └───────┘  │
└─────────────┘      └─────────────┘
        ▲                   ▲
        └───── Istiod ──────┘            ← 控制平面（Control Plane）
```

| 平面 | 组件 | 职责 |
|---|---|---|
| 数据平面 | Envoy Sidecar | 实际流量转发、熔断、mTLS、遥测 |
| 控制平面 | Istiod（Pilot/Citadel/Galley） | 下发配置、证书管理、服务发现 |

## 2. 解决什么问题

| 能力 | 传统（Spring Cloud） | Mesh |
|---|---|---|
| 服务发现 | 各语言各自实现 | Sidecar 统一 |
| 熔断限流 | 库（Resilience4j） | Sidecar 配置 |
| 重试/超时 | 代码/配置 | 声明式配置 |
| 链路追踪 | SDK 埋点 | Sidecar 自动 |
| mTLS | 需集成 | 自动 |
| 多语言 | 需各语言 SDK | **语言无关** |

核心卖点：**把治理能力从应用代码下沉到基础设施**，业务代码零侵入、多语言统一。

## 3. Istio 核心资源

| CRD | 作用 |
|---|---|
| `Gateway` | 入口/出口网关（L4-L6） |
| `VirtualService` | 路由规则（类似 Gateway 的 Route） |
| `DestinationRule` | 目标策略（负载均衡、连接池、熔断、子集） |
| `ServiceEntry` | 注册外部服务到网格 |
| `Sidecar` | 限制 Sidecar 配置范围 |
| `PeerAuthentication` | mTLS 策略 |
| `AuthorizationPolicy` | 授权（RBAC） |
| `EnvoyFilter` | 直接改 Envoy 配置（高级） |
| `Telemetry` | 遥测配置 |

## 4. 示例配置

### 路由与分流（金丝雀）

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: {name: order}
spec:
  hosts: [order-service]
  http:
    - match:
        - headers:
            x-canary: {exact: "true"}
      route:
        - destination: {host: order-service, subset: v2}
    - route:
        - destination: {host: order-service, subset: v1}
          weight: 90
        - destination: {host: order-service, subset: v2}
          weight: 10
      timeout: 3s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: 5xx,reset,connect-failure
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: {name: order}
spec:
  host: order-service
  trafficPolicy:
    connectionPool:
      tcp: {maxConnections: 100}
      http: {http1MaxPendingRequests: 100, maxRequestsPerConnection: 10}
    outlierDetection:            # 熔断/异常驱逐
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels: {version: v1}
    - name: v2
      labels: {version: v2}
```

### 限流与鉴权

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: {name: order-policy}
spec:
  selector: {matchLabels: {app: order}}
  action: ALLOW
  rules:
    - from: [{source: {principals: ["cluster.local/ns/default/sa/gateway"]}}]
      to: [{operation: {methods: [GET, POST]}}]
```

## 5. 与 Spring Cloud 的关系

| 维度 | Spring Cloud | Istio |
|---|---|---|
| 位置 | 应用内（SDK） | 应用外（Sidecar） |
| 语言 | Java 为主 | 任意 |
| 治理粒度 | 方法级（更细） | 服务/流量级 |
| 侵入 | 有 | 无 |
| 运维 | 应用团队 | 平台团队 |
| 学习成本 | 低 | 高 |
| 性能 | 少一跳 | 每请求多 2 跳（~1ms） |

**演进建议**：
- 已有 Spring Cloud 体系 → 不必急着上 Mesh。
- 多语言/大平台 → Mesh 价值大。
- 混合方案：Spring Cloud 做业务级熔断降级，Mesh 做流量治理与安全。
- K8s 环境下可先只用 **K8s Service + Mesh**，逐步去掉 Eureka 等组件。

## 6. 可观测性

Istio + Envoy 自动生成指标（Golden Signals：延迟、流量、错误、饱和度），配合：
- **Prometheus + Grafana**：指标
- **Jaeger / Zipkin / SkyWalking**：链路
- **Kiali**：服务拓扑图

> 注意：Mesh 的追踪需要在应用侧**传递** B3/W3C trace header，应用不传就会断链。

## 7. mTLS

- `PeerAuthentication` 配置 `STRICT` / `PERMISSIVE`。
- Sidecar 自动完成证书签发（Citadel）、轮换、双向 TLS 加密。
- 应用代码零改动即获得传输加密与身份认证。

## 8. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| Sidecar 注入失败 | Pod 只有 1 个容器 | 命名空间打 `istio-injection=enabled` 标签 |
| 追踪断链 | 链路不完整 | 应用需透传 `traceparent`/`b3` 头 |
| 启动顺序 | 应用先于 Envoy 就绪 | 配 `holdApplicationUntilProxyStarts` |
| 健康检查失败 | K8s 探针被 mTLS 拦截 | 探针改端口或用 `sidecar.istio.io/rewriteAppHTTPProbe` |
| 性能损耗 | P99 增加 | 调优连接池、Envoy 资源限制 |
| 配置复杂 | 排障困难 | `istioctl analyze`、`istioctl proxy-config` |
| 与 Spring Cloud 功能重复 | 双重熔断 | 明确分层：Mesh 管流量，应用管业务降级 |

## 9. 面试速答

- **Service Mesh 是什么**：处理服务间通信的专用基础设施层，通常以 Sidecar 代理实现，对应用透明。
- **数据平面与控制平面**：数据平面（Envoy）负责实际流量转发；控制平面（Istiod）负责配置下发与证书管理。
- **与 Spring Cloud 区别**：Spring Cloud 是应用内 SDK（侵入、语言相关），Mesh 是应用外代理（无侵入、语言无关）。
- **要不要上 Mesh**：多语言、大规模、有平台团队时才划算；小团队 Spring Cloud 足够，Mesh 运维成本不低。
