---
name: python-crawler
description: >-
  Python 3.10+ 网络爬虫、逆向解析、反反爬对抗与大规模数据采集工程规范技能。
  涵盖协议逆向优先原则、httpx 异步连接池与并发限流 (Semaphore)、Playwright 动态渲染与资源拦截优化、
  TLS/JA3 指纹伪装 (curl_cffi)、代理池轮换、指数退避重试 (Tenacity) 及强制 Docstring 请求/响应样例工程红线。
---

# Python 3.10+ 网络爬虫与数据采集工程规范技能 (Python Crawler Mastery Skill)

## 概述 (Overview)

本技能定义了研发工程师与 AI 编码助手在构建高可用、高并发、抗反爬的 Python 网络爬虫与数据提取流水线时的技术选型标准、架构模式与工程规范。

### 核心设计原则

1. **逆向协议优先原则 (Reverse-Engineering First)**：抓包分析 XHR/Fetch/WebSocket 接口优先于解析静态 HTML，更优先于 Headless 浏览器渲染。坚决杜绝在未抓包分析的情况下无脑拉起浏览器。
2. **异步高并发与资源节流**：基于 `httpx.AsyncClient` 构建长连接池，结合 `asyncio.Semaphore` 实施严格的并发限流，严禁单线程同步阻塞或无节制并发轰炸目标。
3. **强制 Docstring 真实请求/响应样例（核心红线）**：所有爬虫请求方法与解析器，必须在注释中附带完整的 Request Sample 与 Response Sample，保证解析链路在目标页面变动时具备可核对性。
4. **反反爬与高弹性重试**：TLS/JA3 指纹伪装、真实 Browser Header 指纹矩阵、代理池健康探活与带抖动的指数退避重试（Exponential Backoff with Jitter）。

---

# 1. 采集技术选型矩阵 (Technology Selection Matrix)

| 场景需求 | 推荐方案 | 核心库 / 工具 | 禁用 / 淘汰方案 |
| :--- | :---: | :--- | :--- |
| **标准 REST/JSON 接口** | 纯异步 HTTP | `httpx.AsyncClient` | `requests` (阻塞事件循环) |
| **强反爬 / TLS 指纹校验** | 指纹伪装 HTTP | `curl_cffi` (模拟 Chrome/Safari TLS) | 默认 Python `urllib` / `ssl` |
| **复杂前端加密 / 动态渲染** | 异步 Headless | `playwright.async_api` | `selenium` (沉重、易被特征检测) |
| **HTML/XML DOM 解析** | 高性能 C 引擎 | `selectolax` (Lexbor) / `lxml` | 纯正则表达式解析复杂 HTML |
| **重试与容错控制** | 指数退避调度 | `tenacity` | 无限死循环 `while True` |

---

# 2. 爬虫工程核心红线 (Core Engineering Rules)

## 2.1 强制 Docstring 携带请求与响应样例 (Sample Rule)
- **核心红线**：封装网络请求和数据解析的方法时，**必须在函数 Docstring 中提供真实的 Request 示例与 Response 示例**。
- **正规规范示例**：

```python
async def fetch_product_detail(item_id: str, client: httpx.AsyncClient) -> dict:
    """获取商品详情与实时库存价格
    
    Request Sample:
        Method: GET
        URL: https://api.example.com/v1/products/detail?id=P100293&lang=zh_CN
        Headers:
            User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36
            Accept: application/json
            X-Client-Version: 2.4.0

    Response Sample:
        Status: 200 OK
        Body:
        {
            "code": 0,
            "msg": "success",
            "data": {
                "id": "P100293",
                "title": "无线降噪耳机 Pro",
                "price": 299.00,
                "stock": 142,
                "specs": [{"name": "颜色", "value": "哑光黑"}]
            }
        }
    """
    url = "https://api.example.com/v1/products/detail"
    params = {"id": item_id, "lang": "zh_CN"}
    resp = await client.get(url, params=params)
    resp.raise_for_status()
    return resp.json()
```

