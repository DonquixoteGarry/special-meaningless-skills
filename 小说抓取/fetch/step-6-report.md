## 步骤 6：汇报

向用户按以下结构汇报：

### 6.1 抓取概览
```
小说：{名称}
来源：{网站 URL}
章节：{总数} 章（成功 {成功数} / 失败 {失败数}）
原始文件：{路径}（{文件大小}）
耗时：{分钟} 分钟
```

### 6.2 异常记录
```
抓取失败的章节：
  - 第 X 章（{URL}）：原因
  - 第 Y 章（{URL}）：原因

需要浏览器降级的页面：
  - {页面 URL}：原因
```

### 6.3 清洗结果
```
清洗后文件：{路径}
移除广告：X 处
编码修复：X 处
删除行：X 行
章节数：X 章
```

### 6.4 后续建议
```
- 是否需要进一步拼音残留修复？
- 是否需要查找其他来源补充失败章节？
- 是否要追加新发现的广告模式到规则文件？
```

---

## 常见小说网站结构速查

此部分记录了常见网站的结构特征，随经验持续补充。

### 通用特征
| 特征 | 常见选择器/模式 |
|------|-----------------|
| 目录容器 | `div#list`、`div.list`、`ul.chapter`、`dl` |
| 章节链接 | `<dd><a href="...">`、`<li><a href="...">` |
| 正文容器 | `div#content`、`div#chaptercontent`、`div.content` |
| 章节标题 | `div.bookname h1`、`div.content h1`、`<h1>` |
| 编码 | GBK 为主，少数 UTF-8 |
| 防盗链 | 检查 Referer，部分需要 `Cookie` |

### 已分析网站

| 网站 | 源码系统 | URL 规律 | 目录页容器 | 正文容器 | 编码 | 章节数获取 | 备注 |
|------|----------|----------|-----------|---------|------|-----------|------|
| biquinfo.com | jieqi（杰奇） | `/book/{id}/` → 目录页；`/book/{id}/{cid}.html` → 章节页 | `ul#section-list`（注意: 有 `ul.section-list.fix` 仅显示最新12章，必须用 id 选择） | `div#content` | GB18030（`<meta>` 可能误标为 utf-8） | 从 `ul#section-list` `li` 数 | Referer 必须加，否则内容不完整；TOC URL 必须带尾部 `/` |
| sudugu.org | 通用模板 | `/{id}/` → 目录页；`/{id}/{aid}.html` → 章节页 | `<a>` 列表（直接用 `<a href="/{id}/{aid}.html">第X章...</a>`） | `div.con` + `<p>` 段落，首行含章节标题需剥离 | UTF-8 | 从 `<a>` 链接正则提取 | 内容干净，章节号比 biquge 类多约 3 章；URL 可能不稳定（曾从 `/42_42496/` 变为 `/144/`）；适合做备用站。**⚠️ 目录页可能不完整：仅显示首~1000章和末尾几章，中段章节不可见。不能用于全量备用，只能补个别缺失章节（需通过 CID 扫描发现）** |
| cheyil.cc | 通用模板 | `/book/{id}/` → 目录页；`/book/{id}/{cid}.html` → 章节页（多页: `{cid}_N.html`） | `<div>` 内 `<a>` 列表 | `div#booktxt` + `<p>` 段落，章节标题在 `<h1>` 标签 | UTF-8 | 从目录页 `<a>` 数 | **章节分多页显示**（如【1 / 10】），需依次抓取 `_{2..N}.html` 合并；有 "车毅小说网" 广告行需过滤；适合做备用站补缺失章节 |
| 22biqu PC (www.22biqu.com) | 通用模板 | `/biqu{id}/` → 目录页（分页 `/biqu{id}/{pg}/`）；`/biqu{id}/{cid}.html` → 章节页 | 分页目录，每页~200章+最新12章（重复）。第1页仅展示首末段章节，中间章节在后续分页 | `div.content` + `<p>` 段落 | UTF-8 | 从分页目录页 `<a>` 提取，按章节号去重 | **章节标题混合格式**：早期用中文数字（第二百零一章），后期用阿拉伯数字（第1770章）；有重复章节链接（同章不同URL）；有章节用 `div#content` id 而非 class；**存在服务端数据损坏风险**：个别章节页面可能返回全 `?` 字符（0x3F），需备用源补齐 |
| 22biqu Mobile (m.22biqu.net) | 通用模板 | `/biqu{id}/{cid}.html` → 章节页；分页目录 `/biqu{id}/{pg}/` | 分页（`/biqu{id}/2/`）第1页仅展示首末段章节（~50章 + ~5章），中间大量章节在后续分页 | `div#chaptercontent` + `<p>` 段落 | UTF-8 | 从分页目录页 `<a>` 提取 | **CID 不稳定**，需每次从目录页或反向遍历发现；章节页**只有"上一章"链接**；"上一章"可能指向作者感言；有 **+3 章节偏移**（22biqu chN = qidian ch(N-3)）；章节内容可能分多页（`_2.html`）；Python SSL 会报 EOF 错误，用 curl 或 `ssl.CERT_NONE` |

