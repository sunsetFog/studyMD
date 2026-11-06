# 42 - 最佳实践 Checklist

> 项目启动 / Code Review / 上线前三份清单，逐项可勾选。

## 1. 项目初始化清单

- [ ] `npm create vue@latest`（TS + ESLint + Prettier + Router + Pinia + Vitest）
- [ ] VSCode 插件统一（Vue - Official）并入库 `.vscode/` 配置
- [ ] 路径别名 `@/`（vite + tsconfig 双配置）
- [ ] 目录按 `24-sfc-project-structure.md` 分层
- [ ] api 层封装（拦截器/错误码/重试）
- [ ] 环境变量规范（VITE_ 前缀、多模式）
- [ ] lint-staged + husky 提交卡口
- [ ] CI：vue-tsc + eslint + vitest run + build

## 2. 编码规范清单（Code Review 用）

### 响应式
- [ ] 默认 ref；reactive 不解构不重赋值
- [ ] 第三方实例 markRaw；大数据评估 shallowRef
- [ ] 派生值 computed；watch 只做副作用
- [ ] watch reactive 属性用 getter；异步 watch 有竞态保护

### 组件
- [ ] props/emit/emits 全类型化；默认值工厂函数
- [ ] 不修改 props；单向数据流
- [ ] composable 入参 MaybeRefOrGetter + toValue
- [ ] expose 最小化（行为 + readonly）
- [ ] 浮层 Teleport + 全局 z-index 规范

### 模板
- [ ] v-for 稳定唯一 key；无 v-if/v-for 混用
- [ ] 复杂表达式抽 computed
- [ ] v-html 白名单消毒

### 状态
- [ ] 服务端状态不进 Pinia；store 按业务域拆
- [ ] 组件只调 action 改 store
- [ ] 列表筛选/分页进 URL query

### 安全
- [ ] token 内存存储；localStorage 白名单
- [ ] 生产 drop console；sourcemap 私有化
- [ ] redirect 校验；接口后端鉴权

## 3. 性能清单

- [ ] 路由懒加载（非首页全部）
- [ ] 重组件 defineAsyncComponent
- [ ] 万级列表虚拟滚动或分页
- [ ] 首屏请求并行 + 骨架屏
- [ ] 高频输入防抖
- [ ] bundle analyze 通过且 vendor 合理
- [ ] Web Vitals 有基线（LCP<2.5s / INP<200ms / CLS<0.1）

## 4. 可靠性清单

- [ ] 全局 errorHandler + unhandledrejection + 资源 error 三路捕获
- [ ] 错误上报带版本 + sourcemap 体系
- [ ] 路由级 ErrorBoundary + 重试
- [ ] api 层重试（指数退避）+ 401 去重处理
- [ ] chunk 加载失败兜底（router.onError + reload）
- [ ] 定时器/监听/第三方实例清理配对

## 5. 上线前清单

- [ ] vue-tsc 类型检查通过
- [ ] ESLint 零 error
- [ ] 单测通过 + 关键链路 E2E 通过
- [ ] 生产构建产物 analyze 无意外
- [ ] sourcemap 处理策略执行（私有上传或关闭）
- [ ] 错误监控/埋点接入验证
- [ ] 灰度/回滚方案明确
- [ ] 真机低版本浏览器回归（legacy 需求确认）
- [ ] 敏感信息扫描（token/密钥/内网地址不入库）

## 6. KeepAlive / 路由相关

- [ ] 缓存组件显式 defineOptions name
- [ ] KeepAlive 有 include/exclude + max
- [ ] 缓存页数据刷新放 onActivated
- [ ] 动态权限路由：注册后 return fullPath 重入、登出清理
- [ ] redirect 白名单校验

## 7. 团队协作

- [ ] 分支模型与 commit 规范（conventional commits）
- [ ] PR 模板含自检清单
- [ ] 组件库变更走 semver + changelog
- [ ] ADR（架构决策记录）沉淀重大选型

## 使用建议

- 初始化项目：走清单 1，一次性配齐
- 日常 review：清单 2 按模块抽查
- 发版前：清单 3/4/5 过一遍，负责人签字
