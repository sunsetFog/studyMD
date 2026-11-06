# 25 - Vite 配置实战

## 1. Vite 心智模型

- **开发**：esbuild 预构建依赖（CommonJS→ESM）+ 原生 ESM 按需编译，启动与热更新毫秒级
- **生产**：Rollup 打包 + terser/esbuild 压缩

开发与生产的行为差异是 Vite 项目线上问题的主要来源（dev 正常 build 后坏）。

## 2. 标准配置模板

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { fileURLToPath } from 'node:url'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) },
  },
  css: {
    preprocessorOptions: {
      scss: {
        api: 'modern-compiler',
        additionalData: `@use "@/styles/variables" as *;`,   // 全局变量注入
      },
    },
  },
  server: {
    port: 5173,
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true, rewrite: p => p.replace(/^\/api/, '') },
    },
  },
  build: {
    target: 'es2018',
    outDir: 'dist',
    sourcemap: false,             // 生产按需（错误监控需要时配合上传私有源码映射）
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('element-plus') || id.includes('echarts')) return 'vendor-heavy'
            return 'vendor'
          }
        },
      },
    },
    chunkSizeWarningLimit: 1000,
  },
})
```

## 3. 环境与模式

```
.env                # 所有模式
.env.development    # dev
.env.production     # build
.env.staging        # 自定义模式 build --mode staging
```

- 变量必须 `VITE_` 前缀才暴露给客户端
- `import.meta.env.MODE / PROD / DEV`
- HTML 中 `%VITE_APP_TITLE%`

## 4. 代理与后端联调

```ts
proxy: {
  '/api': {
    target: 'https://test.example.com',
    changeOrigin: true,
    secure: false,             // 自签证书
    rewrite: path => path.replace(/^\/api/, ''),
  },
  '/ws': { target: 'ws://localhost:8080', ws: true },
}
```

规范：前端统一 `/api` 前缀开发，生产由网关/nginx 承接，代码零改动切换环境。

## 5. 代码分割策略

| 手段 | 场景 |
|---|---|
| 路由懒加载 | 页面级自动分包 |
| defineAsyncComponent | 重组件 |
| manualChunks | vendor 大库单独包（利于长缓存） |
| import.meta.glob | 批量注册/模块化动态 |

manualChunks 注意：**不要按 entry 拆太碎**——首屏请求变多反而慢；核心原则「变与不变分离」（业务代码常变，vendor 稳定利于 CDN 缓存）。

## 6. 预构建（optimizeDeps）

```ts
optimizeDeps: {
  include: ['lodash-es', 'axios'],     // 动态导入的依赖预构建防二次加载
  exclude: ['some-cjs-lib'],
}
```

症状识别：页面初次打开某模块时报「Failed to resolve import」或刷新后短暂全量重载 → include 缺失。

## 7. 构建产物分析

```bash
npm i -D rollup-plugin-visualizer
```

```ts
import { visualizer } from 'rollup-plugin-visualizer'
plugins: [vue(), visualizer({ open: true, gzipSize: true })]
```

优化常规路径：echarts/lodash 按需引入 → 大图片转 CDN/压缩 → moment 换 dayjs → 检查重复依赖。

## 8. PWA 与多页应用

```ts
// 多页
build.rollupOptions.input = { main: 'index.html', admin: 'admin.html' }
```

## 9. 常用插件清单

| 插件 | 用途 |
|---|---|
| @vitejs/plugin-vue | SFC |
| @vitejs/plugin-vue-jsx | JSX |
| unplugin-auto-import | API 自动导入（团队统一使用才开，显式 import 优先） |
| unplugin-vue-components | 组件库自动按需引入 |
| vite-plugin-svg-icons | 雪碧图 |
| vite-plugin-compression | gzip/br 预压缩 |
| rollup-plugin-visualizer | 产物分析 |
| vite-plugin-vue-devtools | 内嵌调试 |

auto-import 的取舍：开发体验 + 但隐式依赖增加理解成本，中大型团队建议显式 import（配合 ESLint no-undef 检查）。

## 常见坑

1. **dev 正常 build 坏**：CJS/ESM 互操作问题、运行时代码依赖 dev 特有行为（如 `import.meta.glob` 写法不标准）。
2. **node 内置模块在浏览器代码 import**：build 报错 polyfill 缺失；确认代码分层。
3. **代理 rewrite 前缀和后端路径不匹配**：接口 404，先 curl target 验证。
4. **manualChunks 循环依赖告警**：vendor 拆太细导致；合并回粗粒度。
5. **scss additionalData 注入整段样式**：每文件重复样式；只注入变量/mixin。
6. **sourcemap 全量上传公网**：源码泄漏；私有化存储或按需开启。
7. **低版本浏览器报语法错**：build.target 配置与 @vitejs/plugin-legacy 取舍确认。

## Checklist

- [ ] alias、scss 注入、proxy 配置齐备
- [ ] vendor 分包与缓存策略明确
- [ ] 生产禁 sourcemap 或私有存储
- [ ] analyze 过一次产物且无意外大包
- [ ] legacy 需求（低版本浏览器）已决策
- [ ] 环境变量 VITE_ 前缀规范且不入库密钥
