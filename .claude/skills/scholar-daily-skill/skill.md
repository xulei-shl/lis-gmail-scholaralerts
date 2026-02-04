---
name: scholar-daily-skill
description: "自动处理 Google Scholar Alerts 邮件,基于 MEMORY.md 过滤论文,生成 Markdown 日报。使用场景:用户说'生成scholar日报'、'处理scholar邮件'、'今天的scholar日报',或使用/scholar-daily命令。"
allowed-tools: "Read, Write, Bash, Skill, Task, AskUserQuestion"
---

# Google Scholar 日报生成器

作为**主智能体（指挥官）**，协调 Gmail 访问、邮件解析、语义过滤和报告生成。

## 涉及的组件

| 组件 | 类型 | 角色 | 文件位置 |
|------|------|------|----------|
| `gmail-skill` | Skill | Gmail 访问工具 | `.claude/skills/gmail-skill/SKILL.md` |
| `scholar-email-processor` | Subagent | 邮件过滤专家 | `.claude/agents/scholar-email-processor.md` |
| `email_formatter.py` | Script | 邮件解析工具 | `.claude/skills/scholar-daily-skill/scripts/email_formatter.py` |

## 快速开始

```bash
# 生成今天的日报
/scholar-daily

# 生成指定日期的日报
/scholar-daily 2026-02-03
```

## 职责分工

| 组件 | 职责 | 读取文件 |
|------|------|----------|
| **主流程** | 编排任务、汇总结果、生成日报 | 仅读取子代理返回的 JSON 结果 |
| **Subagent** | 读取论文数据、读取研究兴趣、语义过滤 | `papers_*.json` + `MEMORY.md` |

**关键原则**：
- ✅ 主流程**不读取** `MEMORY.md` - 避免冗余和上下文浪费
- ✅ 子代理自主读取 `MEMORY.md` - 每个过滤任务独立完成
- ✅ 主流程只汇总子代理返回的 JSON 结果 - 保持简洁

## 工作流

### Step 1: 解析日期并搜索邮件

```bash
# 构建日期查询 (使用 after:YYYY/M/D-1 before:YYYY/M/D+1 精确匹配当天)
target_date="2026-02-04"
prev_date="2026/2/3"
next_date="2026/2/5"
query="from:scholaralerts-noreply@google.com after:${prev_date} before:${next_date}"

# 搜索邮件
python .claude/skills/gmail-skill/scripts/gmail_skill.py search "${query}"
```

**日期查询技巧**：
- 搜索 `2026-02-04` → `after:2026/2/3 before:2026/2/5`
- 前后各扩展 1 天确保覆盖当天邮件

### Step 2: 并行读取邮件（单条消息）

```bash
# 使用 --output 参数直接保存到 temps 目录
{baseDir}="项目根目录"
temps_dir="{baseDir}/outputs/temps"

# 在单条响应中并行调用多个 Bash
Bash(command=f"python ... read {id1} --output {temps_dir}/email_{id1}.json")
Bash(command=f"python ... read {id2} --output {temps_dir}/email_{id2}.json")
# ... 所有邮件
```

### Step 3: 并行解析论文（单条消息）

```bash
# 并行调用 email_formatter.py
Bash(command=f"python ... email_formatter.py {temps_dir}/email_{id1}.json --output {temps_dir}/papers_{id1}.md --json-output {temps_dir}/papers_{id1}.json")
Bash(command=f"python ... email_formatter.py {temps_dir}/email_{id2}.json --output {temps_dir}/papers_{id2}.md --json-output {temps_dir}/papers_{id2}.json")
# ... 所有邮件
```

**输出文件**：
- `papers_{id}.md` - Markdown 格式（人工查看用）
- `papers_{id}.json` - JSON 格式（过滤阶段用）

### Step 4: 并行过滤邮件

