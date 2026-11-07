# 20 - API 设计规范（全栈整合）

## 1. RESTful 设计（必考⭐）

| 动作 | 方法+路径 | 状态码 |
|---|---|---|
| 列表 | GET /orders | 200 |
| 详情 | GET /orders/{id} | 200 / 404 |
| 创建 | POST /orders | 201 |
| 全量更新 | PUT /orders/{id} | 200 |
| 局部更新 | PATCH /orders/{id} | 200 |
| 删除 | DELETE /orders/{id} | 204 |
| 非资源动作 | POST /orders/{id}/cancel | 200 |

- 名词复数、层级 ≤2、过滤用 query（?status=1&page=1）、版本化（/v1/ 或 Header）
- 务实原则：国内团队多采用"RESTful 风格 + 统一响应体"折中——动作语义复杂时 POST + 动词子路径完全可接受，讲清 trade-off 即可

## 2. 统一响应体（前后端契约）

```json
{ "code": 0, "message": "ok", "data": { }, "traceId": "abc123" }
```

- code 体系：0/200 成功；4xx 客户端（参数、未登录 401、无权限 403、业务 1xxx）；5xx 服务端
- 业务错误码枚举类管理，禁止魔法数字散落
- traceId 贯穿：前端报障带上即可后端定位
- 分页约定：{ list, total, pageNum, pageSize }

## 3. 错误处理契约

- 4xx 带可读 message（面向开发者）+ errors 字段（字段级校验错误）
- 5xx 不暴露内部细节（堆栈/SQL），日志留痕 + traceId
- 前端统一 axios 拦截：code 分流（401 去登录、403 提示、5xx 兜底提示）

## 4. 接口文档

- SpringDoc（OpenAPI 3）：@Operation/@Schema 注解生成 swagger-ui
- 契约优先（加分）：先写 OpenAPI yaml → 生成前后端代码（保证联调一致）
- 变更管理：废弃字段保留一个版本 + 文档标注 @Deprecated

## 5. 接口版本化与兼容

| 策略 | 适用 |
|---|---|
| URL /v1/v2 | 直观，主流 |
| Header 版本 | URL 干净，调试不便 |
| 字段兼容优先 | 加字段不删字段、不改语义，可长期不升版本 |

## 6. 前后端联调工程实践（高频追问）

1. **Mock**：Apifox/YApi 依据 OpenAPI 生成 mock，前端不等后端
2. **环境分层**：dev/test/staging/prod + Vite 代理解决本地跨域
3. **接口变更流程**：文档先行 → 评审 → 后端实现 → 联调 → 回归
4. **响应裁剪**：列表页与详情页分开接口（避免大而全）
5. **防重放/幂等**：写接口携带 requestId，后端唯一键兜底（见 18）

## 常见追问

1. GET 带 body？——不符合语义，axios 也不支持，用 query 或改 POST
2. POST 幂等吗？——不幂等，需业务层 token/唯一键保障
3. 长轮询 vs WebSocket 选什么？——见 21

## Checklist

- [ ] RESTful 动词与状态码对应关系清晰
- [ ] 统一响应体 + 错误码体系能画出结构
- [ ] 有契约优先/Mock 联调的实践认知
