# 全栈面试考点体系索引

> 技术栈：前端 Vue3 / React 双轨 + TypeScript，后端 Java 17 + SpringBoot 3，中间件 Redis/MySQL/RabbitMQ。
> 本系列定位：**面试考点浓缩**，原理深挖见仓库内 Java/、SpringBoot/、Mysql/、Vue3/、React/ 系列专题。

## 学习路线

```
通用基础 → 前端框架双轨 → 后端核心 → 全栈整合 → 系统设计 → 面试实战
```

## 目录

### 一、通用基础（01-04）
| 篇目 | 文件 | 核心考点 |
|---|---|---|
| 01 | [01-http-network.md](01-http-network.md) | HTTP/HTTPS、TCP 握手、状态码、缓存、CDN |
| 02 | [02-js-core.md](02-js-core.md) | 闭包、原型链、事件循环、Promise、防抖节流 |
| 03 | [03-typescript.md](03-typescript.md) | 类型系统、泛型、工具类型、type vs interface |
| 04 | [04-browser.md](04-browser.md) | 渲染原理、重绘回流、存储、跨域、Web 安全 |

### 二、前端框架双轨（05-10）
| 篇目 | 文件 | 核心考点 |
|---|---|---|
| 05 | [05-vue3-core.md](05-vue3-core.md) | 响应式、组合式 API、v-model、生命周期 |
| 06 | [06-vue3-advanced.md](06-vue3-advanced.md) | Proxy 原理、diff 策略、keep-alive、性能 |
| 07 | [07-react-core.md](07-react-core.md) | 组件、Props、状态、事件、受控非受控 |
| 08 | [08-react-advanced.md](08-react-advanced.md) | Hooks、Fiber、重渲染、状态管理、性能优化 |
| 09 | [09-framework-compare.md](09-framework-compare.md) | Vue3 vs React 深度对比（高频开放题） |
| 10 | [10-frontend-engineering.md](10-frontend-engineering.md) | Vite/Webpack、首屏优化、SSR、微前端 |

### 三、后端核心（11-18）
| 篇目 | 文件 | 核心考点 |
|---|---|---|
| 11 | [11-java-core.md](11-java-core.md) | 集合、HashMap、并发、线程池、锁 |
| 12 | [12-jvm.md](12-jvm.md) | 内存区域、GC、类加载、排查 |
| 13 | [13-spring-core.md](13-spring-core.md) | IoC/AOP、Bean 生命周期、事务失效、循环依赖 |
| 14 | [14-springboot-web.md](14-springboot-web.md) | 自动配置、参数校验、全局异常、拦截器 |
| 15 | [15-mysql-mybatis.md](15-mysql-mybatis.md) | 索引、事务、MVCC、慢 SQL、MyBatis |
| 16 | [16-redis.md](16-redis.md) | 数据结构、缓存三问、分布式锁、持久化 |
| 17 | [17-mq.md](17-mq.md) | 消息可靠性、幂等、积压、选型 |
| 18 | [18-distributed.md](18-distributed.md) | 分布式锁/ID/事务、限流熔断、幂等设计 |

### 四、全栈整合（19-21）
| 篇目 | 文件 | 核心考点 |
|---|---|---|
| 19 | [19-auth-security.md](19-auth-security.md) | Session vs JWT、OAuth2、SSO、XSS/CSRF |
| 20 | [20-api-design.md](20-api-design.md) | RESTful 规范、统一响应、错误码、版本化 |
| 21 | [21-realtime-file.md](21-realtime-file.md) | WebSocket、文件上传下载、大文件分片 |

### 五、系统设计与实战（22-25）
| 篇目 | 文件 | 核心考点 |
|---|---|---|
| 22 | [22-system-design-basics.md](22-system-design-basics.md) | 高并发方法论、缓存/异步/分库分表 |
| 23 | [23-design-cases.md](23-design-cases.md) | 秒杀、短链、Feed 流、支付、排行榜拆解 |
| 24 | [24-devops-deploy.md](24-devops-deploy.md) | CI/CD、Docker、Nginx、监控告警 |
| 25 | [25-interview-strategy.md](25-interview-strategy.md) | 答题方法论、项目包装、高频追问应对 |

## 全局规则（只写一次）

- **事件循环**（JS 宏微任务）：见 02，浏览器篇 04 互引
- **缓存三问**（穿透/击穿/雪崩）：见 16，系统设计 22/23 引用
- **事务失效**：见 13，SpringBoot 专题 14 详述
- **JWT 完整方案**：见 19，SpringBoot 专题 21 有实现
- **秒杀全链路**：见 23，涉及 16/17/18 各篇机制
- 深挖指向仓库专题系列，本系列保持"考点 + 答题框架"密度
