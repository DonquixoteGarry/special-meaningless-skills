## 步骤 4：组装与输出

### 4.1 全文组装

```python
import os
import json

def assemble_novel(chapters, output_dir, novel_name):
    """将所有章节拼接为单个 TXT 文件"""
    os.makedirs(output_dir, exist_ok=True)
    output_path = os.path.join(output_dir, f'{novel_name}.txt')

    with open(output_path, 'w', encoding='utf-8') as f:
        for i, ch in enumerate(chapters, 1):
            # 写入章节标题
            f.write(f'\n\n{ch["title"]}\n\n')
            # 写入正文
            if ch.get('content'):
                f.write(ch['content'])
            # 进度
            if i % 100 == 0:
                print(f'已写入 {i}/{len(chapters)} 章')

    print(f'输出文件: {output_path}')
    return output_path
```

### 4.2 抓取状态记录

每抓一章后，将状态记录到 JSON 文件：

```json
{
  "novel": "小说名",
  "source_url": "来源站",
  "total_chapters": 1516,
  "fetched": 1516,
  "failed": 0,
  "output_file": "路径",
  "fetch_date": "2026-05-19",
  "encoding": "utf-8"
}
```

### 4.3 失败章节处理

```
- 抓取失败的章节，记录其 index 和 URL 到 failed_chapters.json
- 全部抓取完成后，对失败章节进行第二轮重试（同上 3.3 节的重试机制）
- 仍失败的，在汇报中说明
- 可以选择：跳过（保留目录标题，正文留空）/ 终止流程告知用户
```

---

## 步骤 4.5：污染抽检

主站全量抓取组装完成后，在进入清洗前，先对内容做**抽样语义检查**，确认是否存在内容污染（站方在章节中混入其他小说的段落）。

### 4.5.1 抽样策略

抽样密度不宜过高，以能发现污染为准：

| 总章节数 | 抽样量 | 方式 |
|---------|-------|------|
| < 100 章 | 5 章 | 开头/中间/结尾各抽 + 2 章随机 |
| 100-500 章 | 8-10 章 | 均匀间隔抽 5 章 + 随机 3-5 章 |
| > 500 章 | 15 章 | 每 100 章抽 1 章 + 随机 5 章 |

### 4.5.2 语义检查方法

对每章抽样，检查以下异常信号：

**信号 1：章节长度异常**
```python
# 计算各章节行数，抽出行数明显偏离平均值的章节
ch_lines = {ch: len(content.split('\\n')) for ch, content in chapters.items()}
avg = statistics.mean(ch_lines.values())
std = statistics.stdev(ch_lines.values()) if len(ch_lines) > 1 else 0
suspect = {ch for ch, n in ch_lines.items() if n < avg - 2 * std}
# 章节行数显著偏短（< 平均值 - 2σ）可能是污染
```

**信号 2：内容风格突变**
翻阅抽样章节的正文前 3-5 行，检查是否与小说背景一致：
- 古代背景小说出现现代词汇（作训服、导演、微信、跑车）
- 历史小说出现修仙/奇幻设定（魔法学院、魔皇、王母、玉帝）
- 同一章节内段落之间人物/场景/时代跳跃无过渡

**信号 3：关键词交叉碰撞**
```python
# 检查是否混入明显不属于本书的关键词
# 用小说简介/前10章提取核心词汇作为"白名单"
# 在抽样章节中搜索白名单外的异常名词
CONTAMINATION_PATTERNS = [
    # 现代元素
    '香港', '金庸', '导演', '娱乐圈', '微信', '支付宝', '跑车', '上市公司',
    # 西方奇幻
    '魔法学院', '魔皇', '骑士', '魔兽', '狂武',
    # 东方修仙
    '王母', '玉帝', '灭世血狱', '窍穴', '金丹', '元婴', '魂力',
    # 其他小说角色名（需根据上下文判断）
    '陈铭', '作训服', '赛西斯', '卡丽妲',
    # 其他小说的标志性台词/设定
    '平行位面', '本位面', '被动效果',
]
```

### 4.5.3 判断标准

```
污染判定条件（任一满足即判定为污染）：
1. 同一章节内出现 ≥3 个不相关关键词（如上表）
2. 章节行数 < 正常章节的 1/3 且内容不完整
3. 连续多个章节出现内容风格突变的段落
4. 抽样章节中混入了明显不属于该小说的完整段落（如金庸、修仙设定）
```

---

## 步骤 4.6：污染定位与修复

确认污染后，定位污染范围并用备用站替换。

### 4.6.1 定位污染范围

