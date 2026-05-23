## 步骤 0：预扫描（Pre-scan）

**目标：** 在运行任何清洗前，全面了解文件状况

**⚠️ 不可跳步：** 首次清洗和每次源文件修复后，都必须完整执行步骤 0 的全部子步骤。不要因为"已经扫描过了"就跳过修复后的二次预扫描——第一轮可能因关键词覆盖不全漏掉邻近章节的污染，修复后再扫可能发现新问题。

### 0.1 基本信息采集

- 文件路径、大小（超过 100MB 会触发警告）
- 总行数（`wc -l`）
- 编码检测：优先 UTF-8 → GB18030 → chardet 兜底
- 文件扩展名：TXT / EPUB / PDF 走不同提取路径

### 0.2 原文件健康检查

- 文件末尾是否完整（通常含"全书完"、后记、番外等标记，`tail -3`）
- 文件头部是否有元数据（书名、作者等）
- 是否包含 null bytes（`\x00`）—— 若有则编码识别可能出错

### 0.3 扫描明显问题

使用和代码一致的检查逻辑：

**URL/域名：** `https?://`、`www\.\S+\.\w+`、`\.(com|net|org|cc|cn|tw)\b`
**HTML 标签：** `<[^>]+>`、`&\w+;`、`&#\d+;`
**广告关键词：** 笔趣阁 / biqu\d{3} / 更多精彩 / 本书来自 / 爱下电子书 / 速读谷 / 飘天文 / 24K小说 / 严正声明 / Txt,Epub,Mobi / qinkan / 章节内容开始 / 章节内容结束 / 蓝薬提醒 / 阅读更多小说最新章节 / 51read
**导航文本残留：** `上一章\s*←` / `目录\s*→` / `下一页` — 抓取自 biquge 系列站点的章节间常有此导航文本，应在清洗阶段移除
**裸 URL 行：** `^https?://\S+$`
**裸域名行：** `(?:https?://|www\.)[a-zA-Z0-9.-]+\.[a-z]{2,}`
**装饰分隔线：** `^[=\-_*]{10,}$`

### 0.4 章节统计

- 章检测正则：`^第(\s*[一二三四五六七八九十百千零\d]+\s*)([章节])\s*(.*)$`
- 备选格式：`^(\d{3,})\s+(.*)$`、`^=== (\d+)【(.+)】\s*===\s*$`
- 卷检测正则：`^第(\s*[一二三四五六七八九十百千零\d]+\s*)卷\s*(.*)$`
- 统计章/卷数量

### 0.5 拼音残留预检

搜索单字母 `[a-zāáǎàōóǒòēéěèīíǐìūúǔùǖǘǚǜ]` 出现在中文字符之间（含声调字母）：

```
[一-鿟》）」][a-zāáǎàōóǒòēéěèīíǐìūúǔùǖǘǚǜ][一-鿟《（「]
```

**说明：** 中文小说中出现 ASCII 小写字母或带声调字母，几乎 100% 是编码损坏或拼音替代屏蔽词。

### 0.6 屏蔽词预检

- `\*{2,}`：搜索 `**` / `***` 等显式屏蔽标记
- `口口`：注意区分是屏蔽盒（通常单独出现，替换原文中的一个词）还是正常词语（口口相传、口口声声、一口口）
- `哔` / `哔——`：拟声替代屏蔽

### 0.7 跨书污染预检（重要！）

盗版源文件常混入其他小说的段落甚至整章内容。在开始清洗前必须检查：

**搜索现代元素（对历史/古代背景小说尤其重要）：**
```
电脑 / 手机 / 耳机 / 网页 / 网站 / 微信 / 电话 / 汽车 / 飞机 / 作训服
```

**搜索知名角色名（跨书穿帮）：**
```
卡卡西 / 佐助 / 火影 / 白绝 / 秽土转生       # 火影同人
林辰 / 韩宥 / 陈铭 / 皇甫雪 / 赵芊羽 / 星野冰  # 其他小说主角
```

**搜索异常题材信号：**
```
赏金猎人 / 弹幕时间 / 宠兽空间 / 野熊 / 魂力   # 游戏/玄幻
美苏冷战 / 戴林 / 工业园区                      # 近现代/工业
```

