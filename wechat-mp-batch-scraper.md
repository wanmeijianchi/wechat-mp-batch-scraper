# 微信公众号文章正文批量抓取技术方案（含反爬绕过、链接获取与试错记录）

> 适用场景：需要把微信公众号文章正文批量扒下来，回填到表格某一列或做归档。
> 本文记录从「直接抓 → 踩坑 → 跑通」的完整过程，以及最终方案。**第一版实测发现「获取文章链接列表」才是真正卡点，本版重点补上「人在回路」的解法。** 由 AI（Workbuddy）基于实操整理，供参考。

> 🤖 **怎么用：复制一句话，Agent 就帮你干**
>
> 下面这段是「你发给 Agent 的唯一一句话」，**已内置技术方案链接**。复制下面代码块内容，粘给你的任意 Agent（Workbuddy / Claude Code / Codex / Hermes / openClaw 等）即可。**里面不含任何具体公众号——Agent 读完方案会反过来问你要抓哪个号、哪个合集。**
>
> ```
> 请先阅读并"安装"这个技能：https://raw.githubusercontent.com/wanmeijianchi/wechat-mp-batch-scraper/main/wechat-mp-batch-scraper.md
> 读完之后，用你学到的流程主动向我确认需求：我想抓取哪个微信公众号？是该号全部文章还是某个合集（合集名是？）？只要链接列表还是连正文一起要？文章链接怎么提供给你（给我 mptext.top 的 X-Auth-Key / 我直接给链接或 Excel / 你去 mptext.top 后台导出）？确认清楚后按方案执行。注意：你不要自己去爬搜索引擎拿链接。版权仅限个人用途。
> ```
>
> Agent 读完方案会像「装了 Skill」一样先向你确认参数，再开工。

> 💬 **Agent 读完后会怎么跟你谈需求（这是它应该主动说的）**
>
> 它应当先说类似这样的话，把需求聊清楚再动手：
>
> > 我已掌握公众号文章批量抓取方案。请告诉我：① 目标公众号名称？② 抓该号全部文章，还是某个合集（合集名）？③ 只要链接列表，还是连正文一起要？④ 文章链接怎么提供给我——给我 mptext.top 的 X-Auth-Key、你直接给链接/Excel，还是你去 mptext.top 后台导出？确认后我就开工。
>
> 你照着回它就行，全程不用知道任何链接或代码。

> 📌 **示例（仅作演示，可换成任意公众号）**：比如你回「抓 *神奇老太* 的合集 *神奇老太探店记* 的正文」，那对应上面 ①=神奇老太 ②=神奇老太探店记 ③=要正文 ④=按你当时给的链接来源。示例只是说明用法，复制块本身不含这些信息。

> 🔗 **获取链接前置（人在回路，必做）**：方案本身不假设能免登录自动拿别人号的链接。详见下文「零、获取文章链接列表」一节——需要先去 mptext.top 登录拿 `X-Auth-Key`，或你直接给链接/Excel。

---

## 零、获取文章链接列表（前置必做，关键卡点）

### 为什么这是第一版的头号卡点

第一版让 Agent 直接开干，结果**拿不到文章链接列表**——自动尝试的 6 种方法全失败：

| 方法 | 结果 | 原因 |
|------|------|------|
| mptext.top API（无 key） | 400 / 空 | 所有端点需 `X-Auth-Key`，未登录无 key |
| mptext.top 网页端 | 拿不到 | Nuxt.js SPA，curl 取不到内容 |
| 搜狗微信搜索 | 空 | 搜索结果不返回文章链接 |
| 百度 / Bing / Google | 超时 / 被过滤 | 反爬 / 区域限制 |

**结论**：跨账号获取别人公众号（尤其某个合集）的文章链接，没有免认证的可靠自动方式。必须由**用户介入**。

### 解法：用户在 mptext.top 登录后提供 key 或链接

mptext.top（`down.mptext.top`）提供公众号文章导出能力，底层是 `wechat-article-exporter`。所有 API 需在 Header 带 `X-Auth-Key`（用户微信扫码登录后，在网站 API 页面查看，有效期 **4 天**）。

**方式 1（推荐，最稳）**：用户登录 mptext.top → 后台「公众号管理」添加该号并同步 → 「文章下载」页勾选目标合集的文章 → 右上角「导出」选 Excel/JSON → 把文件/链接列表直接给 Agent。Agent 拿到链接就抓正文。

**方式 2（Agent 用 API 拉）**：用户把 `X-Auth-Key` 给 Agent，Agent 自己调 API 拉链接（见下）。

**兜底**：两者都没有时，Agent **不要瞎爬搜索引擎**，直接问用户要链接列表/Excel。

