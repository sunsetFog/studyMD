# Webpack 知识点总结

## 1. 简介

Webpack 是一个模块打包工具（module bundler），把项目中的各种资源（JS、CSS、图片、字体等）当作模块处理，打包成浏览器可用的静态文件。

核心作用：
- 模块化打包
- 处理各种资源（Loader）
- 代码优化、压缩、分割
- 开发服务器与热更新

## 2. 核心概念

Webpack 有 5 个核心概念：entry、output、loader、plugin、mode。

```
entry    入口：从哪个文件开始打包
output   输出：打包结果放到哪
loader   加载器：处理非 JS 文件（CSS、图片等）
plugin   插件：扩展功能（压缩、生成 HTML 等）
mode     模式：development / production
```

## 3. 安装

```
# 初始化项目
npm init -y

# 安装 webpack 和 CLI
npm install webpack webpack-cli --save-dev
```

## 4. 入口 entry

指定打包的起点。

```
// webpack.config.js
module.exports = {
    entry: './src/index.js',        // 单入口
}
```

```
// 多入口
module.exports = {
    entry: {
        main: './src/index.js',
        admin: './src/admin.js'
    }
}
```

## 5. 输出 output

指定打包结果的存放位置和命名。

```
const path = require('path')

module.exports = {
    entry: './src/index.js',
    output: {
        path: path.resolve(__dirname, 'dist'),  // 输出目录（绝对路径）
        filename: 'bundle.js',                  // 输出文件名
        clean: true                             // 每次构建前清空 dist
    }
}
```

## 6. 模式 mode

```
module.exports = {
    mode: 'development',   // 开发模式（不压缩，可读性好）
    // mode: 'production', // 生产模式（压缩优化）
}
```

## 7. Loader 概念

Loader 处理非 JS 文件，把各种资源转换成 webpack 能识别的模块。

```
// loader 写法
module.exports = {
    module: {
        rules: [
            {
                test: /\.css$/,   // 匹配文件
                use: ['style-loader', 'css-loader']  // 从右到左执行
            }
        ]
    }
}
```

## 8. 常用 Loader

```
css-loader       解析 CSS 中的 import 和 url()
style-loader     把 CSS 注入到 <style> 标签
sass-loader      编译 SCSS/Sass
babel-loader     转译 ES6+ 语法
file-loader      处理文件（图片、字体）
url-loader       小文件转 base64，大文件走 file-loader
```

## 9. 处理 CSS

```
// 安装
npm install style-loader css-loader --save-dev

module.exports = {
    module: {
        rules: [
            {
                test: /\.css$/,
                use: ['style-loader', 'css-loader']  // 从右往左：先 css-loader 再 style-loader
            }
        ]
    }
}
```

## 10. 处理图片（asset module）

Webpack 5 用 asset module 替代 file-loader/url-loader。

```
module.exports = {
    module: {
        rules: [
            {
                test: /\.(png|jpg|gif|svg)$/,
                type: 'asset',                  // 自动选择 resource 或 inline
                parser: {
                    dataUrlCondition: {
                        maxSize: 8 * 1024       // 小于 8KB 转 base64
                    }
                }
            }
        ]
    }
}
```

```
# asset 类型
asset/resource  输出文件，返回 URL（原 file-loader）
asset/inline    转 base64 内联（原 url-loader）
asset/source    导出源码
asset           自动在 resource 和 inline 间选择
```

## 11. 处理 ES6（babel-loader）

```
npm install babel-loader @babel/core @babel/preset-env --save-dev

module.exports = {
    module: {
        rules: [
            {
                test: /\.js$/,
                exclude: /node_modules/,   // 排除 node_modules
                use: {
                    loader: 'babel-loader',
                    options: {
                        presets: ['@babel/preset-env']
                    }
                }
            }
        ]
    }
}
```

## 12. Plugin 概念

Plugin 在打包的各个生命周期阶段执行，扩展 webpack 功能。

