## 步骤 0：目标确认与准备工作

### 0.1 确认目标

- **小说名称 / 作者**
- **目标网站 URL**（目录页 URL）
- **目标范围**：从头开始 / 从指定章节 / 部分章节（如仅某卷）
- **输出目录**：默认 `工作目录/小说名/`

### 0.2 技术路线选择

```
HTTP + 解析器 (首选)                    浏览器自动化 (降级)
   ↓                                         ↓
- 网站返回静态 HTML                  - 网站有 Cloudflare/反爬
- 章节链接可直接提取                  - 内容通过 JS 动态加载
- 正文在 HTML 标签内                  - 需要点击"展开"等交互
- 无严格反爬                          - 页面需等待渲染
```

**判断方法：** 先用 `curl` 获取目录页查看 HTML 结构，如果返回了正确内容但关键内容缺失，试浏览器。

### 0.3 文件位置

- **本指引：** `../index.md`
- **清洗指引：** `../../小说清洗/index.md`
- **专用脚本：** `~/.claude/tools/fetch_{站点标识}.py`（本地路径）

### 0.4 工具链说明

| 工具 | 用途 | 命令/引用 |
|------|------|-----------|
| `curl` | HTTP 请求 | 系统自带，配合 `-L` `-A` `-e` 参数 |
| `python3` + `html.parser` | HTML 解析 | 标准库，无需额外安装 |
| `bs4` (BeautifulSoup) | HTML 解析增强 | `python3 -c "from bs4 import BeautifulSoup"` |
| `agent-browser` skill | 浏览器自动化 | 需 Skill 工具调用 |
| `clean_novel.py` | 清洗单文件 | `~/.claude/tools/clean_novel.py`（本地路径） |
| `batch_clean.py` | 批量清洗 | `~/.claude/tools/batch_clean.py`（本地路径） |

---

## 步骤 0.5：准备备用站点

在开始抓取前，预先找到 2-3 个备用源站，以便主站内容污染时切换。

### 0.5.1 搜索备用站

```bash
# 搜索小说名称，找不同的小说网站
# 优先选 biquge 系列（结构相似，复用抓取逻辑）、其次是 sudugu/iqxs 等
```

备选站选择标准：
- 章节数 ≥ 主站（表明内容完整）
- 正文容器可识别（有明确的 content div）
- 编码以 UTF-8 / GBK 为主
- 非 JS 渲染（静态 HTML）

### 0.5.2 快速验证

对每个备用站：
1. 打开目录页，确认章节列表完整
2. 抽取 1-2 章验证正文可提取
3. 记录目录页 URL、正文容器选择器、编码

### 0.5.3 记录结果

```json
{
  "primary": {
    "name": "主站名",
    "toc_url": "...",
    "content_selector": "div#content",
    "encoding": "gbk"
  },
  "backups": [
    {
      "name": "sudugu.org",
      "toc_url": "https://www.sudugu.org/{id}/",
      "content_selector": "div.con p",
      "encoding": "utf-8",
      "chapter_prefix": "https://www.sudugu.org/{id}/",
      "notes": "章节号可能与其他站偏移 ±3"
    }
  ]
}
```

---