**判断方法：**
- 如果小说是古代/历史题材，出现以上任何一个 → 大概率是跨书污染
- 如果出现在章节中部而非结尾 → 不是作者彩蛋，是源文件排版错误
- 多段落连续出现 → 该章节内容完全来自其他小说，需从备用站重新抓取替换
- 单段落出现 → 该章节部分内容混入，需找干净源覆盖

**处理方案：** 不要试图用清洗规则覆盖（语义问题无法用正则解决），应借助抓取功能从备用站重新获取被污染章节替换源文件。

**替换操作注意事项（重要）：**
- **核对章节编号偏移：** 不同源站的章节编号可能不一致（如 sudugu 从 169 章起比 22biqu 多 3 章偏移）。替换前必须比较前后各 5 章的标题内容，确认 mapping 正确后再执行替换。
- **剥离作者感言/题外话：** 备用站章节末尾可能包含作者感言（如"最近睡不着有点焦躁…蓝战非"、"下一章今晚上写不完…求月票"），这些不属于正文内容，替换时必须去除。
- **作者感言关键词要同时配置简繁体：** "蓝戰非"（繁体）不匹配"蓝战非"（简体），`AUTHOR_SIGS` 中必须同时包含两种写法。同理，"请假"作为作者感言检测词有大量误报（故事中角色请假情节），应只在章节末尾位置使用。
- **替换后重新预扫描：** 每轮替换后必须重新执行完整步骤 0 预扫描（含 0.7 + 0.8），检测是否还有残留污染或新暴露的问题。不能假设修了几章就完事。

**dict-key 覆盖 Bug（重要）：**
当源文件存在重复章节时（同一章号出现两次），用 dict 以章节号为 key 存储位置会**被后一次覆盖**，导致替换打在错误位置。示例：
```python
# ❌ 错误：重复章号时后一次覆盖前一次
ch_ranges = {}
for i, line in enumerate(lines):
    m = re.match(r'^第(\d+)[章节]\s', line)
    if m:
        ch = int(m.group(1))
        if ch in targets:
            ch_ranges[ch] = {'start': i}  # 第172章出现两次，第二次覆盖第一次
# ch_ranges[172] 指向错误位置（第二个172，非原位）

# ✅ 正确：用 list 记录每次出现
ch_ranges = []
for i, line in enumerate(lines):
    m = re.match(r'^第(\d+)[章节]\s', line)
    if m:
        ch = int(m.group(1))
        if ch in targets:
            ch_ranges.append((ch, i))  # 每个出现都记录
```

**章节边界检测陷阱：**
污染内容中可能包含假章节标题（如混入的其他小说段落中含有"第XX章"），用 regex 扫描寻找下一章边界会被假标题截断。解决方案：
1. 不使用 regex 从污染区内部搜索边界
2. 改为扫描已知的下一章标题位置（基于预扫瞄统计的完整章节表）
3. 或直接以备份中已知的章节位置为边界

**22biqu 标题格式差异：**
sudugu 使用阿拉伯数字标题："第169章 丝丝入扣，光前启后"
22biqu 使用中文数字标题："第一百六十九章 高屋建瓴，函幽育明"
`clean_content()` 中的标题剥离 regex 须同时匹配两种格式：
```python
# ✅ 正确：同时匹配"第169章"和"第一百六十九章"
if re.match(r'^第[一二三四五六七八九十百千零\d]+章\s', l): continue
# ❌ 错误：只匹配阿拉伯数字
if re.match(r'^第\d+章\s', l): continue
```

### 0.8 异常长度章节检测（拓展：空章节 + 长度突变检测 + 分段污染扫描）

源文件中存在空章节（标题后无正文）意味着抓取阶段遗漏或源站缺页。此外，若某章字数与相邻章节差距超过 **1.7 倍**，该章可能存在污染（混入/缺失内容）。清洗前必须做此检测。

**⚠️ 分段污染扫描说明：** 污染可能只出现在章节的后半段或中间段落，前半段内容完全正常。因此必须将异常章节按行分割为 **前 30%、中 40%、后 30%** 三段分别检测，避免"前面几百字正常、后面污染"的漏检。

