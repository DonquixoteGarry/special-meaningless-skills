---
name: workflow-clean-novel
description: 小说清洗全流程操作指引（主索引）。按步骤惰性加载子指引，避免一次加载过多上下文。与抓取管道配合使用。
---

# 小说清洗全流程指引（主索引）

## 流程概要

每次清洗小说时，按以下 **6 步**依次执行，**不可跳步、不可提前结束**。

```
预扫描 → 规则清洗 → 拼音残留 → 屏蔽词 → 重跑 → 验证 → 汇报
```

## 子指引索引

每个步骤对应一个子指引文件，存放在 `clean/` 目录下。**只在执行该步骤时加载对应文件**，不要一次加载全部。

| 步骤 | 子指引文件 | 内容概要 | 预估行数 |
|------|-----------|---------|---------|
| 步骤 0 | `clean/step-0-prescan.md` | 基本信息采集、健康检查、问题扫描(URL/HTML/广告)、章节统计、拼音残留预检、屏蔽词预检、跨书污染预检、空章节/异常短章节检测 | ~450行 |
| 步骤 1 | `clean/step-1-rule-cleaning.md` | 运行 `batch_clean.py` 或 `clean_novel.py`，规则驱动清洗(ad_patterns→HTML→encoding_fixes→line_removal→chapter) | ~60行 |
| 步骤 2 | `clean/step-2-pinyin.md` | 拼音残留扫描(单字母嵌中文模式)、评估(确认属实vs风格化)、修复(encoding_fixes/手动/全文替换bypass) | ~70行 |
| 步骤 3 | `clean/step-3-blocked.md` | 屏蔽词扫描(`**`/口口/哔)、确认是否可修复、屏蔽词模式追加入口 | ~40行 |
| 步骤 4 | `clean/step-4-rerun.md` | 增量规则追加后重新运行 batch_clean、全量重扫确认 | ~20行 |
| 步骤 5 | `clean/step-5-verify.md` | 全部检测项重新验证(URL/HTML/广告/拼音/屏蔽词/分隔线)、章节完整性验证(章数+空章)、污染验证(跨书/语义) | ~75行 |
| 步骤 6 | `clean/step-6-report.md` | 清洗汇报结构、规则文件更新协定(ad_patterns/line_remove/encoding_fixes)、多格式支持、质量升级路径 | ~90行 |

## 加载规则

```
1. 收到清洗请求 → 先读本索引文件
2. 执行步骤 0（预扫描）→ 读 clean/step-0-prescan.md
3. 执行步骤 1 → 读 clean/step-1-rule-cleaning.md
4. 依次类推，每步只读一个子指引
5. 步骤 4（重跑）后需回到步骤 0 验证 → 重新读 step-0-prescan.md
6. 步骤 0 最重要且最长（~450行），不可跳步
```

## 与抓取管道的关系

- 抓取指引主索引：`../novel-fetching/index.md`
- 抓取子指引：`fetch/` 目录
- 先抓取 → 后清洗，不要跳过
- 清洗阶段发现污染/空章节 → 回到抓取阶段（步骤 4.6-4.8）从备用站补齐

## 工具链

| 工具 | 用途 | 路径 |
|------|------|------|
| `clean_novel.py` | 单文件清洗引擎 | `~/.claude/tools/clean_novel.py`（本地路径） |
| `batch_clean.py` | 批量清洗入口+闭环检查 | `~/.claude/tools/batch_clean.py`（本地路径） |
| `clean_novel_rules.json` | 清洗规则（单一来源） | `~/.claude/tools/clean_novel_rules.json`（本地路径） |
| `clean_filenames.py` | 文件名清洗 | `~/.claude/tools/clean_filenames.py`（本地路径） |

## 核心原则

1. **步骤 0 不可跳过**：首次清洗和每次源文件修复后都必须完整执行
2. **规则驱动**：新盗版模式只需追加 JSON 规则，不改代码
3. **独立输出**：源文件不动，结果输出到 `{源目录}_clean/`
4. **闭环验证**：清洗后自动全量扫描，发现残留即报告
5. **污染闭环**：语义污染无法用正则修复，回抓取阶段从备用站替换
6. **先修内容再修标题**：替换章节时内容优先于标题
