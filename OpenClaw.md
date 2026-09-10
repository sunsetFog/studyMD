# OpenClaw 知识点总结

## 1. 简介

OpenClaw 是一个开源、自托管、多渠道的 AI 智能体网关（agent gateway）。它把 AI 模型接入 WhatsApp、Telegram、Discord、Slack、微信等消息平台，让用户用自然语言下达指令，智能体自主规划并执行任务——不只是「对话」，而是「动手做事」。

```
用户发消息 → Gateway 接收 → Agent 规划 → 调用 Skills 执行 → 结果返回用户
```

## 2. 历史背景

原名 Clawdbot（2025 年 11 月发布，玩 Claude + 龙虾钳的谐音梗），后短暂改名 Moltbot，2026 年 1 月底因商标纠纷定名 OpenClaw。由奥地利开发者 Peter Steinberger 创建，GitHub 星标数迅速超过 React，成为史上增长最快的开源项目之一。

## 3. 与对话式 AI 的区别

| 维度 | 对话式 AI | OpenClaw |
|------|-----------|----------|
| 形态 | 一问一答 | 常驻、自主执行 |
| 能力 | 生成文本 | 发邮件、订日程、跑命令 |
| 运行 | 单次调用 | 长驻 Gateway 进程 |
| 定位 | 聊天助手 | 数字员工 / 私人助理 |

## 4. 核心架构

OpenClaw 架构分四层：Gateway（网关）、Skills（技能）、Memory（记忆）、Agent Brain（智能体大脑）。

```
Gateway     中央枢纽，负责调度、路由、健康检查、鉴权
Skills      能力单元，教智能体如何使用工具
Memory      记忆层，保存跨会话上下文
Agent Brain 模型层，负责规划与推理
```

## 5. 安装与初始化

用 `openclaw onboard` 交互式初始化，或用非交互模式一键配置。

```
# 交互式配置向导
openclaw onboard

# 非交互式（跳过健康检查，需 --accept-risk）
openclaw onboard --non-interactive --accept-risk --skip-health
```

支持 macOS、Windows（推荐 WSL）、Linux、云服务器，甚至树莓派。

## 6. 启动 Gateway

Gateway 是 OpenClaw 的入口进程，启动后提供 CLI 和 Web 控制台。

```
# 启动网关
openclaw gateway start

# 重启
openclaw gateway restart          # Linux
openclaw gateway stop && openclaw gateway start   # Windows

# macOS 后台运行
pkill -f openclaw && nohup openclaw gateway start > ~/.openclaw/logs/gateway.log 2>&1 &
```

## 7. 配置文件 openclaw.json

主配置在 `~/.openclaw/openclaw.json`，包含模型、网关、渠道、技能等设置。

```
{
  "agents": {
    "defaults": { "model": "anthropic/claude-sonnet-4-5" }
  },
  "gateway": {
    "mode": "local",           // 网关模式
    "bind": "127.0.0.1",       // 绑定地址
    "auth": { "token": "xxx" } // 鉴权 token
  },
  "channels": {
    "telegram": { "botToken": "..." },
    "discord": { "token": "..." }
  },
  "skills": {
    "autoLoad": true           // 自动加载技能
  }
}
```

## 8. 模型 Provider 配置

支持 35+ 模型提供商：Anthropic、OpenAI、Ollama、Bedrock、OpenRouter、通义千问、智谱等。

```
# 云端模型
agents.defaults.model = "anthropic/claude-opus-4-5"
# 或
agents.defaults.model = "openai/gpt-4o"

# 本地模型（Ollama）
agents.defaults.model = "ollama/llama3"
```

```
# 本地模型自托管，数据不出本机
# 需先安装 Ollama 并拉取模型
ollama pull llama3
```

## 9. 渠道 Channels

把智能体接入各种消息平台，用户直接在聊天软件里发指令。

```
# 支持的渠道：WhatsApp、Telegram、Discord、Slack、
# iMessage、Signal、Teams、微信、Matrix 等

{
  "channels": {
    "telegram": { "botToken": "123456:ABC..." },
    "discord": { "token": "..." },
    "signal": { "..." }
  }
}
```

