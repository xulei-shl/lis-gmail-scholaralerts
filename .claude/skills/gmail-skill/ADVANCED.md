# Gmail 技能 - 高级用法

## 处理大体积邮件内容

当邮件输出内容超过约40KB时（会显示"输出内容过大"提示），使用 `email_formatter.py` 工具格式化。

### Google Scholar Alerts 专用格式化

自动提取论文标题、作者、来源、摘要等信息。

```bash
# 第一步：搜索获取邮件ID
python {baseDir}/.claude/skills/gmail-skill/scripts/gmail_skill.py search "from:scholaralerts-noreply@google.com after:2026/2/1"

# 第二步：并行读取多封邮件，保存到临时文件
python {baseDir}/.claude/skills/gmail-skill/scripts/gmail_skill.py read "id1" > /tmp/emails.json

python {baseDir}/.claude/skills/gmail-skill/scripts/gmail_skill.py read "id2" >> /tmp/emails.json

# 第三步：使用 email_formatter.py 格式化为 Markdown
python {baseDir}/.claude/skills/gmail-skill/scripts/email_formatter.py /tmp/emails.json

# 输出自动保存到 outputs/emails/YYYY-MM-DD-google-scholar-alerts.md
```

### 使用管道直接处理

```bash
# 单封邮件直接管道
python {baseDir}/.claude/skills/gmail-skill/scripts/gmail_skill.py read "id1" | \
  python {baseDir}/.claude/skills/gmail-skill/scripts/email_formatter.py - \
  --output outputs/emails/my-alert.md
```

### email_formatter.py 参数

| 参数 | 说明 |
|------|------|
| `input` | 输入文件路径，或 `-` 表示从 stdin 读取 |
| `--output, -o` | 指定输出文件路径（可选，默认自动生成） |
| `--output-dir` | 自动生成文件的输出目录（默认：outputs/emails） |

## 并行执行模式

对于多个独立操作，应使用多个独立的 Bash 工具调用实现并行执行：

```bash
# 在单个响应中并行执行多个独立命令
python {baseDir}/.claude/skills/gmail-skill/scripts/gmail_skill.py read "id1"
```

```bash
python {baseDir}/.claude/skills/gmail-skill/scripts/gmail_skill.py read "id2"
```

> **注意**：不要使用 `&`、`&&` 或 `;` 来连接命令。每个命令应作为独立的 Bash 工具调用，以实现真正的并行执行。
