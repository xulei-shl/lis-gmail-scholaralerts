## 项目用途

面向图书情报学（Library and Information Science, LIS）研究者的谷歌学术提醒（Google Scholar Alerts）自动化处理系统。该系统从 Gmail 抓取学术提醒邮件，解析学术论文信息，基于研究兴趣进行语义过滤，并生成按相关性排序的每日 Markdown 格式报告。

## 主要命令

### 面向用户的命令
```bash
/scholar-daily [YYYY-MM-DD]    # 生成指定日期的每日学术报告（默认：当日）
```

### Python 脚本（通过 Bash 调用）
```bash
# Gmail 操作
python .claude/skills/gmail-skill/scripts/gmail_skill.py search "<query>"
python .claude/skills/gmail-skill/scripts/gmail_skill.py read <message_id>
python .claude/skills/gmail-skill/scripts/gmail_skill.py mark-read <message_id>

# 邮件解析
python .claude/skills/scholar-daily-skill/scripts/email_formatter.py <email_id>
```

## 架构概述

### 多智能体工作流程

```
用户命令 (/scholar-daily)
    ↓
scholar-daily-skill（主协调器）
    ├── Gmail Skill → 获取谷歌学术提醒邮件
    ├── email_formatter.py → 将 HTML 解析为 JSON/MD 格式
    ├── scholar-email-processor 智能体（并行执行）→ 语义过滤
    └── 聚合结果 → 生成每日报告
```

### 核心组件

| 组件 | 路径 | 用途 |
|-----------|----------|---------|
| 主协调器 | `.claude/skills/scholar-daily-skill/skill.md` | 协调整个工作流程，管理子智能体 |
| Gmail 客户端 | `.claude/skills/gmail-skill/scripts/gmail_skill.py` | Gmail API 封装（搜索、读取、修改） |
| 邮件解析器 | `.claude/skills/scholar-daily-skill/scripts/email_formatter.py` | 将谷歌学术提醒的 HTML 转换为结构化 JSON |
| 子智能体 | `.claude/agents/scholar-email-processor.md` | 读取 `MEMORY.md`，对论文进行语义过滤 |
| 用户配置文件 | `MEMORY.md` | 存储研究兴趣、纳入/排除关键词 |

### 数据流向

```
Gmail API → email_{id}.json → papers_{id}.json → 子智能体 → filtered_papers.json → 生成报告
```

## 重要文件

### MEMORY.md
定义用户的研究画像和过滤规则。`scholar-email-processor` 子智能体会读取此文件来判断论文的相关性。研究兴趣的任何变更都应同步更新至此文件。

### outputs/ 目录结构
- `outputs/temps/` - 调试用临时文件（报告生成后会清理）
- `outputs/scholar-reports/` - 最终的每日报告（格式：YYYY-MM-DD.md）

## 配置要求

### Gmail OAuth 配置（首次使用）
1. 创建 Google Cloud OAuth 客户端
2. 将凭证保存至 `.claude/skills/gmail-skill/scripts/credentials.json`
3. 运行一次 `gmail_skill.py`，通过浏览器流程完成认证
4. 令牌会保存至 `tokens/token_{email}.json`

注意：`credentials.json` 和 `tokens/` 目录均已加入 git 忽略列表，以保障安全。
> 参考：[google-cloud-setup.md at main · benchflow-ai/skillsbench](https://github.com/benchflow-ai/skillsbench/blob/main/tasks/scheduling-email-assistant/environment/skills/gmail-skill/docs/google-cloud-setup.md)

## 开发说明

### 并行处理模式
系统会并发启动多个 `scholar-email-processor` 子智能体，每个智能体独立处理一封解析后的邮件，最终聚合所有结果。该模式是应对大量邮件时保障性能的关键。

### 语义过滤 vs 关键词过滤
论文相关性由大语言模型（LLM）基于 `MEMORY.md` 内容的语义理解判定，而非简单的关键词匹配。这使得即便使用不同术语表述，也能识别出相关论文。

### 技能复用性
`gmail-skill` 被设计为独立、可复用的功能模块，可单独调用以完成谷歌学术提醒之外的邮件操作。

### Python 依赖
```bash
pip install google-auth google-auth-oauthlib google-auth-httplib2 google-api-python-client requests
```

### 日期查询格式
Gmail 搜索使用 `after:YYYY/M/D-1 before:YYYY/M/D+1` 格式来筛选指定日期发送的邮件，该格式可兼容时区差异。

---

### 总结
1. 核心功能：该系统为图书情报学研究者自动化处理谷歌学术提醒邮件，通过语义过滤生成每日相关性报告；
2. 技术特点：采用多智能体并行处理、语义过滤（而非关键词匹配），且 Gmail 模块可独立复用；
3. 配置关键：首次使用需完成 Gmail OAuth 认证，敏感凭证文件已做 git 忽略处理以保障安全。

---

### 致谢
本项目基于：
- [claude-skills/gmail-skill at main · idanbeck/claude-skills](https://github.com/idanbeck/claude-skills/tree/main/gmail-skill)
- [skillsbench/tasks/scheduling-email-assistant/environment/skills/gmail-skill at main · benchflow-ai/skillsbench](https://github.com/benchflow-ai/skillsbench/tree/main/tasks/scheduling-email-assistant/environment/skills/gmail-skill)