## 2.2 防阻塞与异步连接池复用
- 严禁每次采集请求都新建 `httpx.AsyncClient()` 或 `ClientSession`；
- 必须通过上下文管理器或依赖注入复用 Client 单例，保持 TCP/TLS 连接池热活；
- 必须设定合理的超时时间（建议 `timeout=httpx.Timeout(15.0, connect=5.0)`），严禁使用无超时的默认配置。

---

# 3. 异步并发采集与限流架构 (Concurrency & Rate Limiting)

使用 `asyncio.Semaphore` 精确控制并发窗口，配合 `tenacity` 实现自适应指数退避：

```python
import asyncio
import httpx
from tenacity import retry, stop_after_attempt, wait_exponential_jitter, retry_if_exception_type

class AsyncScraper:
    def __init__(self, max_concurrency: int = 10):
        self.semaphore = asyncio.Semaphore(max_concurrency)
        self.client = httpx.AsyncClient(
            timeout=httpx.Timeout(15.0, connect=5.0),
            limits=httpx.Limits(max_keepalive_connections=20, max_connections=50),
            follow_redirects=True
        )

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential_jitter(initial=1, max=10, jitter=1),
        retry=retry_if_exception_type((httpx.TransportError, httpx.HTTPStatusError)),
        reraise=True
    )
    async def fetch_page(self, url: str) -> str:
        async with self.semaphore:
            headers = {
                "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
                "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8"
            }
            resp = await self.client.get(url, headers=headers)
            # 遇到 429 限流或 5xx 服务端错误时触发重试
            if resp.status_code in (429, 502, 503, 504):
                resp.raise_for_status()
            return resp.text

    async def close(self):
        await self.client.aclose()
```

---

# 4. Playwright 动态渲染与降本优化 (Playwright Optimization)

当页面必须通过浏览器执行 JavaScript 解密时，采用 Playwright 异步驱动，并实施**资源拦截**以降低 70%+ 带宽和 CPU 负载：

```python
from playwright.async_api import async_playwright, Route, Request

async def handle_route_intercept(route: Route, request: Request):
    # 阻断不必要的媒体、字体与监控打点请求，极速提升加载速度
    if request.resource_type in ["image", "media", "font", "stylesheet"]:
        await route.abort()
    elif "analytics" in request.url or "tracker" in request.url:
        await route.abort()
    else:
        await route.continue_()

async def render_dynamic_content(target_url: str) -> str:
    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=True,
            args=[
                "--no-sandbox",
                "--disable-blink-features=AutomationControlled", # 抹除自动化驱动标记
                "--disable-dev-shm-usage"
            ]
        )
        context = await browser.new_context(
            viewport={"width": 1920, "height": 1080},
            user_agent="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36"
        )
        page = await context.new_page()
        # 挂载路由拦截器
        await page.route("**/*", handle_route_intercept)

        await page.goto(target_url, wait_until="networkidle", timeout=30000)
        content = await page.content()
        await browser.close()
        return content
```

---

# 5. 防御性数据解析与清洗 (Defensive Data Extraction)

使用高性能 `selectolax` 进行防御性提取，杜绝因单个页面标签缺失导致批量解析异常中断：

```python
from selectolax.parser import HTMLParser
from pydantic import BaseModel, Field

class ArticleItem(BaseModel):
    title: str = Field(..., description="文章标题")
    author: str = Field(default="未知作者", description="作者名称")
    read_count: int = Field(default=0, description="阅读量")

def parse_article_html(html_text: str) -> list[ArticleItem]:
    tree = HTMLParser(html_text)
    articles: list[ArticleItem] = []

    for node in tree.css("div.article-card"):
        # 安全防御提取：通过安全导航获取文本
        title_node = node.css_first("h2.title")
        if not title_node:
            continue
        title = title_node.text(strip=True)

        author_node = node.css_first("span.author-name")
        author = author_node.text(strip=True) if author_node else "未知作者"

        read_node = node.css_first("span.views")
        raw_views = read_node.text(strip=True) if read_node else "0"
        read_count = int(raw_views) if raw_views.isdigit() else 0

        articles.append(ArticleItem(title=title, author=author, read_count=read_count))

    return articles
```

---

# 6. 爬虫开发 Checklist

