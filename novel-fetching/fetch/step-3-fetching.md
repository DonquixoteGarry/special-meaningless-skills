## 步骤 3：逐章抓取正文

### 3.1 核心抓取函数

```python
import subprocess
import time
import random
import re
from bs4 import BeautifulSoup
from urllib.parse import urljoin

# 默认请求头
HEADERS = 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'

def fetch_page(url, encoding=None):
    """使用 curl 获取页面内容，自动处理编码"""
    cmd = [
        'curl', '-sL',
        '-A', HEADERS,
        '-e', 'https://www.google.com/',
        '--connect-timeout', '30',    # 连接超时放宽到 30s
        '--max-time', '60',           # 总超时放宽到 60s
        url
    ]
    result = subprocess.run(cmd, capture_output=True)
    raw = result.stdout

    if encoding:
        try:
            return raw.decode(encoding, errors='replace')
        except LookupError:
            pass

    # 自动检测编码
    for enc in ['utf-8', 'gb18030', 'gbk', 'gb2312']:
        try:
            return raw.decode(enc)
        except (UnicodeDecodeError, LookupError):
            continue
    return raw.decode('utf-8', errors='replace')

def extract_content(html):
    """从章节 HTML 中提取正文文本（无 bs4 依赖，用正则提取）"""
    # 1. 定位正文容器
    content = None
    for pattern in [r'<div id="booktxt">(.*?)</div>',
                    r'<div id="chaptercontent"[^>]*>(.*?)</div>',
                    r'<div class="content"[^>]*>(.*?)</div>',
                    r'<div id="content"[^>]*>(.*?)</div>',
                    r'<div class="con">(.*?)</div>',
                    r'<article[^>]*>(.*?)</article>']:
        m = re.search(pattern, html, re.DOTALL)
        if m:
            content = m.group(1)
            break

    if not content:
        return None

    # 2. <p>→换行必须在剥离 HTML 标签之前做
    #    否则 <p>...</p><p>...</p> 会合并成一行
    clean = re.sub(r'</p>\s*<p[^>]*>', '\n', content)
    clean = re.sub(r'^<p[^>]*>', '', clean)
    clean = re.sub(r'</p>\s*$', '', clean)

    # 3. 剥离剩余 HTML 标签
    clean = re.sub(r'<[^>]+>', '', clean)
    clean = re.sub(r'<br\s*/?>', '\n', clean)
    clean = re.sub(r'&nbsp;', ' ', clean)

    # 4. 分行并过滤广告行
    lines = [l.strip() for l in clean.split('\n') if l.strip()]
    ad_keywords = ['手机用户', '请收藏', '推荐票', '月票',
                   'https?://', 'www\.', '本站域名',
                   '一秒记住', '车毅小说网', '本章未完']
    filtered = []
    for line in lines:
        if any(re.search(kw, line) for kw in ad_keywords):
            continue
        filtered.append(line)

    # 5. 剥离嵌入章节标题（sudugu 模式：div.con 首行是章节标题）
    if filtered and re.match(r'^第\d+[章节]\s', filtered[0]):
        filtered.pop(0)

    return '\n'.join(filtered)
```

### 3.2 抓取策略与反爬对抗

```
┌─────────────────────────────────────────────────────┐
│ 策略                   │ 适用场景                     │
├─────────────────────────┼─────────────────────────────┤
│ 1. 随机 User-Agent      │ 基础反爬，始终开启           │
│ 2. 并行抓取 + 随机延时   │ 提速同时规避频率检测         │
│ 3. 添加 Referer         │ 防盗链，始终开启             │
│ 4. 连接/总超时放宽       │ 30s 连接 + 60s 总超时        │
│ 5. IP 被封 → 换代理      │ curl --proxy ...            │
│ 6. JS 渲染 → 浏览器      │ 降级到 agent-browser        │
│ 7. Python stdout 缓冲     │ 始终用 python3 -u 或 flush=True │
└─────────────────────────────────────────────────────┘
```

**Python stdout 缓冲陷阱：** 抓取脚本的输出默认被 Python 缓冲，后台运行时输出文件可能长时间为空（0字节），无法看到进度。解决方案：
- 运行时加 `-u` 参数：`python3 -u fetch_xxx.py`
- 或在脚本中所有 `print()` 添加 `flush=True` 参数
- **不可使用 `sleep N && tail` 轮询输出，属阻塞操作**

### 3.3 并行抓取（先验证 → 再并行）

**核心流程：先单次验证内容提取逻辑，确认返回非空后再并行批量抓取。** 单章行数为 0 时不确认原因就直接并行会浪费大量请求。