## 10. SOUL.md：定义「是谁」

SOUL.md 定义智能体的角色、人格、行为准则——回答「这个智能体是谁」。

```
# ~/.openclaw/agents/my-agent/SOUL.md
你是一个私人助理，名叫小爪。
- 语气友好、简洁
- 处理邮件、日程、提醒
- 不确定时先询问再行动
```

## 11. SKILL.md：定义「怎么做」

Skill 是能力单元，用 SKILL.md 教智能体如何执行某个工作流——回答「怎么做事」。

```
~/.openclaw/skills/<skill-name>/SKILL.md
```

```
# SKILL.md 结构 = YAML frontmatter + Markdown 正文
# frontmatter：元信息（何时用、怎么触发）
# 正文：操作步骤（做什么、怎么调用脚本、怎么格式化输出）
```

## 12. SKILL.md frontmatter

```
---
name: weather-query              # 小写+连字符，须与目录名一致
description: 查询城市天气。当用户问天气时使用。
user-invocable: true             # 用户可否手动触发
triggers:                        # 触发提示词
  - 天气
  - 温度
metadata:
  version: "1.0.0"
  author: "tom"
  required_binaries:
    - python3
---
```

## 13. Skill 目录结构

每个技能是一个目录，SKILL.md 是入口，可带脚本和参考资料。

```
~/.openclaw/skills/
  weather-query/
    SKILL.md            # 技能定义（必需）
    query.py            # 辅助脚本
    references/
      api-doc.md        # 参考资料
```

## 14. 安装技能（ClawHub）

ClawHub 是官方技能市场（「AI 技能的 App Store」），700+ 技能、24 个分类。

```
# 方式一：ClawHub CLI
npm i -g clawhub
clawhub login                    # 登录
clawhub search "csv"             # 搜索
clawhub install org/csv-summary --version 1.2.3   # 安装并锁定版本

# 方式二：OpenClaw 原生命令
openclaw skills search browser
openclaw skills install browser-control
openclaw skills verify browser-control --card
openclaw skills list --installed
```

## 15. 手动安装技能

```
# ZIP 导入：解压到技能目录，装依赖
cp skill.zip ~/.openclaw/skills/ && unzip skill.zip
cd ~/.openclaw/skills/<skill-name>
npm install          # 或 pip3 install -r requirements.txt

# Git 克隆
git clone <repo-url> ~/.openclaw/skills/<skill-name>
```

```
# 技能目录有文件监听器，自动扫描，无需手动注册
# 重启 runtime 后新技能会被索引加载
```

## 16. 给 Agent 分配技能

技能装好后要分配给具体的智能体角色才生效。

```
# 配置文件方式（编辑 openclaw.json）
{
  "agents": {
    "web_agent": { "skills": ["agent-browser"] }
  }
}
```

```
# CLI 方式
openclaw agents add web_agent --skill agent-browser

# 查看已分配技能
openclaw agents get web_agent --show-skills
```

## 17. 创建、测试、发布技能

```
# 创建（脚手架）
openclaw skills create my-skill

# 开发：在 skills/my-skill/ 下编辑 SKILL.md

# 本地测试
openclaw skills test my-skill

# 发布到 ClawHub
openclaw skills publish my-skill
```

## 18. Agents 多智能体

OpenClaw 支持多智能体，每个 agent 有独立的 SOUL.md、记忆、技能和模型。

```
# 不同角色用不同 agent
# 例如：客服 agent、开发 agent、个人助理 agent

{
  "agents": {
    "customer_service": {
      "model": "anthropic/claude-haiku-4-5",
      "skills": ["crm-lookup", "ticket-triage"]
    },
    "personal_assistant": {
      "model": "anthropic/claude-sonnet-4-5",
      "skills": ["calendar", "email", "reminder"]
    }
  }
}
```

## 19. Memory 记忆层

记忆层保存跨会话的上下文，让智能体「记得」用户偏好和历史。

```
# 记忆让多轮、跨会话协作成为可能
# 例如：记住用户的地址、常用联系人、偏好设置
# 与 SOUL.md（静态角色）不同，记忆是动态积累的
```

