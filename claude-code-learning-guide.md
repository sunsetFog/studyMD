# Claude Code 深度学习清单

> 一份从入门到精通的 Claude Code 系统学习指南，涵盖 CLI、IDE、提示词工程、实战场景等全方位内容。

---

## 📋 目录

- [一、基础入门](#一基础入门)
- [二、CLI 命令行深度掌握](#二cli-命令行深度掌握)
- [三、IDE 集成与工作流](#三ide-集成与工作流)
- [四、提示词工程（核心能力）](#四提示词工程核心能力)
- [五、代码理解与生成](#五代码理解与生成)
- [六、代码编辑与重构](#六代码编辑与重构)
- [七、调试与问题排查](#七调试与问题排查)
- [八、测试驱动开发](#八测试驱动开发)
- [九、Git 与版本控制](#九git-与版本控制)
- [十、项目架构与设计](#十项目架构与设计)
- [十一、文档生成](#十一文档生成)
- [十二、多语言与框架专项](#十二多语言与框架专项)
- [十三、安全与合规](#十三安全与合规)
- [十四、高级技巧与最佳实践](#十四高级技巧与最佳实践)
- [十五、常见实战场景](#十五常见实战场景)
- [十六、性能优化](#十六性能优化)
- [十七、CI/CD 与 DevOps](#十七cicd-与-devops)
- [十八、团队协作规范](#十八团队协作规范)
- [十九、常见陷阱与解决方案](#十九常见陷阱与解决方案)
- [二十、持续进阶路线图](#二十持续进阶路线图)

---

## 一、基础入门

### 1.1 什么是 Claude Code
- [ ] 理解 Claude Code 的定位：Anthropic 推出的 AI 编程助手
- [ ] 了解与其他 AI 编程工具的差异（vs Copilot / Cursor / Codeium）
- [ ] 阅读官方文档与 Release Notes
- [ ] 了解支持的模型版本（Claude 3.5 Sonnet / Claude 3 Opus 等）
- [ ] 理解 Token 限制与上下文窗口机制

### 1.2 安装与配置
- [ ] macOS / Linux / Windows 各平台安装
  ```bash
  # macOS
  brew install claude-code
  
  # npm 全局安装
  npm install -g @anthropic-ai/claude-code
  
  # 验证安装
  claude --version
  ```
- [ ] API Key 配置与认证
  ```bash
  export ANTHROPIC_API_KEY="your-api-key"
  # 或在 ~/.claude/config.json 中配置
  ```
- [ ] 代理设置（企业环境）
- [ ] 多账号/多项目配置管理
- [ ] VS Code / JetBrains 插件安装

### 1.3 第一次使用
- [ ] 在终端中运行 `claude` 进入交互模式
- [ ] 理解会话（Session）的概念与生命周期
- [ ] 学会 `/help` 查看内置命令
- [ ] 尝试第一个简单任务：让 Claude Code 解释某个文件
- [ ] 了解权限模型（读取/写入/执行 的确认机制）

### 1.4 核心概念
- [ ] **上下文窗口**：什么是上下文、如何管理
- [ ] **System Prompt**：系统提示词的作用
- [ ] **Tool Use**：Claude Code 可以调用哪些工具
- [ ] **MCP（Model Context Protocol）**：模型上下文协议基础
- [ ] **权限与安全边界**：什么操作需要用户确认

---

## 二、CLI 命令行深度掌握

### 2.1 基本命令
- [ ] `claude` — 启动交互式会话
- [ ] `claude "帮我..."` — 单次非交互式执行
- [ ] `claude -p "prompt"` — 指定提示词
- [ ] `claude --resume` — 恢复上次会话
- [ ] `claude --continue` — 继续最近的会话
- [ ] `claude -c` — 从 stdin 读取内容

### 2.2 文件与目录操作
- [ ] `claude --files "src/**/*.ts"` — 指定上下文文件
- [ ] `claude --add-dir ./src` — 添加目录到上下文
- [ ] `claude --no-auto-context` — 禁用自动上下文收集
- [ ] 理解 `.claudeignore` 文件（类比 .gitignore）
- [ ] 使用 glob 模式精确控制上下文范围

### 2.3 会话管理
- [ ] `claude --list-sessions` — 列出历史会话
- [ ] `claude --resume <session-id>` — 恢复指定会话
- [ ] `claude --delete-session <id>` — 删除会话
- [ ] `claude --rename-session <id> <name>` — 重命名会话
- [ ] 会话导出与分享

### 2.4 输出格式控制
- [ ] `--output-format json` — JSON 格式输出
- [ ] `--output-format stream-json` — 流式 JSON 输出
- [ ] `--no-color` — 禁用颜色输出
- [ ] `--verbose` — 详细日志模式
- [ ] 管道与重定向组合使用

### 2.5 配置管理
- [ ] `~/.claude/config.json` 配置文件结构
- [ ] 项目级 `.claude/settings.json` 配置
- [ ] 环境变量大全
  - `ANTHROPIC_API_KEY`
  - `ANTHROPIC_BASE_URL`
  - `CLAUDE_CODE_MAX_TOKENS`
  - `CLAUDE_CODE_MODEL`
- [ ] `claude config` 交互式配置向导

### 2.6 高级 CLI 技巧
- [ ] 结合 `fzf` / `peco` 进行交互式文件选择
- [ ] 使用 Shell 别名提升效率
  ```bash
  alias cc="claude"
  alias cce="claude --editor"
  alias ccr="claude --resume"
  ```
- [ ] 管道组合：`cat error.log | claude "分析这些错误日志"`
- [ ] 定时任务与批处理脚本集成
- [ ] 与 `tmux` / `screen` 配合长时间会话

---

## 三、IDE 集成与工作流

### 3.1 VS Code 深度集成
- [ ] Claude Code VS Code 插件安装与配置
- [ ] 侧边栏聊天面板使用
- [ ] 内联代码建议（Inline Suggestions）
- [ ] 选中代码 → 右键菜单操作
  - 解释代码
  - 优化代码
  - 添加注释
  - 生成测试
  - 查找 Bug
- [ ] 快捷键绑定自定义
- [ ] 与 Copilot 共存策略

### 3.2 JetBrains 集成
- [ ] IntelliJ / PyCharm / WebStorm 插件安装
- [ ] 右键上下文菜单使用
- [ ] 代码审查（Code Review）辅助
- [ ] 重构菜单集成

### 3.3 终端编辑器集成
- [ ] Vim / Neovim 集成方案
- [ ] Emacs 集成方案
- [ ] Helix 编辑器集成

### 3.4 工作流最佳实践
- [ ] **Task-Driven Workflow**：以任务为导向使用 Claude Code
- [ ] **Review-First Workflow**：AI 生成 → 人工审查 → 确认合并
- [ ] **Pair Programming Workflow**：AI 作为结对编程伙伴
- [ ] **Exploration Workflow**：用 Claude Code 探索陌生代码库
- [ ] **Refinement Workflow**：迭代优化提示词获取最佳结果

---

## 四、提示词工程（核心能力）

### 4.1 提示词基础
- [ ] **清晰具体**：模糊 vs 精确的描述对比
  ```
  ❌ "优化这段代码"
  ✅ "将这段 React 组件的 useState 重构为 useReducer，
      并提取自定义 hook，处理 loading/error/success 三种状态"
  ```
- [ ] **提供上下文**：相关文件、依赖、业务逻辑
- [ ] **指定输出格式**：期望的代码风格、命名规范
- [ ] **设定约束条件**：性能要求、兼容性要求

### 4.2 角色设定技巧
- [ ] 设定专业角色提升输出质量
  ```
  "你是一位拥有 15 年经验的 Rust 系统编程专家，
   专注于高性能网络服务开发。请审查以下代码..."
  ```
- [ ] 多角色协作模拟
- [ ] 角色切换与组合

### 4.3 结构化提示词模板

#### 通用任务模板
```
【任务】：[一句话描述]
【背景】：[项目背景、业务场景]
【约束】：[技术栈限制、性能要求、兼容性]
【输入】：[相关代码/文件]
【期望输出】：[格式、风格、详细程度]
【参考示例】：[如果有的参考实现]
```

#### 代码审查模板
```
【审查目标】：[文件名/功能模块]
【关注点】：[安全性、性能、可读性、边界条件]
【代码规范】：[遵循的规范，如 Airbnb Style Guide]
【审查深度】：[简要审查 / 逐行分析 / 架构级审查]
```

#### Bug 修复模板
```
【Bug 描述】：[现象、复现步骤]
【影响范围】：[影响的功能/用户]
【相关日志】：[错误日志/堆栈信息]
【环境信息】：[OS、版本、依赖]
【尝试过的方案】：[已尝试但未解决的方案]
```

### 4.4 进阶提示词策略
- [ ] **Chain of Thought（思维链）**：让 AI 先分析再行动
  ```
  "请先分析这个问题的根因，列出可能的解决方案及优缺点，
   然后推荐最优方案并实现"
  ```
- [ ] **Few-Shot Prompting**：提供 2-3 个示例引导输出风格
- [ ] **Self-Consistency**：多次生成取最佳结果
- [ ] **Step-by-Step 分解**：复杂任务拆解为子步骤
  ```
  "请按以下步骤操作：
   1. 先阅读 src/utils/ 下所有文件
   2. 识别重复代码模式
   3. 提取公共函数到 shared.ts
   4. 更新所有引用"
  ```
- [ ] **Negative Prompting**：明确说不要什么
- [ ] **Iterative Refinement**：多轮迭代优化策略

### 4.5 提示词管理与复用
- [ ] 建立个人提示词库（Prompt Library）
- [ ] 按场景分类：
  - 代码生成类
  - 代码审查类
  - 调试类
  - 重构类
  - 文档类
  - 测试类
  - 架构设计类
- [ ] 使用 Claude Code 的 `/prompt` 命令管理
- [ ] 团队共享提示词库

---

## 五、代码理解与生成

### 5.1 代码库理解
- [ ] 让 Claude Code 分析项目结构
  ```
  "请分析这个项目的目录结构，解释各模块职责和依赖关系"
  ```
- [ ] 理解陌生代码库的策略
  1. 先让 AI 画出架构图
  2. 逐模块深入理解
  3. 追踪关键数据流
  4. 识别核心入口与出口
- [ ] 快速上手开源项目的流程

### 5.2 代码生成
- [ ] **CRUD 生成**：常见的增删改查代码
- [ ] **API 端点生成**：RESTful / GraphQL API
- [ ] **数据模型生成**：ORM 模型 / 类型定义
- [ ] **UI 组件生成**：React / Vue / Svelte 组件
- [ ] **工具函数生成**：通用工具库代码
- [ ] 要求遵循特定设计模式
  ```
  "用 Repository 模式为这个 User 实体生成完整的数据库操作层"
  ```

### 5.3 代码翻译（Language Translation）
- [ ] TypeScript → Python
- [ ] JavaScript → TypeScript（添加类型）
- [ ] Python 2 → Python 3
- [ ] Class Component → Functional Component（React）
- [ ] Options API → Composition API（Vue）
- [ ] 跨框架翻译的注意事项

### 5.4 样板代码（Boilerplate）消除
- [ ] 识别项目中的样板代码模式
- [ ] 让 AI 生成 CLI 工具自动生成样板代码
- [ ] 使用 Claude Code 创建项目脚手架

---

## 六、代码编辑与重构

### 6.1 精确编辑
- [ ] 指定行号范围的精确修改
- [ ] 函数级别的替换
- [ ] 多文件同步修改
- [ ] 使用 diff 格式输出变更

### 6.2 重构技巧

#### 函数级重构
- [ ] 提取函数（Extract Function）
- [ ] 内联函数（Inline Function）
- [ ] 重命名（Rename）
- [ ] 参数对象化（Introduce Parameter Object）

#### 类级重构
- [ ] 提取类（Extract Class）
- [ ] 提取接口（Extract Interface）
- [ ] 用组合替代继承
- [ ] 将 God Class 拆分为多个职责单一的类

#### 条件逻辑重构
- [ ] 分解条件表达式
- [ ] 用多态替换条件
- [ ] 引入 Null Object 模式
- [ ] 用策略模式替换 switch

#### 架构级重构
- [ ] 单体拆分为微服务的设计建议
- [ ] 分层架构梳理
- [ ] 依赖反转实现
- [ ] 模块解耦策略

### 6.3 代码风格统一
- [ ] 配置 ESLint/Prettier 规则并让 AI 遵循
- [ ] 全项目统一命名规范
- [ ] 统一导入顺序
- [ ] 统一错误处理模式

---

## 七、调试与问题排查

### 7.1 错误分析
- [ ] 粘贴错误日志让 AI 分析根因
  ```bash
  # 直接管道传入
  npm run build 2>&1 | claude "分析构建错误并给出修复方案"
  ```
- [ ] 堆栈追踪解读
- [ ] 运行时错误定位
- [ ] 内存泄漏分析
- [ ] 竞态条件（Race Condition）识别

### 7.2 性能问题诊断
- [ ] N+1 查询问题识别
- [ ] 不必要的重渲染分析（React）
- [ ] 死循环检测
- [ ] 资源泄漏定位

### 7.3 日志分析
- [ ] 让 Claude Code 分析大量日志
- [ ] 提取关键错误模式
- [ ] 生成日志分析报告

### 7.4 调试辅助
- [ ] 添加战略性日志
- [ ] 生成条件断点建议
- [ ] 模拟边界条件测试

---

## 八、测试驱动开发

### 8.1 单元测试
- [ ] Jest / Vitest / Pytest 测试生成
- [ ] 边界条件测试用例生成
- [ ] Mock / Stub / Spy 的正确使用
- [ ] 测试覆盖率提升建议
- [ ] 从函数签名自动生成测试骨架

### 8.2 集成测试
- [ ] API 端点测试
- [ ] 数据库交互测试
- [ ] 消息队列测试
- [ ] 第三方服务 Mock

### 8.3 端到端测试
- [ ] Playwright / Cypress 测试脚本生成
- [ ] 用户流程测试设计
- [ ] 视觉回归测试（Visual Regression）

### 8.4 TDD 工作流
- [ ] 让 AI 先写测试，再写实现
- [ ] 红-绿-重构循环中 AI 的角色
- [ ] 测试用例评审

---

## 九、Git 与版本控制

### 9.1 Commit 管理
- [ ] 生成规范的 Commit Message
  ```
  "根据这些改动生成 Conventional Commits 格式的提交信息"
  ```
- [ ] 分析 diff 生成 PR 描述
- [ ] 自动生成 Changelog

### 9.2 代码审查
- [ ] PR Review 辅助
  ```bash
  git diff main | claude "审查这些改动，关注安全性、性能和可维护性"
  ```
- [ ] 安全检查清单
- [ ] 最佳实践对比

### 9.3 冲突解决
- [ ] 合并冲突分析
- [ ] 智能冲突解决方案建议

### 9.4 Git 历史分析
- [ ] 分析代码变更趋势
- [ ] 识别技术债务积累模式
- [ ] Blame 信息解读辅助

---

## 十、项目架构与设计

### 10.1 架构设计
- [ ] 新项目技术选型讨论
- [ ] 系统架构图生成（Mermaid/PlantUML）
- [ ] 数据流设计
- [ ] API 设计（RESTful / GraphQL / gRPC）

### 10.2 设计模式应用
- [ ] 单例 / 工厂 / 建造者 / 原型
- [ ] 适配器 / 桥接 / 装饰器 / 代理
- [ ] 观察者 / 策略 / 模板方法 / 命令
- [ ] 在现有代码中识别并优化设计模式使用

### 10.3 数据库设计
- [ ] SQL DDL 生成
- [ ] 索引优化建议
- [ ] 数据迁移脚本（Migration）生成
- [ ] ER 图描述与生成

### 10.4 系统设计面试模拟
- [ ] 设计 URL 短链接服务
- [ ] 设计聊天系统
- [ ] 设计限流器
- [ ] 设计分布式 ID 生成器
- [ ] 让 AI 扮演面试官追问

---

## 十一、文档生成

### 11.1 代码文档
- [ ] JSDoc / Docstring 自动生成
- [ ] README 生成与更新
- [ ] API 文档生成（OpenAPI/Swagger）
- [ ] 架构决策记录（ADR）生成

### 11.2 项目文档
- [ ] CONTRIBUTING.md 生成
- [ ] CHANGELOG.md 自动维护
- [ ] 项目 Wiki 生成
- [ ] 新人 Onboarding 文档

### 11.3 技术文档
- [ ] 技术方案文档
- [ ] 故障复盘报告（Postmortem）
- [ ] 性能测试报告

---

## 十二、多语言与框架专项

### 12.1 TypeScript / JavaScript
- [ ] React 组件开发与优化
- [ ] Next.js 全栈应用开发
- [ ] Vue 3 Composition API
- [ ] Node.js 后端服务（Express / Fastify / NestJS）
- [ ] 类型体操（高级 TypeScript 类型）
- [ ] Zustand / Redux / Jotai 状态管理

### 12.2 Python
- [ ] FastAPI / Django / Flask API 开发
- [ ] Pandas 数据处理脚本
- [ ] 异步编程（asyncio）
- [ ] 类型提示（Type Hints）补全

### 12.3 Go
- [ ] 标准库最佳实践
- [ ] 并发模式（Goroutine / Channel）
- [ ] 错误处理范式
- [ ] 项目布局规范

### 12.4 Rust
- [ ] 所有权（Ownership）与借用（Borrowing）理解
- [ ] 错误处理（Result / Option）
- [ ] 异步编程（Tokio）
- [ ] unsafe 代码审查

### 12.5 Java / Kotlin
- [ ] Spring Boot 应用开发
- [ ] JPA / MyBatis 数据层
- [ ] 多线程与并发
- [ ] Maven / Gradle 配置

### 12.6 其他语言
- [ ] Swift / SwiftUI
- [ ] Kotlin Multiplatform
- [ ] C# / .NET
- [ ] Ruby on Rails
- [ ] PHP / Laravel

### 12.7 基础设施即代码
- [ ] Terraform 配置生成
- [ ] Dockerfile 编写与优化
- [ ] Kubernetes YAML 生成
- [ ] GitHub Actions / GitLab CI 配置

---

## 十三、安全与合规

### 13.1 安全审查
- [ ] SQL 注入检测与修复
- [ ] XSS 漏洞识别
- [ ] CSRF 防护检查
- [ ] 敏感信息硬编码检测
- [ ] 不安全的依赖版本识别

### 13.2 认证与授权
- [ ] JWT 实现审查
- [ ] OAuth 2.0 流程验证
- [ ] RBAC / ABAC 权限模型审查
- [ ] API Key 管理最佳实践

### 13.3 数据安全
- [ ] 加密算法使用审查
- [ ] 数据脱敏实现
- [ ] 日志中的敏感信息排查
- [ ] GDPR / 个人信息保护合规检查

---

## 十四、高级技巧与最佳实践

### 14.1 上下文管理
- [ ] 精准控制上下文窗口（不过载也不缺失）
- [ ] 分段处理大文件策略
- [ ] 关键文件优先加载
- [ ] `.claudeignore` 精细配置

### 14.2 多轮对话策略
- [ ] 渐进式细化（从粗略到精确）
- [ ] 分步骤确认（每步完成后确认再继续）
- [ ] 分支探索与合并（探索多个方案后选择最优）
- [ ] 回退策略（不满意时回退重来）

### 14.3 MCP（Model Context Protocol）
- [ ] MCP 概念与架构理解
- [ ] 配置 MCP Server
- [ ] 连接数据库的 MCP 使用
- [ ] 连接 API 的 MCP 使用
- [ ] 自定义 MCP Server 开发

### 14.4 自定义指令（Custom Instructions）
- [ ] 项目级自定义指令配置
- [ ] 用户级偏好设置
- [ ] 为不同项目设定不同的 AI 行为

### 14.5 Hooks 与自动化
- [ ] Pre-generation Hooks
- [ ] Post-generation Hooks
- [ ] 与 Lint/Format 工具集成
- [ ] 与 CI/CD 流水线集成

### 14.6 成本控制
- [ ] Token 消耗估算
- [ ] 减少不必要的上下文
- [ ] 缓存策略利用
- [ ] 选择合适的模型（Opus vs Sonnet vs Haiku）

---

## 十五、常见实战场景

### 15.1 场景一：接手遗留代码
1. [ ] 让 AI 画出项目模块依赖图
2. [ ] 识别核心业务流程
3. [ ] 标记技术债务
4. [ ] 制定渐进式重构计划
5. [ ] 逐个模块添加测试

### 15.2 场景二：从零搭建项目
1. [ ] 技术选型与架构讨论
2. [ ] 项目脚手架生成
3. [ ] 核心模块实现
4. [ ] CI/CD 配置
5. [ ] 部署配置

### 15.3 场景三：性能优化
1. [ ] 让 AI 识别性能瓶颈
2. [ ] 生成 Profiling 代码
3. [ ] 优化方案 `代码:` 化
4. [ ] 基准测试（Benchmark）代码生成
5. [ ] 优化前后对比分析

### 15.4 场景四：大型重构
1. [ ] 制定重构策略文档
2. [ ] 提取共享模块
3. [ ] 逐步替换旧实现（Strangler Fig Pattern）
4. [ ] 兼容性保证
5. [ ] 验证清单

### 15.5 场景五：Bug 紧急修复
1. [ ] 错误上下文快速提供
2. [ ] 根因分析
3. [ ] 修复方案生成
4. [ ] 回归测试生成
5. [ ] 影响范围评估

### 15.6 场景六：代码迁移
1. [ ] 框架版本升级（如 React 17 → 18）
2. [ ] JavaScript → TypeScript 迁移
3. [ ] REST → GraphQL 迁移
4. [ ] 单体 → 微服务拆分

### 15.7 场景七：学习新技术
1. [ ] 让 AI 用你熟悉的语言解释新概念
2. [ ] 生成交互式学习代码
3. [ ] 对比你已掌握的技术进行类比教学
4. [ ] 生成练习题与项目实战

---

## 十六、性能优化

### 16.1 前端性能
- [ ] 大列表虚拟滚动实现
- [ ] 图片懒加载
- [ ] 代码分割（Code Splitting）
- [ ] Bundle 分析优化
- [ ] 渲染性能优化（React.memo / useMemo / useCallback）

### 16.2 后端性能
- [ ] 数据库查询优化（索引 / 查询计划分析）
- [ ] 缓存策略设计（Redis / Memcached）
- [ ] 连接池配置优化
- [ ] 异步处理与消息队列

### 16.3 算法优化
- [ ] 时间复杂度分析与优化
- [ ] 空间换时间策略
- [ ] 数据结构选择建议

---

## 十七、CI/CD 与 DevOps

### 17.1 CI/CD 配置
- [ ] GitHub Actions Workflow 生成
- [ ] GitLab CI Pipeline 配置
- [ ] Jenkins Pipeline 脚本
- [ ] Docker 多阶段构建优化

### 17.2 部署相关
- [ ] Docker Compose 编排
- [ ] Kubernetes 资源清单
- [ ] Helm Chart 编写
- [ ] 云服务配置（AWS / GCP / Azure）

### 17.3 监控与告警
- [ ] 健康检查端点实现
- [ ] 结构化日志最佳实践
- [ ] Metrics 埋点建议
- [ ] 告警规则设计

---

## 十八、团队协作规范

### 18.1 团队使用规范
- [ ] AI 生成代码的审查标准
- [ ] 什么场景下应该/不应该使用 AI
- [ ] 代码归属与版权说明
- [ ] AI 工具使用培训计划

### 18.2 代码质量保障
- [ ] AI 生成代码的质量门禁
- [ ] 必须通过的检查项
- [ ] 人工审查要点清单

### 18.3 知识沉淀
- [ ] 有效的提示词归档
- [ ] AI 辅助的最佳实践文档
- [ ] 定期回顾与改进

---

## 十九、常见陷阱与解决方案

### 19.1 质量陷阱
| 陷阱 | 描述 | 解决方案 |
|------|------|----------|
| 幻觉代码 | AI 生成不存在的 API | 始终验证，先查文档 |
| 过度工程 | AI 倾向生成过于复杂的方案 | 明确要求"保持简单" |
| 安全盲区 | 忽略输入校验、权限检查 | 安全审查清单 |
| 不一致性 | 多文件修改时风格不统一 | 提前明确代码规范 |
| 上下文断裂 | 长对话中 AI "忘记"之前约定 | 关键约定在提示词中重复 |

### 19.2 效率陷阱
- [ ] 不要一次要求太多（超出上下文窗口）
- [ ] 不要跳过审查直接使用 AI 代码
- [ ] 不要在完全不了解的领域盲目信任 AI
- [ ] 不要忽略错误信息中的线索

### 19.3 依赖陷阱
- [ ] AI 可能使用过时的 API 版本
- [ ] 依赖包版本锁定建议
- [ ] 定期让 AI 检查依赖更新

---

## 二十、持续进阶路线图

### 第一阶段：入门（第 1-2 周）
- [ ] 完成基础安装配置
- [ ] 每天用 Claude Code 完成一个小任务
- [ ] 积累 10 个常用提示词模板
- [ ] 阅读官方文档全部内容

### 第二阶段：熟练（第 3-4 周）
- [ ] 掌握 CLI 全部常用命令
- [ ] 建立个人提示词库
- [ ] 熟练使用 IDE 集成
- [ ] 能够用 AI 完成中等复杂度功能开发

### 第三阶段：精通（第 5-8 周）
- [ ] 掌握 MCP 协议及自定义开发
- [ ] 能够设计复杂的多轮对话策略
- [ ] 形成个人最佳实践体系
- [ ] 开始指导团队成员使用

### 第四阶段：专家（持续）
- [ ] 自定义 Hooks 与自动化工作流
- [ ] 为特定领域训练/调优使用方式
- [ ] 分享经验（博客/演讲/内部分享）
- [ ] 参与社区建设

---

## 📚 推荐资源

### 官方资源
- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook)
- [MCP 官方仓库](https://github.com/modelcontextprotocol)

### 社区资源
- [r/ClaudeAI](https://reddit.com/r/ClaudeAI) - Reddit 社区
- [Anthropic Discord](https://discord.gg/anthropic) - 官方 Discord
- [Claude Code GitHub Issues](https://github.com/anthropics/claude-code)

### 学习建议
- **每天至少使用一次 Claude Code**，形成肌肉记忆
- **每周复盘**：哪些提示词效果好，哪些需要改进
- **保持更新**：Claude Code 更新频繁，关注 Release Notes
- **多重验证**：AI 生成的代码，永远要经过测试和审查

---

> **最后提醒**：Claude Code 是强大的助手，但它不能替代你的判断力、领域知识和工程经验。最好的工程师是那些知道如何与 AI 高效协作的人——既懂得借力 AI，也始终保持批判性思维。

---

*最后更新：2026年7月17日*
