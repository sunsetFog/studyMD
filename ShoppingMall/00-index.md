# 仿淘宝商城（ShoppingMall）需求文档全集 · 总索引

> **性质说明**：本套文档为电商商城（淘宝模式）的学习/练习用需求文档体系，从 0 构建一个
> 综合电商平台（C 端用户 + B 端商家 + 平台运营三方）的完整 PRD。内容为通用电商产品逻辑，
> **非淘宝官方文档**。
>
> 覆盖：账号 → 商品 → 交易 → 售后 → 营销 → 商家 → 履约 → 平台治理 → 支撑系统，共 53 篇。

---

## 阅读顺序建议

| 阶段 | 篇目 | 目标 |
|---|---|---|
| 一、全局认知 | 00-03 | 业务全景、角色体系、文档规范 |
| 二、买家主链路 | 04/08/11/13-16 | 注册→搜索→加购→下单→支付 |
| 三、商家主链路 | 32-35/37 | 入驻→发布（08）→履约→结算 |
| 四、逆向链路 | 20-22/39 | 退款退货→仲裁→客服 |
| 五、增长与运营 | 25-31/44 | 营销玩法、内容、广告 |
| 六、治理与支撑 | 41-43/45-46 | 风控、审核、规则、数据、SLA |

---

## 目录

### 一、全局（00-03）
| 编号 | 文件 | 内容 |
|---|---|---|
| 00 | [00-index.md](00-index.md) | 总索引与阅读路径（本篇） |
| 01 | [01-product-overview.md](01-product-overview.md) | 产品定位、业务架构图、三方角色、核心链路 |
| 02 | [02-doc-conventions.md](02-doc-conventions.md) | PRD 文档规范、模板、命名与评审规则 |
| 03 | [03-roles-permissions.md](03-roles-permissions.md) | 角色定义与权限矩阵（买家/商家/运营/客服/风控） |

### 二、账号域（04-07）
| 编号 | 文件 | 内容 |
|---|---|---|
| 04 | [04-register-login.md](04-register-login.md) | 注册、登录、多端一致、找回密码 |
| 05 | [05-account-security.md](05-account-security.md) | 账号安全中心、异地登录、风控锁定 |
| 06 | [06-membership-points.md](06-membership-points.md) | 会员等级、积分/淘金币体系 |
| 07 | [07-realname-wallet.md](07-realname-wallet.md) | 实名认证、平台钱包/余额 |

### 三、商品域（08-12）
| 编号 | 文件 | 内容 |
|---|---|---|
| 08 | [08-product-publish.md](08-product-publish.md) | 商品发布（SPU/SKU、属性、图文、资质） |
| 09 | [09-product-detail.md](09-product-detail.md) | 商品详情页（信息结构、模块规则） |
| 10 | [10-category-attributes.md](10-category-attributes.md) | 类目与属性体系（后台类目/前台展示） |
| 11 | [11-search.md](11-search.md) | 搜索（召回、排序、筛选、纠错、无结果） |
| 12 | [12-recommendation.md](12-recommendation.md) | 推荐系统产品需求（场景、策略、体验规则） |

### 四、交易域（13-19）
| 编号 | 文件 | 内容 |
|---|---|---|
| 13 | [13-shopping-cart.md](13-shopping-cart.md) | 购物车（加购、合并、失效、促销勾选） |
| 14 | [14-order-create.md](14-order-create.md) | 下单与结算页（地址/优惠/拆单规则） |
| 15 | [15-payment.md](15-payment.md) | 支付（渠道、超时关单、退款路径） |
| 16 | [16-order-management.md](16-order-management.md) | 订单状态机与管理（买家视角） |
| 17 | [17-order-cancel-modify.md](17-order-cancel-modify.md) | 取消订单/修改地址/拦截发货 |
| 18 | [18-inventory.md](18-inventory.md) | 库存体系（占用/回补/超卖防控） |
| 19 | [19-price-discount-engine.md](19-price-discount-engine.md) | 价格与优惠计算引擎（叠加互斥规则） |

### 五、售后域（20-24）
| 编号 | 文件 | 内容 |
|---|---|---|
| 20 | [20-refund-only.md](20-refund-only.md) | 仅退款流程 |
| 21 | [21-return-refund-exchange.md](21-return-refund-exchange.md) | 退货退款/换货 |
| 22 | [22-dispute-arbitration.md](22-dispute-arbitration.md) | 交易纠纷与平台仲裁 |
| 23 | [23-review-rating.md](23-review-rating.md) | 评价体系（评分、晒图、追评、申诉） |
| 24 | [24-questions-answers.md](24-questions-answers.md) | 问大家（UGC 问答） |