```python
# 统计每章行数 + 异常长度检测 + 分段污染扫描
with open('源文件.txt', 'r') as f:
    lines = f.readlines()

import re

# 解析所有章节
chapters = []
current_ch = None
current_start = None
for i, line in enumerate(lines):
    m = re.match(r'^第(\s*[一二三四五六七八九十百千零\d]+\s*)([章节])\s', line)
    if m:
        if current_ch is not None:
            chapters.append((current_ch, i - current_start - 1, current_start + 1, lines[current_start:i]))
        current_ch = m.group(1).strip() + m.group(2)
        current_start = i
if current_ch is not None:
    chapters.append((current_ch, len(lines) - current_start - 1, current_start + 1, lines[current_start:]))

ch_names = [c[0] for c in chapters]
ch_lines = [c[1] for c in chapters]

# ---- 1. 空/过短章节 ----
empty = [(ch_names[i], ch_lines[i]) for i, n in enumerate(ch_lines) if n <= 1]
short = [(ch_names[i], ch_lines[i]) for i, n in enumerate(ch_lines) if 1 < n < 5]

# ---- 2. 异常长度章节（与相邻章节差距 > 1.7x）----
# 分级关键词库：按照确度分级
# HIGH = 几乎肯定是污染（已知其他小说角色名/情节）
# MEDIUM = 对于古代历史题材非常可疑（现代事物）
# LOW = 题材偏离信号（需人工确认）
CONTAMINATION_KEYWORDS = {
    'HIGH': [
        # 已知其他小说角色名（来自之前检测到的污染源）
        '聿修白', '万俟陇西', '苏尘', '陶世茹', '田歆',
        '林佳佳', '傅世瑾', '夜离殇', '李世民和李元霸',  # 从上一本混入
        '婆娑那', '秦戈', '陆羽', '杨志明', '迦太坚',
        '杨青帝', '孙千思', '叶若欢', '傅奕简', '飞鸟', '绿川麻衣',
        # 知名跨书角色
        '卡卡西', '佐助', '白绝', '秽土转生', '鸣人', '宇智波',
        '林辰', '韩宥', '陈铭', '皇甫雪', '赵芊羽', '星野冰',
        '欧阳癫', '戴林',
        # 占位章标记
        '本章由于字数太少',
    ],
    'MEDIUM': [
        # 现代科技/社会（古代小说中不应出现）
        '电脑', '手机', '互联网', '微信', 'QQ', '微博', '抖音',
        '网页', '网站', '网址', '邮箱', '宽带', 'WiFi',
        '蓝牙', '芯片', '软件', 'App', 'APP', '程序',
        '高铁', '飞机', '汽车', '摩托车', '地铁', '电梯',
        '空调', '冰箱', '洗衣机', '电视机', '微波炉',
        '信用卡', '支付宝', '微信支付', '扫码', '外卖',
        # 现代概念
        '心理医生', '物业', '保安', '公司', '上市', '股价',
        '分钟', '点钟', '小时',  # 古代用"时辰""刻"
        '摄氏度', '平方米', '公里',  # 古代用不同单位
    ],
    'LOW': [
        # 修仙/玄幻（古代历史小说中可疑）
        '灵气', '丹田', '元婴', '筑基', '金丹', '渡劫', '飞升',
        '斗气', '魂力', '武魂', '魔力', '魔法', '精灵',
        # 西方/日式
        '恶魔', '骑士', '天使', '教会', '圣光',
        # 现代动作/游戏
        '赏金猎人', '弹幕', '宠兽空间', '作训服',
        # 近现代/工业
        '美苏冷战', '工业园区', '工厂', '机械',
        # 作者感言/通知（不应出现在正文章节）
        '随缘更新', '请假', '明天更新', '下章', '求月票', '求推荐',
    ],
}
# ---- 已知误报模式（穿越/历史小说的特殊语境）----
# 以下关键词在分级库中属于 MEDIUM/LOW 级别，但在特定语境下不是污染：
#
# 穿越文主角回忆现代生活：
#   "冰箱"/"手机"/"电脑"/"软件"/"微信"/"公司" → 主角上辈子记忆中的现代事物，属剧情需要
#   "司法局"/"刑警"/"物业"/"保安" → 同上，现代职业/机构
#
# 古代/政治语境中的歧义词：
#   "程序" → 朝政/司法流程中的"程序"（如"走程序"），非计算机程序
#   "天使" → "钦差天使"（天子使者），非西方宗教天使
#   "小时" → "小时候"中的"小时"是通用词，非时间单位
#
# 修辞/典故：
#   "魔法" → "只有魔法才能打败魔法"属于修辞引用，非玄幻设定
#   "骑士" → 可能出现在中外交流的语境中
#
# 判断原则：
# - 上述词语仅在古代背景小说中是"可疑"信号 → 所以放在 MEDIUM/LOW 而非 HIGH
# - 如果搭配其他更明确的污染词（如其他小说角色名）出现 → 应升级为污染判断
# - 多段落连续出现 + 其他污染词 → 大概率是真实污染
# - 单段落出现 + 其他上下文正常 → 穿越文回忆/作者文风，可忽略

def get_expected(i):
    """用相邻最多 4 章的中位数作为期望行数"""
    neighbors = []
    for offset in [-2, -1, 1, 2]:
        idx = i + offset
        if 0 <= idx < len(ch_lines) and ch_lines[idx] > 5:
            neighbors.append(ch_lines[idx])
    if not neighbors:
        return ch_lines[i]
    neighbors.sort()
    return neighbors[len(neighbors) // 2]

def segment_hits(lines_slice):
    """对一组行扫描关键词，返回 (high_hits, medium_hits, low_hits, keyword_list)"""
    text = ''.join(lines_slice)
    hits = {'HIGH': [], 'MEDIUM': [], 'LOW': []}
    for level, kws in CONTAMINATION_KEYWORDS.items():
        for kw in kws:
            if kw in text:
                hits[level].append(kw)
    return hits

THRESHOLD = 1.7
anomalies = []
for i, n in enumerate(ch_lines):
    if n <= 5:
        continue
    expected = get_expected(i)
    if expected <= 5:
        continue
    ratio = n / expected
    if ratio > THRESHOLD or ratio < 1 / THRESHOLD:
        ch_lines_data = chapters[i][3]  # actual lines of this chapter
        total = len(ch_lines_data)
        # 分成3段：前30%, 中40%, 后30%
        p1_end = max(1, int(total * 0.3))
        p2_end = max(p1_end + 1, int(total * 0.7))
        seg1 = ch_lines_data[:p1_end]
        seg2 = ch_lines_data[p1_end:p2_end]
        seg3 = ch_lines_data[p2_end:]

        hits1 = segment_hits(seg1)
        hits2 = segment_hits(seg2)
        hits3 = segment_hits(seg3)

        # 合并所有命中（去重，用于传统输出）
        all_hits = {}
        for seg_hits in [hits1, hits2, hits3]:
            for level, kws in seg_hits.items():
                for kw in kws:
                    all_hits[kw] = level

        # 判断污染是否集中在某一段
        seg_counts = [
            len(hits1['HIGH']) + len(hits1['MEDIUM']),
            len(hits2['HIGH']) + len(hits2['MEDIUM']),
            len(hits3['HIGH']) + len(hits3['MEDIUM']),
        ]
        total_hits = sum(seg_counts)
        # 如果某一段集中了过半的命中
        concentrated = ''
        if total_hits > 1:
            for si, sc in enumerate(seg_counts):
                if sc >= total_hits * 0.5:
                    pos = ['前30%', '中40%', '后30%'][si]
                    concentrated = f'（集中于{pos}）'

        # 分段详细输出
        seg_detail = ''
        if total_hits > 0:
            parts = []
            for si, seg_data in enumerate([hits1, hits2, hits3]):
                seg_labels = []
                for level in ['HIGH', 'MEDIUM', 'LOW']:
                    if seg_data[level]:
                        seg_labels.append(f'{level}:{",".join(seg_data[level][:3])}')
                if seg_labels:
                    parts.append(f'段{si+1}({["前30%","中40%","后30%"][si]}):{";".join(seg_labels)}')
            if parts:
                seg_detail = ' | ' + ', '.join(parts)

        anomalies.append({
            'chapter': ch_names[i],
            'line_no': chapters[i][2],
            'lines': n,
            'expected': expected,
            'ratio': round(ratio, 2),
            'action': '偏长' if ratio > THRESHOLD else '偏短',
            'keywords': all_hits,
            'seg_counts': seg_counts,
            'concentrated': concentrated,
            'seg_detail': seg_detail,
        })

# ---- 输出 ----
print('📊 行数统计: 最少=%d, 最多=%d, 中位数=%d' % (
    min(ch_lines), max(ch_lines), sorted(ch_lines)[len(ch_lines)//2]))

if empty:
    print(f'\n❌ 空章节（正文行数≤1）: {len(empty)} 章')
    for ch, n in empty[:20]:
        print(f'  {ch} ({n}行)')
    if len(empty) > 20:
        print(f'  ...共 {len(empty)} 章')

if short:
    print(f'\n⚠️  过短章节（2-4行）: {len(short)} 章')
    for ch, n in short[:10]:
        print(f'  {ch} ({n}行)')
    if len(short) > 10:
        print(f'  ...共 {len(short)} 章')

if anomalies:
    print(f'\n🔍 异常长度章节（与邻居差距>%0.0f%%）: {len(anomalies)} 章' % ((THRESHOLD - 1) * 100))
    for a in anomalies:
        kw = ''
        if a['keywords']:
            high = [k for k, v in a['keywords'].items() if v == 'HIGH']
            med = [k for k, v in a['keywords'].items() if v == 'MEDIUM']
            low = [k for k, v in a['keywords'].items() if v == 'LOW']
            parts = []
            if high: parts.append(f'HIGH:{"+".join(high[:3])}')
            if med: parts.append(f'MED:{"+".join(med[:3])}')
            if low: parts.append(f'LOW:{"+".join(low[:3])}')
            kw = ' ⚠️' + '; '.join(parts)
        print(f'  {a["action"]} {a["chapter"]}: L{a["line_no"]} ({a["lines"]}行 vs 期望{a["expected"]}行, {a["ratio"]}x){kw}{a["concentrated"]}{a["seg_detail"]}')
else:
    print(f'\n✅ 无异常长度章节')
```

