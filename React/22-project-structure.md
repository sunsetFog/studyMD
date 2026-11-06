# 22 - 项目结构与 Vite

## 1. 推荐目录结构

```
src/
├── main.tsx
├── App.tsx
├── routes/               # 路由定义（按模块拆）
├── pages/                # 页面（路由级容器）
│   ├── user/
│   │   ├── UserListPage.tsx
│   │   └── components/   # 页面私有组件
│   └── order/
├── components/
│   ├── common/           # 通用展示组件（Button/Modal 包装）
│   ├── business/         # 业务组件（UserCard/OrderItem）
│   └── layout/
├── hooks/                # 自定义 hooks（primitives/queries/features）
├── queries/              # TanStack Query 封装（key 工厂 + hooks）
├── api/                  # 请求层（http 实例 + 按域接口）
├── stores/               # Zustand/Redux slices
├── contexts/             # Context Provider 模块
├── utils/                # 纯函数工具（无状态）
├── types/                # 全局类型
├── constants/
└── styles/
```

规则：
- **pages 私有组件就近**，跨页复用才晋升 components/business
- api 层类型定义在 api 模块导出，组件 import type
- utils 无状态无副作用；有状态逻辑进 hooks

## 2. 路径别名

```ts
// vite.config.ts
import path from 'node:path'
resolve: { alias: { '@': path.resolve(__dirname, 'src') } }
// tsconfig.json
"paths": { "@/*": ["./src/*"] }
```

层级约定：pages 之间禁止互相 import 内部组件（循环依赖 + 隐式耦合）。

## 3. Vite 配置模板

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'node:path'

export default defineConfig({
  plugins: [react()],        // babel 热更新（含 fast refresh）
  resolve: { alias: { '@': path.resolve(__dirname, 'src') } },
  server: {
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true, rewrite: p => p.replace(/^\/api/, '') },
    },
  },
  build: {
    target: 'es2018',
    sourcemap: false,
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('antd') || id.includes('echarts')) return 'vendor-heavy'
            return 'vendor'
          }
        },
      },
    },
  },
})
```

React 专属插件说明：`@vitejs/plugin-react` 用 Babel（支持 fast refresh 与 JSX 转换）；追求更快构建可用 `@vitejs/plugin-react-swc`（SWC 替代 Babel）。

## 4. 环境变量

```bash
# .env.development / .env.production
VITE_API_BASE=/api
```

```ts
const base = import.meta.env.VITE_API_BASE
// 类型增强 env.d.ts
interface ImportMetaEnv { readonly VITE_API_BASE: string }
```

## 5. 代码规范工具链

```bash
npm i -D eslint typescript-eslint eslint-plugin-react-hooks prettier
```

关键规则（React 专属）：
- `react-hooks/rules-of-hooks`（error）——Hooks 顺序纪律
- `react-hooks/exhaustive-deps`（error）——依赖诚实
- `@typescript-eslint/no-floating-promises`——异步错误防漏
- react-compiler eslint 插件（可选，提前暴露纯度问题）

lint-staged + husky 提交卡口（同 vue3/24）。

## 6. 组件文件粒度

| 场景 | 粒度 |
|---|---|
| 页面 | >400 行拆子组件/Hook |
| 拆分信号 | 逻辑块独立、复用、需 memo 边界 |
| 一个文件一个组件 | 例外：Compound 小部件可同文件 |

命名：组件文件 PascalCase 与组件同名；hooks `useXxx.ts`；常量 UPPER_SNAKE。

## 7. imports 排序与绝对导入

```tsx
// 统一顺序（eslint-plugin-import + simple-import-sort 自动化）
// 1. react/react-dom  2. 第三方  3. @/ 别名  4. 相对路径  5. 类型/样式
```

## 8. 多环境与部署

```bash
# 构建产物：dist/（静态托管 + nginx fallback 到 index.html for SPA 路由）
# Docker 示例
FROM node:20-alpine AS build
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

SPA fallback：`try_files $uri /index.html;`（history 路由刷新 404 的解）。

## 常见坑

1. **pages 互相引用**：循环依赖；晋升公共组件。
2. **别名只配了 vite 没配 tsconfig**（或反之）：类型报错。
3. **react-hooks lint 未开**：Hooks 规则裸奔，最危险的沉默错误。
4. **manualChunks 拆太碎**：首屏请求变多反而慢。
5. **nginx 无 fallback**：路由刷新 404。
6. **env 泄漏密钥**：VITE_ 前缀全部暴露到客户端。

## Checklist

- [ ] 目录分层清晰，pages 私有组件就近
- [ ] vite + tsconfig 别名同步
- [ ] react-hooks 两条规则 error 级开启
- [ ] vendor 分包与缓存策略
- [ ] 部署 fallback 配置验证
- [ ] 环境变量无敏感信息
