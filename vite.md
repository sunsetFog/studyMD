# Vite 知识点总结

## 1. 简介

Vite 是新一代前端构建工具，基于浏览器原生 ES Module（ESM）实现极速冷启动和热更新。开发时按需编译，生产时用 Rollup 打包。

特点：
- 极快的冷启动（无需全量打包）
- 秒级热更新（HMR）
- 开箱即用，配置简洁
- 支持 Vue、React、TS 等

## 2. 工作原理

```
开发模式：利用浏览器原生 ESM，按需编译，不打包
生产模式：用 Rollup 打包，产出优化后的静态文件
```

```
# 传统工具（Webpack）：启动时全量打包，慢
# Vite：启动时只启动 dev server，请求哪个模块才编译哪个，快
```

## 3. 创建项目

```
# 使用官方脚手架
npm create vite@latest

# 指定模板
npm create vite@latest my-app -- --template vue
npm create vite@latest my-app -- --template react
npm create vite@latest my-app -- --template react-ts
```

```
# 进入项目并安装依赖
cd my-app
npm install
```

## 4. 目录结构

```
my-app/
├── index.html          # 入口 HTML（在根目录，不在 public）
├── vite.config.js      # Vite 配置
├── package.json
├── public/             # 静态资源（原样拷贝，不处理）
└── src/                # 源码
    ├── main.js
    └── ...
```

## 5. 开发服务器

```
# 启动开发服务器
npm run dev
# 默认 http://localhost:5173
```

```
// vite.config.js 配置 dev server
export default defineConfig({
    server: {
        port: 3000,              // 端口
        open: true,              // 自动打开浏览器
        host: true,              // 允许局域网访问
        proxy: {                 // 代理
            '/api': 'http://localhost:8080'
        }
    }
})
```

## 6. 构建 build

```
# 生产构建
npm run build
# 输出到 dist 目录
```

```
// vite.config.js 构建配置
export default defineConfig({
    build: {
        outDir: 'dist',          // 输出目录
        sourcemap: true,         // 生成 source map
        minify: 'esbuild'        // 压缩方式
    }
})
```

## 7. 预览 preview

构建后本地预览生产产物。

```
# 预览 dist
npm run preview
```

## 8. vite.config.js 配置

```
import { defineConfig } from 'vite'

export default defineConfig({
    plugins: [],        // 插件
    resolve: {},        // 解析
    server: {},         // 开发服务器
    build: {},          // 构建
})
```

## 9. 路径别名 alias

```
import { defineConfig } from 'vite'
import path from 'path'

export default defineConfig({
    resolve: {
        alias: {
            '@': path.resolve(__dirname, 'src')   // @ 指向 src
        }
    }
})
```

```
# 使用
import utils from '@/utils'   // 等价于 ./src/utils
```

## 10. 处理 CSS

Vite 原生支持 CSS、SCSS、Less，无需额外配置。

```
// 直接引入 CSS
import './style.css'

// 引入 SCSS（需安装 sass）
import './style.scss'
```

```
# 安装 sass
npm install -D sass
```

## 11. CSS Modules

`.module.css` 文件自动启用 CSS Modules，样式局部作用域。

```
// Button.module.css
.button {
    color: red;
}
```

```
import styles from './Button.module.css'

// 使用
<div className={styles.button}>按钮</div>
```

## 12. 静态资源处理

```
# public 目录：原样拷贝，用 / 直接引用
<img src="/logo.png" />

# src 内资源：import 引入，会被处理（hash、压缩、小图转 base64）
import logo from './logo.png'
<img src={logo} />
```

## 13. 环境变量

环境变量放在 `.env` 文件，以 `VITE_` 开头才会暴露给前端。

```
# .env
VITE_API_URL=https://api.example.com
VITE_APP_TITLE=My App
```

```
# 使用
console.log(import.meta.env.VITE_API_URL)
```

## 14. import.meta.env 内置变量

```
import.meta.env.MODE        // 当前模式：development / production
import.meta.env.DEV         // 是否开发环境
import.meta.env.PROD        // 是否生产环境
import.meta.env.BASE_URL    // 部署基础路径
```

## 15. 多环境配置

不同环境用不同 .env 文件。

```
.env                  # 所有环境通用
.env.development      # 开发环境
.env.production       # 生产环境
.env.local            # 本地覆盖（不提交）
```

```
# 加载规则
npm run dev  → 加载 .env + .env.development
npm run build → 加载 .env + .env.production
```

## 16. 热更新 HMR

Vite 内置 HMR，修改代码即时生效，无需刷新页面。

```
# 默认开启，无需配置
# 修改 Vue/React 组件、CSS 都会热更新，保留状态
```

## 17. 插件 plugins

Vite 基于 Rollup 插件体系，用 plugins 数组配置。

```
import vue from '@vitejs/plugin-vue'

export default defineConfig({
    plugins: [vue()]
})
```

