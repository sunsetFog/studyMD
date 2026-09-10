# Claude Code 知识点总结

## 1. 什么是 Claude Code

Claude Code 是 Anthropic 官方的命令行编程智能体（agentic coding tool），直接在终端里运行。它能读取整个代码库、执行命令、编辑文件、搜索网络，并和开发者对话协作完成任务。

```
$ claude
Claude Code 启动，进入交互式会话（REPL）
```

## 2. 工作原理：Agent 循环与内置工具

基于「感知-决策-行动」的 Agent 循环工作：读取上下文 → 调用内置工具 → 观察结果 → 决定下一步，直到任务完成。

```
Claude 自主调用：Read 读文件 → Edit 修改 → Bash 跑测试 → 循环直至完成
```

内置工具包括：Read（读文件）、Write（写文件）、Edit（编辑）、Bash（执行命令）、Grep（搜索）、Glob（查找文件）、WebSearch、WebFetch 等。

## 3. 安装方式一：npm 全局安装

通过 npm 安装，需要 Node.js 18+。

```
npm install -g @anthropic-ai/claude-code
```

## 4. 安装方式二：原生安装脚本

官方原生安装脚本安装独立二进制（不依赖 Node），并注册 claude 命令。

```
curl -fsSL https://claude.ai/install.sh | bash
```

## 5. 升级与健康检查

```
claude update        # 更新到最新版本
claude doctor        # 检查安装、PATH、配置是否正常
claude --version     # 查看版本
```

## 6. 登录与认证

```
claude login                          # 浏览器 OAuth 登录
export ANTHROPIC_API_KEY=sk-ant-xxxx  # 或使用 API Key
claude logout                         # 退出登录
```

## 7. 基本用法

```
claude                                # 交互模式
claude "帮我看看这个项目的架构"        # 一次性 headless 调用
```

## 8. 内建 Slash 命令

交互会话中以 `/` 开头的内置命令。

```
/help        帮助          /clear       清空当前会话上下文
/compact     压缩上下文     /config      打开交互式配置界面
/init        生成 CLAUDE.md  /memory     管理记忆
/model       切换模型      /permissions 管理权限
/status      会话状态      /resume      继续之前的会话
/cost        会话费用      /usage       订阅用量
/login       登录          /logout      退出登录
/doctor      环境诊断      /mcp         管理 MCP 服务器
/agents      管理子代理     /add-dir     把目录加入工作区
```

## 9. CLAUDE.md：项目记忆文件

项目根目录的 CLAUDE.md 是给 Claude 看的「项目说明书」，每次启动自动加载，适合写构建命令、代码风格、项目结构、易错点。

```
# CLAUDE.md
运行测试：npm test
代码风格：使用 TypeScript，变量命名用 camelCase
注意：不要修改 public/ 下的文件
```

## 10. CLAUDE.md 的层级

支持全局、项目、子目录三级，就近覆盖、逐层合并。

```
~/.claude/CLAUDE.md               # 用户级，全局生效
<project>/CLAUDE.md               # 项目级
<project>/src/utils/CLAUDE.md     # 子目录级，在该目录下工作时加载
```

## 11. 自动记忆系统（MEMORY.md）

完成任务后 Claude 会把可复用经验写入记忆文件，存放于 `~/.claude/projects/<项目路径编码>/`，核心文件为 MEMORY.md，用 `/memory` 查看编辑。

```
/memory
# 打开记忆管理器，查看/编辑跨会话记忆
# 记忆文件位置：~/.claude/projects/<项目路径编码>/MEMORY.md
```

## 12. Skills：可复用的技能包

Skill 把「某类任务的完整操作方法」打包成 SKILL.md，放在 `.claude/skills/<名称>/`，用 frontmatter 的 name/description 描述，Claude 根据描述自动判断何时使用。

```
.claude/skills/unit-test/SKILL.md
---
name: unit-test
description: 为指定函数生成单元测试。当用户要求"写测试"时使用。
---
(步骤正文：如何定位函数、用什么测试框架、断言风格等)
```

## 13. 子代理 Subagents

`.claude/agents/*.md` 定义专门子代理角色，有独立 system prompt、可选模型和工具白名单，通过 Task 工具委派任务。