```
# 关键：并行启动 subagent
Task(
    subagent_type="scholar-email-processor",
    description=f"过滤邮件: {email1_subject}",
    prompt=f"""请过滤以下邮件中的相关论文:

邮件 ID: {email_id}
主题: {subject}

已解析论文文件: {temps_dir}/papers_{email_id}.json

任务:
1. 读取 {temps_dir}/papers_{email_id}.json 获取论文列表
2. 读取 {baseDir}/MEMORY.md 了解研究兴趣
3. 对每篇论文进行语义过滤,判断相关度
4. 返回 JSON 格式结果

论文已解析完成,无需调用 email_formatter.py。
"""
)

# 邮件2、3、4、5... 同样在同一条消息中并行调用
```

**重要提示**：
- 论文已在 Step 3 解析完成，subagent 只需读取 JSON
- Subagent 只负责过滤，不调用 `gmail read` 或 `email_formatter.py`

### Step 5: 汇总结果

此步骤只需处理Step 4子代理返回的 JSON 结果

```python
# 等待所有 subagent 完成，收集结果
total_emails = len(results)
total_papers = sum(r["total_papers"] for r in results)
relevant_papers = []
for r in results:
    relevant_papers.extend(r["relevant_papers"])

# 按相关度排序（星级从高到低）
def star_to_number(star_str):
    return star_str.count('★')

relevant_papers.sort(key=lambda x: star_to_number(x.get("relevance_score", "★☆☆☆☆")), reverse=True)
```

### Step 6: 生成日报

基于 Step 5 汇总的结果生成日报

日报包含三个章节：

1. **统计摘要** - 邮件数、论文数、相关论文数
2. **汇总摘要** - LLM 生成的综合性摘要（150-300字）
3. **相关论文详情** - 按相关度排序的论文列表

完整模板见 [REFERENCE.md](REFERENCE.md#日报模板)。

### Step 7: 归档已处理邮件

```bash
# 批量归档（使用逗号分隔多个 ID）
python .claude/skills/gmail-skill/scripts/gmail_skill.py mark-done "id1,id2,id3,id4"
```

**错误处理**：归档失败仅记录警告，不影响日报生成完成状态。

### Step 8: 清理临时文件

```bash
# 清理 temps 目录下的所有内容
rm -rf ${temps_dir}/*
```

## 输出格式

### 成功输出模板

```
✅ Scholar Alerts 日报生成完成

📊 统计:
- 处理邮件: X 封
- 总论文数: Y 篇
- 相关论文: Z 篇

📁 日报路径:
outputs/scholar-reports/scholar-report-YYYY-MM-DD.md
```

### 高光论文（可选）

如果有 5 星论文，可以简要列出标题：

```
⭐ 今日高光论文:
- [论文标题1] (★★★★★)
- [论文标题2] (★★★★★)
```

**不要输出**：完整的论文列表、所有论文的详细摘要。

## 错误处理

| 错误类型 | 处理方式 |
|----------|----------|
| 无邮件 | 提示"未找到 {date} 的 scholaralerts 邮件" |
| Subagent 失败 | 记录错误,继续处理其他邮件 |
| 无相关论文 | 提示"今日无相关论文",生成空日报（仅统计摘要） |
| 归档邮件失败 | 记录警告,不影响日报生成 |

## 文件流程

```
Gmail search → 邮件 ID 列表
        ↓
并行读取 → outputs/temps/email_{id}.json
        ↓
并行解析 → outputs/temps/papers_{id}.md + papers_{id}.json
        ↓
并行过滤 → Subagent 读取 .json, 返回过滤结果
        ↓
汇总生成日报 → outputs/scholar-reports/{date}-scholar-daily.md
        ↓
归档已处理邮件
        ↓
清理 temps 目录
```

## 参考文档

| 文档 | 内容 |
|------|------|
| [REFERENCE.md](REFERENCE.md) | 日报模板、输出格式详细说明 |
| [CONFIG.md](CONFIG.md) | 常量定义、配置说明 |
| [gmail-skill](../gmail-skill/SKILL.md) | Gmail 访问能力 |
| [scholar-email-processor](../../agents/scholar-email-processor.md) | Subagent 详细说明 |
