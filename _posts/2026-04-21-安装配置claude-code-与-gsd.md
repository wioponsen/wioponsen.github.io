---
title: '安装配置claude code 与 GSD'
date: 2026-04-21 09:38:00 +0000
author: wioponsen
categories: [博客迁移]
tags: [迁移]
math: true
mermaid: true
---

refs:
1. https://code.claude.com/docs/zh-CN/quickstart
2. https://blog.csdn.net/2301_80863610/article/details/150963139
3. https://zhichai.net/topic/177168657
其他工具：opencode, Cursor

### 准备工作
1. 配置node.js (v16+)
安装完成后使用cmd指令测试
```cmd
node -v
npm -v
```

2. VSCode安装claude code 插件
插件名称： Claude Code for VS Code

### 安装 Claude code
```
# Node.js 18+⭐️              
/*Universal Method       */ npm install -g @anthropic-ai/claude-code
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# Windows
/* Via CMD               */ npm install -g @anthropic-ai/claude-code
/* Via Powershell        */ irm https://claude.ai/install.ps1 | iex
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# WSL/GIT
/* Via Terminal          */ npm install -g @anthropic-ai/claude-code
/* Via Terminal          */ curl -fsSL https://claude.ai/install.sh | bash
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# MacOS                  */ brew install node && npm install -g @anthropic-ai/claude-code
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# Linux 
/* Via Terminal          */ sudo apt update && sudo apt install -y nodejs npm
/* Via Terminal          */ npm install -g @anthropic-ai/claude-code
/* Via Terminal          */ curl -fsSL https://claude.ai/install.sh | bash
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# Arch                     
/* Via Terminal          */ yay -S claude-code*/ 
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# Docker 
/* Windows (CMD)         */ docker run -it --rm -v "%cd%:/workspace" -e ANTHROPIC_API_KEY="sk-your-key" node:20-slim bash -lc "npm i -g @anthropic-ai/claude-code && cd /workspace && claude"
/* macOS/Linux (bash/zsh)*/ docker run -it --rm -v "$PWD:/workspace" -e ANTHROPIC_API_KEY="sk-your-key" node:20-slim bash -lc 'npm i -g @anthropic-ai/claude-code && cd /workspace && claude'
/* No bash Fallback      */ docker run -it --rm -v "$PWD:/workspace" -e ANTHROPIC_API_KEY="sk-your-key" node:20-slim sh -lc 'npm i -g @anthropic-ai/claude-code && cd /workspace && claude'
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# Check if claude is installed correctly
/* Linux                 */ which claude 
/* Windows               */ where claude
/* Universal             */ claude --version
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
# Common Management
/*claude config          */ Configure settings
/*claude mcp list        */ Setup MCP servers, you can also replace "list" with add/remove
/*claude /agents         */ Configure/Setup Subagents for different tasks
/*claude update          */ Update to latest

```
验证是否成功安装，使用指令测试：
```
claude --version
# 2.1.116 (Claude Code)
```



### 配置claude
在国内使用claude code 需要使用梯子，否则无法访问。(或者通过配置setting.json跳过，settings.json可以在后面安装gsd等skills的时候创建，也可以用cc-switch配置)

使用指令 `claude` 进入首次环境配置，如果开启了VPN仍旧返回连接问题：
```bat
Welcome to Claude Code v2.1.116
…………………………………………………………………………………………………………………………………………………………

     *                                       █████▓▓░
                                 *         ███▓░     ░░
            ░░░░░░                        ███▓░
    ░░░   ░░░░░░░░░░                      ███▓░
   ░░░░░░░░░░░░░░░░░░░    *                ██▓░░      ▓
                                             ░▓▓███▓▓░
 *                                 ░░░░
                                 ░░░░░░░░
                               ░░░░░░░░░░░░░░░░
       █████████                                        *
      ██▄█████▄██                        *
       █████████      *
…………………█ █   █ █………………………………………………………………………………………………………………

 Unable to connect to Anthropic services

 Failed to connect to api.anthropic.com: ERR_BAD_REQUEST

 Please check your internet connection and network settings.

 Note: Claude Code might not be available in your country. Check supported countries at
 https://anthropic.com/supported-countries
```
则可以尝试手动输入vpn代理地址：
```powershell
# powershell

# 设置当前会话的代理
$env:HTTP_PROXY="http://127.0.0.1:7890"
$env:HTTPS_PROXY="http://127.0.0.1:7890"

# 再次尝试运行
claude

```
正常第一次会进行配置：
```
Welcome to Claude Code v2.1.116
…………………………………………………………………………………………………………………………………………………………

     *                                       █████▓▓░
                                 *         ███▓░     ░░
            ░░░░░░                        ███▓░
    ░░░   ░░░░░░░░░░                      ███▓░
   ░░░░░░░░░░░░░░░░░░░    *                ██▓░░      ▓
                                             ░▓▓███▓▓░
 *                                 ░░░░
                                 ░░░░░░░░
                               ░░░░░░░░░░░░░░░░
       █████████                                        *
      ██▄█████▄██                        *
       █████████      *
…………………█ █   █ █………………………………………………………………………………………………………………

 Let's get started.

 Choose the text style that looks best with your terminal
 To change this later, run /theme

   1. Auto (match terminal)
 ❯ 2. Dark mode ✔
   3. Light mode
   4. Dark mode (colorblind-friendly)
   5. Light mode (colorblind-friendly)
   6. Dark mode (ANSI colors only)
   7. Light mode (ANSI colors only)

 ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
  1  function greet() {
  2 -  console.log("Hello, World!");
  2 +  console.log("Hello, Claude!");
  3  }
 ╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
  Syntax theme: Monokai Extended (ctrl+t to disable)


# 回车后：
Claude Code can be used with your Claude subscription or billed based on API usage through your Console account.

 Select login method:

 ❯ 1. Claude account with subscription · Pro, Max, Team, or Enterprise

   2. Anthropic Console account · API usage billing

   3. 3rd-party platform · Amazon Bedrock, Microsoft Foundry, or Vertex AI



```
按步骤配置即可， 也可以直接从本地json配置api，主要是配置以下三个字段：
- ANTHROPIC_BASE_URL : API地址
- ANTHROPIC_AUTH_TOKEN : API密钥
- ANTHROPIC_MODEL : 指定模型

