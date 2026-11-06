# LLMOps 知识体系（大语言模型应用的运维与工程）

> 目标：让 AI 应用「跑得稳、烧得起、可排查、能回滚」。
> 定位：面向 **AI 应用开发者**（不是算法工程师），补齐传统后端不会遇到的一整套运维问题：
> 概率性输出、按 token 计费的持续成本、模型版本漂移、Prompt 也是代码。
> 结构：概览架构 → 成本性能 → 可靠性降级 → Prompt/版本管理 → 可观测性 → 评估保障 → 部署与工具。
> 前置：建议先读 `java777/`（Spring/后端基础）与 `Product/32-36`（AI 产品的可行性、UX、Evals 定义）。

---

## 与 Product 篇的分工

| 主题 | Product 篇（定义侧） | LLMOps 篇（工程侧） |
|---|---|---|
| 成本 | 定价、单位经济、毛利 | token 计量、降本手段、预算熔断 |
| 质量 | evals 集怎么建、指标怎么定 | evals 怎么自动化、怎么进 CI |
| Prompt | prompt 怎么写、怎么评审 | prompt 怎么版本化、灰度、回滚 |
| 安全 | 合规、信任设计 | 护栏工程、密钥、审计落地 |

---

## 目录

### 一、概览与架构
| 编号 | 文件 | 内容 |
|---|---|---|
| 01 | [01-llmops-overview.md](01-llmops-overview.md) | LLMOps 是什么、与 DevOps/MLOps 的区别、核心挑战 |
| 02 | [02-app-lifecycle.md](02-app-lifecycle.md) | AI 应用全生命周期与各环节的工程活动 |
| 03 | [03-architecture.md](03-architecture.md) | 典型架构：网关/编排/向量库/缓存/异步 |
| 04 | [04-model-gateway.md](04-model-gateway.md) | 模型网关、多模型抽象、路由与供应商隔离 |
| 05 | [05-config-secrets.md](05-config-secrets.md) | 环境、配置与密钥（API Key）管理 |

### 二、成本与性能
| 编号 | 文件 | 内容 |
|---|---|---|
| 06 | [06-token-cost-model.md](06-token-cost-model.md) | Token 计量、计费模型、成本计算与预算 |
| 07 | [07-cost-optimization.md](07-cost-optimization.md) | 降本手段：压缩、路由、批量、小模型 |
| 08 | [08-caching.md](08-caching.md) | 精确缓存、语义缓存、embedding 缓存 |
| 09 | [09-latency-optimization.md](09-latency-optimization.md) | 延迟优化：流式、并行、上下文裁剪 |
| 10 | [10-rate-limit-quota.md](10-rate-limit-quota.md) | 限流、配额、预算熔断与防刷 |

### 三、可靠性与降级
| 编号 | 文件 | 内容 |
|---|---|---|
| 11 | [11-fallback-degradation.md](11-fallback-degradation.md) | 降级策略、多模型容灾、兜底内容 |
| 12 | [12-retry-timeout.md](12-retry-timeout.md) | 重试、超时、退避、幂等重试 |
| 13 | [13-circuit-breaker.md](13-circuit-breaker.md) | 熔断、舱壁隔离、背压 |
| 14 | [14-idempotency.md](14-idempotency.md) | 幂等性设计、去重与一致性 |
| 15 | [15-async-task-queue.md](15-async-task-queue.md) | 异步任务、队列、长耗时任务与回调 |

### 四、Prompt 与版本管理
| 编号 | 文件 | 内容 |
|---|---|---|
| 16 | [16-prompt-versioning.md](16-prompt-versioning.md) | Prompt 版本化、存储结构与变更管理 |
| 17 | [17-prompt-cicd.md](17-prompt-cicd.md) | Prompt 的 CI/CD、灰度与一键回滚 |
| 18 | [18-model-upgrade.md](18-model-upgrade.md) | 模型版本升级、迁移与兼容处理 |
| 19 | [19-knowledge-base-ops.md](19-knowledge-base-ops.md) | 知识库运维：数据管道、增量更新、重建索引 |

### 五、可观测性
| 编号 | 文件 | 内容 |
|---|---|---|
| 20 | [20-observability.md](20-observability.md) | LLM 可观测性总览：trace/span/会话 |
| 21 | [21-logging-tracing.md](21-logging-tracing.md) | 日志规范与全链路追踪落库 |
| 22 | [22-metrics-dashboard.md](22-metrics-dashboard.md) | 指标设计与看板（质量/成本/延迟） |
| 23 | [23-alerting.md](23-alerting.md) | 告警规则、分级与值班响应 |
| 24 | [24-drift-detection.md](24-drift-detection.md) | 漂移检测：模型/数据/输入/成本 |

### 六、评估与质量保障
| 编号 | 文件 | 内容 |
|---|---|---|
| 25 | [25-eval-automation.md](25-eval-automation.md) | 离线 evals 自动化流水线 |
| 26 | [26-online-eval.md](26-online-eval.md) | 线上评估：影子流量、A/B、金丝雀集 |
| 27 | [27-guardrails.md](27-guardrails.md) | 护栏工程：输入输出过滤、注入防护、审核 |
| 28 | [28-release-gate.md](28-release-gate.md) | 回归测试与发布门禁 |

### 七、部署与基础设施
| 编号 | 文件 | 内容 |
|---|---|---|
| 29 | [29-deployment-arch.md](29-deployment-arch.md) | 部署架构选型：容器/Serverless/自建推理 |
| 30 | [30-container-k8s.md](30-container-k8s.md) | 容器化与 K8s（AI 服务的特殊配置） |
| 31 | [31-ci-cd.md](31-ci-cd.md) | CI/CD 流水线与发布策略 |
| 32 | [32-security-compliance.md](32-security-compliance.md) | 安全与合规运维：密钥、数据、审计 |
| 33 | [33-llmops-tooling.md](33-llmops-tooling.md) | 工具链与平台选型（自建 vs 托管） |
| 34 | [34-runbook-cheatsheet.md](34-runbook-cheatsheet.md) | 故障手册、应急预案与速查清单 |

---

## 学习路径（6 周）

| 周次 | 主题 | 篇目 |
|---|---|---|
| W1 | 建立 LLMOps 视角 | 01-05 |
| W2 | 把成本管住（最紧急） | 06-10 |
| W3 | 让服务不崩 | 11-15 |
| W4 | Prompt 与知识库工程化 | 16-19 |
| W5 | 看得见才修得快 | 20-24 |
| W6 | 质量保障与上线部署 | 25-34 |

## 阅读约定

- 代码块中的 `// ...` 表示省略的无关代码。
- 「坑」小节列的是真实项目中高频事故点。
- 「Checklist」为可直接落地的行动清单。
- 示例技术栈以 **Python/TS + 云端模型 API** 为主，兼容 Java/Spring 场景会特别标注。
