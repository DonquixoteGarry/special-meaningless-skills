## 步骤 1：运行规则清洗（Rule Cleaning）

### 1.1 确认规则文件

路径：`~/.claude/tools/clean_novel_rules.json`（本地路径）

规则文件分为五个部分（代码 `load_rules` 的合并逻辑）：

1. **`ad_patterns`**：`[[regex, replacement], ...]`
   - 内置默认 + 外部合并
   - 编译为 `re.compile(pat_str)` 后对全文执行 `sub()`
   - **一条正则可能有多个替换结果，eg. `(?<=[一-鿿])i(?=[一-鿿])` → `日`**
   - 按 `len(pat.findall(text))` 计数后逐个替换

2. **`line_remove_patterns`**：`[pattern_str, ...]`
   - 内置默认 + 外部合并
   - 编译为 `re.compile(p, re.IGNORECASE)` 逐行匹配
   - 匹配的行整行删除

3. **`encoding_fixes`**：`{wrong: right, ...}`
   - 内置默认 + 外部合并（外部覆盖/追加）
   - 使用 `text.replace(wrong, right)` 逐对精确替换
   - **不是正则**，只做精确子串替换

4. **章节正则**：`chapter_re` / `volume_re` / `separator_re`
   - 内置默认可被外部覆盖

5. **`alt_chapter_patterns`**：备用章节格式

### 1.2 执行命令

**批量清洗（推荐）：**
```bash
python3 ~/.claude/tools/batch_clean.py <目录>  # 本地路径
```

可选参数：`--no-backup`、`--outdir`、`--only`

**单文件清洗：**
```bash
python3 ~/.claude/tools/clean_novel.py <文件路径>  # 本地路径
```

可选：`-o 输出路径`、`--outdir 输出目录`、`--no-backup`、`--dump-rules`

### 1.3 记录清洗日志

批处理输出中会记录：
- `🗑️  广告「pat」→ replacement: 替换 X 处`
- `🏷️  HTML 标签剥离: 移除 X 个标签`
- `🔧 编码修复: 'wrong' → 'right' (X 处)`
- 章节数、文件大小变化
- `🔍 质量抽查: X 处/步长步长×3轮，✅ 无残留问题`
- `🤖 LLM 语义样本: X 段/每段 ~1000字`

**逐条记录这些数字，步骤 4 中需要验证它们是累加的。**

---