### mptext.top API 用法（需 X-Auth-Key）

```python
import requests

BASE = "https://down.mptext.top"
AUTH = "你的X-Auth-Key"          # 用户登录后网站 API 页面查看，有效期 4 天
H = {"X-Auth-Key": AUTH}

# 1) 搜公众号，拿 fakeid
r = requests.get(f"{BASE}/api/public/v1/account", params={"keyword": "神奇老太"}, headers=H, timeout=20)
print(r.json())                  # 从返回里取该号的 fakeid
fakeid = "xxxxxxxx"              # 手动填上面拿到的 fakeid

# 2) 分页拉文章列表（size 最大 20，靠 begin 翻页）
links = []
begin = 0
while True:
    r = requests.get(f"{BASE}/api/public/v1/article",
                     params={"fakeid": fakeid, "begin": begin, "size": 20}, headers=H, timeout=20)
    items = r.json().get("data", {}).get("list", [])
    if not items:
        break
    for it in items:
        links.append(it.get("url") or it.get("link") or it.get("article_url"))
    begin += 20

# 3)（可选）直接拿正文，连自己抓都省了——带 key 时绕开微信风控
# txt = requests.get(f"{BASE}/api/public/v1/download",
#                    params={"url": links[0], "format": "text"}, headers=H).text
```

> 注：API 返回的字段名（url/link/article_url）以实际响应为准，先 `print(r.json())` 看一眼结构再取值。合集筛选在 UI 里勾选最省事；用 API 拿全号后也可按标题关键词过滤。

### Agent 开局引导话术（skill 式，推荐）

Agent 读完本方案后，应**先向用户确认**而不是自作主张：

1. 目标公众号名 / 合集名确认（合集 ≠ 整个公众号，只抓指定合集）；
2. 文章链接怎么来：有 `X-Auth-Key` 吗？还是你直接给链接/Excel？或去 mptext.top 后台导出？
3. 输出到哪个文件 / 路径。

---

## 一、背景与目标

有一份 Excel（`老太文章列表.xlsx`），共 147 行：

- 第 3 列 `链接`：每篇公众号文章的 `https://mp.weixin.qq.com/s/xxx` URL
- 第 18 列 `文章内容`：目标列，初始为空，需把对应正文填进去

目标：批量抓取 147 篇文章正文，回填到 `文章内容` 列，输出一份新表。

---

## 二、踩坑记录（试错过程）

一开始我以为「直接 requests 抓 URL 解析 HTML」就能搞定，结果连踩三个坑。

### 坑 1：直接抓取，正文是空的

用 `requests` 抓文章页，页面返回约 2.3MB，BeautifulSoup 取 `#js_content` 能定位到元素，但 `get_text()` 出来是**空字符串**。

原因：微信文章正文是**前端 JS 动态注入**，静态 HTML 里 `js_content` 是空壳。

### 坑 2：风控拦截「环境异常」

换另一篇试，直接返回**只有 18KB 的验证页**，文案：

> 环境异常，当前环境异常，完成验证后即可继续访问。

这是微信对「非微信客户端 + 陌生 IP（如云沙箱出口 IP）」的**反爬风控**。

试过无效：换移动端 UA、单纯加重试。

### 坑 3：`og:description` 不可靠

部分文章 `<meta property="og:description">` 里竟**完整装着正文**，但验证后发现不能依赖：有的有（695 字全文）、有的空（长度 0）。属偶发行为。

---

## 三、最终成熟方案（抓正文）

### 3.1 核心：预热会话绕过风控 ✅

**先用同一会话访问微信域名根 `https://mp.weixin.qq.com/` 拿到 `ua_id` cookie，再带着它抓文章**，风控放行，返回正常 3MB 页。

```python
import requests

def make_session():
    s = requests.Session()
    s.headers.update({
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                      "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36",
        "Accept-Language": "zh-CN,zh;q=0.9",
    })
    s.get("https://mp.weixin.qq.com/", timeout=15)  # 预热，拿 ua_id cookie
    return s
```

抓单篇务必带 `Referer`：

```python
s.get(url, headers={"Referer": "https://mp.weixin.qq.com/"}, timeout=25)
```

### 3.2 正文提取（容器优先 + 兜底）

```python
from bs4 import BeautifulSoup
import re

def extract_body(html):
    soup = BeautifulSoup(html, "lxml")
    el = (soup.select_one("#js_content")
           or soup.select_one(".rich_media_content")
           or soup.select_one("#js_article"))
    if el:
        t = el.get_text("\n", strip=True)
        if len(t) >= 20:
            return t
    m = re.search(r'<meta property="og:description"[^>]*content="([^"]*)"', html)
    if m:
        return m.group(1).replace("\x0a", "\n").strip()
    return ""
```

