# 前端 AI Coding 面试题清单

> 覆盖 AI 辅助编程、Prompt Engineering、上下文工程、MCP 协议、Vibe Coding、AI 代码审查、Agent 开发等核心考点。每题包含详细原理、代码示例和面试要点。

---

## 目录

1. [一、AI Coding 入门与工具生态](#一ai-coding-入门与工具生态)
2. [二、Prompt Engineering for Coding](#二prompt-engineering-for-coding)
3. [三、AI 代码生成策略与最佳实践](#三ai-代码生成策略与最佳实践)
4. [四、上下文工程 (Context Engineering)](#四上下文工程-context-engineering)
5. [五、AI 辅助代码审查与质量保障](#五ai-辅助代码审查与质量保障)
6. [六、AI 在测试中的应用](#六ai-在测试中的应用)
7. [七、AI 驱动的代码重构](#七ai-驱动的代码重构)
8. [八、MCP 协议与工具集成](#八mcp-协议与工具集成)
9. [九、前端 AI Coding 实战](#九前端-ai-coding-实战)
10. [十、Vibe Coding 与 Agent 模式](#十vibe-coding-与-agent-模式)
11. [十一、安全、隐私与合规](#十一安全隐私与合规)
12. [十二、团队协作与工程化](#十二团队协作与工程化)

---

## 一、AI Coding 入门与工具生态

### 1. 什么是 AI Coding？它与传统开发模式有何不同？

**AI Coding（AI 辅助编程）** 是指利用人工智能技术（主要是大语言模型 LLM）辅助软件开发的各个环节，包括代码生成、补全、审查、测试、重构、文档编写等。

**与传统开发模式的核心区别：**

| 维度 | 传统开发 | AI Coding |
|------|---------|-----------|
| 代码编写方式 | 逐行手写 | 自然语言描述 → AI 生成 + 人工审查 |
| 信息检索 | 搜索引擎/文档 | AI 直接理解上下文并给出答案 |
| 代码补全 | IDE 基于语法规则 | AI 理解语义，生成完整函数/模块 |
| 代码审查 | 完全人工 | AI 先审 + 人工复核 |
| 测试编写 | 手动编写用例 | AI 根据代码逻辑自动生成 |
| 文档维护 | 手动撰写 | AI 一键生成并保持同步 |
| 学习曲线 | 新手需要大量搜索 | AI 降低知识门槛，即时解答 |

**典型工作流变化：**

```
传统模式：思考 → 手写代码 → 搜索文档 → 调试 → 提交
AI模式：  自然语言描述 → AI生成 → 人工审查修正 → 调试 → 提交
```

**面试要点：** 强调 AI Coding 不是"AI 取代开发者"，而是"开发者 + AI"的协作模式，人的决策、审查和架构设计能力更加重要。

---

### 2. 列举主流 AI Coding 工具，并分析各自的适用场景。

**主流 AI Coding 工具对比：**

| 工具 | 类型 | 核心能力 | 适用场景 |
|------|------|---------|---------|
| **GitHub Copilot** | IDE 插件 | 行级/函数级代码补全、Chat | 日常编码，各种语言和框架 |
| **Cursor** | AI-native IDE | 多文件编辑、Agent 模式、Composer | 全栈开发、复杂重构 |
| **Windsurf (Codeium)** | AI-native IDE | Cascade 流式思考、多文件编辑 | 大型项目理解、架构级任务 |
| **CodeBuddy** | IDE 插件/Agent | 多 Agent 协作、自动化任务 | 复杂多步骤任务、团队协作 |
| **Cline / Roo Code** | VS Code 插件 | Agent 自主执行、工具调用 | 完全自主的开发代理 |
| **Claude Code** | CLI 工具 | 终端 Agent、全项目理解 | 命令行开发、CI/CD |
| **Aider** | CLI 工具 | Git-aware 编辑、多模型支持 | Git 工作流集成 |
| **Continue** | IDE 插件 | 开源、本地模型支持 | 隐私敏感场景 |
| **v0 / Bolt / Lovable** | Web 平台 | 自然语言→完整应用 | 快速原型、MVP 搭建 |

**选择策略：**

```
场景一：日常业务开发
→ GitHub Copilot（补全快、文档全）

场景二：重构大型遗留项目
→ Cursor/Windsurf（多文件理解、Agent 模式）

场景三：隐私要求高（金融/医疗）
→ Continue + 本地模型（离线运行）

场景四：从零搭建应用原型
→ v0 / Bolt（生成速度快）

场景五：CI/CD 自动化
→ Claude Code / Aider（CLI first）

场景六：团队多人协作
→ CodeBuddy（多 Agent 协作）
```

**面试要点：** 能根据实际项目需求推荐合适工具；理解 IDE 插件、AI-native IDE、CLI Agent 三种形态的区别。

---

### 3. GitHub Copilot 的补全原理是什么？它如何理解你的代码上下文？

**GitHub Copilot 工作原理：**

GitHub Copilot 基于 OpenAI Codex 模型（GPT 系列的代码特化版本），采用了 **Fill-in-the-Middle (FIM)** 训练策略：

```
[前缀代码上下文] + [光标位置] + [后缀代码上下文] → [补全建议]
```

**上下文收集策略（关键）：**

```typescript
// Copilot 会收集以下信息作为上下文：

// 1. 当前文件内容（按相关性排序）
// 2. 相邻打开的标签页（Neighbor Tabs）
// 3. 项目中的同类型文件
// 4. 导入/引用的模块内容
// 5. 注释和文档字符串

// 示例：Copilot 根据上下文推断类型
interface User {
  id: number;
  name: string;
  email: string;
}

// 当你写注释时，Copilot 根据以上 interface 推断
// 生成的代码会使用 User 类型
function getUserById(id: number): Promise<User> {
  // Copilot 自动补全:
  return fetch(`/api/users/${id}`).then(res => res.json());
}
```

**FIM 训练详解：**

```
训练时将代码片段切分为三部分：
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Prefix   │  │  Middle  │  │  Suffix  │
│  (前缀)   │  │  (被挖掉) │  │  (后缀)  │
└──────────┘  └──────────┘  └──────────┘

训练目标：给定 Prefix + Suffix，预测 Middle

这种训练方式使模型天然适合 IDE 补全场景——光标前后都有代码
```

**面试要点：** 理解 FIM 训练、Neighbor Tabs 上下文机制、了解如何通过注释和代码结构"引导" Copilot 生成更好的结果。

---

### 4. AI Coding 工具的"幻觉（Hallucination）"问题是什么？在前端开发中如何识别和防范？

**AI 幻觉的定义：** AI 模型生成看似合理但实际错误或不存在的代码。在前端开发中表现为：

```typescript
// 幻觉示例 1：不存在的 API
// AI 可能生成：
localStorage.getAsync('key');  // 不存在此方法！

// 幻觉示例 2：不存在的 CSS 属性
.element {
  font-color: red;  // 正确的是 color
  text-wrap: balance; // 虽然存在但兼容性差，AI 不提示
}

// 幻觉示例 3：不存在的库函数
import { useOptimistic } from 'react';  // React 18 前不存在
// AI 在 React 17 项目中也可能生成这个

// 幻觉示例 4：错误的类型定义
interface Config {
  compilerOptions: {
    // AI 编造不存在的 TypeScript 配置选项
    enableExperimentalDecorators: boolean;
  };
}
```

**防范策略：**

```typescript
// 策略 1：类型系统作为第一道防线
// 定义清晰的类型，AI 生成的代码需要符合类型约束
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

// 策略 2：ESLint + TypeScript strict 模式
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true
  }
}

// 策略 3：对 AI 生成的代码建立审查清单
const reviewChecklist = {
  syntaxValid: true,          // 语法是否正确
  typesCorrect: true,         // 类型是否正确
  apiExists: true,            // 使用的 API 是否存在
  browserSupport: true,       // 浏览器兼容性
  securityImplication: false, // 是否有安全隐患
  logicCorrect: true,         // 业务逻辑是否正确
};

// 策略 4：针对关键逻辑编写立即执行的测试
describe('UserService - AI Generated', () => {
  it('should not contain hallucinated APIs', () => {
    const service = new UserService();
    expect(typeof service.fetchUser).toBe('function');
  });
});
```

**面试要点：** 展示对 AI 幻觉的具体认知（不是泛泛而谈），能够举例说明前端场景下的典型幻觉，并有系统化的防范方案。

---

### 5. 什么是 Prompt Engineering？它在 AI Coding 中有多重要？

**Prompt Engineering（提示工程）** 是设计和优化输入给 AI 模型的指令，以获得高质量、准确输出的技术和实践。

**在 AI Coding 中的核心地位：**

```
输入质量          模型能力          输出质量
┌──────────┐    ┌──────────┐    ┌──────────┐
│ 好的 Prompt│ → │ GPT-4o  │ → │ 好代码    │
│ 差的 Prompt│ → │ GPT-4o  │ → │ 差代码    │
└──────────┘    └──────────┘    └──────────┘

同样的模型，Prompt 质量决定输出质量
```

**Prompt 框架：CRISP**

```
C - Context（上下文）：项目技术栈、现有代码架构
R - Role（角色）: "你是一个资深 React 开发专家"
I - Intent（意图）: 具体要实现什么功能
S - Specs（规格）: 输入输出、边界条件、性能要求
P - Preferences（偏好）: 代码风格、库偏好、命名规范
```

**面试要点：** 不只停留在理论，能够给出具体的 Prompt 框架；能够对比同一需求下好坏 Prompt 的产出差异。

---

### 6. 在 AI Coding 中，注释（Comment）的作用发生了什么变化？

**传统开发 vs AI Coding 中的注释：**

| 维度 | 传统开发 | AI Coding |
|------|---------|-----------|
| 主要受众 | 人类开发者 | 人类 + AI |
| 注释作用 | 解释代码意图 | 引导 AI 生成代码 |
| 注释时机 | 通常是事后补充 | 先写注释，后生成代码 |
| 注释粒度 | 解释 why 而非 what | 同时解释 what 和 how |
| 注释价值 | 降低认知负担 | 充当"迷你 Prompt" |

**注释作为 Prompt 的实践：**

```typescript
// 传统注释（对 AI 帮助有限）
// 这是一个获取用户列表的函数
function getUsers() {
  return fetch('/api/users');
}

// AI Coding 优化后的注释（充当 Prompt）
/**
 * 获取分页用户列表
 * @param page - 当前页码（从 1 开始）
 * @param pageSize - 每页数量，默认 20，最大 100
 * @param keyword - 搜索关键词（匹配 name 和 email 字段）
 * @returns 返回 { data: User[], total: number, hasMore: boolean }
 * @throws 网络错误时抛出 ApiError
 */
// AI 根据以上注释自动生成完整实现
async function fetchUsers(params: {
  page: number;
  pageSize?: number;
  keyword?: string;
}): Promise<{ data: User[]; total: number; hasMore: boolean }> {
  const { page, pageSize = 20, keyword } = params;
  const query = new URLSearchParams({
    page: String(page),
    pageSize: String(Math.min(pageSize, 100)),
  });
  if (keyword) query.set('keyword', keyword);
  const res = await fetch(`/api/users?${query}`);
  if (!res.ok) throw new ApiError(`Failed to fetch users: ${res.status}`);
  return res.json();
}
```

**面试要点：** 理解注释在 AI Coding 时代的角色转变——从"解释代码"到"引导代码生成"；掌握用 JSDoc + 结构化注释作为 Prompt 的技巧。

---

### 7. AI Coding 工具如何处理大型代码仓库？有哪些上下文管理策略？

**大仓库面临的挑战：**

```
挑战：仓库有 5000+ 文件、100 万+ 行代码，但 LLM 上下文窗口有限

GPT-4o:     128K tokens（约 10 万词）
Claude 3.5: 200K tokens（约 15 万词）

——显然不能把整个仓库塞进去
```

**上下文管理策略：**

```typescript
// 策略 1：基于相关性的上下文选择
// Cursor/Windsurf 使用 Embedding + 向量搜索
function selectContext(query: string, repo: Repository): ContextFile[] {
  // Step 1: 对查询做 Embedding
  const queryEmbedding = embed(query);
  // Step 2: 在代码库的向量索引中搜索相似文件
  const similarFiles = vectorSearch(queryEmbedding, repo.embeddings, { topK: 10 });
  // Step 3: 分析依赖关系，补充被引用/引用的文件
  const dependencyFiles = analyzeImports(similarFiles);
  // Step 4: 合并去重，按相关性排序
  return rankByRelevance([...similarFiles, ...dependencyFiles]);
}

// 策略 2：RAG (Retrieval-Augmented Generation) 架构
// 用户提问 → 检索模块（搜索代码库、配置文件） → Prompt 组装 → LLM → 回答

// 策略 3：代码库索引（Codebase Index）
// 构建时对代码库做结构化分析：符号信息、导入关系图、调用关系图、语义向量索引

// 策略 4：文件片段（File Chunking）
// 大型文件被切分为语义完整的片段，每个片段 2000 tokens 左右，有重叠
```

**面试要点：** 理解 LLM 上下文窗口限制；掌握 RAG 在 AI Coding 中的应用；知道主流工具（Cursor/Windsurf）的上下文管理方式。

---

### 8. 什么是 AI Coding 中的"代码索引"？它和传统的静态分析有什么不同？

**代码索引在 AI Coding 中的定义：**

代码索引是为 AI 模型构建的、可语义搜索的代码库表示，通常以向量嵌入（Embedding）+ 结构化元数据的混合方式存储。

**关键区别：**

| 维度 | 传统静态分析 | AI 代码索引 |
|------|------------|------------|
| 搜索方式 | 精确匹配（名称/类型） | 语义搜索（"查询数据库的代码"） |
| 理解层次 | 语法/类型级别 | 意图/语义级别 |
| 关系发现 | 静态引用关系 | 隐式语义关联 |
| 适用场景 | 跳转定义、重构 | 代码理解、影响分析 |
| 存储格式 | AST + 符号表 | 向量 + 结构化数据 |
| 更新方式 | 编译时构建 | 增量更新 |

**实际应用示例：**

```
传统搜索："找到所有调用 fetchUser 的地方"
→ 返回所有直接引用 fetchUser 的位置

AI 索引搜索："找到所有获取用户数据的代码"
→ 返回调用 fetchUser、getUserProfile、loadUserInfo
   甚至包含手动 fetch('/api/users/...') 的代码
   AI 能理解这些代码的语义意图相同
```

**面试要点：** 区分传统静态分析和 AI 语义索引；理解 Embedding 在代码索引中的作用；了解构建代码索引的流程。

---

### 9. 在 AI Coding 中，如何平衡"信任 AI 生成代码"和"保持代码质量"？

**核心原则：Trust but Verify（信任但要验证）**

```typescript
// 信任层级模型（Trust Pyramid）

//         ┌──────────────┐
//         │  架构决策     │ ← 永远不交给 AI
//         │  (完全人工)   │
//         ├──────────────┤
//         │  业务逻辑     │ ← AI 建议 + 人工审查
//         │  (人审 AI)    │
//         ├──────────────┤
//         │  通用工具函数  │ ← AI 生成 + 自动测试验证
//         │  (测试驱动)   │
//         ├──────────────┤
//         │  样板代码     │ ← 可高度信任 AI
//         │  (模板/配置)  │
//         └──────────────┘
```

**质量保障工作流：**

```yaml
# .github/copilot-instructions.md 或类似的配置
ai-coding-guidelines: |

  ## 代码生成规则
  1. 所有生成的代码必须通过 ESLint + TypeScript strict 检查
  2. 涉及网络请求必须包含错误处理
  3. 涉及用户输入必须包含边界检查
  4. 禁止使用 any 类型
  5. 必须使用 async/await 而非 Promise.then()
  6. React 组件必须使用函数式组件

  ## 审查规则
  1. AI 生成的代码提交前必须通过 CI 管道
  2. 包含对应的测试
  3. 不引入新的 lint 警告
  4. 关键逻辑必须有至少一条人工编写的测试
```

**面试要点：** 展示分层信任的思维模型；理解不同代码类型需要不同审查深度；能够设计 AI + 人工的协作工作流。

---

### 10. 介绍 Cursor、Windsurf、Copilot 三者的核心差异，什么场景选哪个？

**架构层面的根本差异：**

```typescript
// GitHub Copilot — 补全型 IDE 插件
// 架构：在现有 IDE 内注入 AI 能力
// 核心价值：不改变工作流，增量增强
const CopilotArch = {
  mode: "Inline Enhancement",       // 行内增强
  contextWindow: "Limited (Neighbor Tabs)", // 有限的上下文
  strength: "Fast Completions",     // 快速补全
  weakness: "Multi-file Tasks",     // 跨文件任务弱
  learning: "Low, familiar workflow", // 学习成本低
};

// Cursor — AI-native IDE
// 架构：以 AI 为中心重新设计 IDE
// 核心价值：AI 作为一等公民，深度融合
const CursorArch = {
  mode: "Agent + Composer + Chat",  // 多种交互模式
  contextWindow: "Full Project",    // 全项目上下文
  strength: "Multi-file Refactoring", // 跨文件重构
  weakness: "Heavier than Copilot",  // 比 Copilot 重
  learning: "Medium, new shortcuts",  // 学习成本中等
};

// Windsurf (Codeium) — Cascade 流式思考 AI IDE
// 架构：引入"思考过程可见"的 Cascade 模式
const WindsurfArch = {
  mode: "Cascade (Think + Act)",    // 思考-行动流
  contextWindow: "Full Project",    // 全项目上下文
  strength: "Reasoning Transparency", // 推理过程可见
  weakness: "Newer Ecosystem",      // 生态较新
  learning: "Medium-Low",           // 学习成本中低
};
```

**实战对比表：**

| 场景 | 推荐工具 | 原因 |
|------|---------|------|
| 日常 CRUD 业务开发 | Copilot | 快速补全，不打断心流 |
| 重构 500 行+ 的组件 | Cursor Agent | 多文件理解 + 自动编辑 |
| 从 0 搭建新功能模块 | Cursor Composer | 一句话生成完整模块 |
| 理解/分析大型陌生项目 | Windsurf Cascade | 思考过程透明，易理解 |
| 修复跨多个文件的 Bug | Cursor Agent | 自动搜索定位 + 批量修改 |
| 团队统一使用（成本） | Copilot | 价格低，兼容性好 |
| 隐私要求高 | Continue + 本地模型 | 代码不出本地 |

**面试要点：** 不要停留在"都很好用"的表面评价；能具体说明架构层面的差异；根据不同场景给出具体的工具推荐并提供理由。

---

## 二、Prompt Engineering for Coding

### 11. AI Coding 中的 Prompt 有哪些核心组成要素？

**高质量 Prompt 的六大要素：**

```markdown
## 要素 1：系统角色设定 (System Role/Persona)

You are a Senior Frontend Engineer with 10 years of experience in
React, TypeScript, and performance optimization. You write clean,
testable code following SOLID principles.

## 要素 2：明确的任务描述 (Task Description)

Implement a debounce hook for React with the following behavior:
- Delays function execution until a specified wait time has elapsed
since the last invocation
- Supports leading-edge execution option
- Automatically cleans up on unmount

## 要素 3：技术栈约束 (Technical Constraints)

- React 18+ with TypeScript 5.x
- Use functional components and hooks
- No external dependencies
- Follow React Strict Mode compatibility

## 要素 4：输入输出规格 (I/O Specification)

Input:
  fn: (...args: any[]) => void | Promise<void>
  wait: number — delay in milliseconds (default: 300)
  options?: { leading?: boolean }

Output:
  A debounced version of the input function
  A cancel method to clear pending executions

## 要素 5：边界条件 (Edge Cases)

- Handle rapid consecutive calls correctly
- Clean up timeouts on component unmount
- Handle async functions properly
- Work with React 18 Strict Mode (double invocation)

## 要素 6：代码风格要求 (Style Preferences)

- Use named exports
- Add JSDoc comments
- Prefer explicit types over inference for public APIs
- Use useRef for mutable values, useCallback for stable references
```

**Prompt 质量自检表：**

```
□ 角色是否明确？
□ 任务是否具体（非模糊的"帮我写个..."）？
□ 技术栈是否指明？
□ 输入输出是否清楚？
□ 边界条件是否列出？
□ 代码风格是否有要求？

缺失越多 → AI 输出质量越低
```

**面试要点：** 展示系统化的 Prompt 设计方法；至少掌握 4-5 个核心要素；能够快速识别一个 Prompt 的不足之处。

---

### 12. 举例说明"Chain-of-Thought Prompting"在代码生成中的应用。

**Chain-of-Thought (CoT) — 让 AI "先思考再写代码"：**

```markdown
## 不使用 CoT 的 Prompt：
"实现一个 LRU 缓存"

## 使用 CoT 的 Prompt：
"实现一个 LRU 缓存。请按以下步骤思考：
Step 1: 选择数据结构（Map vs Object + 双向链表）
Step 2: 分析各操作的复杂度要求（get O(1), put O(1)）
Step 3: 设计接口
Step 4: 实现代码
Step 5: 提供测试用例"
```

**实际效果：**

```typescript
// AI 的思考过程（使用 CoT）：

// Step 1: 数据结构分析
// - JavaScript Map 保持插入顺序 → 天然适合 LRU
// - 不需要额外实现双向链表
// - Map.prototype.keys() 返回插入顺序的迭代器

// Step 2: 复杂度分析
// - get(k): map.has() → O(1), delete + set 也是 O(1)
// - put(k,v): 检查容量 → delete 最老 → set → O(1)

// Step 3: 接口设计
// interface LRUCache<K, V> { get(key: K): V | undefined; put(key: K, value: V): void; readonly size: number; }

// Step 4: 实现
class LRUCache<K = string, V = unknown> {
  private cache = new Map<K, V>();
  private capacity: number;

  constructor(capacity: number) {
    if (capacity <= 0) throw new Error('Capacity must be positive');
    this.capacity = capacity;
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) return undefined;
    // Map 不直接支持"移动到末尾"，技巧：delete + set 重新建立插入顺序
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  put(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.capacity) {
      const oldestKey = this.cache.keys().next().value as K;
      this.cache.delete(oldestKey);
    }
    this.cache.set(key, value);
  }

  get size(): number {
    return this.cache.size;
  }
}
// 关键洞察：CoT 让 AI 先做了架构分析，选对了 Map 这个数据结构
```

**CoT 在其他场景的应用：**

```markdown
## 场景：调试 Bug
"请按以下步骤帮我调试这个 React 组件的无限循环问题：
1. 先读取错误信息和组件代码
2. 识别所有 useEffect 和它们的依赖数组
3. 分析哪个依赖可能导致循环更新
4. 提出修复方案
5. 给出修复后的代码"

## 场景：性能优化
"请按以下步骤优化这个页面的性能：
1. 先用 React DevTools Profiler 的思路分析可能的瓶颈
2. 检查不必要的重渲染
3. 检查大列表是否缺少虚拟化
4. 检查是否有可以拆分的 bundle
5. 按优先级给出优化建议"
```

**面试要点：** 理解 CoT 的价值（提升复杂任务的准确率）；能举例说明在代码生成、调试、优化场景中的应用。

---

### 13. 解释"Few-Shot Prompting"在代码生成中如何提升结果质量。

**Few-Shot Prompting** = 在 Prompt 中提供 2-3 个示例，让 AI 学习你期望的模式和风格。

```typescript
// Zero-Shot（无示例）→ 生成质量不可控
// Prompt: "用 TypeScript 写一个 API 请求函数"
// 输出可能：用 axios? 用 fetch? 不确定。没有错误处理。没有类型定义。风格随机。

// Few-Shot（提供示例）→ 一致的代码风格
// Prompt: "请按照以下示例的风格，实现 fetchUserRoles 和 fetchPermissions 函数。"

// 示例代码：
type ApiResponse<T> = { data: T; error?: never } | { data?: never; error: string };

async function fetchCurrentUser(): Promise<ApiResponse<User>> {
  try {
    const res = await fetch('/api/user/current', {
      headers: { 'Content-Type': 'application/json' },
    });
    if (!res.ok) {
      return { error: `HTTP ${res.status}: ${res.statusText}` };
    }
    const data: User = await res.json();
    return { data };
  } catch (err) {
    return { error: err instanceof Error ? err.message : 'Unknown error' };
  }
}

// 需要实现的函数:
// 1. fetchUserRoles(userId: string): Promise<ApiResponse<Role[]>>
// 2. fetchPermissions(roleId: string): Promise<ApiResponse<Permission[]>>

// AI 生成的代码会高度一致（同样的错误处理、返回类型、代码组织方式）
```

**Few-Shot 在前端开发中的典型应用场景：**

1. **统一组件结构** - 给 AI 看 2 个现有组件，要求新组件保持相同模式
2. **统一测试风格** - 给 AI 看一个完整测试文件，为其他模块生成类似测试
3. **统一错误处理** - 给 AI 看 2-3 个 API handler 的错误处理模式
4. **代码迁移** - 给 AI 看 "旧写法 → 新写法" 的对比示例

**最佳实践：**

```
1. 示例数量：2-3 个最佳
   少于 2 个 → 模式不明显
   多于 5 个 → 浪费 token，边际收益递减

2. 示例应该覆盖多样性
   ✓ 包含正常情况和边界情况
   ✓ 包含成功路径和失败路径

3. 示例应标注关键模式
   标注命名规范、返回类型约定、错误处理方式
```

**面试要点：** 能解释 Few-Shot 的原理（通过示例建立输出分布约束）；知道 2-3 个示例是最佳数量；能在代码生成、测试、迁移等场景中正确应用。

---

### 14. 在 AI Coding 中，如何利用"规则/指令文件"（如 .cursorrules）来统一团队编码规范？

**规则文件（Rules File）的作用：**

规则文件是放在项目根目录或 `.cursor/rules/` 下的配置文件，作为每次 AI 交互的"系统级 Prompt 后缀"，确保 AI 生成的代码符合团队规范。

```markdown
# .cursorrules 示例（完整版）

## 角色定义
You are a Senior Frontend Engineer specializing in React, TypeScript,
and Next.js. Always prioritize code quality, type safety, and performance.

## 技术栈
- React 18 (functional components + hooks only)
- TypeScript 5.x (strict mode)
- Next.js 14 (App Router)
- Tailwind CSS (utility-first)
- Zustand (state management)
- TanStack Query (server state)
- Vitest + React Testing Library (testing)

## 代码规范

### TypeScript
- 禁止使用 `any` 类型
- 优先使用 `interface` 而非 `type`（除非需要联合类型）
- 导出函数必须标注返回类型

### React
- 使用函数声明（`function Component()`）而非箭头函数定义组件
- Props 使用 `interface XxxProps { ... }` 单独定义
- 自定义 Hook 以 `use` 开头
- 不要使用 `React.FC` 类型标注

### 命名规范
- 组件: PascalCase (UserProfile.tsx)
- Hook: camelCase with 'use' (useUserData.ts)
- 常量: UPPER_SNAKE_CASE (MAX_RETRY_COUNT)

### 文件组织
- 每个组件一个文件
- index.ts 仅用于 barrel export
- 测试文件与被测文件同目录

## 禁止事项
- 不要生成 console.log（使用项目定义的 logger）
- 不要使用 class 组件
- 不要使用 default export（统一 named export）
- 不要修改 prettier/eslint 配置
- 不要引入项目未使用的依赖
```

**主流工具的规则文件支持：**

```
Cursor:      .cursorrules 或 .cursor/rules/*.mdc
Windsurf:    .windsurfrules
Copilot:     .github/copilot-instructions.md
CodeBuddy:   .codebuddy/rules.md
Cline:       .clinerules
```

**面试要点：** 理解规则文件是 AI Coding 工程化的关键一环；能够设计覆盖技术栈、代码规范、命名、禁止事项的完整规则；了解不同工具对规则文件的命名和位置要求。

---

### 15. 什么是"结构化 Prompt 模板"？为 AI Coding 设计一个可复用的 Prompt 模板。

**结构化 Prompt 模板** 是将常见 AI Coding 任务抽象成可复用的模板，确保每次交互都包含必要信息，提升输出一致性。

```markdown
# 通用 AI Coding Prompt 模板

## 基本信息
- **任务类型**: [新功能 / Bug修复 / 重构 / 测试 / 代码审查 / 文档]
- **优先级**: [高 / 中 / 低]
- **相关模块**: [模块名称]

## 背景上下文
[简要说明为什么要做这个改动，业务背景是什么]
现有代码路径: `src/components/UserList/index.tsx`
相关依赖: 依赖的 hook: `useUsers`, 依赖的类型: `User`, `FilterParams`

## 具体需求
### 输入
- [参数名]: [类型] - [说明]
### 输出
- [返回值]: [类型] - [说明]
### 行为逻辑
1. [行为步骤 1]
2. [行为步骤 2]
### 边界条件
- [ ] 空数据
- [ ] 加载状态
- [ ] 错误状态
- [ ] 边界值

## 技术约束
- 技术栈: React 18 + TypeScript + Tailwind CSS
- 兼容性: Chrome 90+, Safari 15+, Edge 90+

## 输出要求
- [x] 包含完整的 TypeScript 类型定义
- [x] 包含错误处理
- [x] 包含必要的 JSDoc 注释
```

**场景化模板示例：**

```markdown
# 模板 A：Bug 修复 Prompt
## Bug 描述
**现象**: [用户看到的错误表现]
**复现步骤**: 1. [步骤 1] 2. [步骤 2]
**期望行为**: [正确应该是怎样的]
**环境**: Chrome 120 / React 18 / macOS

## 修复要求
1. 保持向后兼容
2. 添加对应的测试确保不再复现
3. 解释根因

# 模板 B：代码审查 Prompt
请审查以下代码，关注：
1. 潜在的 bug 和边界情况
2. 性能问题（不必要的重渲染、内存泄漏）
3. 安全问题（XSS、CSRF、敏感信息泄露）
4. 可维护性问题（过长的函数、重复代码）
5. 与项目规范的一致性

评分维度（1-5）：正确性、性能、安全性、可读性、可测试性

# 模板 C：重构 Prompt
## 重构目标
**当前问题**: [代码的什么问题]
**期望效果**: [重构后要达到什么效果]
## 约束
- [x] 不改变外部行为
- [x] 不改变公共 API
- [ ] 保持现有测试通过
```

**面试要点：** 展示模板化思维——不是每次从零写 Prompt；能够为不同任务类型（生成/Bug/审查/重构/测试）设计专用模板。

---

### 16. AI 生成的代码中常见的"反模式（Anti-Pattern）"有哪些？如何通过 Prompt 预防？

**常见 AI 生成代码反模式：**

```typescript
// 反模式 1：过度使用 any
// ❌ AI 常见输出
function processData(data: any): any {
  return data.items.map((item: any) => item.name);
}
// ✅ Prompt 预防: "禁止使用 any，不确定用 unknown + 类型守卫"

// 反模式 2：缺少错误处理
// ❌ AI 常见输出
async function fetchUser(id: string) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}
// ✅ Prompt 预防: "所有 async 函数必须包含 try-catch，所有 fetch 调用必须检查 res.ok"

// 反模式 3：不必要的状态/Effect
// ❌ AI 常见输出
function UserSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  // ... 手写 fetch 逻辑 + loading/error 管理
// ✅ Prompt 预防: "使用 TanStack Query 管理服务端状态，不要手动管理 loading/error"

// 反模式 4：useEffect 滥用
// ❌ AI 常见输出
function Component({ userId }: { userId: string }) {
  const [userName, setUserName] = useState('');
  useEffect(() => { setUserName(fetchUserName(userId)); }, [userId]);
  // 派生状态完全可以计算
// ✅ Prompt 预防: "派生状态应该直接计算，不要用 useEffect + useState"

// 反模式 5：内存泄漏
// ❌ AI 常见输出
function AutoRefresh() {
  useEffect(() => { setInterval(() => { refetch(); }, 5000); }, []);
  // 组件卸载后 interval 继续运行
// ✅ Prompt 预防: "所有定时器/订阅必须在 useEffect cleanup 中清理"
```

**反模式汇总与 Prompt 防御语句：**

| 反模式 | Prompt 防御语句 |
|--------|---------------|
| 过度 `any` | "禁止使用 any，不确定用 unknown + 类型守卫" |
| 缺少错误处理 | "所有 async 函数必须 try-catch，所有网络请求检查 ok" |
| 滥用 useEffect | "派生状态直接计算，网络请求用 React Query" |
| 无 cleanup | "所有 effect 必须返回 cleanup 函数" |
| 无 key 的列表 | "所有 map 渲染必须使用稳定的 key" |
| 状态更新依赖旧值 | "修改状态依赖旧值时使用函数式更新 setState(prev => ...)" |
| 大依赖注入 | "useEffect/useCallback/useMemo 的依赖数组必须准确" |

**面试要点：** 能列举 5+ 个 AI 常见反模式；不仅指出问题，还能给出 Prompt 层面的预防方案；体现了对 AI 生成代码"特征性缺陷"的深入了解。

---

### 17. 如何设计"迭代式 Prompt"来完成复杂的前端功能开发？

**迭代式 Prompt** = 不试图一次 Prompt 解决所有问题，而是通过多轮对话逐步构建复杂功能。

**案例：实现一个带搜索、分页、排序的数据表格**

```typescript
// ❌ 一次性的巨型 Prompt（容易翻车）:
// "帮我写一个数据表格组件，支持：服务端分页、排序、搜索、列宽拖拽、行选择、导出CSV、
//  列显示隐藏、固定列、虚拟滚动...使用 React + TypeScript + Tailwind..."
// 问题：太长容易遗漏、一次性生成耦合度高、某个需求没做好需全部重来

// ✅ 迭代式 Prompt 流程

// 迭代 1：核心骨架
// "创建 DataTable 组件的基础结构：定义 Props 接口、实现基础表格渲染、支持 customRender"

// 迭代 2：分页
// "为 DataTable 添加分页功能：pagination props、分页组件、pageSize 变化重置到第 1 页"

// 迭代 3：排序
// "添加列排序：sortable 配置、点击表头切换排序状态、排序图标"

// 迭代 4：搜索
// "在 DataTable 上方添加搜索栏：防抖 300ms、受控/非受控模式、清除按钮"

// 迭代 5：行选择
// "添加行选择功能：复选框/单选模式、全选、已选行高亮"
```

**迭代式 Prompt 的原则：**

```
原则 1：每次只加一个功能维度
  Iter 1: 基础渲染 → Iter 2: + 分页 → Iter 3: + 排序 → ...

原则 2：每轮迭代产出可运行的代码
  迭代后能编译通过、能渲染 = 可验证

原则 3：后一轮引用前一轮的具体文件
  "为 DataTable 添加..."（引用已有组件名）

原则 4：明确新增 vs 修改的文件
  新增：src/components/DataTable/PaginationBar.tsx
  修改：src/components/DataTable/index.tsx
```

**面试要点：** 理解迭代式 Prompt 比一次性 Prompt 更可靠的原因；能用具体案例演示迭代过程；掌握"逐渐累加功能"的 Prompt 策略。

---

### 18. 在 AI Coding 中，如何用 Prompt 引导 AI 编写"可测试的代码"？

**关键思路：** 在 Prompt 中内嵌可测试性要求，让 AI 生成的代码天然适合测试。

```markdown
## Prompt 模板：生成可测试代码

### 1. 依赖注入友好
- 外部依赖通过参数传入，不要硬编码导入
- 如果使用全局 API（如 fetch、localStorage），封装成可注入的 service

### 2. 纯函数优先
- 核心逻辑应该是纯函数（相同输入 → 相同输出，无副作用）
- 副作用集中在外层

### 3. 单一职责
- 每个函数只做一件事

### 4. 显式状态
- 不隐藏内部状态，状态变化可观测
- 返回结果而非 mutate 参数

### 5. 边界明确
- 明确定义输入输出类型
- 处理所有可能的输入（null, undefined, 空数组等）
```

**具体示例：**

```typescript
// ❌ AI 默认生成（难以测试）
class UserService {
  async getActiveUsers(): Promise<User[]> {
    const res = await fetch('/api/users');  // fetch 硬编码
    const users: User[] = await res.json();
    return users.filter(u => u.status === 'active');  // 过滤逻辑和网络请求耦合
  }
}

// ✅ 用 Prompt 引导生成（易于测试）
// Prompt: "HTTP Client 通过构造函数注入，过滤逻辑提取为纯函数"

interface IHttpClient {
  get<T>(url: string): Promise<T>;
}

function filterActiveUsers(users: User[]): User[] {  // 纯函数，独立可测试
  return users.filter(u => u.status === 'active');
}

class UserService {
  constructor(private http: IHttpClient) {}  // 依赖注入
  async getActiveUsers(): Promise<Result<User[]>> {
    try {
      const users = await this.http.get<User[]>('/api/users');
      return { data: filterActiveUsers(users) };
    } catch (err) {
      return { error: err instanceof Error ? err.message : 'Unknown error' };
    }
  }
}

// 测试变得简单：
// test filterActiveUsers: expect(filterActiveUsers([...])).toEqual([...])
// test UserService: const mockHttp = { get: vi.fn().mockResolvedValue(mockUsers) };
//   const service = new UserService(mockHttp);
//   const result = await service.getActiveUsers();
```

**可测试性 Prompt 速查：**

```
□ "外部依赖通过参数/构造函数注入，使用接口定义"
□ "核心逻辑提取为纯函数"
□ "使用 Result<T, E> 类型代替 throw/try-catch 满天飞"
□ "函数做一件事，命名体现其行为"
□ "副作用集中在一处，其余逻辑为纯函数"
□ "避免隐式依赖全局对象（window, localStorage, process.env）"
```

**面试要点：** 理解 AI 默认生成代码和可测试代码的差距；能在 Prompt 中内嵌可测试性约束；熟悉依赖注入、纯函数、Result 类型等可测试性模式。

---

### 19. 如何通过 Prompt 让 AI 生成符合无障碍（Accessibility/A11y）标准的代码？

```markdown
## A11y Prompt 模板

请实现 [组件名]，确保符合 WCAG 2.1 AA 标准：

### Semantic HTML（语义化 HTML）
- 使用正确的语义标签（<nav>, <main>, <article>, <aside> 等）
- 标题层级递进（h1 → h2 → h3）
- 列表使用 <ul>/<ol>/<dl>

### ARIA 属性
- 交互元素有明确的 role 属性
- 动态内容使用 aria-live 区域
- 展开/折叠使用 aria-expanded
- 自定义组件实现正确的 aria 属性

### 键盘可访问
- 所有交互元素可通过 Tab 聚焦
- 焦点顺序符合视觉顺序
- 自定义组件实现正确的键盘交互（Enter/Space/Arrow）
- 模态框打开时焦点移入，关闭时焦点还原

### 视觉
- 颜色不是传达信息的唯一方式（配合图标/文字）
- 对比度符合标准（正文 4.5:1，大文字 3:1）
- 焦点指示器可见（outline 不要 none）

### 表单
- 所有 input 有关联的 <label>
- 错误信息与表单字段关联（aria-describedby）
- 必填字段有明确标识
```

**AI 根据 A11y Prompt 生成的正确示例（Tabs 组件）：**

```tsx
function AccessibleTabs({ tabs, activeTab, onTabChange }: AccessibleTabsProps) {
  // 键盘导航：Arrow 键切换 Tab
  const handleKeyDown = (e: React.KeyboardEvent) => {
    let newIndex = activeIndex;
    if (e.key === 'ArrowRight') newIndex = (activeIndex + 1) % tabs.length;
    else if (e.key === 'ArrowLeft') newIndex = (activeIndex - 1 + tabs.length) % tabs.length;
    else if (e.key === 'Home') newIndex = 0;
    else if (e.key === 'End') newIndex = tabs.length - 1;
    else return;
    e.preventDefault();
    onTabChange(tabs[newIndex].id);
  };

  return (
    <div>
      <div role="tablist" aria-orientation="horizontal" onKeyDown={handleKeyDown}>
        {tabs.map((tab) => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={tab.id === activeTab}
            aria-controls={`panel-${tab.id}`}
            tabIndex={tab.id === activeTab ? 0 : -1}
            onClick={() => onTabChange(tab.id)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {tabs.map((tab) => (
        <div
          key={tab.id}
          role="tabpanel"
          aria-labelledby={`tab-${tab.id}`}
          hidden={tab.id !== activeTab}
        >
          {tab.content}
        </div>
      ))}
    </div>
  );
}
// A11y checklist ✅：语义化 role、aria-selected、键盘导航、tabIndex 管理
```

**面试要点：** 不只会说"要做无障碍"，能在 Prompt 层面给出具体可执行的要求；熟悉 WCAG 核心标准和 ARIA 模式；理解 Prompt 可以降低开发者写无障碍代码的门槛。

---

### 20. 在 AI Coding 中，"Negative Prompting"（否定式提示）如何帮助排除不良代码模式？

**Negative Prompting** = 明确告诉 AI "不要做什么"，比仅说"要做什么"更能有效排除常见问题。

**常见 Negative Prompt 清单：**

```markdown
## TypeScript 相关
- 不要使用 `any`
- 不要使用 `as` 类型断言（除非必要）
- 不要使用 `!` 非空断言

## React 相关
- 不要使用 class 组件
- 不要使用 `React.FC` 类型
- 不要直接在 JSX 中定义内联对象/函数（会导致不必要的重渲染）
- 不要在 render 中创建新的组件
- 不要忘记给列表项加 key
- 不要使用 index 作为 key（除非列表是静态的）

## 通用
- 不要使用 var
- 不要使用 console.log（用项目的 logger 工具）
- 不要注释掉代码（删除即可，历史在 Git）

## 样式
- 不要使用内联 style 对象（用 Tailwind classes）
- 不要使用 !important
- 不要硬编码颜色值，使用设计系统的 CSS 变量

## 安全
- 不要在前端代码中硬编码密钥/token
- 不要使用 dangerouslySetInnerHTML（除非明确需要且已 sanitize）
- 不要把用户输入直接拼接到 URL 中
```

**面试要点：** 理解 Negative Prompting 是对正面 Prompt 的精准补充；能够针对不同技术栈（TS/React/样式/安全）准备相应的 Negative 清单；知道 Negative Prompting 对防止 AI 常见幻觉特别有效。

---
## 三、AI 代码生成策略与最佳实践

### 21. AI Coding 中"代码生成"的常见策略有哪些？各适用于什么场景？

**代码生成策略全景：**

```typescript
// 策略 1：注释驱动生成（Comment-Driven）
// 最常用：先写注释描述需求，让 AI 根据注释生成实现

/**
 * 将扁平数组转为树形结构
 * @param items - 扁平数据，每项有 id 和 parentId
 * @param rootId - 根节点 parentId 的值（默认 null）
 * @returns 树形结构数组
 * @complexity O(n) - 使用 Map 索引优化
 */
// AI 根据上面的 JSDoc 生成下面的实现：
function buildTree<T extends { id: string; parentId: string | null }>(
  items: T[], rootId: string | null = null
): (T & { children: T[] })[] {
  const map = new Map<string, T & { children: T[] }>();
  const result: (T & { children: T[] })[] = [];
  for (const item of items) {
    map.set(item.id, { ...item, children: [] });
  }
  for (const item of items) {
    const node = map.get(item.id)!;
    if (item.parentId === rootId) {
      result.push(node);
    } else {
      const parent = map.get(item.parentId!);
      if (parent) parent.children.push(node);
    }
  }
  return result;
}
// 适用场景：工具函数、数据处理、算法实现

// 策略 2：接口先行生成（Interface-First）
// 先定义完整接口/类型，让 AI 基于类型生成实现
interface TodoService {
  getAll(): Promise<Todo[]>;
  getById(id: string): Promise<Todo>;
  create(input: CreateTodoInput): Promise<Todo>;
  update(id: string, input: UpdateTodoInput): Promise<Todo>;
  delete(id: string): Promise<void>;
}
// Prompt: "基于 TodoService 接口，实现 TodoServiceImpl 类"
// → AI 根据 interface 约束生成实现，类型天然正确
// 适用场景：API 服务、数据仓库、接口驱动开发

// 策略 3：参考代码生成（Reference-Based）
// 提供一个已有实现作为参考
// "参考 fetchUsers 的实现风格，实现 fetchProducts"
// 适用场景：批量类似模块开发

// 策略 4：测试驱动生成（TDD with AI）
// 先写测试用例，让 AI 根据测试生成满足条件的代码
describe('formatCurrency', () => {
  it('formats number to CNY', () => {
    expect(formatCurrency(1234.56)).toBe('¥1,234.56');
  });
  it('handles zero', () => {
    expect(formatCurrency(0)).toBe('¥0.00');
  });
  it('handles negative', () => {
    expect(formatCurrency(-99.9)).toBe('-¥99.90');
  });
});
// Prompt: "基于以下测试用例实现 formatCurrency"
// → 适用场景：核心工具函数、财务计算

// 策略 5：渐进式生成（Progressive）
// 从骨架开始，逐步细化
// Round 1: "创建 DashboardPage 的骨架"
// Round 2: "实现 StatsCards 子组件"
// Round 3: "添加数据加载和 loading 状态"
// 适用场景：复杂页面、大功能模块
```

**策略选择指南：**

| 策略 | 最佳场景 | 代码质量 | 审查难度 |
|------|---------|---------|---------|
| 注释驱动 | 工具函数、算法 | 高 | 低 |
| 接口先行 | API 层、服务层 | 高（类型约束） | 低 |
| 参考生成 | 批量类似模块 | 高（一致性） | 低 |
| TDD 驱动 | 核心逻辑、金融计算 | 最高 | 低 |
| 渐进生成 | 复杂页面/模块 | 中 → 高 | 中 |

**面试要点：** 不只说"用 AI 生成代码"，能区分不同策略和适用场景；理解不同类型的代码应该用不同策略生成。

---

### 22. 如何使用 AI 生成符合团队规范的代码？介绍"Coding Standards Injection"技术。

**Coding Standards Injection** = 将团队规范以结构化方式注入 AI 上下文，确保生成代码一致性。

```typescript
// 方式 1：项目级规则文件（最推荐）
// .cursorrules 或 .github/copilot-instructions.md
// 写法和传统 ESLint 配置类似，但面向 AI

// 方式 2：Context Window 注入
// 在每次 AI 对话时，把规范文件作为上下文附上
// Prompt 开头加入："请在生成代码前，先阅读以下代码规范：[粘贴规范]"

// 方式 3：Check-Then-Fix（生成后校验）
// AI 生成代码 → ESLint/Prettier 检查 → AI 修复违规
async function aiCodeWithLintCheck(prompt: string): Promise<string> {
  const code = await aiGenerate(prompt);       // 1. AI 生成代码
  const lintResult = await runESLint(code);    // 2. 写入临时文件并运行 ESLint
  if (lintResult.errors.length > 0) {          // 3. 如果有违规，交给 AI 修复
    const fixPrompt = `修复以下代码的 ESLint 错误：
错误列表：${lintResult.errors.map(e => `- Line ${e.line}: ${e.message}`).join('\n')}
代码：${code}`;
    return aiGenerate(fixPrompt);
  }
  return code;
}

// 方式 4：Rule-as-Code 约束（最严格）
// 用 zod 定义规范约束，AI 生成代码后自动校验
const ComponentSchema = z.object({
  hasPropsInterface: z.literal(true),
  noAnyType: z.literal(true),
  isNamedExport: z.literal(true),
  noConsoleLog: z.literal(true),
});
function validateGeneratedCode(code: string): ValidationResult {
  return {
    hasPropsInterface: /interface\s+\w+Props/.test(code),
    noAnyType: !/\bany\b/.test(code),
    isNamedExport: /export\s+(function|const)/.test(code),
    noConsoleLog: !/console\.log/.test(code),
  };
}
// 如果验证不通过，让 AI 重新生成
```

**面试要点：** 不只依赖 AI "自动遵守规范"，而是设计系统化的规范注入方案；能够从规则文件、上下文注入、事后校验、编码约束四个层面构建规范保障体系。

---

### 23. 使用 AI 生成代码时，如何确保安全性？列举前端安全相关的 Prompt 策略。

**安全 Prompt 模板（OWASP Top 10 前端相关）：**

```typescript
// XSS 防护
// Prompt 追加:
// - 不要使用 dangerouslySetInnerHTML/innerHTML 插入用户内容
// - 如果必须插入 HTML，使用 DOMPurify 先清理
// - 渲染用户输入的 URL 时校验协议（仅允许 http/https）

// ❌ AI 可能生成
function UserComment({ text }: { text: string }) {
  return <div dangerouslySetInnerHTML={{ __html: text }} />;
}
// ✅ 安全要求下 AI 应该生成
function UserComment({ text }: { text: string }) {
  return <div>{text}</div>;  // 纯文本渲染，天然防 XSS
}

// CSRF 防护
// Prompt: "所有数据修改请求必须附带 CSRF token"
function getCsrfToken(): string {
  const meta = document.querySelector<HTMLMetaElement>('meta[name="csrf-token"]');
  return meta?.content ?? '';
}

// 敏感信息泄露防护
// Prompt: "不要在代码中硬编码 API Key，敏感配置从环境变量读取"
const STRIPE_KEY = import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY;
// 注意：只有 VITE_ 前缀的变量会暴露给前端

// 客户端存储安全
// Prompt: "敏感数据不要存在 localStorage，Token 优先使用 httpOnly cookie"
```

**安全 Prompt 速查清单：**

```markdown
## 数据输入
□ 用户输入是否经过校验/转义？
□ URL 参数是否经过校验？
□ 文件上传是否校验类型和大小？

## 数据输出
□ 用户内容是否用文本渲染（而非 HTML）？
□ 富文本是否经过 sanitize？
□ 错误信息是否隐藏了内部细节？

## 认证与授权
□ Token 是否存储在安全位置？
□ API 请求是否附带认证头？
□ 是否有前端路由守卫？

## 数据传输
□ 是否使用 HTTPS？
□ 是否使用 Content-Security-Policy？
```

**面试要点：** 能具体说出 5+ 种前端安全风险以及对应的 AI Prompt 预防策略；不空谈"安全重要"，而是演示如何通过 Prompt 让 AI 生成安全的代码。

---

### 24. 什么是"代码审查式 Prompt"？如何使用 AI 审查 PR 中的代码？

**代码审查 Prompt 模板：**

```markdown
请以资深前端工程师的身份审查以下 PR 代码。

## 审查维度

### 1. 正确性
- 逻辑是否正确？边界条件是否处理？异步操作是否正确？

### 2. 安全性
- 是否有 XSS 风险？用户输入是否校验？敏感信息是否泄露？

### 3. 性能
- 是否有不必要的重渲染？是否有内存泄漏？

### 4. 可维护性
- 函数/组件是否过长？是否有重复代码？命名是否清晰？

### 5. 代码规范
- 是否符合项目 ESLint 规则？类型定义是否充分？

## 输出格式
对每个问题标注:
- 严重程度: 🔴 阻塞 / 🟡 建议 / 🟢 小优化
- 文件:行号
- 问题描述
- 建议修复方案（含代码）
```

**实际审查输出示例：**

```markdown
### 审查结果

#### 🔴 阻塞（必须修复）

**文件**: src/components/UserProfile.tsx:23
**问题**: useEffect 缺少依赖项，可能导致闭包陷阱
**建议修复**:
```tsx
useEffect(() => {
  fetchUserData(userId);
}, [userId]);  // ← 添加缺失的依赖
```

**文件**: src/utils/validation.ts:45
**问题**: 对 null/undefined 输入无防护
**建议修复**: 在函数开头添加 `if (!email) return false;`

#### 🟡 建议

**文件**: src/hooks/useDebounce.ts:12
**问题**: debounce 实现未在 unmount 时清理 timeout
**建议**: 在 useEffect 中返回 cleanup 函数
```

**面试要点：** 展示结构化的审查维度（不止"看看有没有 bug"）；能给出有严重程度标注、文件行号、代码修复方案的专业审查输出。

---

### 25. 如何使用 AI 为已有代码生成单元测试？有哪些注意事项？

**测试生成 Prompt 模板：**

```markdown
请为以下代码生成完整的单元测试。

## 测试要求
- 测试框架: Vitest
- Mock 库: vitest (vi.fn, vi.mock)
- 工具库: @testing-library/react (组件测试)

### 覆盖要求
- 正常路径（Happy Path）
- 错误路径（Error Path）
- 边界条件（空输入、极大值、null/undefined）
- 用户交互（如适用）

### 注意事项
- 只 mock 外部依赖，被测函数的内部逻辑不 mock
- 异步测试使用 async/await
- 测试名称以 "should" 开头
```

**AI 生成的测试示例：**

```typescript
// src/utils/search.ts
export function searchItems<T extends { name: string; tags?: string[] }>(
  items: T[], keyword: string
): T[] {
  if (!keyword.trim()) return items;
  if (!items.length) return [];
  const lowerKeyword = keyword.toLowerCase();
  return items.filter(item => {
    if (item.name.toLowerCase().includes(lowerKeyword)) return true;
    if (item.tags?.some(tag => tag.toLowerCase().includes(lowerKeyword))) return true;
    return false;
  });
}

// AI 生成的测试
describe('searchItems', () => {
  describe('正常路径', () => {
    it('should return items matching name (case insensitive)', () => {
      const result = searchItems(mockItems, 'react');
      expect(result).toHaveLength(1);
    });
    it('should return items matching tags', () => {
      const result = searchItems(mockItems, 'layout');
      expect(result).toHaveLength(1);
    });
  });
  describe('边界条件', () => {
    it('should return all items when keyword is empty', () => {
      expect(searchItems(mockItems, '')).toEqual(mockItems);
    });
    it('should return empty array when items is empty', () => {
      expect(searchItems([], 'search')).toEqual([]);
    });
    it('should return empty array when no match', () => {
      expect(searchItems(mockItems, 'zzz_not_found')).toEqual([]);
    });
    it('should handle items without tags', () => {
      const itemsNoTags = [{ name: 'Simple Item' }];
      expect(searchItems(itemsNoTags, 'simple')).toHaveLength(1);
    });
  });
});
```

**注意事项（审查 AI 生成的测试）：**

```
⚠️ AI 测试常见问题：
1. 测试太浅：只测 happy path，不测错误/边界
2. 过度 mock：把不该 mock 的也 mock 了
3. 假阳性：测试通过了但实际测的是 mock 而非真实逻辑
4. 测试耦合：多个 it 之间有执行顺序依赖

✅ 审查清单：
□ 每条测试是否独立可运行？
□ 是否覆盖了所有出口分支？
□ Mock 是否必要且最小化？
□ 是否有边界测试（空/零/null/undefined）？
□ 异步测试是否有 await？
```

**面试要点：** 能给出结构化的测试生成 Prompt；能够审查 AI 生成的测试（找出假阳性、过度 mock 等问题）；理解 TDD 和 AI 的结合方式。

---

### 26. AI 生成的代码如何处理国际化（i18n）？有哪些 Prompt 策略？

```typescript
// i18n Prompt 模板
/*
请生成符合国际化要求的代码，使用 react-i18next。

## i18n 要求
- 所有用户可见的文本必须使用 t() 函数
- 不使用任何硬编码的中文/英文字符串
- 翻译 key 命名规范：模块.子模块.含义
- 数字使用 i18n 格式化，日期使用 i18n 格式化
- 使用 t() 的 count 参数处理复数
- <html lang="{locale}"> 动态设置
*/

// AI 应生成：
import { useTranslation } from 'react-i18next';

function OrderList() {
  const { t } = useTranslation();
  return (
    <div>
      <h1>{t('order.list.title')}</h1>
      <p>{t('order.list.count', { count: 5 })}</p>
      {orders.length === 0 ? (
        <EmptyState message={t('order.list.emptyText')} />
      ) : (
        <ul>
          {orders.map(order => (
            <li key={order.id}>
              {t('order.item.total', {
                amount: t('currency.cny', { value: order.total })
              })}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**面试要点：** 不只是在 Prompt 中说"支持 i18n"；能够给出具体的 i18n 约束——key 命名规范、格式化方式、RTL 支持。

---

## 四、上下文工程 (Context Engineering)

### 27. 什么是"上下文工程（Context Engineering）"？为什么它是 AI Coding 的核心技能？

**上下文工程** = 系统性地设计、管理和优化提供给 AI 模型的上下文信息，以最大化输出质量。

**三层模型：**

```
┌─────────────────────────────────────┐
│  Layer 3: 即时上下文               │
│  - 当前问题/任务描述               │
│  - 相关代码片段                    │
│  - 最近对话历史                    │
├─────────────────────────────────────┤
│  Layer 2: 项目上下文               │
│  - .cursorrules / rules.md         │
│  - 技术栈和约定                    │
│  - 项目结构和架构                  │
├─────────────────────────────────────┤
│  Layer 1: 领域上下文               │
│  - 业务领域知识                    │
│  - 行业最佳实践                    │
│  - 安全/合规要求                   │
└─────────────────────────────────────┘
```

**为什么重要（实验数据）：**

```
1. 无上下文 Prompt: "实现分页逻辑" → 通过率 45%
2. 基础上下文（技术栈）: → 通过率 65%
3. 完整上下文（技术栈 + 规范 + 参考代码 + 现有 API）: → 通过率 92%

上下文质量每提升一级，生成代码可用性提升 20-30%
```

**核心实践：**

```typescript
// 实践 1：主动提供依赖信息
// 不要只说："实现一个 Modal"
// 要说：
/*
项目已有的相关代码：
- src/hooks/useLockScroll.ts — 锁定 body 滚动
- src/hooks/useFocusTrap.ts — 焦点陷阱
请使用这些现有工具实现 Modal 组件，不要重新实现 scroll lock 和 focus trap。
*/

// 实践 2：提供 API 契约
/*
后端 API 返回类型：
interface UserListResponse {
  items: User[];
  pagination: { currentPage: number; totalPages: number; totalItems: number; pageSize: number; };
}
请使用 UserListResponse 作为 useQuery 的返回类型。
*/

// 实践 3：引用项目常量/枚举
/*
项目常量：ORDER_STATUS = { PENDING: 'pending', PROCESSING: 'processing', ... }
请基于 ORDER_STATUS 实现状态筛选组件。
*/
```

**面试要点：** 能超越"写好 Prompt"，讨论上下文结构设计；理解三个层次上下文的区别；知道主动提供依赖、API、常量信息来提升 AI 输出质量。

---

### 28. 如何设计和维护项目级的 AI 规则文件，使其在团队中持续有效？

```yaml
# .cursor/rules/02-tech-stack.mdc 示例
---
description: 当前项目技术栈约束
globs: "**/*.{ts,tsx}"
alwaysApply: true
---

# 技术栈
- React 18.2+ (函数组件 + Hooks)
- TypeScript 5.3+ (strict mode)
- Next.js 14 (App Router)
- State: Zustand (client) + TanStack Query (server)
- Style: Tailwind CSS 3.4+ + shadcn/ui
- Form: React Hook Form + Zod
- Test: Vitest + React Testing Library

# 不使用的技术
- 不要引入 Redux (已有 Zustand)
- 不要引入 styled-components (已有 Tailwind)
- 不要引入 axios (已有 fetch 封装)
```

**规则文件设计原则：**

```
分层结构:
.cursor/rules/
├── 01-global.mdc       # 全局规范
├── 02-tech-stack.mdc   # 当前技术栈
├── 03-naming.mdc       # 命名规范
├── 04-patterns.mdc     # 代码模式/模板
├── 05-testing.mdc      # 测试规范
└── 06-security.mdc     # 安全要求

内容原则:
DO: 具体可执行的规则、正面+反面示例、团队约定、禁止事项
DON'T: 通用的编程道理、过于冗长的反例、与ESLint重复的格式规则
```

**团队协作机制：**

```
1. 规则文件纳入 PR Review（和代码同等地位）
2. 新人 on-boarding 包含规则文件解读
3. 每月回顾规则文件是否需要更新
4. 当 AI 反复生成某类错误时 → 在规则文件中加一条禁止
5. 规则文件变更通过 PR + 团队讨论
```

**面试要点：** 不只是"写个 rules 文件"；能设计分层结构、给持续维护方案；理解规则文件是活的文档，需要团队共同维护。

---

### 29. 解释 AI Coding 中的"In-Context Learning"（上下文学习）与前端的结合。

**In-Context Learning (ICL)** = AI 模型通过 Prompt 中提供的示例和上下文，在不更新参数的情况下"学会"完成新任务。

**ICL 在前端 AI Coding 中的三种形式：**

```typescript
// 形式 1：模式学习（Pattern Learning）
// .cursorrules 中定义代码风格偏好
/*
## 组件风格
✅ 推荐写法：
interface GreetingProps { name: string; age?: number; }
export function Greeting({ name, age }: GreetingProps) {
  return <p>Hello, {name}{age && ` (${age})`}</p>;
}

❌ 避免写法：
const Greeting: React.FC<{ name: string }> = ({ name }) => { ... };
// 不要用 React.FC，不要用 default export
*/
// AI 学到：用 interface 单独定义 Props、named export、条件渲染风格

// 形式 2：组件模式学习（通过打开多个 Tab）
// Tab 1: UserCard.tsx, Tab 2: ProductCard.tsx
// Prompt: "参考打开的 UserCard 和 ProductCard 的写法，实现 OrderCard"
// AI 学到：Props 命名方式、导出写法、样式组织、空状态处理

// 形式 3：对比学习
// 展示好的和坏的写法，比只展示好的更快见效
```

**ICL 的关键洞察：**

```
1. 示例质量 > 示例数量（2-3 个典型示例最佳）
2. 对比学习有效（✅ vs ❌）
3. 就近原则（最近对话中的示例权重更高）
4. 一致性引导（AI 对"与上下文保持一致"有强烈倾向）
```

**面试要点：** 区分 ICL 和微调（Fine-tuning），ICL 不需要训练；理解前端中 ICL 的三种形式；知道通过示例教会 AI 新模式的技巧。

---

### 30. 如何利用 AI Coding 工具处理"遗留代码"（Legacy Code）的理解和迁移？

```typescript
// 场景 1：理解遗留代码
// Prompt:
/*
请分析以下遗留代码，给出：
1. 功能概述（这段代码做什么）
2. 数据流图（输入 → 处理 → 输出）
3. 潜在问题清单（bug、性能、安全问题）
4. 依赖关系（引用了哪些模块/API）
5. 测试建议
*/

// 场景 2：遗留代码迁移
// Prompt:
/*
请将以下 jQuery 代码迁移到 React 18 + TypeScript + TanStack Query。
要求：
1. 保持完全相同的功能
2. 使用函数组件 + Hooks
3. 使用 useQuery 管理数据加载
4. 添加 loading/error 状态
5. 添加搜索防抖（300ms）
6. 响应式设计
*/

// 场景 3：增量迁移策略
/*
Prompt:
"当前项目正在从 Vue 2 迁移到 React 18。给出增量迁移方案和对应的 AI Prompt 策略。"

AI 建议：
1. 使用 Micro-Frontend 架构并行运行新旧应用
2. 每次迁移一个模块
3. AI Prompt 中提供：原组件代码 + 已有新代码参考 + 概念映射表（computed→useMemo, watch→useEffect）
*/
```

**面试要点：** 不只说"AI 可以辅助迁移"；能演示从"理解"到"迁移"到"验证"的完整流程；理解增量迁移的重要性。

---

## 五、AI 辅助代码审查与质量保障

### 31. 如何使用 AI 做"预防性代码审查"（Proactive Code Review）？

**预防性代码审查** = 在代码提交前就用 AI 发现潜在问题，而非等到 CI/CD 报错或生产事故。

**三层审查体系：**

```
Step 1: 本地开发阶段（pre-commit）
  → git commit 前自动调用 AI 审查暂存区变更

Step 2: CI 阶段的 AI 审查
  → PR 创建时自动分析 diff，生成审查报告

Step 3: 智能门禁判断
  → 根据 AI 审查结果的严重程度决定是否阻断 PR
```

**门禁判断逻辑：**

```typescript
interface AIReviewResult {
  overall: 'pass' | 'warn' | 'fail';
  issues: Array<{
    severity: 'blocker' | 'major' | 'minor';
    file: string;
    line: number;
    message: string;
    suggestion: string;
  }>;
}

function shouldBlockPR(review: AIReviewResult): boolean {
  const hasBlocker = review.issues.some(i => i.severity === 'blocker');
  const majorCount = review.issues.filter(i => i.severity === 'major').length;
  return hasBlocker || majorCount >= 3;  // 有 blocker 或 3+ major 则阻断
}
```

**预防性审查的关注点：**

```markdown
## 安全性（每次必查）
- [ ] 用户输入是否经过校验/转义？
- [ ] 是否有硬编码的密钥/token？
- [ ] dangerouslySetInnerHTML 是否安全？

## 性能（React 项目）
- [ ] 是否有不必要的 re-render？
- [ ] 大列表是否有虚拟化？图片是否懒加载？

## 可访问性
- [ ] 交互元素是否可通过键盘访问？
- [ ] 图片有 alt 吗？表单有 label 吗？

## 错误处理
- [ ] 所有 async 操作有错误处理？
- [ ] 边界条件（空、null、undefined）处理了吗？
```

**面试要点：** 不只是"用 AI 审查代码"；能设计 pre-commit → CI → 门禁的三层审查体系；理解预防性审查（提交前）比事后审查更有价值。

---

### 32. AI 在 CI/CD 管道中可以扮演什么角色？设计一个 AI 增强的 CI/CD 流水线。

```yaml
# AI-Enhanced CI/CD Pipeline 设计

jobs:
  # Job 1: 代码质量门禁（快速检查）
  quality-gate:
    steps:
      - run: npm run lint   # 传统检查
      - run: npm run type-check
      - run: npm run test -- --coverage
      - run: npx ai-coverage-analyzer  # AI 增强：覆盖率不足检测
      - run: npx ai-bundle-analyzer    # AI 增强：智能大小检查

  # Job 2: AI 深度审查（并行运行）
  ai-deep-review:
    steps:
      - run: git diff origin/main...HEAD | ai-review \
               --aspects security,performance,patterns,accessibility

  # Job 3: AI 测试生成（自动补全缺失测试）
  ai-test-generation:
    steps:
      - run: npx ai-test-gen --changed-files "..." --min-coverage 80

  # Job 4: 部署风险评估
  deploy-risk-assessment:
    steps:
      - run: git log --oneline -10 | ai-deploy-risk

  # Job 5: 部署 + AI 监控
  deploy:
    steps:
      - run: npm run deploy
      - run: npx ai-health-check --url https://app.example.com
```

**流水线设计原则：**

```
1. 传统检查先行（lint、type-check、test）— 快速失败
2. AI 检查并行运行 — 不阻塞快速检查路径
3. AI 门禁可配置严重程度阈值
4. 关注点分离：质量门禁 vs 深度审查 vs 风险评估
```

**面试要点：** 能设计出 lint → AI 审查 → AI 测试生成 → 风险评估 → 部署监控的完整流水线；区分哪些阶段需要 AI，哪些用传统工具即可；理解 AI 在 CI/CD 中"辅助增强"而非"完全替代"的定位。

---

## 六、AI 在测试中的应用

### 33. 如何用 AI 生成"有意义的"E2E 测试用例？

```typescript
// E2E 测试生成 Prompt
/*
请为以下页面生成 Playwright E2E 测试。

## 页面描述
URL: /checkout — 电商结账页面
包含：地址表单、支付方式选择、订单明细、提交按钮

## 必须覆盖的场景
1. 正常流程：填写表单 → 选择支付 → 提交成功
2. 表单验证：空字段提交、格式错误
3. 支付方式切换
4. 优惠券应用
5. 网络错误处理
6. 未登录用户拦截

## 测试结构
- 使用 test.describe 按场景分组
- 使用 data-testid 定位元素
- 每个测试用例独立
*/

// AI 生成：
test.describe('Checkout - 正常流程', () => {
  test('should complete checkout with valid information', async ({ page }) => {
    await page.goto('/checkout');
    await page.fill('[data-testid="checkout-name"]', '张三');
    await page.fill('[data-testid="checkout-phone"]', '13800138000');
    await page.click('[data-testid="payment-wechat"]');
    await page.click('[data-testid="submit-order"]');
    await expect(page).toHaveURL(/\/order\/success/);
  });
});

test.describe('Checkout - 表单验证', () => {
  test('should show error for empty name', async ({ page }) => {
    await page.goto('/checkout');
    await page.click('[data-testid="submit-order"]');
    await expect(page.locator('[data-testid="name-error"]')).toBeVisible();
  });
});

test.describe('Checkout - 边界与异常', () => {
  test('should handle network error gracefully', async ({ page }) => {
    await page.route('**/api/orders', route => route.abort('failed'));
    await page.fill('[data-testid="checkout-name"]', '张三');
    await page.click('[data-testid="submit-order"]');
    await expect(page.locator('[data-testid="submit-error"]')).toBeVisible();
  });
});
```

**面试要点：** 不只是"生成测试"；能区分生成的测试哪些有价值（正常流程、边界、异常），哪些是噪音（测实现细节）；理解 E2E 测试的价值在于覆盖用户行为而非代码逻辑。

---

### 34. AI 生成的测试中常见的"反模式"有哪些？

```typescript
// 反模式 1：测试实现细节而非行为
// ❌ AI 常见输出
test('should call setState', () => {
  const setState = vi.fn();
  render(<Component setState={setState} />);
  fireEvent.click(screen.getByText('Submit'));
  expect(setState).toHaveBeenCalled();  // 测试内部实现
});
// ✅ 应该测试
test('should show success message after submit', async () => {
  render(<Component />);
  fireEvent.click(screen.getByText('Submit'));
  await waitFor(() => {
    expect(screen.getByText('提交成功')).toBeVisible();  // 测试用户可见行为
  });
});

// 反模式 2：不验证具体值的测试
// ❌ expect(Array.isArray(result)).toBe(true);  // 太弱
// ✅ expect(result).toEqual([{ status: 'active' }]);  // 验证具体逻辑

// 反模式 3：过度 Mock
// ❌ 把 UserService、formatDate、Logger、Analytics 全 mock 了
// ✅ 只 mock 外部边界（如 HTTP 层）

// 反模式 4：缺少 await 的异步测试
// ❌ expect(screen.getByText('Loaded')).toBeInTheDocument();  // 异步数据还没加载
// ✅ expect(await screen.findByText('Loaded')).toBeInTheDocument();  // findBy 自动重试
```

**面试要点：** 能够列举并解释 AI 生成测试的典型问题；不仅仅指出问题，还能给出正确的测试写法；理解"测试行为而非实现"的核心原则。

---

## 七、AI 驱动的代码重构

### 35. 如何使用 AI 进行安全的大规模重构？

```typescript
// Step 1: 影响分析（重构前）
/*
Prompt:
分析 src/utils/api.ts 中 fetchData 函数的：
1. 所有调用方（搜索整个项目）
2. 调用方传入的参数类型
3. 调用方对返回值的消费方式
4. 与它耦合的测试文件
5. 推荐的重构策略
*/

// Step 2: 制定重构方案
/*
基于影响分析，设计零风险重构方案：
- 现有: function fetchData(url, options): Promise<any>
- 目标: 重写为类型安全的 fetchJson<T>，使用 Result<T> 类型
- 保持向后兼容或给出迁移路径
*/

// Step 3: 分步执行
// Step 1: 创建新函数，保持旧函数不变
// Step 2: 迁移一个简单调用方做验证
// Step 3: 运行测试确认
// Step 4: 逐步迁移剩余调用方
// Step 5: 移除旧函数
// Step 6: 更新测试

// Step 4: 自动化验证
/*
Prompt:
请为我生成一个脚本，自动检查：
1. 项目中是否还有对 fetchData 的引用
2. 新代码是否都使用 fetchJson
3. 没有新的类型错误
*/
```

**面试要点：** 不只说"用 AI 重构"；能展示影响分析 → 方案设计 → 分步执行 → 验证的完整流程；强调安全重构的关键是"小步快跑 + 每步验证"。

---

### 36. 如何使用 AI 进行"代码清理"——删除无用代码和依赖？

```typescript
// 死代码检测（Dead Code Detection）
/*
Prompt:
分析 src/components/ 目录，找出：
1. 未被任何其他文件导入的组件
2. 导出了但未被使用的函数/类型
3. 只有测试文件在引用的代码（可能是过时的）
4. 被注释掉的代码块
*/

// 依赖分析（Dependency Analysis）
/*
Prompt:
分析 package.json 中的依赖：
1. 哪些包使用频率低（< 3 次引用）？
2. 哪些包有更轻量的替代品？
3. 估算每个包对 bundle size 的贡献

AI 可能给出的建议：
| 包名 | 使用次数 | 大小 | 建议 |
| moment | 2 处 | 230KB | 替换为 date-fns (~5KB) |
| lodash | 4 处 | 70KB | 改为 lodash-es (tree-shakable) |
| qs | 1 处 | 8KB | 替换为 URLSearchParams (原生) |
*/
```

**面试要点：** 理解 AI 在代码清理中的独特价值——能快速扫描整个项目找到关联；能给出具体的数据（使用次数、包大小）辅助决策。

---

## 八、MCP 协议与工具集成

### 37. 什么是 MCP（Model Context Protocol）？它解决了 AI Coding 中的什么问题？

**MCP（Model Context Protocol）** 是由 Anthropic 提出的开放协议，标准化了 AI 模型与外部工具/数据源之间的交互方式。

```
没有 MCP 的世界（N×M 集成问题）：
每个 AI 应用都要自己实现与每个工具的集成

MCP 解决后（标准化）：
AI 应用只需实现 MCP Client，工具只需实现 MCP Server
→ 一次开发，所有 AI 应用都能使用
```

**MCP 核心概念：**

```typescript
interface MCPServer {
  // 工具（Tools）: AI 可以调用的函数
  tools: {
    name: string;
    description: string;          // AI 理解这个工具做什么
    inputSchema: JSONSchema;      // 参数定义
    handler: (input: any) => any; // 实际执行逻辑
  }[];

  // 资源（Resources）: AI 可以读取的上下文数据
  resources: {
    uri: string;
    name: string;
    description: string;
    mimeType: string;
  }[];

  // 提示词模板（Prompts）: 预定义的对话模板
  prompts: {
    name: string;
    description: string;
    arguments?: { name: string; description: string }[];
  }[];
}
```

**为什么重要：**

```
1. 互操作性：不同 AI 工具都能用同一套 MCP Server
2. 安全性：工具权限可精细控制，不暴露完整系统
3. 可组合：多个 MCP Server 可同时连接（Git + Database + API）
4. 可扩展：社区可以贡献新的 MCP Server（如 Figma MCP, Jira MCP）
```

**面试要点：** 理解 MCP 解决的是集成标准化问题；知道 Tools/Resources/Prompts 三个核心概念；能举例说明 MCP 在前端开发中的应用。

---

### 38. 你会如何设计一个前端开发专用的 MCP Server？

**前端开发 MCP Server 设计：**

```typescript
const FrontendDevMCPServer = {
  tools: [
    { name: 'generate_component', description: '根据需求描述生成 React 组件' },
    { name: 'search_npm_package', description: '搜索 npm 包信息（下载量、大小、兼容性）' },
    { name: 'add_dependency', description: '安全安装 npm 依赖（检查兼容性和大小）' },
    { name: 'find_similar_code', description: '在项目中搜索语义相似的代码片段' },
    { name: 'analyze_component_deps', description: '分析组件的依赖树和影响范围' },
    { name: 'analyze_test_coverage', description: '分析测试覆盖率，识别未覆盖的关键路径' },
    { name: 'check_browser_support', description: '检查 API 的浏览器兼容性' },
  ],
  resources: [
    { uri: 'project://structure', name: '项目结构', mimeType: 'application/json' },
    { uri: 'project://dependencies', name: '依赖清单', mimeType: 'application/json' },
    { uri: 'project://tsconfig', name: 'TypeScript 配置', mimeType: 'application/json' },
    { uri: 'docs://component-library', name: '组件库文档', mimeType: 'text/markdown' },
  ],
  prompts: [
    { name: 'create_feature', description: '创建新功能的完整流程' },
    { name: 'fix_bug', description: 'Bug 修复流程' },
    { name: 'refactor_component', description: '组件重构流程' },
  ],
};
```

**面试要点：** 能设计出有实用价值的 MCP Server；工具涵盖代码生成、包管理、分析、测试、兼容性等前端关键场景；理解 Resources 和 Prompts 的作用。

---

### 39. MCP 的 Tool 和 Resource 有什么区别？各自在前端 AI Coding 中的应用场景？

| 维度 | Tool | Resource |
|------|------|----------|
| **本质** | 可执行的操作 | 可读取的数据 |
| **类比** | 函数（有副作用） | 文件/数据库（只读） |
| **AI 操作** | 调用执行 | 读取内容 |
| **典型示例** | git_commit, create_file | git://log, file://README.md |
| **安全性** | 需要审批（可能改系统） | 相对安全（只读） |
| **使用时机** | 需要"做某事" | 需要"了解某事" |

```typescript
// ✅ Tool 场景：创建/修改文件、执行命令、git 操作、搜索
// 示例：自动修复 lint 错误的 Tool
{
  name: 'auto_fix_lint',
  description: '运行 ESLint --fix 自动修复可修复的问题',
  handler: async () => {
    const result = execSync('npx eslint src/ --fix --format json').toString();
    return JSON.parse(result);
  }
}

// ✅ Resource 场景：配置文件、文档、静态数据、覆盖率报告
// 示例：把 i18n 翻译文件作为 Resource 暴露
{
  uri: 'i18n://zh-CN',
  name: '中文翻译文件',
  mimeType: 'application/json',
  handler: () => fs.readFileSync('public/locales/zh-CN.json', 'utf-8'),
}
// AI 可理解已有哪些翻译 key，生成代码时直接使用现有的
```

**面试要点：** 清晰区分 Tool（做操作）和 Resource（读数据）；能举例说明前端开发中各自的典型应用；理解 Resource 的只读特性使其更安全。

---
## 九、前端 AI Coding 实战

### 40. 在实际项目中，如何使用 AI Coding 工具提升开发效率？

**日常 AI Coding 工作流：**

```typescript
// 最佳实践 1：有计划地使用 AI（而非无目的地问）
// ❌ 低效模式："帮我写个表单" "不对，换个样式" "加个验证" → 来回多轮，越改越乱
// ✅ 高效模式：先 5 分钟想清楚再一次性 Prompt

// 最佳实践 2：把 AI 当作"结对编程伙伴"而非"代码生成器"
// 不止生成代码："这段代码有什么潜在问题？""有没有更简洁的写法？"
// "这个函数的时间复杂度是多少？""如果输入是 null 会怎样？"

// 最佳实践 3：善用 @-mention 引用代码
// "@UserProfile.tsx 这个组件有什么性能问题？"
// "@folder:src/api 这个文件夹下的所有 API 调用有哪些共同的模式可以提取？"

// 最佳实践 4：建立自己的 Prompt 片段库
const myPrompts = {
  createComponent: '请实现 {componentName} 组件...',
  addTests: '请为以下代码生成完整的单元测试...',
  reviewCode: '请审查以下代码...',
  explainCode: '请解释这段代码的逻辑...',
};

// 最佳实践 5：AI + Git 结合
// Commit 前：AI 审查暂存区变更
// Commit Message：AI 根据 diff 生成
// PR 描述：AI 根据分支 diff 生成

// 最佳实践 6：遇到问题先问 AI，再查文档
// 传统：Google → 看好几篇文章 → 综合理解
// AI 时代：直接给出针对你代码的修复方案
```

**效率提升数据（业内经验总结）：**

```
任务类型          传统耗时    AI辅助耗时   提升
─────────────────────────────────────────────
CRUD API 层       30 分钟     5 分钟      83%
组件编写          45 分钟     10 分钟     78%
单元测试编写       20 分钟     3 分钟      85%
类型定义          10 分钟     1 分钟      90%
工具函数          15 分钟     2 分钟      87%
Bug 修复          60 分钟     20 分钟     67%
代码审查          30 分钟     10 分钟     67%
文档编写          20 分钟     3 分钟      85%
─────────────────────────────────────────────
综合              230 分钟    54 分钟     77%
```

**面试要点：** 不只说"用 AI 能提效"；能给出具体的工作流实践和效率数据；展示的是"开发者 + AI"的高效协作模式，而非"AI 替代开发者"。

---

### 41. 如何在 AI Coding 中保持代码的"人可维护性"？

```typescript
// 策略 1：在 Prompt 中要求可维护性
/*
1. 变量/函数命名使用完整单词（不用缩写）
2. 函数不超过 30 行（超过则拆分）
3. 每个函数/组件有 JSDoc 注释
4. 避免"魔法数字"——使用命名常量
5. 导出函数在文件顶部有统一说明
*/

// 策略 2：生成后立即重构
// AI 生成: function p(d: any[]): any[] { return d.filter((i: any) => i.active); }
// 人工/AI 重构:
/**
 * 过滤出所有状态为 active 的实体
 * @param entities - 待过滤的实体列表
 * @returns 所有 active 状态的实体
 */
function filterActiveEntities<T extends { active: boolean }>(entities: T[]): T[] {
  return entities.filter(entity => entity.active);
}

// 策略 3：建立代码审查门禁
const maintainabilityChecks = {
  maxFunctionLength: 30,        // 函数不超过 30 行
  maxCyclomaticComplexity: 10,  // 圈复杂度不超过 10
  maxParameterCount: 4,         // 函数参数不超过 4 个
  requireJSDoc: true,           // 导出函数必须有 JSDoc
  bannedWords: ['any', 'magic'], // 禁止 any 类型和魔法数字
};

// 策略 4：让 AI 写自解释的代码
// ❌ 需要注释解释的代码：
// Check if user is admin and has active subscription
if (u.r === 'admin' && s.s === 'active' && !s.e) { /* ... */ }
// ✅ 自解释的代码：
const isAdmin = user.role === 'admin';
const hasActiveSubscription = subscription.status === 'active' && !subscription.expired;
if (isAdmin && hasActiveSubscription) { /* ... */ }
```

**面试要点：** 理解 AI Coding 的一大风险是生成"难以维护的代码"；能提出具体的可维护性保障策略。

---

### 42. 如何处理 AI Coding 中"代码不一致"的问题？

**问题表现：** 同一个项目的三个 API 函数，AI 前后生成了三种不同写法（async/await、Promise.then、axios）。

**根本原因：** AI 的每次生成是独立的（除非上下文窗口包含了之前的代码）。没有看到之前生成的代码 → 根据训练数据随机选择一种风格。

**四层防御体系：**

```
Layer 1: 规则文件定义风格（治本）
  .cursorrules 中写明："所有 HTTP 请求必须使用 apiClient，使用 async/await，返回 Result<T>"

Layer 2: Few-Shot 示例（纠正特定交互）
  "参考 src/api/userApi.ts 中 fetchUsers 的写法，实现 fetchProducts"

Layer 3: 上下文窗口包含之前代码（让 AI 能看到）
  在 Cursor 中：打开之前写的 API 文件标签页
  在 Chat 中：@-mention 引用之前的文件

Layer 4: 代码审查 + ESLint（自动化检查）
  ESLint 规则强制 async/await → CI 直接报错 → AI 无法绕过工具检查
```

**面试要点：** 理解不一致的根本原因（AI 无状态生成）；给出规则文件 + Few-Shot + 上下文 + 自动化检查的四层方案。

---

### 43. 在 AI Coding 中使用 TypeScript 有哪些额外优势？

**TypeScript — AI Coding 的最佳搭档：**

```typescript
// 优势 1：类型作为"可执行规范"
// 不给 AI 写文字描述，直接给类型定义
interface RawUserData {
  first_name: string;
  last_name: string;
  birth_date: string;     // "YYYY-MM-DD"
  email_address: string;
  phone_number: string;   // "+8613800138000"
  tags: string;           // comma-separated
}

interface UserProfile {
  displayName: string;      // "LastName FirstName"
  age: number;              // calculated
  email: string;
  phone: { countryCode: string; number: string };
  tags: string[];
}
// AI 精确知道输入和输出，不会偏差

// 优势 2：类型缩小 AI 的"创意空间"
// 没有类型约束 → AI 可以返回 { success: true } 或 { ok: true } 或 { status: 200 }
// 有类型约束 → AI 的输出被锁定
type OrderResult =
  | { success: true; orderId: string; totalAmount: number }
  | { success: false; errorCode: string; errorMessage: string };
// AI 必须返回 OrderResult 类型的值

// 优势 3：类型错误 = AI 生成代码的即时反馈
// AI 生成代码 → TypeScript 编译器检查 → 类型错误立即暴露问题

// 优势 4：泛型约束让 AI 生成更通用的代码
function usePagination<T extends { id: string | number }>(
  fetchFn: (page: number) => Promise<{ data: T[]; totalPages: number }>,
  pageSize = 20
) { /* ... */ }
// 类型安全：T 必须有 id 属性
```

**面试要点：** 理解 TypeScript 在 AI Coding 中的独特价值——类型是"可执行规范"；能演示类型驱动开发的工作流。

---

### 44. 如何处理 AI 生成代码中的"过度工程化"问题？

```typescript
// 症状 1：不必要的抽象
// 需求："显示一个欢迎信息"
// ❌ AI 可能生成：WelcomeConfig interface + WelcomeMessageBuilder class + 多态 + 策略模式
// ✅ 适度工程：
function getWelcomeMessage(userName?: string): string {
  const name = userName || 'User';
  const hour = new Date().getHours();
  const greeting = hour < 12 ? 'Good morning' : hour < 18 ? 'Good afternoon' : 'Good evening';
  return `${greeting}, ${name}!`;
}

// 症状 2：为了未来可能的变更而设计的"灵活性"
// 需求："写一个按钮组件"
// ❌ AI 可能定义 8 种 variant + 6 种 size + 4 种 icon position + 5 种 animation + 5 级 elevation
// ✅ YAGNI (You Ain't Gonna Need It)
type ButtonVariant = 'primary' | 'secondary' | 'danger';
type ButtonSize = 'sm' | 'md' | 'lg';

// Prompt 防御策略：
/*
"使用 YAGNI 原则实现 [功能]：
1. 只实现当前明确需要的功能
2. 不要为'未来可能的扩展'添加代码
3. 先让代码能工作，再考虑优化
4. 抽象只在出现第三次重复时才提取
5. 如果可以用 10 行代码完成，不要写 50 行"
*/
```

**面试要点：** 识别 AI 生成代码中过度工程化的典型表现；在 Prompt 中嵌入 YAGNI/KISS 原则；理解"简单"比"灵活"更重要的工程哲学。

---

### 45. 使用 AI Coding 时，如何避免"技术债"的累积？

**AI Coding 产生的技术债类型：**

```
1. 质量债：AI 生成的功能正确但代码质量差
2. 文档债：没有对应文档/注释
3. 测试债：没有对应测试
4. 一致性债：不同时间生成的代码风格不统一
5. 依赖债：引入了不必要的第三方依赖
6. 理解债：开发者不完全理解 AI 生成的代码逻辑
```

**AI Coding DoD (Definition of Done)：**

```
AI 生成的代码，必须满足以下条件才算"完成"：

□ 通过 TypeScript strict 编译
□ 通过 ESLint 检查（0 个 error）
□ 有对应的单元测试（至少覆盖关键路径）
□ 有必要的 JSDoc / 注释
□ 开发者能口头解释代码逻辑（理解检查）
□ 没有引入项目未使用的新依赖
□ 代码风格与相邻文件一致
□ 所有 TODO/FIXME 转为正式任务
□ 通过了 AI 代码审查（用审查 Prompt 自检）
```

**面试要点：** 不只关注"当下能跑"；能识别 AI Coding 产生的各类技术债；有系统的 DoD 标准来阻断技术债累积。

---

## 十、Vibe Coding 与 Agent 模式

### 46. 什么是"Vibe Coding"？它和传统 AI Coding 有什么区别？

**Vibe Coding** 是由 Andrej Karpathy 提出的概念，指一种更极端的 AI 编程范式：开发者用自然语言描述需求，完全由 AI 生成代码，开发者几乎不直接阅读/修改代码。

```
传统编程：      写代码 → 调试 → 重构
AI Coding：     描述需求 → AI 生成 → 人审查 → 微调
Vibe Coding：   描述需求 → AI 生成 → 运行 → 不行就重新描述 → AI 重新生成

对比：
             代码控制权    代码理解度    适用场景
传统编程       完全          完全         所有
AI Coding      高            高           生产代码
Vibe Coding    低            低           原型/实验
```

**推荐的混合模式：**

```
- 原型阶段：用 Vibe Coding 快速验证想法
- 开发阶段：切换回传统 AI Coding（审查代码）
- 维护阶段：理解代码 + 精准修改
```

**面试要点：** 理解 Vibe Coding 的定义和边界；能区分 Vibe Coding 和工程化 AI Coding 的适用场景；认识到 Vibe Coding 适合原型不适合生产。

---

### 47. 什么是 AI Coding 中的"Agent 模式"？它与 Chat 模式有何不同？

**Agent 模式** = AI 不仅回答问题，还能自主规划、执行多步骤任务，调用工具（文件系统、终端、浏览器），迭代修正。

```
Chat 模式（一问一答）:
你："帮我实现一个分页组件"
AI：返回代码
你：复制粘贴，手动创建文件，运行看效果，再问，再改...

Agent 模式（自主多步执行）:
你："在 Dashboard 页面添加一个分页的数据表格"
AI Agent 自动：
  1. 读取 Dashboard.tsx 了解现有结构
  2. 搜索项目中是否有现成的 Table 组件
  3. 创建 OrderTable.tsx
  4. 在 Dashboard.tsx 中添加 import 和使用
  5. 运行 lint 检查
  6. 如果失败 → 读取错误信息 → 修复 → 重新检查
  7. 报告完成
```

**Agent 的关键能力：**

| 能力 | 说明 |
|------|------|
| 任务规划 | 将复杂任务拆分为子任务，确定执行顺序 |
| 工具使用 | 调用文件系统、终端、LSP、Git 等工具 |
| 上下文收集 | 主动搜索项目代码、读取相关文件 |
| 自我纠错 | 执行失败后分析错误，调整方案重试 |
| 多文件编辑 | 理解文件间依赖，做协调的批量修改 |

**面试要点：** 清晰区分 Chat（一问一答）和 Agent（自主多步执行）；理解 Agent 的工具调用、规划、自我纠错能力。

---

### 48. Agent 模式的"工具调用"是如何实现的？

```typescript
// Agent 工具调用的底层机制

// 1. 工具定义（给 AI 看的描述）
const tools = [
  {
    name: 'read_file',
    description: '读取文件内容。用于理解现有代码。',
    parameters: {
      type: 'object',
      properties: {
        filePath: { type: 'string', description: '文件路径' },
        startLine: { type: 'number', description: '起始行号（可选）' },
      },
      required: ['filePath'],
    },
  },
  {
    name: 'write_file',
    description: '创建或覆盖文件。',
    parameters: {
      properties: {
        filePath: { type: 'string' },
        content: { type: 'string' },
      },
      required: ['filePath', 'content'],
    },
  },
  {
    name: 'replace_in_file',
    description: '在现有文件中执行字符串替换。',
    parameters: {
      properties: {
        filePath: { type: 'string' },
        old_str: { type: 'string', description: '要被替换的原始字符串（必须精确匹配）' },
        new_str: { type: 'string', description: '替换后的新字符串' },
      },
      required: ['filePath', 'old_str', 'new_str'],
    },
  },
  {
    name: 'search_content',
    description: '使用正则表达式搜索文件内容。',
    parameters: {
      properties: {
        pattern: { type: 'string', description: '搜索模式（支持正则）' },
      },
      required: ['pattern'],
    },
  },
  {
    name: 'execute_command',
    description: '执行终端命令。',
    parameters: {
      properties: {
        command: { type: 'string' },
        requires_approval: { type: 'boolean', default: false },
      },
      required: ['command'],
    },
  },
];

// 2. 工具调用流程
/*
User: "在 DashboardPage 添加一个显示最近订单的表格"

Agent 内部思考：
我需要：
1. 先看 DashboardPage 现在的代码 → read_file
2. 看项目的 Order 类型定义 → search_content 找 interface Order
3. 看有没有现成的 Table 组件 → search_content 找 Table export
4. 创建 OrderTable 组件 → write_file
5. 修改 DashboardPage → replace_in_file
*/
```

**面试要点：** 理解 Agent 工具调用的底层机制（定义 → 触发 → 执行 → 反馈）；能列举前端开发中最常用的工具类型；知道工具调用的安全考虑。

---

### 49. 多 Agent 协作在 AI Coding 中如何工作？

**多 Agent 架构** = 将复杂任务分配给多个专业 Agent 并行或串行协作。

```
单 Agent 模式:
User → [Agent] → Result

多 Agent 模式:
User → [Orchestrator Agent] → [Agent 1: 前端]
                             → [Agent 2: 后端]
                             → [Agent 3: 测试]
                             → [Agent 4: 审查]
                             → [Orchestrator Agent] → Result
```

**应用场景：**

```typescript
// 场景 1：全栈功能开发
/*
任务："添加用户头像上传功能"
Orchestrator 拆分：
  子任务 1 (前端 Agent): 实现头像上传 UI（拖拽、预览、裁剪）
  子任务 2 (后端 Agent): 实现文件上传 API + 图片处理
  子任务 3 (类型 Agent): 定义共享的 TypeScript 类型
  子任务 4 (测试 Agent): 为前后端编写集成测试
各 Agent 并行工作，Orchestrator 整合检查
*/

// 场景 2：代码审查 + 修复
/*
任务："审查 PR #42 并自动修复发现的问题"
Review Agent: 审查代码 → 生成问题列表
Fix Agent: 接收问题列表 → 逐个修复
Test Agent: 验证修复后测试通过
*/

// 场景 3：文档生成
/*
Analyzer Agent: 扫描所有组件 → 提取 Props、用途、示例
Writer Agent: 根据分析结果 → 生成 Markdown/Storybook 文档
Reviewer Agent: 检查文档准确性 → 提出修正
*/
```

**多 Agent 的优势与挑战：**

```
✅ 任务并行，速度更快
✅ 专业 Agent 各司其职，质量更高
✅ 复杂任务可分解为简单子任务

⚠️ Agent 间通信开销
⚠️ 子任务结果可能冲突
⚠️ Token 消耗增加
```

**面试要点：** 理解单 Agent 和多 Agent 的架构区别；能给出具体的多 Agent 协作场景。

---

### 50. 在 Agent 模式下，如何设置合适的"权限边界"？

```typescript
enum PermissionLevel {
  AUTO = 'auto',           // 无需审批，自动执行
  NOTIFY = 'notify',       // 自动执行但通知用户
  CONFIRM = 'confirm',     // 需要用户确认
  FORBIDDEN = 'forbidden', // 完全禁止
}

const permissionConfig = {
  // 读取操作（自动执行）
  'read_file':           PermissionLevel.AUTO,
  'search_content':      PermissionLevel.AUTO,
  'list_dir':            PermissionLevel.AUTO,

  // 写入操作（需要确认）
  'write_file':          PermissionLevel.CONFIRM,
  'replace_in_file':     PermissionLevel.CONFIRM,
  'delete_file':         PermissionLevel.CONFIRM,

  // Shell 命令（分级）
  'npm_run_test':        PermissionLevel.AUTO,    // 安全
  'npm_run_lint':        PermissionLevel.AUTO,    // 安全
  'npm_install':         PermissionLevel.CONFIRM, // 需确认
  'git_commit':          PermissionLevel.CONFIRM, // 需确认
  'git_push':            PermissionLevel.CONFIRM, // 需确认
  'git_push_force':      PermissionLevel.FORBIDDEN, // 禁止

  // 危险操作（完全禁止）
  'rm_rf':               PermissionLevel.FORBIDDEN,
  'modify_git_config':   PermissionLevel.FORBIDDEN,
  'access_env_secrets':  PermissionLevel.FORBIDDEN,
};
```

**目录级别权限控制：**

```typescript
const directoryPermissions = {
  'src/':      { read: true, write: true, delete: true },
  'tests/':    { read: true, write: true, delete: true },
  '.env':      { read: false, write: false }, // 敏感文件完全禁止
  '.git/':     { read: false, write: false },
  'package.json': { read: true, write: PermissionLevel.CONFIRM },
  'tsconfig.json': { read: true, write: PermissionLevel.CONFIRM },
};
```

**面试要点：** 能设计分层权限模型（自动/通知/确认/禁止）；理解"最小权限原则"在 Agent 模式下的应用。

---

## 十一、安全、隐私与合规

### 51. 使用 AI Coding 工具时，有哪些安全和隐私风险？

**风险全景图：**

```
风险 1：代码泄露到云端
- 大多数 AI Coding 工具将代码发送到云端 API 处理
- 包含 API Key、内部架构的代码可能被发送
缓解：使用 .cursorignore 排除敏感文件、企业版 Copilot、本地模型

风险 2：AI 生成代码引入安全漏洞
- 硬编码密钥、XSS、代码注入
缓解：安全审查 Prompt + 自动化扫描

风险 3：AI 训练数据污染
- 训练数据可能包含 GPL 等强 Copyleft 许可证的代码
缓解：代码引用过滤、许可证检查

风险 4：过度依赖 AI
- AI 工具不可用时，开发者能力下降
缓解：保持核心技能、定期"无 AI 日"练习
```

**面试要点：** 不只看到 AI Coding 的好处，也要认清风险；能分四个维度分析（泄露、漏洞、污染、依赖）；给出实际可操作的缓解方案。

---

### 52. 企业环境下引入 AI Coding 工具需要做哪些准备工作？

```markdown
## 企业引入 AI Coding 准备清单

### 1. 合规评估
- [ ] 确认工具隐私政策（数据是否用于训练）
- [ ] 确认数据存储位置（合规要求）
- [ ] 法务审核工具使用条款

### 2. 安全配置
- [ ] 配置代码过滤规则（排除敏感文件）
- [ ] 设置网络访问策略
- [ ] 配置 SSO 和访问控制

### 3. 技术准备
- [ ] 选择适合团队的工具
- [ ] 搭建内部规则文件库（组织级 + 项目级）
- [ ] CI/CD 集成（AI 审查、自动化测试）

### 4. 流程调整
- [ ] 更新 Code Review 流程（增加 AI 生成标记）
- [ ] 定义 AI 代码的验收标准
- [ ] 建立 AI Coding 最佳实践文档

### 5. 团队培训
- [ ] AI Coding 基础使用 + Prompt Engineering
- [ ] 安全风险意识
- [ ] 代码审查技巧培训

### 6. 度量与改进
- [ ] 设定效果度量（开发速度、质量、满意度）
- [ ] 定期收集开发者反馈
```

**面试要点：** 不只关注技术层面；能从合规、安全、流程、培训、度量五个维度给出系统化方案。

---

## 十二、团队协作与工程化

### 53. 在团队中如何统一 AI Coding 的最佳实践？

```typescript
// 团队 AI Coding 统一实践框架

// 1. 统一的规则文件（共享到 Git 仓库）
// .cursor/rules/ → 纳入 Git，所有人共用

// 2. 统一的 Prompt 模板库
// docs/ai-prompts/
// ├── create-component.md
// ├── add-tests.md
// ├── fix-bug.md
// └── code-review.md

// 3. AI 生成的代码规范标记
/**
 * @ai-generated 2024-01-15
 * @reviewed-by Zhang San
 * @ai-tool Cursor + Claude 3.5 Sonnet
 */
export function SomeComponent() { /* AI 生成的代码 */ }

// 4. PR 中标记 AI 生成内容
/*
PR Template 新增一节：
## AI Coding 声明
- [ ] 本 PR 包含 AI 生成的代码（占比 ___%）
- [ ] AI 生成的代码已经过人工审查
- [ ] 所有 AI 生成的代码有对应测试
*/

// 5. 团队知识共享
// - 每周 AI Coding 经验分享（15 分钟）
// - 维护团队 AI Prompt 最佳实践库
// - 定期更新规则文件
```

**面试要点：** 展示团队协作视角（而非个人使用）；能给出可操作的统一实践方案。

---

### 54. 如何评估 AI Coding 工具的 ROI（投资回报率）？

```typescript
// ROI 评估维度
interface ROIMetrics {
  // 效率指标
  efficiency: {
    codeGenerationSpeed: number;     // 代码生成速度提升 %
    codeReviewTimeSaved: number;     // 代码审查节省的时间
    testCreationTimeSaved: number;   // 测试编写节省的时间
    bugFixTimeReduced: number;       // Bug 修复时间减少
    onboardingTimeReduced: number;   // 新人上手时间减少
  };
  // 质量指标
  quality: {
    bugRateChange: number;           // Bug 率变化
    testCoverageChange: number;      // 测试覆盖率变化
    securityVulnerabilityChange: number; // 安全漏洞数变化
  };
  // 满意度指标
  satisfaction: {
    developerSatisfaction: number;   // 开发者满意度
    focusTimeIncrease: number;       // 专注时间增加
  };
}

// 简化 ROI 计算（以 10 人团队为例）
/*
投入：
- Copilot Business: $19/人/月 × 10 × 12 = $2,280/年
- 培训时间: 20 小时/人 × $80/小时 × 10 = $16,000（一次性）

产出（保守估计 30% 效率提升）：
- 节省开发时间: 2000 小时/年 × 30% = 600 小时/年
- 价值: 600 × $80 = $48,000/年

ROI ≈ ($48,000 - $2,280) / ($2,280 + $16,000) ≈ 250%（约 2.5 倍回报）
*/

// 定性收益（难以量化但很重要）：
// 1. 开发者满意度提升
// 2. 新人上手速度加快
// 3. 知识壁垒降低
// 4. 技术债减少
// 5. 文档质量提升
```

**面试要点：** 不只说"AI 提效"；能给出量化的 ROI 分析框架；理解投入和产出。

---

### 55. AI Coding 对前端工程师的技能要求有什么变化？

```typescript
// 重要性下降的技能（AI 擅长）:
// ↓ 手写样板代码（CRUD、表单、列表）
// ↓ 记忆 API 语法细节
// ↓ 基础算法实现
// ↓ 写 CSS 布局
// ↓ 正则表达式编写

// 重要性上升的技能（AI 不擅长）:
// ↑ 架构设计（系统级思考）
// ↑ Prompt Engineering（精准描述需求）
// ↑ 代码审查（审查 AI 生成代码的能力）
// ↑ 调试与问题定位（AI 生成的 bug 更难发现）
// ↑ 产品思维（理解业务、用户需求）
// ↑ 安全审计（AI 可能引入安全问题）
// ↑ 性能分析（AI 可能生成性能差的代码）
// ↑ 测试策略设计（决定测什么、怎么测）
// ↑ 跨领域协作（AI 无法替代人际沟通）
// ↑ 技术选型与权衡决策
// ↑ 上下文工程（设计和管理 AI 的上下文）
```

**未来前端开发者的新能力模型：**

```typescript
const futureFrontendDevSkills = {
  // 核心编程能力（依然重要，但重心转移）
  core: {
    architectureDesign: '架构设计能力',     // ↑↑↑
    debugging: '调试与问题定位',            // ↑
    codeReview: '代码审查（含 AI 代码）',   // ↑↑
    systemThinking: '系统性思维',            // ↑↑
  },
  // AI 协作能力（全新）
  aiCollaboration: {
    promptEngineering: 'Prompt 工程',             // 核心新技能
    contextEngineering: '上下文工程',             // 核心新技能
    aiCodeReview: 'AI 生成代码审查',             // 核心新技能
    toolOrchestration: '工具编排（MCP/Agent）',   // 进阶新技能
    aiSafetyAwareness: 'AI 安全意识',            // 必要新技能
  },
  // 软技能（持续重要）
  soft: {
    communication: '沟通能力',
    productThinking: '产品思维',
    businessUnderstanding: '业务理解',
    mentorship: '指导能力',
  },
};
```

**面试准备建议：**

```
1. 不要停止基础学习：AI 不能替代对 JS/TS/CSS/React 的深入理解
2. 练习 Prompt Engineering：这是一个可训练的技能
3. 培养架构思维：AI 能生成代码，但架构需要人设计
4. 强化调试能力：AI 生成的 bug 更难排查
5. 学会审查 AI 代码：这不只是看代码，是看"AI 可能犯的错误"
6. 保持好奇心：AI 工具每周都在进化
```

**面试要点：** 不只是列出技能；能区分"上升"和"下降"的技能方向；理解 AI 是增强而非替代开发者。

---

> **本文共计 55 题**，覆盖 AI Coding 的 12 大核心模块：工具生态、Prompt 工程、代码生成策略、上下文工程、代码审查、AI 测试、代码重构、MCP 协议、前端实战、Vibe Coding & Agent、安全隐私、团队工程化。
>
> 每道题均结合前端开发实际场景，包含详细代码示例、对比表格和面试要点，帮助面试者在 AI Coding 领域建立系统化的知识体系。
>
> 已生成的面试题文件：Vue 100 题 + React 105 题 + 前端通用 100 题 + **AI Coding 55 题 = 360 题**，全面覆盖现代前端开发面试的高频考点。

---

*最后更新：2026年7月*