```
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
    plugins: [
        new HtmlWebpackPlugin()   // 自动生成 HTML 并引入打包文件
    ]
}
```

## 13. 常用 Plugin

```
HtmlWebpackPlugin      生成 HTML，自动引入打包后的 JS
CleanWebpackPlugin     清空输出目录
MiniCssExtractPlugin   把 CSS 提取成独立文件
CssMinimizerPlugin     压缩 CSS
TerserPlugin           压缩 JS
DefinePlugin           定义全局常量
```

```
// 完整示例
const HtmlWebpackPlugin = require('html-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')

module.exports = {
    plugins: [
        new HtmlWebpackPlugin({
            template: './src/index.html',   // 使用模板
            title: 'My App'
        }),
        new MiniCssExtractPlugin({
            filename: 'css/[name].[contenthash].css'
        })
    ]
}
```

## 14. devServer 开发服务器

```
module.exports = {
    devServer: {
        port: 3000,             // 端口
        open: true,             // 自动打开浏览器
        hot: true,              // 开启热更新
        compress: true,         // 开启 gzip
        historyApiFallback: true,  // SPA 路由刷新 404 解决
        proxy: {                // 代理接口
            '/api': 'http://localhost:8080'
        }
    }
}
```

```
# 启动开发服务器
npx webpack serve
```

## 15. HMR 热更新

修改代码后浏览器局部更新，不刷新整页。

```
module.exports = {
    devServer: {
        hot: true
    }
}
```

```
// 模块内接收热更新
if (module.hot) {
    module.hot.accept()
}
```

## 16. Source Map

方便调试，把打包后代码映射回源码。

```
module.exports = {
    devtool: 'source-map'       // 完整映射（生产）
    // devtool: 'eval-cheap-module-source-map'  // 开发推荐，速度快
}
```

```
# 常用 devtool
eval              开发，最快
source-map        生产，最完整
inline-source-map 内联映射
```

## 17. 代码分割 Code Splitting

把代码拆成多个 chunk，按需加载。

```
module.exports = {
    optimization: {
        splitChunks: {
            chunks: 'all'       // 提取公共依赖
        }
    }
}
```

## 18. 懒加载（动态导入）

用 `import()` 实现按需加载。

```
// 点击时才加载模块
button.addEventListener('click', () => {
    import('./heavyModule').then(module => {
        module.default()
    })
})
```

## 19. 缓存 contenthash

文件名带 hash，内容不变则文件名不变，利于浏览器缓存。

```
module.exports = {
    output: {
        filename: 'js/[name].[contenthash:8].js'   // 内容哈希
    }
}
```

```
# hash 类型
hash           整个项目哈希（任何改动都变）
chunkhash      单个 chunk 哈希
contenthash    内容哈希（内容不变则不变，最常用）
```

## 20. Tree Shaking

去掉未使用的代码（摇树优化），生产模式默认开启。

```
// math.js
export const add = (a, b) => a + b
export const mul = (a, b) => a * b   // 未使用，会被 tree shake 掉

// index.js
import { add } from './math'
add(1, 2)   // 只用了 add
```

```
# Tree Shaking 前提
1. 使用 ES Module（import/export）
2. 生产模式（mode: production）
```

## 21. resolve 配置

```
const path = require('path')

module.exports = {
    resolve: {
        alias: {                    // 路径别名
            '@': path.resolve(__dirname, 'src')
        },
        extensions: ['.js', '.jsx', '.json']   // 省略扩展名
    }
}
```

```
# 使用别名
import utils from '@/utils'   // 等价于 ./src/utils
```

## 22. 多页面配置

```
module.exports = {
    entry: {
        index: './src/index.js',
        about: './src/about.js'
    },
    output: {
        filename: '[name].bundle.js'   // 生成 index.bundle.js 和 about.bundle.js
    },
    plugins: [
        new HtmlWebpackPlugin({
            filename: 'index.html',
            chunks: ['index']          // 只引入 index chunk
        }),
        new HtmlWebpackPlugin({
            filename: 'about.html',
            chunks: ['about']
        })
    ]
}
```