```python
# 第一步：单次验证
test_html = fetch_page(target[0]['url'])
test_content = extract_content(test_html)
if not test_content or len(test_content) < 100:
    raise RuntimeError(f'内容提取失败：第1章仅 {len(test_content) if test_content else 0} 字符，需检查提取逻辑')

# 第二步：验证通过后再并行
import concurrent.futures

results = [None] * n  # 预分配列表，按 index 位置填充

with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as executor:
    futs = []
    for i, ch in enumerate(target):
        delay = None if i == 0 else random.uniform(1.5, 3.0)
        futs.append(executor.submit(fetch_one, (i, ch, delay)))

    for fut in concurrent.futures.as_completed(futs):
        idx, content = fut.result()
        results[idx] = content  # 按索引写回，天然保序

# 组装时按顺序遍历 results
with open(output_path, 'w') as f:
    for i, ch in enumerate(target):
        f.write(f'\n\n{ch["title"]}\n\n{results[i]}\n')
```

### 3.3.1 空章节检测与补救

并行抓取完成后、组装文件之前，必须做空章节检测：

```python
# 第三步（新增）：空章节检测
empty_chapters = []
for i, ch in enumerate(target):
    content = results[i]
    if not content or len(content.strip()) == 0:
        empty_chapters.append((i, ch['title'], '内容为空'))
    elif len(content.strip()) < 50:
        empty_chapters.append((i, ch['title'], f'内容过短({len(content.strip())}字)'))

if empty_chapters:
    print(f'\n⚠️  发现 {len(empty_chapters)} 个空章节:')
    for idx, title, reason in empty_chapters:
        print(f'  [{reason}] {title} (index {idx})')

    # 逐一重试空章节
    print('\n🔄 第二轮: 逐一重试空章节...')
    for idx, title, reason in empty_chapters:
        url = target[idx]['url']
        for attempt in range(1, 4):
            html = fetch_page(url)
            content = extract_content(html)
            if content and len(content.strip()) >= 50:
                results[idx] = content
                print(f'  ✅ {title}: 重试第{attempt}次成功')
                break
            else:
                print(f'  ⚠️  {title}: 第{attempt}次仍为空')
                time.sleep(attempt * 5)  # 递增等待

    # 仍为空的章节标记备用站需求
    still_empty = [(i, target[i]['title'])
                   for i, _ in empty_chapters
                   if not results[i] or len(results[i].strip()) < 50]
    if still_empty:
        print(f'\n❌ {len(still_empty)} 章仍为空，需从备用站获取:')
        for idx, title in still_empty:
            print(f'  {title} (index {idx})')
        print('→ 进入步骤 4.6 备用站补齐流程')
```

**异常长度检测（污染线索）：**

除空章节外，还要检查各章节长度是否与相邻章节差距过大（> 1.7 倍）。长度突变可能意味着内容混入或缺失：

```python
# 第四步（新增）：异常长度检测
lengths = [len(r.strip()) if r else 0 for r in results]
total = len(lengths)
if total > 10:
    THRESHOLD = 1.7
    anomalies = []
    for i in range(total):
        if lengths[i] < 50:
            continue  # 已在空章节处理
        # 取前后各2章的正常行数作为参考
        neighbors = [lengths[j] for j in range(max(0, i-2), min(total, i+3))
                     if j != i and lengths[j] > 200]
        if not neighbors:
            continue
        expected = sum(neighbors) / len(neighbors)
        ratio = lengths[i] / expected if expected > 0 else 0
        if ratio > THRESHOLD or (ratio > 0 and ratio < 1 / THRESHOLD):
            action = '偏长' if ratio > THRESHOLD else '偏短'
            anomalies.append((i, target[i]['title'], lengths[i],
                              int(expected), round(ratio, 2), action))

    if anomalies:
        print(f'\n🔍 异常长度章节（与邻居差距>{int((THRESHOLD-1)*100)}%）：{len(anomalies)} 章')
        for i, title, l, exp, r, act in anomalies[:15]:
            print(f'  {act} {title}: {l}字 vs 期望{exp}字 ({r}x)')
        # 对偏长的章节做污染关键词检查
        CONTAM_KEYWORDS = ['电脑', '手机', '网站', '卡卡西', '陈铭', '韩宥']
        long_anomalies = [(i, title) for i, title, l, exp, r, act in anomalies
                          if act == '偏长']
        if long_anomalies:
            for i, title in long_anomalies:
                text = results[i]
                found = [kw for kw in CONTAM_KEYWORDS if kw in text]
                if found:
                    print(f'  ⚠️  {title}: 检出污染词 {found}')
        if len(anomalies) > 15:
            print(f'  ...共 {len(anomalies)} 章')
```

