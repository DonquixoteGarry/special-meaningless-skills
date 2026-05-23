## 步骤 4：重跑清洗（Re-run）

### 4.1 修改规则文件后必须重跑

```bash
python3 ~/.claude/tools/batch_clean.py <目录> --no-backup  # 本地路径
```

**原因：** 代码的清洗流程有严格的顺序（ad_patterns → HTML → encoding_fixes → line_removal → chapter），手动修改文件可能会破坏这个管道逻辑。

### 4.2 验证新规则生效

检查重跑输出的日志，确认：
- 新增的 `🔧 编码修复` 记录出现，且计数不为 0
- 之前的修复计数仍然在（没有被覆盖）

注意：`batch_clean` 会把清洗后的文件输出到 `{目录}_clean/`，**覆盖**上一次的输出。

---