**异常长度判断与处理：**

| 情况 | 原因 | 处理方案 |
|------|------|---------|
| 空章节（正文 ≤ 1 行） | 抓取遗漏或源站缺页 | 从备用站补齐（见抓取指引 4.6） |
| 过短（2-4 行）且含"本章由于字数太少" | 盗版站占位符 | 从备用站获取真实内容替换 |
| 过短但不含占位词 | 可能是过渡章或内容缺失 | 查看上下文判断 |
| 偏长 > 1.7x + 检出污染关键词 | 该章混入其他小说内容 | 从备用站重新获取替换 |
| 偏长 > 1.7x + 污染集中在后30%段 | "前半章正常、后半章污染"型污染 | 从备用站获取替换（关键词可能不全，分段扫描更可靠） |
| 偏长 > 1.7x 但无污染关键词 | 可能是正常长章或内容重复 | 抽样验证，确认无误则忽略 |
| 偏短 < 0.59x 但有污染关键词 | 内容被截断 | 从备用站重新获取 |
| 偏短 < 0.59x + 前段正常后段混入 | 结尾被替换为其他小说内容 | 从备用站重新获取 |
| 大片长短异常（> 30% 章节） | 提取逻辑问题或全站污染 | 不要清洗修复，回抓取阶段排查 |

