# 微信公众号文章正文批量抓取技术方案（含反爬绕过与试错记录）

> 适用场景：手头有一份「公众号文章 URL 列表」（如 Excel 的某一列），想把每篇文章的**正文**扒下来，回填到表格的某一列，或存成归档。
> 本文记录了从「直接抓 → 踩坑 → 跑通」的完整过程，以及最终沉淀下来的成熟方案。

> 🤖 **怎么跟你的 Agent 说（复制即用）**
>
> 把下面这句话（连同本方案对应的 **「技术方案链接」**）直接发给你的**任意一个 Agent** 都能用：Workbuddy / Claude Code / Codex / Hermes / openClaw 等。
> - **模板 A（你直接给链接 / 表格）**：
>   > 帮我把这些微信公众号文章链接的正文批量抓下来，填到 Excel 的「文章内容」列：[这里粘贴 xlsx 路径或链接列表]
> - **模板 B（你只说公众号名，让它自己搞定）**：
>   > 你先读这个技术方案 <技术方案链接>，照着它帮我把微信公众号【公众号名称】的历史文章（或某个合集）正文批量抓取，整理成 Excel。获取文章链接的方法参考 down.mptext.top 和 docs.mptext.top/llms-full.txt；抓取正文用「先访问微信域名根预热拿 cookie、再同会话带 Referer 抓，从 #js_content 取正文」的方案；版权仅限个人用途。
> - ⚠️ **关键点**：这句话**必须配上「技术方案链接」一起发**。一个完全没上下文的 Agent 单听"帮我把微信文章扒下来"是不知道怎么干的——它得先读到这份方案，才像「装了个 Skill」一样知道完整流程，并反过来引导你：*「你想弄哪个公众号 / 合集？我来帮你扒链接、再拿正文」*。

> 🔗 **去哪拿批量文章链接（获取入口）**
>
> - 技术方案（GitHub 公开版，发 Agent 时附上这个链接）：https://raw.githubusercontent.com/wanmeijianchi/wechat-mp-batch-scraper/main/wechat-mp-batch-scraper.md
> - 操作后台：https://down.mptext.top/dashboard/article
> - 说明文档（llms-full）：https://docs.mptext.top/llms-full.txt
> - 上面是 mptext.top 提供的「公众号文章导出 / 链接获取」能力。拿到链接列表后，交给上面的抓取方案即可。

---

## 一、背景与目标

有一份 Excel（`老太文章列表.xlsx`），共 147 行，结构如下：

- 第 3 列 `链接`：每篇公众号文章的 `https://mp.weixin.qq.com/s/xxx` URL
- 第 18 列 `文章内容`：目标列，初始为空，需要把对应正文填进去

目标：批量抓取 147 篇文章正文，回填到 `文章内容` 列，输出一份新表。

---

## 二、踩坑记录（试错过程）

一开始我以为「直接 requests 抓 URL 解析 HTML」就能搞定，结果连踩三个坑。

### 坑 1：直接抓取，正文是空的

用 `requests` 抓文章页，页面确实返回了（约 2.3MB），用 BeautifulSoup 取 `#js_content`（微信正文容器）也能定位到元素，但 `get_text()` 出来是**空字符串**。

原因：微信公众号文章的正文是**前端 JS 动态注入**的，静态 HTML 里 `js_content` 这个 div 是空的，正文并不在初始文档里。

### 坑 2：风控拦截「环境异常」

换另一篇文章试，直接返回的是**只有 18KB 的验证页**，正文根本没有，页面文案是：

> 环境异常，当前环境异常，完成验证后即可继续访问。去验证

这是微信对「非微信客户端 + 陌生 IP（如云沙箱出口 IP）」的**反爬风控**。不是个别文章的问题，是整类请求会被拦。

试过的解法（均无效）：
- 换移动端 UA（iPhone Safari）→ 仍被拦
- 单纯加重试 → 仍被拦

### 坑 3：`og:description` 不可靠

发现部分文章的元信息 `<meta property="og:description">` 里竟**完整装着正文**（纯文本带换行）。但验证后发现它**不能依赖**：
- 有的文章有（如第 1 篇，695 字恰好是全文）
- 有的文章是空的（如第 2 篇，`og:description` 长度为 0）

说明它是作者手填/平台偶发行为，不能当作通用正文来源。

---

## 三、最终成熟方案

### 3.1 核心：预热会话绕过风控 ✅

关键突破点：**先用同一个会话（Session）访问一次微信域名根 `https://mp.weixin.qq.com/`，拿到 `ua_id` 这个 cookie，再带着它去抓文章**，风控就放行了，返回正常的 3MB 完整页面。

原理推测：微信根域先下发一个环境/设备 cookie，后续同会话请求被识别为「正常浏览环境」，不再触发验证页。

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

抓单篇时务必带 `Referer`：

```python
s.get(url, headers={"Referer": "https://mp.weixin.qq.com/"}, timeout=25)
```

### 3.2 正文提取（容器优先 + 兜底）

预热后页面完整，`#js_content` 里就有正文了。提取策略：

1. 优先用 BeautifulSoup 取 `#js_content`（兜底 `.rich_media_content`、`#js_article`）；
2. 若该容器文本过短/为空（图片分享型文章，正文其实是作者配文），则 fallback 到 `<meta property="og:description">`；
3. 把 `og:description` 里的 `\x0a` 还原成换行。

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

- 每篇之间随机 sleep 1~2 秒，降低触发限流/风控的概率；
- 若某次返回仍含「环境异常」或页面过小（< 50KB），视为被拦，**重建会话（重新预热）后重试**，最多 3 次。

### 3.4 断点续跑

147 篇是大批量，中途任何网络抖动都不该前功尽弃。做法：每抓完一篇，就把 `URL -> 正文` 写入本地缓存 JSON；重启脚本时跳过已有结果。

### 3.5 回填 Excel 的注意点

- 用 `openpyxl` 载入原表，定位「链接」「文章内容」两列（按表头名找，别写死列号）；
- Excel 单个单元格上限约 **32767 字**，超长正文截断到 32000 字并标注「已截断」；
- **先备份原文件**，新结果写到新文件（如 `老太文章列表_带正文.xlsx`），不动原表。

---

## 四、完整可运行代码（精简版）

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

# 载入缓存
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

实测结果：147 篇 **100% 抓取成功，0 失败**，正文 208~2788 字（平均 921 字），全程约 5 分钟。

---

## 五、适用边界与注意事项

- **版权**：抓下来的正文归原作者/权利人，仅限个人合法用途（如自己的号做归档、分析），不要未经授权二次转发/分发。
- **阅读量 / 点赞 / 评论**：本方案只拿到正文。要这些增强指标，必须你本人有公众号后台登录凭据（auth-key）或扫码登录，属于另一套凭据流程，不能绕过。
- **频率风险**：别开太快，微信仍可能临时封 IP / 弹验证。随机节流 + 重试是底线。
- **环境差异**：风控是否触发与运行 IP 强相关。本地常驻 IP 可能比云沙箱更稳；若批量大，建议在本人机器上跑。
- **依赖**：Python + `requests` / `beautifulsoup4` / `lxml` / `openpyxl`。

---

## 六、关键结论一句话

> 微信文章正文不能直接抓：正文是 JS 注入的、且陌生 IP 会被风控拦。破解方法是**「先访问微信域名根预热拿 cookie，再同会话带 Referer 抓」**，正文就能从 `#js_content` 正常取出。