---

## 跨站章节 ID 发现与章节号偏移

使用 CID 或数字 ID 标识章节的站点（如 biquge 系列），需要通过目录页或导航发现 ID，不支持直接拼接。

### 目录页分页提取

```python
def discover_chapter_ids(toc_base_url, page_pattern='/page/{}/'):
    """从目录页分页发现章节 ID→章节号映射。page_pattern 接受 {} 替换为页码。"""
    import re
    id_map = {}
    page = 1
    while True:
        url = toc_base_url + page_pattern.format(page) if page > 1 else toc_base_url
        html = fetch_page(url)
        links = re.findall(r'<a[^>]*href="[^"]*?(\d+)\.html"[^>]*>(第\d+章[^<]+)</a>', html)
        if not links:
            break
        for cid, text in links:
            m = re.search(r'第(\d+)章', text)
            if m:
                ch = int(m.group(1))
                if ch not in id_map:
                    id_map[ch] = cid
        # 检查下一页
        next_m = re.search(r'下一页', html)
        if not next_m:
            break
        page += 1
    return id_map
```

### 反向遍历（仅有"上一章"链接时）

部分站点的章节页只有"上一章"链接，没有"下一章"，需反向遍历：

```python
def walk_backward(start_id, base_url_pattern):
    """从已知 ID 反向遍历。base_url_pattern 接受 {} 替换为 ID。"""
    id_map = {}
    cid = start_id
    while True:
        url = base_url_pattern.format(cid)
        html = fetch_page(url)
        m = re.search(r'第(\d+)章\s*([^<]{2,40})', html)
        if m:
            id_map[int(m.group(1))] = cid
        # 若标题不含"第N章"，跳过（可能是作者感言等非章节页）
        prev = re.search(r'href="[^"]*?(\d+)\.html"[^>]*>上一章', html)
        if prev and prev.group(1) != str(cid):
            cid = prev.group(1)
        else:
            break
    return id_map
```

### 跨站章节号偏移

不同盗版源的章节编号可能不一致（某站多插入了番外/感言导致后续整体偏移）。修复时必须先确认偏移量：

1. 取两个源的章节标题列表
2. 找连续 3 章以上标题一致的区间确认对齐点
3. 偏移区间的内容替换使用整体区间重建（避免逐章替换的 dict-key 覆盖 Bug）

---

## 脚本化建议

---

## 脚本化建议

如果用户经常从同一网站抓取，可针对该网站编写专用 Python 脚本，存放于 `~/.claude/tools/` 目录，遵循以下规范：

```
- 脚本名：fetch_{站点标识}.py
- 入口：python3 fetch_{站点标识}.py <目录页URL> [起始章节] [结束章节]
- 输出：当前目录/{小说名}.txt
- 日志：输出到 stdout，包含进度和错误
- 错误处理：重试 3 次，失败章节记录到 failed_chapters.json
```

---

## 核心原则

1. **先分析后抓取**：先摸清网站结构再动手，不盲目猜测
2. **礼貌抓取**：合理延时、设置 User-Agent、不过度请求
3. **降级有序**：HTTP → 浏览器，逐级降级，不一开始就用浏览器
4. **保留原始**：始终保留未经修改的原始抓取文件
5. **管道对接**：抓取只是上游，清洗才是出品质量的保障
6. **污染闭环**：全书抓取完成后必须抽样语义检查，发现污染则从备用站补齐，确保内容纯净
7. **增量积累**：新站点的分析结果记录到速查表，后续复用