**空章节/异常长度的常见原因：**
- **提取逻辑问题**（最常见）：正则匹配不到正文容器、`<p>`→换行顺序错误（必须在去标签之前）、嵌套 div 闭合计算错误。此时大量章节（而非个别）返回空 → 修复 `extract_content()` 函数后重新抓取
- **单章页面结构不同**：个别章节的 HTML 结构特殊（如无 `div.con` 改用 `div.content`）→ 在 `extract_content()` 中添加回退匹配模式
- **源站反爬/频率限制**：少数章节因请求被限返回空白 → 降低 workers 或增加延时后重试
- **源站数据缺失**：该章节在源站确实不存在 → 标记备用站

**重要：** 如果超过 50% 的章节返回空，几乎可以确定是提取逻辑的问题，不要盲目重试，应排查 `extract_content()` 函数。

**workers 数量建议：**
- 目标站无明显反爬：5-10 个 workers
- 目标站有频率限制：2-3 个 workers
- 遇大量 429/503 时减少 workers

```python
def fetch_with_retry(url, max_retries=3, encoding=None):
    for attempt in range(1, max_retries + 1):
        try:
            html = fetch_page(url, encoding)
            if html and len(html) > 100:  # 有实质内容
                return html
            print(f'  内容过短 ({len(html)} 字节)，第 {attempt} 次重试...')
        except Exception as e:
            print(f'  请求失败: {e}，第 {attempt} 次重试...')
        if attempt < max_retries:
            time.sleep(attempt * 3)  # 递增等待
    return None
```

### 3.4 降级到浏览器（agent-browser）

当 HTTP 方式遇到以下情况时，降级到 `agent-browser` skill：
- 页面返回空白或极小内容（<100 字节）
- 页面内容含 «请使用浏览器访问» 等提示
- 302 到验证码/挑战页面
- 内容明显是 JS 渲染后的空白壳

**降级流程：**

```markdown
1. 用 Skill 调用 agent-browser，导航到目标章节 URL
2. 等待页面完全加载
3. 用 page.content 获取完整 HTML
4. 传入 3.1 节的 extract_content() 解析
```

agent-browser 示例（skill 加载后如何指示）：

```
使用 agent-browser 打开 {URL}
等待页面加载完成
获取页面 HTML
查找正文内容区域，提取所有文本
返回提取的文本给我
```

### 3.4.5 多页章节处理

部分网站（如 cheyil.cc）将一章内容拆成多页显示（标题含【1 / 10】标记）。必须在抓取时检测并合并。

**检测信号：**
- 章节标题含 `【1 / N】` 或 `(1/N)` 等分页标记
- 正文末尾含「本章未完，请点击下一页继续阅读」
- 页面中存在形如 `{base}_2.html`、`{base}_3.html` 的翻页链接

**处理流程：**

```python
def fetch_multipage_chapter(base_url, total_pages, extractor):
    """抓取多页章节并合并"""
    all_lines = []
    for page in range(1, total_pages + 1):
        url = f'{base_url}_{page}.html' if page > 1 else f'{base_url}.html'
        html = fetch_page(url)
        lines = extractor(html)
        if lines:
            all_lines.extend(lines)
        time.sleep(0.3)  # 页间合理延时
    return '\n'.join(all_lines)
```

**分页数如何获取：**
- 从 H1 标题中正则提取：`【(\d+) / (\d+)】` → 取第二个数字
- 或从翻页链接数量推断：有 `_2.html`、`_3.html` 则总页数 ≥ 3
- 遍历直到 404 或内容为空：尝试递增 `_N.html` 直到请求失败

**注意：** 合并后需要过滤每页的尾部标记行（"本章未完，请点击下一页继续阅读"）和各页顶部的网站广告行，只保留纯正文。

### 3.5 章节级广告过滤（抓取阶段）

在每章被抓取后、写入文件前，对正文行进行基础过滤：

```python
AD_LINE_PATTERNS = [
    r'^请.*?推荐.*?收藏.*$',
    r'^.*?手机用户.*?阅读.*$',
    r'^.*(?:https?://|www\.).*$',
    r'^第[一二三四五六七八九十百千零\d]+章.*$',  # 小心：这是章节标题，跳过
]

def filter_ad_lines(text):
    """过滤单行广告（保留章节标题）"""
    lines = text.split('\n')
    clean = []
    for line in lines:
        line = line.strip()
        if not line:
            continue
        # 跳过广告行
        is_ad = False
        for pat in AD_LINE_PATTERNS:
            if re.match(pat, line):
                is_ad = True
                break
        if not is_ad:
            clean.append(line)
    return '\n'.join(clean)
```

**重要：** 此阶段的广告过滤只是**初步清理**，目的是减少存储体积。
**精细清洗（广告正则、编码修复、HTML 剥离）交给步骤 5 的清洗管道。**

---