**可选强化：全量章节后30%段扫描**
异常长度检测只覆盖了与邻居差距 > 1.7x 的章节，但**正常长度章节的后半段也可能藏有污染**（篇幅不长所以不触发异常，但实际内容已被替换）。建议在异常检测完毕后，额外扫描所有章节的后30%段，查找 HIGH/MEDIUM 命中：

```python
print('--- 全量扫描：后30%段 HIGH/MEDIUM 集中 ---')
extra_hits = []
for i, n in enumerate(ch_lines):
    if n <= 10:
        continue
    ch_lines_data = chapters[i][3]
    total = len(ch_lines_data)
    p2_end = max(1, int(total * 0.7))
    seg3 = ch_lines_data[p2_end:]
    hits3 = segment_hits(seg3)
    total_h3 = len(hits3['HIGH']) + len(hits3['MEDIUM'])
    if total_h3 > 0:
        extra_hits.append((ch_names[i], n, hits3))
if extra_hits:
    for ch, n, h in extra_hits:
        print(f"  {ch} ({n}行): 后30%段 HIGH={h['HIGH']}, MEDIUM={h['MEDIUM']}")
else:
    print('  ✅ 所有章节后30%段无 HIGH/MEDIUM 命中')
```

**注意：** 全量扫描的命中中有大量 MEDIUM 误报（穿越文回忆、修辞），需结合"已知误报模式"逐一确认。仅当命中集中在少数章节且伴有 HIGH 级别关键词时才判定为污染。