```bash
# 扫描所有章节，标记污染行
python3 -c "
import re
with open('源文件.txt', 'r') as f:
    lines = f.readlines()

current_ch = ''
contaminated = {}
check_patterns = ['金庸', '香港', '导演', '魔法学院', ...]

for i, line in enumerate(lines, 1):
    stripped = line.strip()
    # 检测章节标题
    m = re.match(r'^第(\d+)[章节]', stripped)
    if m:
        current_ch = m.group(1)
        continue
    for pat in check_patterns:
        if pat in stripped:
            contaminated.setdefault(current_ch, []).append((i, pat, stripped[:80]))

print(f'污染章节: {len(contaminated)} 个')
for ch, items in sorted(contaminated.items()):
    print(f'  第{ch}章: {len(items)} 处污染')
"
```

### 4.6.2 从备用站获取正确内容

备用站章节号可能偏移，**不要用章节号直接匹配**，用标题关键词匹配定位。

```python
import re, urllib.parse

def find_chapter_on_backup(keyword, backup_toc_url, backup_base, backup_encoding='utf-8'):
    """
    在备用站目录中搜索与主站相同标题的章节。
    keyword: 主站章节标题的前 2-4 字（如 '交口称赞'），避免精确匹配受异体字影响。
    """
    toc_html = fetch_page(backup_toc_url, backup_encoding)
    # 提取所有<a>链接及其文本
    links = re.findall(r'<a[^>]*href=\"([^\"]+)\"[^>]*>([^<]+)</a>', toc_html)
    for href, title in links:
        if keyword in title:
            return {
                'title': title.strip(),
                'url': urllib.parse.urljoin(backup_base, href)
            }
    return None
```

**关键字匹配策略（按优先级）：**
1. 精确匹配整句标题（最理想，但可能因异体字失败，如 輶→𬨎、瞶→瞆、著→着）
2. **前 4 字匹配**（推荐，平衡精确度和容错，如 "腹热心煎" 匹配 "第252章 腹热心煎，樛葛缠牵"）
3. **前 2-3 字匹配**（低精度但容错高，需要手动确认，适用于异体字较多的章节）
4. 章节号直接匹配（仅当确认两站章节号一致时使用）

### 4.6.3 替换污染/丢失内容

替换操作分两种场景：

**场景 A：替换污染章节（内容不干净）**
保留主站章节标题行，只替换正文内容。

**场景 B：补全空章节（内容完全丢失）**
章节标题本身可能也被误删（如"本章由于字数太少"占位章被清洗规则整行删除），需要从备用站获取完整内容（含标题）。

```python
def replace_chapter_content(source_file, chapter_num, new_content, include_title=False):
    """
    替换源文件中指定章节的内容。
    chapter_num: 章节号
    new_content: 替换正文（不含标题行）
    include_title: 是否同时替换标题行（为空章节修复使用）
    """
    with open(source_file, 'r') as f:
        text = f.read()

    if include_title:
        # 替换整个章节（含标题）：找到 "第N章\n\n" 到 "第N+1章\n\n" 之间的内容
        pattern = rf'(^第{chapter_num}章\s*\n\n)(.*?)(\n\n第{chapter_num+1}章)'
        replacement = rf'\1{new_content}\3'
    else:
        # 只替换正文（保留原标题行）
        pattern = rf'(^第{chapter_num}章\s*\n\n).*?(\n\n第{chapter_num+1}章)'
        replacement = rf'\1{new_content}\2'

    new_text = re.sub(pattern, replacement, text, count=1, flags=re.DOTALL | re.MULTILINE)

    # 备份原文件
    import shutil
    shutil.copy2(source_file, source_file + '.bak')

    with open(source_file, 'w') as f:
        f.write(new_text)
    print(f'第{chapter_num}章已替换')
```

**备用站正文提取注意事项：**
- **sudugu**：`div.con` 首行是章节标题（如 "第200章 交口称赞，犯上作乱"），必须在提取后剥离此行
- **cheyil.cc**：章节分多页显示，需用前一节的 3.4.5 多页处理流程合并；内容含 "车毅小说网" 广告行需过滤
- **标题剥离检测**：`if first_line matches r'^第\d+[章节]\s': strip it`

### 4.6.4 章节号偏移与跨站映射

不同盗版站的章节编号可能不一致，常见偏移类型：

| 偏移类型 | 示例 | 原因 |
|---------|------|------|
| 整体偏移 +N | biquinfo 第199章 → sudugu 第202章 | 某站多收录了几章番外/序章 |
| 局部缺失 | sudugu 缺第247、252章 | 源站同步不及时或源文件缺页 |
| 跨站相同 | biquinfo 第1-199章 ≈ sudugu 第1-199章 | 两站使用相同源，后续章节数差异来自更新进度 |