## 18. 常用官方插件

```
@vitejs/plugin-vue       Vue 3 支持
@vitejs/plugin-vue-jsx   Vue JSX 支持
@vitejs/plugin-react     React 支持（Fast Refresh）
@vitejs/plugin-legacy    兼容旧浏览器
```

```
# Vue 项目
npm install -D @vitejs/plugin-vue

import vue from '@vitejs/plugin-vue'
export default defineConfig({
    plugins: [vue()]
})
```

## 19. 自定义插件

Vite 插件是带钩子函数的对象。

```
// 自定义插件：在文件变化时打印日志
export default function myPlugin() {
    return {
        name: 'my-plugin',     // 插件名
        transform(code, id) {
            // 处理模块代码
            return code
        },
        handleHotUpdate(ctx) {
            console.log('热更新文件：', ctx.file)
        }
    }
}
```

## 20. 代码分割与懒加载

Vite 支持动态 import 自动代码分割。

```
// 动态导入，按需加载
const loadModule = async () => {
    const { default } = await import('./heavyModule')
    return default
}
```

## 21. 代理 proxy

开发时把接口请求代理到后端，解决跨域。

```
export default defineConfig({
    server: {
        proxy: {
            '/api': {
                target: 'http://localhost:8080',   // 后端地址
                changeOrigin: true,                // 修改请求源
                rewrite: (path) => path.replace(/^\/api/, '')  // 重写路径
            }
        }
    }
})
```

## 22. TypeScript 支持

Vite 原生支持 TS，无需额外配置。

```
# 创建 TS 项目
npm create vite@latest my-app -- --template vue-ts

# 直接写 .ts 文件，Vite 自动转译
```

```
# 注意：Vite 只转译不类型检查
# 类型检查用 vue-tsc / tsc 单独跑
```

## 23. 多页面配置

```
import { resolve } from 'path'

export default defineConfig({
    build: {
        rollupOptions: {
            input: {
                main: resolve(__dirname, 'index.html'),
                about: resolve(__dirname, 'about.html')
            }
        }
    }
})
```

## 24. 构建优化（分包）

把第三方库拆成独立 chunk，利于缓存。

```
export default defineConfig({
    build: {
        rollupOptions: {
            output: {
                manualChunks: {
                    vendor: ['vue', 'vue-router'],   // 手动分包
                    lodash: ['lodash']
                }
            }
        }
    }
})
```

## 25. 库模式 build.lib

把项目打包成可发布的库。

```
export default defineConfig({
    build: {
        lib: {
            entry: 'src/index.js',      // 库入口
            name: 'MyLib',              // 全局变量名
            fileName: 'my-lib',         // 输出文件名
            formats: ['es', 'umd']      // 输出格式
        }
    }
})
```

## 26. 与 Webpack 对比

| | Vite | Webpack |
|--|------|---------|
| 冷启动 | 极快（按需编译） | 慢（全量打包） |
| 热更新 | 极快 | 较快 |
| 生态 | 较新、快速发展 | 成熟、插件多 |
| 配置 | 简洁 | 复杂 |
| 生产构建 | Rollup | 自身 |
| 适用 | 新项目 | 老项目、复杂场景 |

## 27. 常用命令

```
npm run dev        # 启动开发服务器
npm run build      # 生产构建
npm run preview    # 预览构建产物
npx vite           # 直接运行 vite
npx vite build     # 直接构建
```

## 28. 配置速查表

| 配置项 | 作用 |
|--------|------|
| plugins | 插件 |
| resolve.alias | 路径别名 |
| server.port | 端口 |
| server.proxy | 代理 |
| server.open | 自动打开 |
| build.outDir | 输出目录 |
| build.minify | 压缩方式 |
| build.sourcemap | Source Map |
| css.preprocessorOptions | CSS 预处理器配置 |
| define | 定义全局常量 |

## 29. 完整配置示例

```
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'path'

export default defineConfig({
    plugins: [vue()],
    resolve: {
        alias: {
            '@': path.resolve(__dirname, 'src')
        }
    },
    server: {
        port: 3000,
        open: true,
        proxy: {
            '/api': 'http://localhost:8080'
        }
    },
    build: {
        outDir: 'dist',
        sourcemap: false,
        rollupOptions: {
            output: {
                manualChunks: {
                    vendor: ['vue', 'vue-router']
                }
            }
        }
    }
})
```

## 30. 常见问题速查

```
# 问题：修改配置不生效
# 重启 dev server（vite.config.js 变更需重启）

# 问题：环境变量读不到
# 检查是否以 VITE_ 开头

# 问题：接口 404
# 检查 server.proxy 代理配置

# 问题：打包后资源路径错误
# 配置 base 字段（部署到子路径时）
export default defineConfig({
    base: '/my-app/'    # 部署路径
})

# 问题：生产环境 console 报错
# 用 build.minify 和 sourcemap 排查
```