### 六、营销与内容域（25-31）
| 编号 | 文件 | 内容 |
|---|---|---|
| 25 | [25-coupons.md](25-coupons.md) | 优惠券（类型、发放、核销、对账） |
| 26 | [26-flash-sale.md](26-flash-sale.md) | 秒杀/限时抢购（库存预热、防超卖） |
| 27 | [27-presale-groupbuy.md](27-presale-groupbuy.md) | 预售与拼团 |
| 28 | [28-promotion-center.md](28-promotion-center.md) | 营销活动中心（满减/满赠/跨店活动） |
| 29 | [29-live-commerce.md](29-live-commerce.md) | 直播电商（挂链、讲解、交易闭环） |
| 30 | [30-content-feed.md](30-content-feed.md) | 内容种草（图文/短视频/逛逛流） |
| 31 | [31-store-decoration.md](31-store-decoration.md) | 店铺装修（商家自运营页面） |

### 七、商家与履约域（32-38）
| 编号 | 文件 | 内容 |
|---|---|---|
| 32 | [32-merchant-onboarding.md](32-merchant-onboarding.md) | 商家入驻（资质审核、保证金、类目授权） |
| 33 | [33-store-management.md](33-store-management.md) | 店铺管理（信息、经营设置、子账号） |
| 34 | [34-order-fulfillment.md](34-order-fulfillment.md) | 商家发货履约（打单、发货、拦截） |
| 35 | [35-settlement-billing.md](35-settlement-billing.md) | 结算与账单（货款、佣金、账期） |
| 36 | [36-store-scores.md](36-store-scores.md) | 店铺评分体系（DSR、指标、处罚联动） |
| 37 | [37-logistics-tracking.md](37-logistics-tracking.md) | 物流与跟踪（对接、轨迹、异常件） |
| 38 | [38-address-management.md](38-address-management.md) | 收发货地址（库表、校验、隐私面单） |

### 八、平台治理与支撑域（39-46）
| 编号 | 文件 | 内容 |
|---|---|---|
| 39 | [39-customer-service.md](39-customer-service.md) | 客服系统（机器人、人工、工单） |
| 40 | [40-message-notification.md](40-message-notification.md) | 消息通知中心（触达矩阵、频控） |
| 41 | [41-risk-control.md](41-risk-control.md) | 风控反作弊（薅羊毛、刷单、账号盗用） |
| 42 | [42-product-compliance-review.md](42-product-compliance-review.md) | 商品审核与合规（机审+人审、资质校验） |
| 43 | [43-platform-rules.md](43-platform-rules.md) | 平台规则体系（违规类型、处罚梯度、申诉） |
| 44 | [44-advertising.md](44-advertising.md) | 广告投放（搜索竞价、展示广告、计费） |
| 45 | [45-data-analytics.md](45-data-analytics.md) | 数据分析与报表（指标树、看板、埋点） |
| 46 | [46-performance-sla.md](46-performance-sla.md) | 非功能需求：性能、可用性、SLA、降级 |

### 九、收尾（47-52）
| 编号 | 文件 | 内容 |
|---|---|---|
| 47 | [47-app-information-architecture.md](47-app-information-architecture.md) | App 信息架构与导航（底部 Tab、页面清单） |
| 48 | [48-personal-center.md](48-personal-center.md) | 个人中心（我的订单/资产/工具） |
| 49 | [49-api-edge-cases.md](49-api-edge-cases.md) | 关键接口约定与异常边界（幂等、并发、超时） |
| 50 | [50-launch-checklist.md](50-launch-checklist.md) | 上线检查清单（首版 MVP 范围 + 验收） |
| 51 | [51-open-questions.md](51-open-questions.md) | 开放问题与后续迭代清单 |
| 52 | [52-glossary.md](52-glossary.md) | 术语表（SPU/SKU/DSR/GBM...） |

---

## MVP 优先级（首版范围标注）

各 PRD 中用「P0/P1/P2」标注需求优先级：
- **P0**：MVP 必备（跑通交易闭环的最小集）
- **P1**：首年迭代
- **P2**：远期规划

P0 核心集：04、08、10、11、13、14、15、16、18、20、32、34、37、42、46。
