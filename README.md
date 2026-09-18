# python-crawler

> Python 3.10+ 网络爬虫、逆向解析、反反爬对抗与大规模数据采集工程规范技能库。

## 🌟 核心特性 (Features)

- **逆向协议优先原则**：严格执行 API 抓包 > 静态 HTML 解析 > Headless 浏览器的评估链路。
- **强制 Docstring 样例红线**：所有请求/解析函数必须具备真实的 Request 与 Response 示例。
- **高并发防反爬架构**：`httpx.AsyncClient` 连接池复用、信号量限流、TLS/JA3 伪装与代理池轮换。
- **Playwright 深度优化**：动态渲染无头浏览器抹除特征，路由拦截静态资源节省 70%+ 开销。
- **弹性重试与容错机制**：基于 Tenacity 的带抖动指数退避，防御 429 与 5xx 瞬时抖动。

## 📦 安装与多 Agent 使用指南 (Installation & Multi-Agent Usage)

本项目遵循开放 Agent 规范，支持在 **Gemini / Antigravity**、**Anthropic Claude**、**Cursor / Codex** 等各类主流 Agent 环境中一键安装与激活：

### 1. Google Antigravity / Gemini Code Assist
- **全局安装（推荐）**：
  ```bash
  git clone git@github.com:Garfield247/agent-skill-python-crawler.git ~/.gemini/config/skills/python-crawler
  ```
- **项目工作区局部引入**：
  ```bash
  mkdir -p .agents/skills
  git clone git@github.com:Garfield247/agent-skill-python-crawler.git .agents/skills/python-crawler
  ```

### 2. Anthropic Claude (Claude Code / Claude Projects)
- **Claude Code (CLI 终端智能体)**：
  克隆至 Claude 全局技能库：
  ```bash
  mkdir -p ~/.claude/skills
  git clone git@github.com:Garfield247/agent-skill-python-crawler.git ~/.claude/skills/python-crawler
  ```
  *或者在项目根目录的 `CLAUDE.md` 中追加引入：*
  ```markdown
  See detailed engineering specifications in: ~/.claude/skills/python-crawler/SKILL.md
  ```
- **Claude Projects (Web / 桌面端)**：
  直接将仓库中的 `SKILL.md` 内容复制并粘贴至 Project 的 **Project Knowledge (项目知识库)** 或 **Custom Instructions (自定义指令)** 中。

### 3. Cursor / GitHub Copilot / OpenAI Codex
- **Cursor (现代 MDC 规则体系)**：
  在项目根目录创建或链接规则：
  ```bash
  mkdir -p .cursor/rules
  # 克隆或软链接为 Cursor 专有规则文件
  git clone git@github.com:Garfield247/agent-skill-python-crawler.git .cursor/rules/python-crawler
  ```
- **GitHub Copilot / Codex**：
  将本技能规范注入 Copilot 指令集：
  ```bash
  mkdir -p .github
  cat << 'EOF' >> .github/copilot-instructions.md
  # 引入本技能核心规则
  EOF
  cat path/to/SKILL.md >> .github/copilot-instructions.md
  ```

## 📄 开源协议 (License)
本项目采用 [MIT License](LICENSE) 授权。