```
.claude/agents/code-reviewer.md
---
name: code-reviewer
description: 专门做安全审查。在合并代码前用。
tools: Read, Grep, Bash
model: claude-sonnet-4-5
---
你是一位资深安全工程师，重点检查注入、越权、敏感信息泄露……
```

## 14. Hooks：生命周期钩子

Hooks 在特定事件发生时执行自定义命令，如工具调用前拦截、任务结束后通知。事件名包括 PreToolUse、PostToolUse、UserPromptSubmit、Notification、Stop、SessionStart、SessionEnd、PreCompact 等。

```
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash",
        "hooks": [ { "type": "command", "command": "echo \"run bash\" >> /tmp/h.log" } ] }
    ],
    "Stop": [
      { "hooks": [ { "type": "command", "command": "say 'task done'" } ] }
    ]
  }
}
```

## 15. MCP 服务器

MCP（Model Context Protocol）是标准化工具协议，让 Claude Code 接入外部工具/数据源（数据库、浏览器、GitHub、内部 API 等）。

```
# 添加 MCP 服务器
claude mcp add github https://api.githubcopilot.com/mcp/

# stdio 类型（本地进程）
claude mcp add --transport stdio filesystem -- npx -y @modelcontextprotocol/server-filesystem /tmp
```

```
# 配置文件方式（.mcp.json，随项目共享）
{
  "mcpServers": {
    "my-tools": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "my-mcp-server"]
    }
  }
}
```

## 16. 权限系统：allow / deny / ask

三级权限规则控制工具调用：allow 放行、deny 拒绝、ask 询问，配置在 settings.json 的 permissions 字段。

```
{
  "permissions": {
    "allow": ["Bash(npm run test)", "Read", "Edit"],
    "deny":  ["Bash(rm -rf *)", "Bash(git push --force)"],
    "ask":   ["Bash(git push)"]
  }
}
```

## 17. 权限模式

会话可切换不同权限模式，通过 `--permission-mode` 启动或 Shift+Tab 切换。

```
claude --permission-mode plan                  # 只读规划
claude --permission-mode bypassPermissions     # 跳过所有确认（高危）
```

权限模式：default（逐条询问）、acceptEdits（自动接受编辑）、plan（只读规划）、bypassPermissions（跳过所有确认）。

## 18. settings.json 配置层级

配置优先级：企业托管 > `~/.claude/settings.json`（用户级）> `.claude/settings.json`（项目共享）> `.claude/settings.local.json`（项目本地，不提交）。

```
.claude/settings.json
{
  "model": "claude-sonnet-4-5",
  "env": { "MY_VAR": "value" },
  "permissions": { "allow": ["Read", "Edit"] },
  "defaultMode": "default",
  "hooks": { "Stop": [ { "hooks": [ { "type": "command", "command": "echo done" } ] } ] }
}
```

## 19. CLI 常用标志

```
claude --continue                      # 继续上一个会话
claude --resume <session-id>           # 恢复指定会话（-r 简写）
claude --model claude-sonnet-4-5       # 指定模型
claude --permission-mode plan          # 指定权限模式
claude --add-dir ../shared             # 额外把目录加进工作区
claude --mcp-config .mcp.json          # 指定 MCP 配置
claude --settings ./custom-settings.json
```

## 20. Headless / 脚本模式

`-p`（--print）非交互式处理一次提示，适合脚本化、管道和 CI，配合 `--output-format` 输出结构化结果。

```
claude -p "解释这个 repo 的架构" --output-format text
claude -p "为 utils.js 写单测并运行" --output-format json
cat bug-report.txt | claude -p "根据内容修复 bug"
```

## 21. 跳过全部权限确认（高危）

`--dangerously-skip-permissions` 跳过所有权限确认，用于完全自动化的 CI/脚本场景。切勿在不受信任的项目或 root 下使用。

```
claude -p "部署到生产" --dangerously-skip-permissions
```

## 22. 交互界面与快捷键

```
Enter       提交输入        Shift+Enter   换行
Esc         中断任务        Tab           切换对话/后台任务
Shift+Tab   切换权限模式     Ctrl+R        恢复历史会话
```

编辑文件后可用 diff 视图逐条接受/拒绝改动。

## 23. Plan 模式（规划模式）

Plan 模式下 Claude 只能读和规划，不能改文件、跑命令，适合先出方案、人工确认后再执行。