## 20. ACP 外部智能体

OpenClaw 可接入外部 harness（ACP agents），复用 Codex、Claude Code、Gemini CLI 等现成工具。

```
# 支持 14+ 外部 harness
# 把 Codex / Claude Code / Gemini CLI 作为子智能体挂载
# 由 OpenClaw 统一调度
```

## 21. 消息发送

用 CLI 直接发消息触发智能体，常用于测试和自动化。

```
# 发送消息触发技能（验证技能是否正常加载）
openclaw message send --content "北京今天天气怎么样？"
```

## 22. Web 控制台

Gateway 启动后提供 Web 控制台（dashboard），可视化管理智能体、技能、渠道。

```
openclaw gateway start
# 浏览器访问控制台
# 可查看技能状态：All / Ready / Needs Setup / Disabled
```

| 状态 | 含义 |
|------|------|
| Ready | 已配置，可用 |
| Needs Setup | 已装但缺 API key/配置 |
| Disabled | 已装但关闭 |

## 23. 技能配置与密钥

技能可能需要 API key 或配置项，安装时提示填写，安全存储。

```
# API key 存进 agent 的 vault（密钥库）
# 绝不硬编码在 SKILL.md 或工作区里

# 配置项在安装流程（install recipe）或控制台详情页提示
```

## 24. 安全：把技能当第三方代码

ClawHub 曾出现恶意技能事件（2025 年 ClawHavoc 事件，发现 847 个恶意技能），安装前必须审查。

```
# 安装前检查
1. 作者信誉：只看已验证发布者
2. 安装量：优先 1000+ 或 5000+ 安装
3. 更新时间：避免 6 个月以上未更新的废弃技能
4. 读 SKILL.md 源码：看有没有未知 URL 或数据外泄
5. 权限范围：天气技能不该要文件系统/shell 权限
```

## 25. 安全最佳实践

```
# 锁定精确版本，不用 latest
clawhub install org/csv-summary --version 1.2.3

# API key 存 vault，不硬编码
# 最小权限原则
# 含代码的技能先扫描
npx carapace scan <skill-dir>
# 或 VirusTotal 扫描

# 生产前沙箱测试
```

## 26. 本地模型（隐私优先）

接入 Ollama 等本地模型，数据不出本机。

```
# 安装 Ollama 并拉取模型
ollama pull llama3

# openclaw.json 配置本地模型
{
  "agents": {
    "defaults": { "model": "ollama/llama3" }
  }
}
```

## 27. 自动化与插件

OpenClaw 支持插件、自动化 hooks、cron 任务、webhook，实现定时和事件驱动。

```
# cron：定时执行
# webhook：外部事件触发
# hooks：生命周期钩子
# 插件：扩展功能
```

## 28. 生态

围绕 OpenClaw 形成了完整生态。

```
# NVIDIA NemoClaw：安全/沙箱层
# ClawHub：技能市场（700+ 技能）
# Microsoft ClawPilot / Project Lobster：内部探索
# 各社区技能仓库、教程
```

## 29. 常用命令速查

| 命令 | 作用 |
|------|------|
| openclaw onboard | 初始化配置 |
| openclaw gateway start/stop/restart | 网关管理 |
| openclaw message send | 发送消息 |
| openclaw skills search/install/list | 技能管理 |
| openclaw skills create/test/publish | 技能开发 |
| openclaw agents add/get | 智能体管理 |
| clawhub login/search/install | 市场操作 |

## 30. 快速上手流程

```
# 1. 安装并初始化
openclaw onboard

# 2. 配置模型和渠道（编辑 openclaw.json）

# 3. 启动网关
openclaw gateway start

# 4. 安装技能
openclaw skills install browser-control

# 5. 分配给 agent
openclaw agents add my-agent --skill browser-control

# 6. 在消息平台或 CLI 发指令
openclaw message send --content "帮我查一下明天的会议"
```

---

> 说明：OpenClaw 是 2025 年底出现、2026 年初爆发的新项目，命令和配置项可能随版本快速演进，建议以官方文档和 `openclaw --help` 为准。