**映射方法：**
```python
# 按标题关键词映射，不用章节号
def build_cross_site_mapping(primary_links, backup_links):
    """
    primary_links: [(chapter_num, title), ...] 来自主站
    backup_links: [(chapter_num, title), ...] 来自备用站
    返回: {primary_num: backup_url}
    """
    mapping = {}
    backup_by_keyword = {}
    for bn, title in backup_links:
        # 用前 4 字做索引
        keywords = title.replace('第', '').replace('章', '').strip()[:4]
        backup_by_keyword[keywords] = (bn, title)

    for pn, title in primary_links:
        keywords = title.replace('第', '').replace('章', '').strip()[:4]
        if keywords in backup_by_keyword:
            mapping[pn] = backup_by_keyword[keywords]

    return mapping
```

替换完成后，重新运行污染抽检（步骤 4.5），确认：
- 原污染关键词消失
- 替换后的章节行数正常（50-200 行）
- 替换后的内容与前后章节连贯

### 4.6.5 备份策略

```
每次替换操作前自动备份原文件为 {文件名}.bak
保留替换记录到 contaminated_fix.json
```

---

## 步骤 4.7：跨书排版污染修复

盗版小说网站常将多本小说的内容混入同一个文件中（同一段落后的章节中出现另一本书的段落）。清洗规则无法处理此类语义污染，必须借助抓取功能从备用站获取正确内容替换。

### 4.7.1 发现污染

污染来源：盗版源站拼接多本书的内容到同一文件中。常见信号：

**预扫描阶段（清洗流程步骤 0.7）：**
- 古代小说中出现"电脑/手机/网站"等现代词汇
- 出现其他小说知名角色名（卡卡西、佐助、林辰、韩宥等）
- 章节标题与内容不匹配（如历史文出现玄幻标题）

**清洗后阶段（清洗流程步骤 5.4）：**
- 全量扫描通过，但阅读样本时发现剧情跳跃、人物名字突变
- 某几章的人名、地名、设定与整体小说完全不符

### 4.7.2 定位污染章节

```bash
# 找到污染词所在行号
grep -n "污染关键词" 源文件.txt

# 找到该行最近的章节标题
# 向上搜索 "^第X章" 找到章节号
```

### 4.7.3 从备用站重新抓取污染章节

1. **确认备用站 URL**（使用 0.5 节准备的备用站或搜索新站）
2. **验证单章内容**：抓取 1 章确认正文提取逻辑可用，且内容干净（无跨书污染）
3. **并行抓取**：用 `ThreadPoolExecutor` 批量抓取所有污染章节
4. **组装标准格式**：`第N章 标题\n\n{正文}`

```python
# 核心流程伪代码
targets = {168: "/book/xxx/168.html", 173: "/book/xxx/173.html"}
results = {}

# 单次验证
test_html = fetch_page(targets[168])
test_body = extract_content(test_html)
assert len(test_body) > 100, "提取失败"

# 并行抓取
with ThreadPoolExecutor(max_workers=4) as executor:
    futs = {executor.submit(fetch_one, num, url): num
            for num, url in targets.items()}
    for fut in as_completed(futs):
        num, text = fut.result()
        results[num] = text

# 替换源文件
for ch_num in sorted(results):
    text = f"第{ch_num}章 {title}\n\n{body}"
    replace_chapter_in_file(source_lines, ch_num, text)
```

**空章节检测 + 异常长度检测（替换前必须做）：**

```python
# 1. 空章节检测
empty = [num for num, text in results.items()
         if not text or len(text.strip()) < 50]
if empty:
    print(f'❌ {len(empty)} 章抓取结果为空: {empty}')
    # 大面积空 = 提取逻辑问题
    if len(empty) > len(results) / 2:
        raise RuntimeError('超过半数章节为空，检查 extract_content() 逻辑')
    for num in empty:
        del results[num]

# 2. 异常长度检测（抓取到的内容长度与邻居差距 > 1.7x 时标记污染风险）
if len(results) >= 5:
    nums = sorted(results)
    THRESHOLD = 1.7
    for i, num in enumerate(nums):
        text = results[num]
        if not text or len(text) < 50:
            continue
        # 取前后各 2 章长度为参考
        neighbors = []
        for j in range(max(0, i-2), min(len(nums), i+3)):
            if j != i:
                t = results[nums[j]]
                if t and len(t) > 100:
                    neighbors.append(len(t))
        if not neighbors:
            continue
        expected = sum(neighbors) / len(neighbors)
        ratio = len(text) / expected
        if ratio > THRESHOLD or ratio < 1 / THRESHOLD:
            action = '偏长' if ratio > THRESHOLD else '偏短'
            # 偏长的章节查污染词
            found = [kw for kw in
                     ['电脑', '手机', '网站', '卡卡西', '陈铭', '韩宥', '白绝']
                     if kw in text]
            kw_tag = f' ⚠️污染词: {found}' if found else ''
            print(f'  🔍 {action} 第{num}章: {len(text)}字 vs 期望{int(expected)}字 ({ratio:.1f}x){kw_tag}')
```

