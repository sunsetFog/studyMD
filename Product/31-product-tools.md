# 31 · 产品工具链：用什么把工作跑起来

## 1. 工具选择原则

```
① 能协作（多人实时/异步）
② 能沉淀（可检索、可回溯）
③ 与团队现有习惯兼容（推广成本最低）
④ 不要为了工具而工具（工具解决不了流程问题）
```

> 换工具的成本远高于工具本身的收益。**团队统一 > 工具先进**。

## 2. 各环节的常用工具

### 需求与文档

| 工具 | 特点 | 适用 |
|---|---|---|
| Notion | 灵活、数据库强、模板多 | 中小团队首选 |
| 飞书文档/钉钉文档 | 国内协作、与 IM 一体 | 国内团队 |
| Confluence | 企业 Wiki，与 Jira 集成 | 大团队 |
| 语雀 | 中文文档体验好 | 国内团队 |
| Google Docs | 实时协作强 | 海外团队 |
| Markdown + Git | 版本可控，适合技术团队 | ★ 开发者友好 |

### 原型与设计

| 工具 | 特点 |
|---|---|
| Figma | 事实标准，协作 + 组件 + 交付 |
| 即时设计 / 墨刀 / MasterGo | 国内替代，上手快 |
| Axure RP | 复杂交互逻辑（B 端） |
| Sketch | macOS 传统方案 |
| Framer | 高保真 + 可发布 |
| Excalidraw / Whimsical | 流程图、草图 |

### 项目管理

| 工具 | 特点 | 适用 |
|---|---|---|
| Jira | 强大、可配置、重 | 大团队/研发流程规范 |
| Linear | 现代、快、开发者友好 | 技术型小团队 |
| Trello | 看板极简 | 小团队/个人 |
| Notion Database | 灵活可定制 | 中小团队 |
| 飞书项目 / Tapd / 禅道 | 国内 | 国内团队 |
| GitHub Projects | 与代码库一体 | 开源/小团队 |

### 数据与埋点

| 类型 | 工具 |
|---|---|
| 行为分析 | Google Analytics、Mixpanel、Amplitude、神策、GrowingIO |
| 埋点管理 | 神策、Sensors、自建埋点平台 |
| 可视化 | Metabase、Superset、Tableau、PowerBI、Grafana |
| 会话回放 | Hotjar、FullStory、Clarity |
| A/B 实验 | Optimizely、GrowthBook、火山 AB、自建 |
| 用户反馈 | 问卷星、Typeform、Intercom、Zendesk |

### ★ AI 产品专用

| 用途 | 工具 |
|---|---|
| LLM 开发框架 | LangChain / LangGraph、LlamaIndex、Vercel AI SDK、Spring AI |
| 向量库 | pgvector、Milvus、Qdrant、Pinecone、Chroma |
| 观测与评估 | LangSmith、LangFuse、Helicone、Weights & Biases、Arize |
| Prompt 管理 | PromptLayer、LangFuse、自建（Git + 配置中心） |
| 低成本原型 | Dify、Coze、n8n、Flowise |
| 模型网关 | One API、LiteLLM、OpenRouter |

## 3. 最小可行工具栈（小团队推荐）

```
文档协作：飞书文档 / Notion
原型：Figma（或 即时设计）
项目管理：Linear / 飞书项目 / Notion Database
数据分析：GA4 + Metabase（自建看板）
用户反馈：微信群 + 表单 + 应用商店评论
AI 专用：LangFuse（观测）+ 自建 evals 集
代码与文档版本：Git
```

## 4. 文档组织建议

```
Product/
├── 00-index.md            # 导航
├── strategy/              # 战略与规划
│   ├── 2026-Q4-roadmap.md
│   └── positioning.md
├── requirements/          # 需求
│   ├── backlog.md         # 需求池（表格）
│   ├── PRD-xxx.md
│   └── decisions/         # 决策记录 ADR
├── research/              # 用户研究
│   ├── interviews/
│   └── competitors/
├── metrics/               # 指标定义与报告
│   ├── metric-dictionary.md
│   └── weekly-report/
└── releases/              # 版本记录
    └── v1.2.0.md
```

**关键文档**（必须有）：

| 文档 | 作用 |
|---|---|
| 指标字典 | 统一口径（最重要，避免扯皮） |
| 需求池 | 唯一的需求来源 |
| 决策记录 | 避免重复争论 |
| 版本记录 | 追溯变更 |
| 术语表 | 统一语言 |

## 5. 自动化提效

```
① 数据日报自动推送（SQL → 定时 → 群机器人）
② 用户反馈自动汇总（表单 → 表格 → 定期邮件）
③ 线上异常自动告警（监控 → 告警群）
④ 发布自动通知（CI → 群）
⑤ 竞品更新监控（RSS / 更新日志订阅）
⑥ ★ AI 成本日报（成本超阈值告警）
```

## 6. ★ AI 项目的工具补充建议

| 需求 | 建议 |
|---|---|
| **Prompt 版本管理** | prompt 存 Git 或专用平台，带版本号与变更记录，支持回滚 |
| **Evals 集管理** | 测试集也入库（输入 + 期望输出 + 评分标准），可复用 |
| **成本看板** | 每日成本、单次成本、单用户成本，设阈值告警 |
| **日志与追溯** | 每次请求记录 prompt/completion/token/耗时/模型版本，可检索 |
| **Bad case 库** | 失败案例单独归档，定期复盘 |
| **人工标注** | 简单的标注界面（👍👎 + 原因），让 PM 也能参与 |

## 7. 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 工具太多 | 信息散在 8 个系统 | 收敛到 3-4 个核心工具 |
| 文档写了找不到 | 没人能检索到 | 有 index + 统一命名 + 搜索 |
| 需求池多处维护 | 三个地方三个版本 | 单一事实来源 |
| 没有指标字典 | 各部门数字打架 | 建立并维护指标字典 |
| 工具推不动 | 团队不用 | 选大家已经在用的，或强制 + 培训 |
| 埋点平台没用起来 | 买了不用 | 先明确要看什么指标再选工具 |
| AI 项目没日志 | 出问题无法复现 | 全量记录请求与响应（注意脱敏） |

## 8. Checklist

- [ ] 团队统一使用同一套工具（文档/项目/设计/数据）
- [ ] 有指标字典、需求池、决策记录三个核心文档
- [ ] 文档有 index 与命名规范，可检索
- [ ] 关键报告自动化（数据日报、成本告警、异常告警）
- [ ] AI 项目：prompt 版本化、evals 集入库、请求日志可追溯
