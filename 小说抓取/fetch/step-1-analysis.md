## 步骤 1：网站结构分析

### 1.1 获取目录页 HTML

```bash
curl -sL -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
  "目标目录页URL" -o /tmp/toc_analysis.html
```

**参数说明：**
- `-s`：静默模式
- `-L`：跟随重定向
- `-A`：设置 User-Agent（模拟浏览器）
- `-o`：输出到文件

**编码检测：**
```bash
# 检查 HTML 中声明的编码
grep -i 'charset' /tmp/toc_analysis.html | head -5
```

如果声明为 `gbk` / `gb2312` / `gb18030`，在 curl 时指定：
```bash
curl -sL -A "..." "URL" | iconv -f gbk -t utf-8 > /tmp/toc_analysis.html
```
或 Python 内做编码转换。

### 1.2 提取目录页关键结构

使用 Python 分析目录页结构：

```bash
python3 -c "
from bs4 import BeautifulSoup
import re

with open('/tmp/toc_analysis.html', 'r', encoding='utf-8', errors='replace') as f:
    soup = BeautifulSoup(f.read(), 'html.parser')

# 1. 查看所有含链接的标签，猜目录区域
link_blocks = soup.find_all(['ul', 'ol', 'div', 'dl'], class_=re.compile(r'(chapter|list|mulu|index|book)', re.I))
for b in link_blocks[:3]:
    links = b.find_all('a')
    if links:
        print(f'=== 发现 {len(links)} 个链接的块: {b.get(\"class\", \"\")} id={b.get(\"id\",\"\")} ===')
        for a in links[:5]:
            print(f'  {a.get_text(strip=True)[:50]} -> {a.get(\"href\",\"\")}')

# 2. 如果上面没找到，用更广泛的搜索
if not link_blocks:
    print('=== 未找到明显目录块，扫描所有链接 ===')
    all_links = soup.find_all('a')
    for a in all_links[:20]:
        text = a.get_text(strip=True)
        if '章' in text or '节' in text or '卷' in text:
            print(f'  {text[:50]} -> {a.get(\"href\",\"\")}')
"
```

### 1.3 分析正文页结构

```bash
# 任取一个章节 URL（从步骤 1.2 的输出中取），查看正文结构
curl -sL -A '...' '某章节URL' -o /tmp/chapter_analysis.html

python3 -c "
from bs4 import BeautifulSoup
with open('/tmp/chapter_analysis.html', 'r', encoding='utf-8', errors='replace') as f:
    soup = BeautifulSoup(f.read(), 'html.parser')

# 查看页面文本量最大的 div/article/section
for tag in ['div', 'article', 'section', 'main']:
    for el in soup.find_all(tag):
        text = el.get_text(strip=True)
        if len(text) > 200:
            cls = el.get('class', '')
            print(f'<{tag}> class={cls} id={el.get(\"id\",\"\")} 文本量={len(text)} 字')
            print(f'  前200字: {text[:200]}')
            print()
"
```

### 1.4 记录分析结果

将以下信息记录到分析暂存文件（如 `/tmp/novel_site_analysis.json`）：

```json
{
  "toc_url": "目录页 URL",
  "toc_selector": "目录提取的 CSS 选择器或查找方法",
  "chapter_link_prefix": "章节链接前缀（相对→完整 URL）",
  "content_selector": "正文提取的查找方法",
  "encoding": "页面编码",
  "chapter_title_selector": "章节标题提取方法",
  "has_anti_scrape": true/false,
  "needs_browser": true/false
}
```

---