## 23. 环境变量

用 DefinePlugin 注入环境变量，或通过 mode 判断。

```
const webpack = require('webpack')

module.exports = {
    plugins: [
        new webpack.DefinePlugin({
            'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV)
        })
    ]
}
```

```
# 命令行传环境变量
NODE_ENV=production npx webpack
```

## 24. 生产环境优化

```
const TerserPlugin = require('terser-webpack-plugin')
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin')

module.exports = {
    mode: 'production',
    optimization: {
        minimizer: [
            new TerserPlugin(),          // 压缩 JS
            new CssMinimizerPlugin()     // 压缩 CSS
        ],
        splitChunks: {
            chunks: 'all'
        }
    }
}
```

## 25. 配置分离

把配置拆成开发、生产、公共三份。

```
webpack.common.js   公共配置
webpack.dev.js      开发配置
webpack.prod.js     生产配置
```

```
# 用 webpack-merge 合并
npm install webpack-merge --save-dev

// webpack.dev.js
const { merge } = require('webpack-merge')
const common = require('./webpack.common.js')

module.exports = merge(common, {
    mode: 'development',
    devtool: 'eval-cheap-module-source-map',
    devServer: { hot: true }
})
```

## 26. 完整配置示例

```
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')
const MiniCssExtractPlugin = require('mini-css-extract-plugin')

module.exports = {
    mode: 'production',
    entry: './src/index.js',
    output: {
        path: path.resolve(__dirname, 'dist'),
        filename: 'js/[name].[contenthash:8].js',
        clean: true
    },
    resolve: {
        alias: { '@': path.resolve(__dirname, 'src') },
        extensions: ['.js', '.jsx']
    },
    module: {
        rules: [
            {
                test: /\.js$/,
                exclude: /node_modules/,
                use: 'babel-loader'
            },
            {
                test: /\.css$/,
                use: [MiniCssExtractPlugin.loader, 'css-loader']
            },
            {
                test: /\.(png|jpg|gif)$/,
                type: 'asset',
                parser: { dataUrlCondition: { maxSize: 8192 } }
            }
        ]
    },
    plugins: [
        new HtmlWebpackPlugin({ template: './src/index.html' }),
        new MiniCssExtractPlugin({ filename: 'css/[name].[contenthash:8].css' })
    ],
    optimization: {
        splitChunks: { chunks: 'all' }
    },
    devServer: {
        port: 3000,
        hot: true
    }
}
```

## 27. 常用命令

```
# 开发构建
npx webpack --mode development

# 生产构建
npx webpack --mode production

# 监听模式（文件变化自动构建）
npx webpack --watch

# 启动开发服务器
npx webpack serve

# 指定配置文件
npx webpack --config webpack.prod.js
```

## 28. 配置速查表

| 配置项 | 作用 |
|--------|------|
| entry | 入口 |
| output | 输出 |
| module.rules | Loader 规则 |
| plugins | 插件 |
| mode | 模式 |
| devServer | 开发服务器 |
| resolve | 模块解析 |
| optimization | 优化（压缩、分割） |
| devtool | Source Map |

## 29. 与 Vite 对比

| | Webpack | Vite |
|--|---------|------|
| 构建方式 | 全量打包 | 按需编译（ESM） |
| 冷启动 | 慢（打包多） | 快 |
| 热更新 | 较快 | 极快 |
| 生态 | 成熟、插件多 | 较新、发展快 |
| 适用 | 复杂老项目 | 新项目、开发体验优先 |

## 30. 常见问题速查

```
# 问题：打包后 CSS 不生效
# 检查 loader 顺序（从右到左执行）

# 问题：图片路径 404
# 检查 output.publicPath 配置

# 问题：ES6 语法不兼容
# 检查 babel-loader 是否配置正确

# 问题：修改代码浏览器不更新
# 检查 devServer.hot 是否开启
```