### 4.7.4 替换注意事项

1. **章节标题可能也不同**：被污染的章节标题可能来自另一本书，从备用站获取时一并替换
2. **换后行号偏移**：替换后每章行数变化，后续章节的查找位置会变。采用"从文件头开始逐章顺序替换"或"每次从完整文件列表重新查找"可避免偏移问题
3. **替换前备份**：替换前自动备份源文件为 `{文件名}_before_repair.txt`
4. **替换后必须重跑全部预扫描**：第一轮预扫描可能漏掉邻近章节的污染（"幸存者偏差"——只修了检测到的章节，但相邻章节可能含同类污染或占位内容）。替换后必须重新执行完整步骤 0 预扫描（含 0.7 污染检测 + 0.8 空章节检测），确认无残留后再进入清洗。
5. **替换后必须重跑清洗**：替换后的文件需重新执行完整清洗管道

### 4.7.5 与清洗管道的衔接

```
清洗预扫描发现污染/空章节
    ↓
回到抓取阶段 → 备用站获取正确章节 → 替换源文件
    ↓
再次执行完整预扫描（0.7 污染检测 + 0.8 空章节检测） ←─ 闭环验证
    ↓
仍有问题？─→ 重复抓取修复（可能遗漏了相邻章节）
    ↓
无问题 → 进入清洗管道
    ↓
清洗后再次验证（5.4 污染验证 + 5.5 空章节验证）
    ↓
通过
```

**关键原则：**
1. **清洗规则解决不了语义污染**（其他小说的段落嵌入）。发现此类污染后应**立即停止加清洗规则**，转用抓取功能从备用站重新获取正确内容。
2. **修复后必须闭环验证**：第一轮预扫描检测到的污染可能只是冰山一角（只检测了已知关键词，但邻近章节可能含未被关键词覆盖的同类污染）。替换后必须重新全量预扫描，不能假设修了几章就完事了。
3. **占位章容易被忽略**："本章由于字数太少"类占位章不属于"污染"范畴，但同样需要从备用站获取真实内容。空章节检测（0.8）会捕获此类问题。

---

## 步骤 4.8：源站数据损坏修复

源站个别章节页面可能出现服务端数据丢失，返回内容全部是 `?` 字符（ASCII 0x3F）。这与编码错误不同——不是解码问题，而是源站服务器端存储已损坏。

### 4.8.1 检测信号

```
- 章节内容大量连续 `???`（50个以上）
- 所有编码（UTF-8/GBK/GB18030/latin-1）解码结果相同（都是?）
- 原始字节中对应位置为 `3F 3F 3F`（ASCII 问号）
```

### 4.8.2 修复策略（按优先级）

1. **尝试移动版**：同一网站的移动版（如 m.22biqu.net）可能有独立的数据源，内容可能完好
2. **尝试浏览器渲染**：使用 agent-browser 渲染页面，JS 可能从其他 API 加载内容
3. **扫描备用站 CID**：备用站目录页不完整时（如 sudugu 中段缺失），根据 AID 范围线性估算，在 ±100 范围内扫描
4. **搜索其他源站**：用搜索引擎搜索「小说名 第X章 章节标题」找其他盗版站
5. **接受丢失**：如果只有个别章节（< 总章节数 0.1%），标记为缺失，在清洗文件中保留占位

### 4.8.3 备用站 CID 估算公式

当备用站目录页只显示首尾章节时，用线性插值估算中段章节的 ID：

```python
# 已知: ch1 的 aid=A1, chN 的 aid=A2
# 求 chX 的估计 aid:
aid_per_ch = (A2 - A1) / (N - 1)
estimated_aid = A1 + (X - 1) * aid_per_ch
# 在 estimated_aid ± 100 范围内扫描
```

**注意：** AID 步长不固定（实测每章 AID 增幅在 3~10 之间波动），必须扫描一个区间而非直接拼接。

---

### 5.1 启动清洗

抓取完成后的原始 TXT 文件，通过清洗管道处理：

```bash
# 单文件清洗
python3 ~/.claude/tools/clean_novel.py "{output_path}"  # 本地路径

# 整个目录清洗
python3 ~/.claude/tools/batch_clean.py "{output_dir}"  # 本地路径
```

### 5.2 清洗后的操作

清洗完成后，遵循 `../../novel-cleaning/index.md` 中的后续步骤：
1. **拼音残留扫描与修复**（步骤 2）
2. **屏蔽词评估**（步骤 3）
3. **规则追加与重跑**（步骤 4）
4. **全量验证**（步骤 5）
5. **汇报**（步骤 6）

### 5.3 原始文件留存

- **始终保留**抓取的原始文件（备份）
- 在原始文件所在目录下创建 `_clean/` 子目录存放清洗结果
- 不要用 `--no-backup` 选项（除非用户明确要求）

---