### 3.3 节流与重试

- 每篇随机 sleep 1~2 秒；
- 仍含「环境异常」或页面过小（< 50KB），**重建会话（重新预热）重试**，最多 3 次。

### 3.4 断点续跑

每抓完一篇把 `URL -> 正文` 写本地缓存 JSON；重启跳过已完成。

### 3.5 回填 Excel 注意点

- `openpyxl` 按表头名定位列（别写死列号）；
- 单元格上限约 **32767 字**，超长截断到 32000 并标注；
- **先备份原文件**，结果写新文件。

---

## 四、完整可运行代码（精简版，已有链接时）

```python
import requests, re, time, json, random
from bs4 import BeautifulSoup
import openpyxl

SRC = "老太文章列表.xlsx"
OUT = "老太文章列表_带正文.xlsx"
CACHE = "/tmp/wechat_body_cache.json"

def make_session():
    s = requests.Session()
    s.headers.update({
        "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) "
                      "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36",
        "Accept-Language": "zh-CN,zh;q=0.9",
    })
    try: s.get("https://mp.weixin.qq.com/", timeout=15)
    except Exception: pass
    return s

def extract_body(html):
    soup = BeautifulSoup(html, "lxml")
    el = (soup.select_one("#js_content")
           or soup.select_one(".rich_media_content")
           or soup.select_one("#js_article"))
    if el and len(el.get_text("\n", strip=True)) >= 20:
        return el.get_text("\n", strip=True)
    m = re.search(r'<meta property="og:description"[^>]*content="([^"]*)"', html)
    return m.group(1).replace("\x0a", "\n").strip() if m else ""

def get_body(s, url):
    for _ in range(3):
        try:
            r = s.get(url, headers={"Referer": "https://mp.weixin.qq.com/"}, timeout=25)
        except Exception:
            s = make_session(); time.sleep(1); continue
        html = r.text
        if "环境异常" in html or len(html) < 50000:
            s = make_session(); time.sleep(1.5); continue
        return extract_body(html), s
    return "", s

cache = json.load(open(CACHE, encoding="utf-8")) if __import__("os").path.exists(CACHE) else {}

wb = openpyxl.load_workbook(SRC)
ws = wb.active
hdr = [c.value for c in ws[1]]
uc, bc = hdr.index("链接") + 1, hdr.index("文章内容") + 1

s = make_session()
for row in ws.iter_rows(min_row=2):
    url = row[uc - 1].value
    if not url or not str(url).startswith("http"): continue
    key = str(url).strip()
    if key in cache and cache[key]: continue
    body, s = get_body(s, key)
    cache[key] = body
    json.dump(cache, open(CACHE, "w", encoding="utf-8"), ensure_ascii=False)
    time.sleep(random.uniform(1.0, 2.0))

for row in ws.iter_rows(min_row=2):
    key = str(row[uc - 1].value).strip() if row[uc - 1].value else ""
    if key in cache and cache[key]:
        row[bc - 1].value = cache[key][:32000]

wb.save(OUT)
```

实测：147 篇 **100% 成功，0 失败**，正文 208~2788 字（平均 921 字），约 5 分钟。

> 若你有 mptext.top 的 `X-Auth-Key`，正文也可直接用它的 `/api/public/v1/download?format=text` 拿，彻底绕开微信风控（见「零」节）。

---

## 五、适用边界与注意事项

- **获取链接必须人在回路**：别人号的链接列表无免认证自动方式，需 mptext.top 登录拿 key / 后台导出，或用户直接提供。
- **合集 ≠ 整个公众号**：抓指定合集要在 mptext.top 勾选该合集，或拿全号后按标题过滤。
- **版权**：正文归原作者，仅限个人合法用途，勿未经授权转发。
- **阅读量 / 点赞 / 评论**：本方案只拿正文；增强指标需本人公众号后台凭据，属另一套流程。
- **频率风险**：别开太快，微信仍可能临时封 IP / 弹验证；建议在本机跑。
- **依赖**：Python + `requests` / `beautifulsoup4` / `lxml` / `openpyxl`。

---

## 六、关键结论一句话

> 微信文章正文不能直接抓（JS 注入 + 陌生 IP 风控），破解是「先访问域名根预热拿 cookie，再同会话带 Referer 抓」。但**更难的是「拿链接列表」**——必须用户在 mptext.top 登录拿 `X-Auth-Key` 或后台导出，Agent 别妄图爬搜索引擎。
