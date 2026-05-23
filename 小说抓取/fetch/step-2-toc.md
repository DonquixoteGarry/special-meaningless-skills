## 步骤 2：抓取目录与章节列表

### 2.1 完整解析章节列表

```python
# 核心逻辑示例
def parse_chapters(html, base_url):
    from bs4 import BeautifulSoup
    import re
    from urllib.parse import urljoin

    soup = BeautifulSoup(html, 'html.parser')
    chapters = []

    # 尝试常见的目录容器
    containers = [
        soup.find('div', id=re.compile(r'(list|chapter|mulu)', re.I)),
        soup.find('ul', class_=re.compile(r'(chapter|list)', re.I)),
        soup.find('div', class_=re.compile(r'(list|chapter|mulu|book)', re.I)),
        soup.find('dl', class_=re.compile(r'(chapter|list)', re.I)),
    ]

    # 也尝试 id=title="章节列表" 或类似的常见模式
    containers = [c for c in containers if c is not None]
    if not containers:
        # 兜底：找包含"章"字的链接最多的容器
        all_divs = soup.find_all(['div', 'ul', 'dl'])
        best = None
        max_chapters = 0
        for d in all_divs:
            ch_links = [a for a in d.find_all('a')
                       if re.search(r'[第\d].*[章节]', a.get_text())]
            if len(ch_links) > max_chapters:
                max_chapters = len(ch_links)
                best = d
        if best:
            containers = [best]

    seen_urls = set()
    for container in containers:
        for a_tag in container.find_all('a'):
            href = a_tag.get('href', '')
            title = a_tag.get_text(strip=True)
            if not href or not title:
                continue
            if not re.search(r'[第\d]', title) and len(chapters) > 5:
                continue  # 已收集足够章节后，无章节特征的跳过
            full_url = urljoin(base_url, href)
            if full_url in seen_urls:
                continue
            seen_urls.add(full_url)
            chapters.append({
                'title': title,
                'url': full_url,
                'index': len(chapters) + 1
            })

    print(f'解析到 {len(chapters)} 个章节')
    return chapters
```

### 2.2 分页目录处理

如果目录分多页，依次抓取并合并：

```python
def parse_paginated_toc(base_url, page_param='page', max_pages=10):
    all_chapters = []
    for page in range(1, max_pages + 1):
        url = f'{base_url}?{page_param}={page}'
        html = fetch_page(url)
        chapters = parse_chapters(html, base_url)
        if not chapters:
            break
        all_chapters.extend(chapters)
        print(f'目录第 {page} 页: {len(chapters)} 章')
    return all_chapters
```

### 2.3 章节列表验证

```
- 章节总数是否合理？（一般 200-2000 章，低于 20 可能解析不完整）
- 标题是否有明显缺失或重复？
- 章节 URL 格式是否一致？
- 对比小说简介中标注的总章节数（如有）
```

---