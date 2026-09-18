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