windows : C:\Users\你的用户名\.claude\settings.json
maxos/linux: ~/.claude/settings.json


示例:
```json
# ollama

{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:11434",
    "ANTHROPIC_AUTH_TOKEN": "ollama",
    "ANTHROPIC_MODEL": "minimax-m2.7:cloud",
    "ANTHROPIC_SMALL_FAST_MODEL": "minimax-m2.7:cloud",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  },
  "hooks": {

...

},
...
}
 
```

或者ps中手动设定环境：
```powershell
# 1. 设置 MiniMax 的 API 地址 (针对中国大陆优化)
$env:ANTHROPIC_BASE_URL="https://api.minimaxi.com/anthropic"

# 2. 设置你的 MiniMax API Key
$env:ANTHROPIC_AUTH_TOKEN="你的_MINIMAX_API_KEY"

# 3. 设置默认使用的模型名
$env:ANTHROPIC_MODEL="MiniMax-M2.5"

# 4. 启动 Claude Code
claude

```
如果使用本地算，如ollama，可以设置为：



### 安装 GSD
```
# 进入powershell admin
npx get-shit-done-cc@latest
# 选择 Claude code  -> Global
```
验证成功：
```
gsd --version


#claude内
claude 
/gsd:help
```


### CC + GSD 实现项目
GSD（get shit done）是面向Claude Code等AI编程工具的“元提示+上下文工程+规范驱动开发”工作流系统。核心思想是“规划与执行分离”，以文件为唯一真相来源，让AI能够可靠地完成复杂的软件开发项目。
• 元提示 (Meta-prompting)：主动引导AI的思考过程，确保对需求的理解零偏差。
• 上下文工程 (Context Eng.)：将AI状态完全外化到文件中，彻底防止长程信息丢失。
• 规范驱动开发 (Spec-Driven)：坚持“先定义清晰规范，再进行编码”，从源头保证质量。
适合希望将AI高效利用在真实项目中的开发者、独立开发者、追求代码高质量、可维护性的小团队

核心优势：
上下文始终新鲜：每个任务都在干净的上下文中执行，彻底避免上下文污染，确保AI输出的质量稳定可靠。
先规划后执行：严格遵循“规范驱动开发”模式，在动手前明确目标与路径，显著降低返工率，保持项目结构清晰。
全流程可追溯：所有关键决策和执行过程都记录在文件中，配合Git原子化提交，任何历史问题都能快速定位与复盘。
人类主导，可控性强：AI仅作为高效的执行工具，人类在讨论、验证等关键节点深度参与决策，始终确保项目的核心方向正确。


核心流程：
```cmd
# 1 初始化 Initialize
# 对于新项目，通过对话确定初始需求和目标
/gsd:new project
# 对于已有项目，扫描整理当前项目结构，自动生成项目规划和架构。在生成codebase之后，可以使用`/gsd:new project`创建项目，提出修改需求
/gsd:map-codebase

# 初始化好了项目，确定需求后，后续可以进行计划调研，细分任务，并行进行

# 2 讨论 Discuss Phase
# 快速开启深度讨论，迭代项目需求，对齐项目的关键内容，沟通细节。N是细化的条目（1,2,3...）
/gsd:discuss-phase [N]


# 3 计划 Plan
# 启动计划阶段，正式进行项目开发，将复杂需求拆解为逻辑严密、可执行的详细子任务列表，产出保存到 planning/plan-N.md
/gsd:plan-phase N


# 4 执行 Execute
# 严格按照原子任务进行开发构建，每个任务在全新的上下文环境中独立运行，并生成独立的Git提交记录，过程清晰、透明且可追溯
/gsd:execute-phase N


# 5 验证 Verify
# 自动运行单元/集成测试，并生成标准化的人工验收清单。也支持人工介入，反馈问题
/gsd:verify-work N

# 6 里程碑 Milestone
# 完成归档，锁定并稳定当前阶段的所有成果
/gsd:complete-milestone
# 开启新篇章，开启下已阶段的开发目标
/gsd:new-milestone
```

高级权限参数:
允许自主执行 Git / IO 操作，大幅提升效率。 仅限在可信项目代码库中使用，确保代码安全
```
claude --dangerously-skip-permissions
```

### 更新
```
npx get-shit-done-cc@latest

npm install -g @anthropic-ai/claude-code@latest
claude update

```