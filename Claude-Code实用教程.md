# Claude Code 实用教程

> 完整指南 - 常用命令、代码编辑、高级技巧与实战案例

**版本**: 3.37.3 | **更新时间**: 2026年4月

---

## 📚 目录

- [快速开始](#%E5%BF%AB%E9%80%9F%E5%BC%80%E5%A7%8B)
- [安装与配置](#%E5%AE%89%E8%A3%85%E4%B8%8E%E9%85%8D%E7%BD%AE)
- [核心命令详解](#%E6%A0%B8%E5%BF%83%E5%91%BD%E4%BB%A4%E8%AF%A6%E8%A7%A3)
- [斜杠命令完整列表](#%F0%9F%93%8B-%E6%96%9C%E6%9D%A1%E5%91%BD%E4%BB%A4%E5%AE%8C%E6%95%B4%E5%88%97%E8%A1%A8)
- [代码编辑功能](#%F0%9F%92%BB-%E4%BB%A3%E7%A0%81%E7%BC%96%E8%BE%91%E5%8A%9F%E8%83%BD)
- [高级技巧](#%F0%9F%94%A7-%E9%AB%98%E7%BA%A7%E6%8A%80%E5%B7%A7)
- [实战案例](#%F0%9F%8E%AE-%E5%AE%9E%E6%88%98%E6%A1%88%E4%BE%8B)
- [快捷键大全](#%E2%8C%A8%EF%B8%8F-%E5%BF%AB%E6%8D%B7%E9%94%AE%E5%A4%A7%E5%85%A8)
- [最佳实践](#%E2%9C%85-%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)
- [故障排查](#%F0%9F%94%8D-%E6%95%85%E9%9A%9C%E6%8E%92%E6%9F%A5)

---

## 🚀 快速开始

### 基础使用

```bash
# 启动交互式会话
claude

# 直接执行查询
claude "解释这个代码库"

# 继续上一次会话
claude -c

# 恢复指定会话
claude -r <session-id>
```

### 三种交互方式

1. **斜杠命令** (`/命令`): 执行内置功能
2. **自然语言输入**: 直接描述需求
3. **Bash命令** (`!命令`): 直接执行Shell命令

---

## ⚙️ 安装与配置

### 安装方法

```bash
# NPM安装（推荐）
npm install -g @anthropic-ai/claude-code

# 原生安装（macOS/Linux/WSL）
curl -sL https://install.anthropic.com | sh

# 原生安装（Windows PowerShell）
irm https://claude.ai/install.ps1 | iex

# 验证安装
claude --version

# 更新到最新版本
claude update
```

### 系统要求

- Node.js 18+（推荐最新LTS版本）
- Windows、macOS（Apple Silicon和Intel）、Linux（x86_64和arm64）
- 4GB RAM最低要求

### 认证方式

```bash
# Claude账户登录（推荐）
claude
# 然后在会话中使用 /login

# API密钥方式
export ANTHROPIC_API_KEY=your_api_key_here
claude
```

### 配置文件位置

| 级别 | macOS/Linux | Windows | 作用域 | Git |
|------|-------------|---------|--------|-----|
| **项目级** | `.claude/` | `.claude\` | 团队 | ✅ |
| **个人级** | `~/.claude/` | `%USERPROFILE%\.claude\` | 个人（所有项目） | ❌ |

**优先级**: 项目级 > 个人级

### .claude/ 文件夹结构

```
.claude/
├── CLAUDE.md              # 本地记忆（git忽略）
├── settings.json          # 团队设置（提交）
├── settings.local.json    # 个人设置覆盖（不提交）
├── agents/                # 自定义代理
├── commands/              # 斜杠命令
├── hooks/                 # 事件脚本
├── rules/                 # 自动加载规则
└── skills/                # 知识模块
```

---

## 🎯 核心命令详解

### CLI命令

| 命令 | 描述 | 示例 |
|------|------|------|
| `claude` | 启动交互式REPL | `claude` |
| `claude "query"` | 带初始提示启动 | `claude "解释这个项目"` |
| `claude -p "query"` | 通过SDK查询后退出 | `claude -p "解释这个函数"` |
| `claude -c` | 继续最近对话 | `claude -c` |
| `claude -r "<id>" "query"` | 按ID恢复会话 | `claude -r "abc123" "完成这个PR"` |
| `claude update` | 更新到最新版本 | `claude update` |
| `claude mcp` | 配置MCP服务器 | 见MCP文档 |

### 常用CLI标志

| 标志 | 描述 | 示例 |
|------|------|------|
| `--add-dir` | 添加额外工作目录 | `claude --add-dir ../apps ../lib` |
| `--allowedTools` | 允许的工具列表 | `"Bash(git log:*)" "Read"` |
| `--disallowedTools` | 禁止的工具列表 | `"Bash(rm:*)" "Bash(sudo:*)"` |
| `--print, -p` | 打印响应（非交互模式） | `claude -p "查询"` |
| `--system-prompt` | 替换整个系统提示 | `claude --system-prompt "你是Python专家"` |
| `--append-system-prompt` | 追加到默认提示 | `claude --append-system-prompt "总是使用TypeScript"` |
| `--output-format` | 输出格式（text/json/stream-json） | `claude -p "query" --output-format json` |
| `--verbose` | 启用详细日志 | `claude --verbose` |
| `--max-turns` | 限制代理轮次 | `claude -p --max-turns 3 "查询"` |
| `--model` | 设置模型 | `claude --model sonnet` |
| `--permission-mode` | 权限模式（plan等） | `claude --permission-mode plan` |
| `--dangerously-skip-permissions` | 跳过权限提示（谨慎使用） | `claude --dangerously-skip-permissions` |

---

## 📋 斜杠命令完整列表

### 会话管理

| 命令 | 功能 | 说明 |
|------|------|------|
| `/help` | 显示帮助 | 列出所有可用命令 |
| `/clear` | 清除对话历史 | 重置上下文 |
| `/compact [指令]` | 压缩对话 | 可选聚焦指令 |
| `/exit` | 退出REPL | 或使用Ctrl+D |
| `/status` | 会话状态 | 显示版本、模型、账户和连接状态 |
| `/context` | 详细Token分解 | 查看上下文使用情况 |
| `/cost` | 显示Token使用统计 | 查看费用信息 |
| `/usage` | 显示计划使用限制 | 仅订阅计划 |
| `/config` | 打开设置界面 | 配置选项卡 |
| `/doctor` | 检查安装健康状态 | 诊断工具 |

### 文件操作

| 命令 | 功能 | 说明 |
|------|------|------|
| `/add-dir` | 添加额外工作目录 | 允许访问其他目录 |
| `/init` | 初始化项目 | 生成CLAUDE.md指南 |
| `/memory` | 编辑记忆文件 | 管理项目记忆 |
| `/permissions` | 查看/更新权限 | 配置IAM权限 |

### Git集成

| 命令 | 功能 | 说明 |
|------|------|------|
| `/review` | 请求代码审查 | 审查代码变更 |
| `/pr_comments` | 查看PR评论 | 拉取请求评论 |

### 模型与设置

| 命令 | 功能 | 说明 |
|------|------|------|
| `/model` | 选择/更改模型 | 切换AI模型 |
| `/vim` | 进入vim模式 | 切换插入和命令模式 |
| `/terminal-setup` | 安装Shift+Enter绑定 | 仅iTerm2和VSCode |

### 高级功能

| 命令 | 功能 | 说明 |
|------|------|------|
| `/agents` | 管理自定义子代理 | 专业化任务 |
| `/mcp` | 管理MCP服务器连接 | OAuth认证 |
| `/bug` | 报告错误 | 发送对话给Anthropic |
| `/login` | 切换Anthropic账户 | 更换登录账户 |
| `/logout` | 登出Anthropic账户 | 退出登录 |
| `/rewind` | 回滚对话和/或代码 | 撤销操作 |

### 自定义斜杠命令

#### 项目级命令

位置: `.claude/commands/`

```bash
# 创建项目命令
mkdir -p .claude/commands
echo "分析此代码的性能问题并建议优化:" > .claude/commands/optimize.md
```

#### 个人级命令

位置: `~/.claude/commands/`

```bash
# 创建个人命令
mkdir -p ~/.claude/commands
echo "审查此代码的安全漏洞:" > ~/.claude/commands/security-review.md
```

#### 命令参数

使用占位符传递动态值：

```bash
# 所有参数使用 $ARGUMENTS
echo '修复 #$ARGUMENTS 问题，遵循我们的编码标准' > .claude/commands/fix-issue.md

# 使用: /fix-issue 123 high-priority

# 单独参数使用 $1, $2 等
echo '审查 PR #$1，优先级 $2，分配给 $3' > .claude/commands/review-pr.md

# 使用: /review-pr 456 high alice
```

#### Bash命令执行

在斜杠命令运行前执行bash命令：

```markdown
---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
description: 创建git提交
---

## 上下文

- 当前git状态: !`git status`
- 当前git差异: !`git diff HEAD`
- 当前分支: !`git branch --show-current`
- 最近提交: !`git log --oneline -10`

## 你的任务

基于以上更改，创建单个git提交。
```

#### 文件引用

使用 `@` 前缀引用文件：

```markdown
# 引用特定文件
审查 @src/utils/helpers.js 中的实现

# 引用多个文件
比较 @src/old-version.js 和 @src/new-version.js
```

#### Frontmatter配置

```yaml
---
allowed-tools: Bash(git add:*), Bash(git status:*)
argument-hint: [message]
description: 创建git提交
model: claude-3-5-haiku-20241022
---

创建git提交，消息: $ARGUMENTS
```

---

## 💻 代码编辑功能

### 文件引用语法

```
@path/to/file.ts        → 引用文件
@agent-name             → 调用代理
!shell-command          → 运行shell命令
```

### IDE快捷键

| IDE | 快捷键 |
|-----|--------|
| VS Code | `Alt+K` |
| JetBrains | `Cmd+Option+K` |

### 代码审查

```bash
# 审查代码
/review

# 审查特定文件
审查 @src/auth/login.ts 中的代码

# 审查变更
审查最近的代码变更，关注安全性和性能
```

### 代码重构

```bash
# 识别需要重构的代码
查找代码库中已弃用的API使用

# 获取重构建议
建议如何重构 utils.js 以使用现代JavaScript特性

# 安全应用更改
重构 utils.js 以使用ES2024特性，同时保持相同行为

# 验证重构
为重构后的代码运行测试
```

### 测试集成

```bash
# 识别未测试代码
查找 NotificationsService.swift 中未覆盖测试的函数

# 生成测试脚手架
为通知服务添加测试

# 添加有意义的测试用例
为通知服务中的边缘条件添加测试用例

# 运行并验证测试
运行新测试并修复任何失败
```

### 文档生成

```bash
# 识别未记录代码
查找auth模块中没有适当JSDoc注释的函数

# 生成文档
为 auth.js 中的未记录函数添加JSDoc注释

# 审查和增强
改进生成的文档，添加更多上下文和示例

# 验证文档
检查文档是否符合我们的项目标准
```

---

## 🔧 高级技巧

### 计划模式（Plan Mode）

**何时使用**:
- 多步骤实现（需要编辑多个文件）
- 代码探索（在更改前彻底研究代码库）
- 交互式开发（与Claude迭代方向）

**如何使用**:

```bash
# 在会话中切换到计划模式
# 按 Shift+Tab 循环权限模式
# Normal → Auto-Accept → Plan Mode

# 以计划模式启动新会话
claude --permission-mode plan

# 在计划模式下运行查询
claude --permission-mode plan -p "分析认证系统并建议改进"
```

**示例：规划复杂重构**

```bash
claude --permission-mode plan

> 我需要将我们的认证系统重构为使用OAuth2。创建详细的迁移计划。
```

Claude将分析当前实现并创建综合计划。使用后续提示进行细化：

```bash
> 向后兼容性如何？
> 我们应该如何处理数据库迁移？
```

**配置计划模式为默认**:

```json
// .claude/settings.json
{
  "permissions": {
    "defaultMode": "plan"
  }
}
```

### 扩展思考（Extended Thinking）

**何时使用**:
- 复杂架构决策
- 棘手bug调试
- 规划多步骤实现
- 理解复杂代码库
- 评估不同方法的权衡

**如何使用**:

```bash
# 启用扩展思考
> 我需要为我们的API实现一个新的OAuth2认证系统。深入思考在我们的代码库中实现此功能的最佳方法。
```

Claude将显示其思考过程（灰色斜体文本）。

**思考深度控制**:

- `think` - 触发基本扩展思考
- `think hard` / `keep thinking` - 触发更深入的思考
- `Tab` - 在会话中切换思考开关

**永久启用**:

```bash
export MAX_THINKING_TOKENS=200000
```

### 模型选择指南

| 任务 | 模型 | 努力程度 |
|------|------|----------|
| 重命名、样板代码、测试生成 | Haiku | low |
| 功能开发、调试、重构 | Sonnet | medium-high |
| 架构、安全审计 | Opus | high-max |

**动态模型切换**:

```bash
# 会话开始（默认Sonnet）
claude

# 遇到复杂功能
> "实现OAuth2流程和PKCE"
/model opus  # 切换到深度推理

# 功能完成，回到常规
/model sonnet  # 速度+成本优化
```

**成本影响**:

| 模型 | 输入 | 输出 | 使用场景 |
|------|------|------|----------|
| Opus | $15/MTok | $75/MTok | 复杂推理（10-20%任务） |
| Sonnet | $3/MTok | $15/MTok | 大多数开发（70-80%任务） |
| Haiku | $0.25/MTok | $1.25/MTok | 简单验证（5-10%任务） |

### 上下文管理

**状态栏监控**:

```
Model: Sonnet | Ctx: 89.5k | Cost: $2.11 | Ctx(u): 56.0%
```

**监控 `Ctx(u):`**:
- >70% → 使用 `/compact`
- >85% → 使用 `/clear`

**上下文阈值**:

| 上下文% | 状态 | 操作 |
|---------|------|------|
| 0-50% | 绿色 | 自由工作 |
| 50-70% | 黄色 | 有选择性 |
| 70-90% | 橙色 | 立即 `/compact` |
| 90%+ | 红色 | 需要 `/clear` |

**按症状操作**:

| 症状 | 操作 |
|------|------|
| 响应简短 | `/compact` |
| 频繁遗忘 | `/clear` |
| >70%上下文 | `/compact` |
| 任务完成 | `/clear` |

### 权限模式

| 模式 | 编辑 | 执行 |
|------|------|------|
| Default | 询问 | 询问 |
| acceptEdits | 自动 | 询问 |
| Plan Mode | ❌ | ❌ |
| dontAsk | 仅在允许规则中 | 仅在允许规则中 |
| bypassPermissions | 自动 | 自动（仅CI/CD） |

**Shift+Tab** 切换模式

### MCP服务器

| 服务器 | 用途 |
|--------|------|
| **Serena** | 索引+会话记忆+符号搜索 |
| **grepai** | 语义搜索+调用图分析 |
| **Context7** | 库文档 |
| **Sequential** | 结构化推理 |
| **Playwright** | 浏览器自动化 |
| **Postgres** | 数据库查询 |
| **doobidoo** | 语义记忆+多客户端+知识图谱 |

**管理MCP连接**:

```bash
/mcp  # 查看所有配置的MCP服务器
```

### 自定义代理（Agents）

创建 `.claude/agents/my-agent.md`:

```markdown
---
name: my-agent
description: 在[触发条件]时使用
model: sonnet
tools: Read, Write, Edit, Bash
---

# 指令在这里
```

**使用代理**:

```bash
# 查看可用代理
/agents

# 自动使用
审查我的代码变更的安全问题

# 显式请求
使用code-reviewer代理检查auth模块
```

### Hooks（钩子）

**Bash (macOS/Linux)**:

```bash
#!/bin/bash
INPUT=$(cat)
# 处理JSON输入
exit 0  # 0=继续, 2=阻止
```

**PowerShell (Windows)**:

```powershell
$input = [Console]].In.ReadToEnd() | ConvertFrom-Json
# 处理JSON输入
exit 0  # 0=继续, 2=阻止
```

### 远程控制（v2.1.51+）

**Pro/Max专用** - 研究预览功能

```bash
# 从终端启动（新会话）
claude remote-control

# 或在活动会话中:
/rc  # 或 /remote-control
```

**从手机/平板/浏览器连接**:

1. 扫描**QR码**（启动后按空格键）
2. 或在浏览器/Claude移动应用中打开**会话URL**
3. 或: `/mobile` → 显示App Store + Play Store链接

**已知限制**:
- 一次只能有一个会话
- 斜杠命令在远程端失效 → 从本地终端使用
- 终端必须保持打开
- 网络超时约10分钟断开 → 会话过期

---

## 🎮 实战案例

### 案例1: 理解新代码库

```bash
# 1. 导航到项目根目录
cd /path/to/project

# 2. 启动Claude Code
claude

# 3. 询问高级概述
> 给我这个代码库的概述

# 4. 深入特定组件
> 解释这里使用的主要架构模式
> 关键数据模型是什么？
> 如何处理认证？
```

### 案例2: 高效修复Bug

```bash
# 1. 分享错误给Claude
> 我运行npm test时看到一个错误

# 2. 询问修复建议
> 建议几种修复user.ts中@ts-ignore的方法

# 3. 应用修复
> 更新user.ts以添加你建议的空检查
```

### 案例3: 创建Pull Request

```bash
# 1. 总结你的更改
> 总结我对认证模块所做的更改

# 2. 用Claude生成PR
> 创建一个PR

# 3. 审查和细化
> 增强PR描述，添加更多关于安全改进的上下文

# 4. 添加测试详情
> 添加关于如何测试这些更改的信息
```

### 案例4: 使用图片工作

```bash
# 1. 添加图片到对话
# - 拖放图片到Claude Code窗口
# - 复制图片并用ctrl+v粘贴到CLI
# - 提供图片路径: "分析这张图片: /path/to/image.png"

# 2. 询问Claude分析图片
> 这张图片显示什么？
> 描述这个截图中的UI元素
> 这个图中有问题元素吗？

# 3. 使用图片作为上下文
> 这是错误的截图。是什么原因导致的？
> 这是我们当前的数据库架构。我们应该如何为新功能修改它？

# 4. 从视觉内容获取代码建议
> 生成匹配这个设计稿的CSS
> 什么HTML结构能重建这个组件？
```

### 案例5: CI/CD集成

```bash
# 非交互执行
claude -p "分析这个文件" src/api.ts

# JSON输出
claude -p "审查" --output-format json

# 经济模型
claude -p "lint" --model haiku

# 自动接受
claude -p "修复拼写错误" --dangerously-skip-permissions
```

### 案例6: 测试驱动开发

```bash
# 首先生成测试
claude --tdd "用户认证模块"

# 更改后运行测试
claude --auto-test "重构数据库层"

# 实现前编写测试
claude --test-first "支付处理"

# 覆盖率分析
claude --coverage "识别未测试代码"
```

### 案例7: 多文件操作

```bash
# 编辑多个文件
claude --multi-edit "更新所有API端点"

# 批量重构
claude --refactor "提取公共逻辑"

# 处理多个文件
find . -name "*.js" -exec claude -p "分析此文件的bug: {}" \; > bug_report.txt
```

---

## ⌨️ 快捷键大全

### 导航

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+P` | 文件选择器 |
| `Ctrl+Shift+F` | 在文件中搜索 |
| `Ctrl+G` | 跳转到行 |
| `Ctrl+Tab` | 切换文件 |
| `Up/Down` | 导航命令历史 |

### 编辑

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+/` | 切换注释 |
| `Ctrl+D` | 多光标 |
| `Ctrl+Shift+L` | 选择所有出现 |
| `Alt+Up/Down` | 移动行 |

### Claude专用

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+Enter` | 提交提示 |
| `Ctrl+K` | 清除提示 |
| `Ctrl+R` | 重试上次命令 |
| `Ctrl+Z` | 撤销AI更改 |
| `Escape` | 取消操作 |
| `Ctrl+C` | 取消当前操作 |
| `Ctrl+D` | 退出Claude Code |
| `Tab` | 自动完成 |
| `Up/Down` | 导航命令历史 |
| `Esc Esc` | 编辑上一条消息（双击ESC） |
| `Shift+Tab` | 切换权限模式 |
| `Ctrl+L` | 清除终端屏幕 |
| `Shift+Enter` | 插入新行（/terminal-setup后） |
| `\ + Enter` | 快速多行转义（所有终端） |
| `Option+Enter` | 插入新行（macOS默认） |
| `Ctrl+J` | 插入换行符用于多行 |
| `Ctrl+B` | 后台任务 |
| `Ctrl+F` | 杀死所有后台代理（双击） |
| `Alt+T` | 切换思考 |
| `Space`（按住） | 语音输入（需要/voice启用） |

---

## ✅ 最佳实践

### 有效提示

**公式**:
```
WHAT: [具体交付物]
WHERE: [文件路径]
HOW: [约束、方法]
VERIFY: [成功标准]
```

**示例**:
```
为登录表单添加输入验证。
WHERE: src/components/LoginForm.tsx
HOW: 使用Zod模式，显示内联错误
VERIFY: 空邮箱显示错误，无效格式显示错误
```

### 上下文管理

- 从最小上下文开始
- 根据需要添加上下文
- 任务间清除不必要上下文
- 使用 `/clear` 频繁重置

### 安全实践

- 关键更改使用审查模式
- 重大更改前备份
- 使用 `--disallowedTools` 处理危险命令
- 避免使用 `--dangerously-skip-permissions`

### 性能优化

- 保持上下文在100k token以下
- 移除不必要的文件
- 使用特定文件模式
- 任务间频繁使用 `/clear`
- 使用 `/compact` 处理长对话

### 工作流程技巧

- 在 `.claude/commands/` 中创建自定义斜杠命令
- 使用 `--output-format json` 进行自动化
- 管道命令用于复杂工作流
- 使用会话ID处理长时间运行的任务

### 反模式

| ❌ 不要 | ✅ 要 |
|--------|------|
| 模糊提示 | 使用@引用指定文件+行 |
| 不读diff就接受 | 阅读每个diff |
| 忽略警告 | 在70%时使用 `/compact` |
| 跳过权限 | 从不在生产环境 |
| 仅负面约束 | 提供替代方案 |

---

## 🔍 故障排查

### 调试命令

```bash
# 启用调试日志
claude --debug

# 详细输出
claude --verbose "复杂任务"

# 检查API状态
claude status

# 清除缓存
claude cache clear

# 重置配置
claude reset

# 性能诊断
/doctor --performance
```

### 常见问题

**权限问题**:
```bash
sudo chown -R $(whoami) ~/.claude-code
```

**更新到最新版本**:
```bash
npm update -g @anthropic-ai/claude-code
```

**重新安装**:
```bash
npm uninstall -g @anthropic-ai/claude-code
npm install -g @anthropic-ai/claude-code
```

### MCP调试

```bash
# MCP调试模式
claude --mcp-debug

# 检查MCP状态
/mcp
```

---

## 📖 参考资料

- **官方文档**: [docs.claude.com/en/docs/claude-code](https://docs.claude.com/en/docs/claude-code)
- **GitHub**: [github.com/anthropics/claude-code](https://github.com/anthropics/claude-code)
- **CLI参考**: [CLI Reference](https://docs.claude.com/en/docs/claude-code/cli-reference)
- **斜杠命令**: [Slash Commands](https://docs.claude.com/en/docs/claude-code/slash-commands)
- **常见工作流**: [Common Workflows](https://docs.claude.com/en/docs/claude-code/common-workflows)

---

## 📝 更新日志

**v3.37.3** (2026年3月)
- 新增远程控制功能（Pro/Max）
- 改进上下文管理
- 新增Tasks API
- 增强MCP集成

---

**提示**: 本教程基于Claude Code官方文档和社区最佳实践整理。建议定期查看官方更新以获取最新功能。

---

*本文档由AI助手协助创建，基于Anthropic官方文档整理。*
