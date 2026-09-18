# python-crawler

> Python 3.10+ 网络爬虫、逆向解析、反反爬对抗与大规模数据采集工程规范技能库。

## 🌟 核心特性 (Features)

- **逆向协议优先原则**：严格执行 API 抓包 > 静态 HTML 解析 > Headless 浏览器的评估链路。
- **强制 Docstring 样例红线**：所有请求/解析函数必须具备真实的 Request 与 Response 示例。
- **高并发防反爬架构**：`httpx.AsyncClient` 连接池复用、信号量限流、TLS/JA3 伪装与代理池轮换。
- **Playwright 深度优化**：动态渲染无头浏览器抹除特征，路由拦截静态资源节省 70%+ 开销。
- **弹性重试与容错机制**：基于 Tenacity 的带抖动指数退避，防御 429 与 5xx 瞬时抖动。

## 📦 安装与加载 (Installation)

### 方式 1: 安装至 Antigravity / Gemini 全局技能库
```bash
git clone git@github.com:Garfield247/python-crawler.git ~/.gemini/config/skills/python-crawler
```

### 方式 2: 在任意项目中作为本地工作区技能引入
```bash
mkdir -p .agents/skills
git clone git@github.com:Garfield247/python-crawler.git .agents/skills/python-crawler
```

## 📄 开源协议 (License)
本项目采用 [MIT License](LICENSE) 授权。
