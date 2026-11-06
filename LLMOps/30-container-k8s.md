# 30 - 容器化与 K8s：AI 服务的特殊配置

> 假设你有既有 K8s/Spring 运维经验（`java777/` 基础）。本篇只讲 AI 服务**不同于普通后端**的容器化配置点。

---

## 1. 长请求对 Pod 配置的影响

LLM 编排服务的请求特征：**长耗时（10-60s+）、流式长连接、突发并发**。连锁影响：

| 配置项 | 传统 API 惯例 | AI 服务的调整 | 原因 |
|---|---|---|---|
| Ingress/网关超时 | 30-60s | 按场景分级，对话类 ≥90s | 长请求被掐断 = 前端报错但模型还在烧钱 |
| max connections | 按 QPS | 按并发占用时长 ×20 | 10s 请求 vs 0.5s 请求，同 QPS 连接数差 20 倍 |
| 优雅终止宽限期 | 5-10s | **30-60s+** | 发布/缩容时不能掐断进行中的 LLM 调用（钱已付，掐断=白烧） |
| 客户端超时 | 统一值 | 分级（09/12 章） | 检索/生成/流式各自预算 |

### 1.1 优雅终止（AI 服务最重要的一条）

```yaml
# terminationGracePeriodSeconds 必须覆盖最长请求
spec:
  terminationGracePeriodSeconds: 75   # > 对话类 P99 耗时
containers:
  - name: orchestrator
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 10"]  # 等 LB 摘流完成
```

配套要求：接到 SIGTERM 后**停止接新请求、进行中请求跑完或迁移**；流式连接客户端感知断开后由前端重连续传（15 章任务状态机兜底）。发布摘流顺序：readiness 切换 → LB 摘流 → 等存量 → 杀进程。

---

## 2. 资源与伸缩配置

### 2.1 资源画像

| 组件 | CPU/内存画像 | 说明 |
|---|---|---|
| 编排服务 | 低 CPU、中内存 | 大头是流式 buffer 与上下文暂存（长 prompt 快照占内存，21 章） |
| 队列 worker | 中 CPU | 解析/后处理，可批处理摊薄 |
| 网关 | 低 CPU、高连接数 | 主要是 IO 转发 + 计量打点 |
| 自建推理 Pod | **GPU 资源独占** | 节点池隔离，不做 bin-packing |

### 2.2 HPA 指标选择（呼应 29 章）

```yaml
# 编排服务：按并发连接数扩容（CPU 指标失真）
metrics:
  - type: External
    external:
      metric:
        name: llm_active_connections
      target:
        averageValue: "200"
```

配合 13 章舱壁：同一 Pod 内按场景/模型分池的并发限制用信号量（Resilience4j），K8s 层不必按场景拆 Deployment——除非场景间资源画像差异巨大。

### 2.3 缩容与突发

- LLM 流量常有峰谷（白天/夜间、营销脉冲）：**缩容速度调慢**（scaleDown stabilization ≥5 分钟），避免把预热好的连接池/缓存（08 章）缩没了。
- 供应商限流是外部硬约束，突发扩容解决不了上游 429（10 章排队 + 舱壁才是正解）。

---

## 3. 配置与密钥挂载（衔接 05 章）

| 项 | 做法 |
|---|---|
| API Key | Secret 挂载，只给网关 Deployment；禁入环境变量明文（ps 可见）、禁入镜像 |
| prompt/护栏/路由配置 | ConfigMap 或配置中心（Nacos/Apollo），支持热更（05 章），**不依赖重启生效** |
| 计价表 | 同上，变更历史化（21 章） |
| 镜像 | 不打 prompt 文件进镜像（否则 prompt 变更=发镜像，16 章原则被破坏）；可打包默认兜底模板 |

---

## 4. 探针设计

```yaml
livenessProbe:   # 只探「进程还活着」：healthz 端点，不碰上游
  httpGet: {path: /actuator/health/liveness, port: 8080}
readinessProbe:  # 探「能服务」：本地依赖就绪（DB/Redis/队列连接）
  httpGet: {path: /actuator/health/readiness, port: 8080}
  periodSeconds: 5
startupProbe:    # 启动慢的服务（连向量库建池、加载配置）防误杀
  failureThreshold: 30
```

**关键陷阱：不要把「模型供应商可达性」放进 readiness**。供应商抖一下 → 所有 Pod 被摘流 → 服务整体下线（11 章降级链正是为这种情况设计的，摘流等于放弃降级能力）。供应商状态由网关熔断器（13 章）处理，不进 K8s 探针。

---

## 5. 发布策略（衔接 31 章）

| 策略 | AI 服务适用性 |
|---|---|
| Rolling Update | 默认选择；配合长宽限期与摘流 |
| Blue/Green | 大版本（模型迁移 18 章）可整簇切换，快速回退 |
| Canary（K8s 层） | **通常不需要**——17 章的灰度在「应用层按用户分流」实现，比 K8s 副本比例灰度精确得多（副本灰度是按请求随机，违反 17 章用户维度原则） |

结论：K8s 层做蓝绿/滚动，**行为灰度在应用层做**，两层职责不要混。

---

## 6. 常见坑

| 坑 | 后果 | 对策 |
|---|---|---|
| 宽限期 10s 直接杀 Pod | 发布时掐断长请求，钱白烧+用户报错 | 宽限期 > P99 请求时长 |
| readiness 探上游供应商 | 供应商抖动引发全服务摘流下线 | 上游状态交熔断器，探针只探本地 |
| CPU-based HPA | 容器扩了也没用，瓶颈在配额 | 并发连接数/队列深度指标 |
| prompt 打进镜像 | prompt 变更被逼走发版流程 | 配置外置热更 |
| 缩容太快 | 连接池/缓存频繁重建，成本延迟抖动 | 缩容稳定窗口 ≥5 分钟 |
| 网关超时 < 场景预算 | 60s 回答被 504 掐断 | 分级超时对齐（09 章） |
| GPU Pod 与普通 Pod 混排 | 调度失败/资源碎片 | 节点池隔离 + taint/toleration |
| K8s 副本灰度当行为灰度用 | 请求级随机分流，违反用户一致性 | 行为灰度在应用层（17 章） |

---

## 7. Checklist

- [ ] terminationGracePeriodSeconds 覆盖对话类 P99，preStop 摘流等待
- [ ] SIGTERM 处理：停止接新、存量跑完、流式断点可续
- [ ] LB/网关超时按 09 章分级表配置
- [ ] HPA 用并发连接数/队列深度，缩容稳定窗口 ≥5 分钟
- [ ] readiness 只探本地依赖，上游供应商状态交熔断器
- [ ] startupProbe 覆盖慢启动组件
- [ ] API Key 仅网关 Secret 挂载；prompt/护栏/计价配置全部热更可达
- [ ] 镜像内不含 prompt 正文（默认兜底模板除外）
- [ ] GPU 推理独立节点池
- [ ] 发布策略：K8s 层滚动/蓝绿，行为灰度留在应用层
