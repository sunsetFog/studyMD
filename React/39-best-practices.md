# 39 - 最佳实践 Checklist

> 项目初始化 / Code Review / 上线前三份清单。

## 1. 项目初始化清单

- [ ] Vite react-ts 模板（或 Next.js App Router 按需）
- [ ] TS strict + noUncheckedIndexedAccess
- [ ] ESLint（react-hooks 两条 error）+ Prettier + lint-staged
- [ ] 目录分层：pages/components/hooks/queries/api/stores/utils
- [ ] TanStack Query + 请求层封装（ApiError/401 去重）
- [ ] 路由懒加载 + 404/errorElement
- [ ] CI：tsc --noEmit + eslint + vitest run + build
- [ ] React Developer Tools 全员安装

## 2. 编码规范清单（Code Review）

### 状态
- [ ] 不可变更新 + 函数式 setState
- [ ] 无镜像/冗余 state；能算不存
- [ ] 状态机联合类型替代布尔组合
- [ ] 重置场景用 key
- [ ] 服务端数据走 Query 不进组件 state/Redux

### Effect
- [ ] effect 只做外部系统同步
- [ ] 所有订阅/定时器/请求有 cleanup
- [ ] 依赖诚实，不屏蔽 lint
- [ ] 无 effect 链 / effect 中 setState 派生

### 组件
- [ ] 渲染期纯函数（无副作用/ref 读写）
- [ ] props 默认值解构、透传 rest、className 合并
- [ ] key 稳定唯一；动态组件大写
- [ ] 条件渲染无 0/NaN 泄漏
- [ ] 复用逻辑走自定义 Hook（规则遵守）

### Context / Store
- [ ] Context 只放低频全局值；value useMemo
- [ ] Zustand selector 粒度 + useShallow
- [ ] 状态修改收口 action

### 安全
- [ ] dangerouslySetInnerHTML 白名单 + DOMPurify
- [ ] token 内存存储；localStorage 白名单脱敏
- [ ] redirect 校验；接口后端鉴权

## 3. 性能清单

- [ ] Profiler 录制过关键路径
- [ ] 状态下沉优先于 memo；memo 三件套配套
- [ ] 万级列表虚拟滚动
- [ ] 昂贵联动用 transition/deferredValue
- [ ] 路由/重组件 lazy
- [ ] Query staleTime 调优 + prefetch
- [ ] Web Vitals 基线（LCP<2.5s / INP<200ms / CLS<0.1）

## 4. 可靠性清单

- [ ] 路由级 ErrorBoundary + resetKeys + 重试
- [ ] 异步错误 showBoundary 转交
- [ ] 全局 unhandledrejection + 资源 error 兜底
- [ ] 上报带版本 + sourcemap 体系
- [ ] retry 区分幂等（GET 才自动重试）
- [ ] chunk 加载失败 reload 兜底

## 5. 上线前清单

- [ ] tsc + eslint + 测试全绿
- [ ] bundle analyze 通过、vendor 合理
- [ ] sourcemap 私有化、drop console
- [ ] 错误监控/埋点验证
- [ ] 灰度与回滚方案
- [ ] 低版本浏览器回归（legacy 决策确认）
- [ ] 敏感信息扫描（token/密钥/内网地址）

## 6. Next.js 项目附加

- [ ] 'use client' 边界最小下沉
- [ ] 数据获取默认 Server Component
- [ ] Server Action 内鉴权
- [ ] revalidate 策略明确
- [ ] 水合错误零容忍

## 7. 团队协作

- [ ] conventional commits + PR 模板含自检
- [ ] 组件库 semver + changesets
- [ ] 技术选型 ADR 记录
- [ ] 新人入读清单：01→05→09→15→19→38

## 使用建议

- 初始化：走清单 1 一次配齐
- Review：清单 2 按模块抽查
- 发版：清单 3/4/5 过一遍签字