### 0.9 污染区批量排查（重要补充）

**背景：** 本次清洗发现 sudugu 源文件 169–199 段存在大面积系统性污染——几乎全部 < 30 行的短章节都是"前 3-5 行正常、后面全部替换"型。关键词检测有一个根本局限：**每个污染源（其他小说）的角色名都不同，关键词库不可能提前覆盖所有可能混入的角色名。**

因此，**仅靠关键词命中与否判断污染，会造成大量漏检。** 需要补充以下方法：

#### 0.9.1 污染区识别

当满足以下条件时，应标记为"污染区"：

1. 同一来源文件中，连续 5 章以上存在 ANY 级别关键词命中（含 LOW）
2. 同一 30 章区间内，出现 3 个以上不同来源的污染词（说明不是单一小说混入，而是源文件段落损坏）
3. 相邻章节中，偏长者来自备用站（干净）、偏短者来自原站（可疑）

**一旦标记为污染区，不要逐章分析关键词，应直接对全部短章节实施替换。**

#### 0.9.2 短章节批量手动检查

对疑似污染区内的所有短章节（< 30 行），不论关键词命中结果，**必须手动抽样查看**。检查方法：

1. 从区间中每隔 1-2 章取一章，打印全文
2. 观察内容风格是否一致：明朝历史对话 → 突然跳到玄幻系统/现代都市/其他小说
3. 如果抽样中 > 50% 短章节存在内容断裂 → 整个区间全污染

**实际案例：** 万历明君 170-199 段，10 个短章节（17-26 行）抽样检查，9 个存在内容断裂。关键词检测仅命中 `魂力`(LOW) 和 `魔法`(LOW)，但实际 90% 的短章节都是污染。

#### 0.9.3 区间批量替换策略

当污染区间确认后：

| 污染范围 | 推荐策略 |
|---------|---------|
| 单章污染 | 从备用站获取该章替换 |
| 几章散布污染 | 逐章从备用站获取替换 |
| **> 30% 章节污染** | **直接从备用站批量获取整个区间的干净内容，整体替换** |

**批量操作流程：**
1. 从备用站逐章获取区间内所有章节，保存为独立文本块
2. 核对章节编号偏移（不同源站编号可能不一致）
3. 剥离作者感言/题外话
4. 用脚本整体替换源文件中的该区间所有章节
5. 重新执行完整步骤 0 预扫描验证

**批量替换脚本要点：**
- **不要用 dict 以章节号为 key**（见 0.7 节 dict-key 覆盖 Bug），用 list 顺序处理
- **不要依赖 regex 检测章节边界**（见 0.7 章节边界检测陷阱），使用备份文件中的已知位置
- **标题行保留源文件原有格式**（如"第169章 丝丝入扣"），22biqu 内容中的标题行需用 regex 剥离
- **作者话剥离在 fetch 阶段和 clean 阶段都要做**：fetch 函数尾部 `while` 循环只去掉了末尾连续的作者感言行，但 22biqu 内容中作者话可能后面跟着空行或"(本章完)"，导致未被捕获而通过 clean 阶段。解决方案：在 clean_content 中也检查作者话模式，覆盖中段和近尾部的作者话。

**实际案例：** 万历明君 169-200 段，逐个替换 6 章污染（第 1 轮脚本）因为边界检测问题导致替换位置错乱。改为整体区间替换（第 2 轮脚本）一次成功，32 章全部从 22biqu 获取，无失败。

---