```
claude --permission-mode plan
# 或会话内 Shift+Tab 切换到 plan
# "先给我一个重构方案，不要动手"
```

## 24. IDE 集成

官方提供 VS Code 扩展和 JetBrains 插件，可在编辑器侧边栏使用。

```
# VS Code：扩展市场搜索 "Claude Code" 安装
# JetBrains：Settings > Plugins > 搜索 "Claude Code"
# 或直接在 IDE 内置终端启动
claude
```

## 25. Bash 工具与后台任务

Claude 通过 Bash 工具执行终端命令，长耗时任务可放后台，让会话继续做别的事。

```
Bash(command="npm run build")                    # 前台执行
Bash(command="npm run e2e", run_in_background=true)  # 后台执行
```

## 26. 动态工作流 / 多代理编排

Workflow 工具可动态生成并并行运行大量子代理，把大任务拆成多路并行的小任务再汇总。

```
Task(描述="审查 api/ 目录下所有控制器")   → 并行子代理
Task(描述="审查 services/ 目录")         → 并行子代理
# 最后汇总每路结果
```

## 27. 模型与成本管理

```
/model      # 选择模型（Opus/Sonnet/Haiku/Max 等别名）
/cost       # 查看会话累计费用
/usage      # 查看订阅用量
/compact    # 压缩长上下文继续对话
```

## 28. 安全与信任

首次进入新目录会询问是否信任；对不可信仓库的 CLAUDE.md 应审查。建议生产环境别用 `--dangerously-skip-permissions`，敏感命令加入 deny 规则。

```
{
  "permissions": {
    "deny": ["Bash(*)", "Bash(rm -rf *)", "Bash(sudo *)"]
  }
}
```

## 29. .claude/ 目录结构

```
<project>/.claude/
  settings.json        项目共享配置
  settings.local.json  项目本地配置（不入库）
  commands/*.md        自定义斜杠命令
  agents/*.md          自定义子代理
  skills/<name>/SKILL.md  自定义技能
<project>/CLAUDE.md    项目说明
<project>/.mcp.json    项目 MCP 配置
~/.claude/
  settings.json        用户级配置
  CLAUDE.md            全局说明文件
  projects/            会话记录与自动记忆
  skills/  agents/     个人技能与子代理
```

## 30. 自定义斜杠命令

`.claude/commands/*.md` 定义自己的斜杠命令，正文即命令的指令，适合固化高频工作流。

```
.claude/commands/release-check.md
---
description: 发布前安全检查
argument-hint: <可选参数>
---
执行：1) 跑测试 2) 检查未提交改动 3) 报告结果
```

## 31. 插件 Plugins

插件是可分发的能力包，可包含 skills、agents、MCP、hooks、命令，通过 marketplace 安装。

```
claude plugin install <插件名>@<marketplace>
/plugins        # 查看和管理已安装插件
```

## 32. 输出样式与脚本输出格式

```
claude -p "重构说明" --output-format json   # 机器可读输出
claude --output-style concise              # 套用简洁排版风格
# 配置：settings.json 里 outputStyle: "concise"
```

## 33. 常用环境变量

```
export ANTHROPIC_API_KEY=sk-ant-...       # API Key 认证
export ANTHROPIC_MODEL=claude-sonnet-4-5  # 默认模型
export ANTHROPIC_BASE_URL=https://...     # 走代理/网关
export HTTP_PROXY=http://127.0.0.1:7890   # 代理
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1  # 关闭非必要网络
```

## 34. 常见工作流与技巧

```
claude /init            # 先让 AI 写项目说明
claude --continue       # 第二天接着昨天的会话
claude -p "..." --dangerously-skip-permissions  # CI 自动化
```

推荐组合：`/init` 生成 CLAUDE.md → `/compact` 控制上下文 → 长任务放后台 → git 分支配合实验；会话中途可 `/model` 换模型、`/clear` 重新开始；疑似 bug 用 `/doctor` 诊断、`/bug` 反馈。

---

> 说明：具体模型 ID 随版本演进（如 claude-sonnet-4-5、claude-opus-4-5 系列），以 `/model` 当前可选列表为准；自动记忆、Output Styles、动态工作流属于较新且仍在演进的功能，不同版本细节略有差异，建议以 `claude --help` 和 `/help` 为准。