- [ ] 是否已优先尝试网络抓包（Network XHR/Fetch/WebSocket）提取结构化接口？
- [ ] 所有爬虫类或方法是否在 Docstring 中包含 Request Sample 与 Response Sample？
- [ ] 是否已配置请求超时时间（Connect/Read Timeout），严禁缺省无超时？
- [ ] 并发请求是否由 `asyncio.Semaphore` 限制数量，防止目标反制或本地资源耗尽？
- [ ] 重试机制是否使用带抖动的指数退避（Jitter Backoff），杜绝死循环？
- [ ] 若使用 Playwright/Headless，是否配置了图片/媒体拦截和自动化标记剔除？

---

# 7. Bug 分析、排查与反爬对抗诊断武器库 (Troubleshooting & Anti-Scraping Diagnostics)

在爬虫任务出现大面积失败、数据解析异常或被目标拦截时，遵循以下排障指引：

### 7.1 阻断类型精准定级与诊断矩阵
| 响应状态 / 现象 | 故障根因分析 (RCA) | 诊断与处置武器 |
| :--- | :--- | :--- |
| **HTTP 403 Forbidden** | 触发 Cloudflare/Akamai/Incapsula WAF 规则，或 TLS/JA3 指纹被特征识别 | 切换至 `curl_cffi` 模拟 Chrome 真实 TLS 握手指纹；检查 User-Agent 与 Client Hints 矩阵是否矛盾。 |
| **HTTP 429 Too Many Requests** | 触发目标针对 IP、Token 或账号的滑动窗口并发频控 | 降低信号量并发（`asyncio.Semaphore`）；注入自适应指数抖动退避重试；轮换健康代理 IP 池。 |
| **HTTP 200 但返回空内容/验证码** | 目标页面返回滑块验证码、风控挑战页（Challenge） | 抓包分析风控下发的校验参数，或切换至 Playwright 真实浏览器环境进行验证码交互。 |
| **网络连接频繁 Reset / 超时** | 目标防火墙丢弃请求包或代理节点失效 | 探测代理连接可用性，检查代理池健康探活心跳。 |

### 7.2 Playwright 动态渲染挂死排查
- **现象**：`page.goto` 耗尽 30s 超时报错 `TimeoutError: Timeout 30000ms exceeded`；
- **根因**：使用了 `wait_until="networkidle"`，但页面内包含持续发送心跳或埋点的长连接，导致网络永远无法真正进入 Idle 状态；
- **修复方案**：降级为 `wait_until="domcontentloaded"`，然后显式使用 `page.wait_for_selector("div.content", timeout=10000)` 等待目标数据节点出现。

### 7.3 解析字段失效 (DOM Mutated) 排查
- **排查红线**：对比现有 HTML 与函数 Docstring 中记录的 **Response Sample**，核对目标页面类名（Class Name）、DOM 结构是否升级换代；
- 优先选择数据接口（XHR/JSON）提取，杜绝过度依赖混淆多变的前端 Class。

---

# 8. 大规模采集进阶：布隆判重、检查点与动态 Stealth (Large-Scale Crawling & Stealth)

### 8.1 布隆过滤器去重与任务检查点 (Bloom Filter & Checkpointing)
面对百万级采集任务，严禁使用 Python 内存 Set 判重（防 OOM）：
1. **Redis 布隆判重**：使用 `pybloom_live` 或 RedisBloom 对 URL / 实体主键进行指纹判重（假阳性率控制在 0.01% 内）；
2. **断点续跑检查点 (Checkpointing)**：每消费完一个批次，向 Redis 保存已消费的游标偏移量（Cursor），支持进程被 kill 后一键无损断点续爬。

### 8.2 Playwright 动态 Stealth 隐身增强
在需要极速通过指纹检测时，挂载 `playwright-stealth` 抹除底层特征：
```python
from playwright.async_api import async_playwright
# 抹除 navigator.plugins, WebGL vendor, AudioContext 指纹
await page.add_init_script("""
    Object.defineProperty(navigator, 'webdriver', {get: () => undefined});
    window.chrome = { runtime: {} };
""")
```
