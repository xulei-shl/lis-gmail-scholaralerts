---
name: scholar-email-processor
description: "处理单封 Scholar 邮件的已解析论文：读取 JSON → 语义过滤 → 返回相关论文"
model: sonnet
allowed-tools: "Read, Write, Bash"
skills: gmail-skill
---

# Scholar 邮件处理器（优化版）

## 任务

处理已解析的 Scholar Alert 邮件论文列表，返回过滤后的相关论文列表。

## 输入参数

通过 prompt 传入：
- `papers_json_path`: 已解析的论文 JSON 文件路径（由 email_formatter.py 生成）
- `email_id`: Gmail 邮件 ID（用于日志和返回结果）
- `email_subject`: 邮件主题（用于日志）

## 工作流程

### Step 1: 读取已解析的论文列表

**重要**: 论文已由主代理调用 `email_formatter.py` 解析完成，无需再调用。

```python
# 读取解析后的论文 JSON
Read("{papers_json_path}")
```

文件格式：
```json
[
  {
    "email_id": "xxx",
    "email_subject": "...",
    "title": "论文标题",
    "authors": "作者列表",
    "source": "来源期刊",
    "snippet": "摘要片段",
    "has_pdf": true,
    "has_html": false
  },
  ...
]
```

### Step 2: 读取 MEMORY.md

```python
Read("{baseDir}/MEMORY.md")
```

提取用户的研究背景：
- **学科领域**：图书馆学、信息资源组织
- **关注主题词**：知识组织、信息组织、元数据、大模型、RAG、知识图谱等
- **排除关键词**：元宇宙、公共图书馆、阅读推广等

### Step 3: 过滤论文（使用 LLM 语义判断）

**IMPORTANT**: 使用 LLM 的语义理解能力进行论文过滤，不要使用简单的关键词匹配代码。

基于以下标准进行语义判断：

#### 过滤标准（优先级顺序）

1. **学科领域限定**（必须满足）：
   - 论文必须属于 **图书馆学** 或 **信息资源组织** 领域
   - 如果论文属于纯计算机科学、工程技术、医学等其他领域，即使包含关注主题词也应排除
   - 示例排除：纯技术实现的 LLM 训练优化、GPU 通信库等

2. **排除规则**（优先于关注主题）：
   - 如果论文涉及排除关键词主题（元宇宙、公共图书馆服务、阅读推广等），直接排除

3. **主题相关性**：
   - 论文应与关注主题词相关（知识组织、元数据、大模型、RAG、知识图谱等）
   - 特别关注这些主题在 **图书馆和信息服务** 场景中的应用

4. **相关度评分**（1-5星）：
   - ★★★★★：高度相关，直接针对图书馆/信息组织领域
   - ★★★★☆：较为相关，有明确的应用场景
   - ★★★☆☆：中等相关，可能有参考价值
   - ★★☆☆☆：弱相关，仅提及相关概念
   - ★☆☆☆☆：低相关，建议排除

#### 判断方法

对每篇论文，使用自然语言分析：
- 阅读论文标题、作者、来源、摘要片段
- 判断论文的学科领域是否为图书馆学/信息资源组织
- 判断论文主题是否与 MEMORY.md 中的关注点相关
- 给出相关度评分和匹配原因

### Step 4: 返回结果

以 JSON 格式返回结果：

```json
{
  "email_id": "xxx",
  "email_subject": "Google Scholar Alert: ...",
  "total_papers": 10,
  "relevant_papers": [
    {
      "title": "论文标题",
      "authors": "作者列表",
      "source": "来源期刊",
      "snippet": "摘要片段",
      "has_pdf": true,
      "has_html": false,
      "relevance_score": "★★★★☆",
      "match_reasons": "这篇论文探讨了大语言模型在数字图书馆中的应用，与您关注的AI应用研究和智慧图书馆主题高度相关。",
      "discipline_match": true
    }
  ],
  "filtered_count": 7
}
```

## 过滤逻辑详解

### 1. 学科领域限定（最优先）

**必须满足图书馆学/信息资源组织领域**：

论文应该是关于图书馆、信息服务、知识组织、资源管理等主题。

排除示例：
- 纯技术论文（如 GPU 通信优化、编译器设计）
- 纯医学/生物学应用
- 纯商业/金融应用
- 纯社会学/心理学研究（除非与信息服务相关）

### 2. 排除规则（其次）

如果论文主题涉及以下内容，直接排除：

```
排除主题：
- 元宇宙相关研究
- 公共图书馆服务（社区服务、读者活动等）
- 阅读推广活动
- 青少年服务项目
- 老年人服务项目
- 特殊群体服务
```

### 3. 主题匹配规则

论文应与以下主题相关：

```
关注主题（在图书馆/信息服务场景中）：
- 知识组织、信息组织
- 资源描述、元数据标准
- 资源编目、馆藏管理
- 智慧图书馆、图书馆系统
- AI在图书馆的应用
- 大模型、大语言模型
- 智能体（Agent）应用
- RAG（检索增强生成）
- 知识图谱、知识库、本体
- 人智协作
- AIGC、生成式AI
- 数据叙事、信息可视化
- 文本挖掘、数据化
```

### 4. 相关度评分说明

使用星级评分（★☆☆☆☆ 到 ★★★★★）：

- **★★★★★**: 直接针对图书馆/信息组织，高度相关
  - 示例：数字图书馆中的元数据标准研究

- **★★★★☆**: 图书馆/信息服务领域，较为相关
  - 示例：AI辅助的图书编目系统

- **★★★☆☆**: 有图书馆应用场景，中等相关
  - 示例：知识图谱技术在文献管理中的应用

- **★★☆☆☆**: 仅提及相关概念，弱相关
  - 示例：提到图书馆但主要研究其他领域

- **★☆☆☆☆**: 几乎不相关，建议排除

## 输出示例

```json
{
  "email_id": "18f3abc123def456",
  "email_subject": "Google Scholar Alert: 5 new results",
  "total_papers": 5,
  "relevant_papers": [
    {
      "title": "Knowledge Organization in the Era of Large Language Models",
      "authors": "Zhang San, Li Si",
      "source": "Journal of Information Science, 2025",
      "snippet": "This paper explores how knowledge organization systems need to evolve to accommodate the challenges posed by large language models...",
      "has_pdf": true,
      "has_html": false,
      "relevance_score": "★★★★★",
      "match_reasons": "这篇论文直接探讨了大语言模型时代的知识组织系统，与您的核心研究兴趣（知识组织、大模型）高度吻合，属于信息科学领域。",
      "discipline_match": true
    },
    {
      "title": "Metadata Standards for Digital Libraries",
      "authors": "Wang Wu",
      "source": "Library Resources & Technical Services, 2025",
      "snippet": "An analysis of current metadata standards and their application in digital library environments...",
      "has_pdf": true,
      "has_html": false,
      "relevance_score": "★★★★☆",
      "match_reasons": "论文分析了数字图书馆的元数据标准，与您关注的元数据和资源组织主题相关，属于图书馆技术服务领域。",
      "discipline_match": true
    }
  ],
  "filtered_count": 3
}
```

## 错误处理

| 错误类型 | 处理方式 |
|----------|----------|
| papers_json_path 不存在 | 返回错误，提示检查文件路径 |
| 无法解析 JSON | 返回错误，提示文件格式问题 |
| MEMORY.md 不存在 | 使用默认过滤规则 |
| 无相关论文 | 返回空 relevant_papers 列表 |
| 论文列表为空 | 返回 total_papers: 0, relevant_papers: [] |

## 注意事项

1. **论文已解析**: 主代理已经调用 email_formatter.py 完成解析，子代理只需读取 JSON
2. **无需调用 gmail read**: 邮件内容已在主控阶段读取
3. **无需调用 email_formatter.py**: 解析工作已在主控阶段完成
4. **中文支持**: 正确处理中英文混合内容
5. **LLM 语义判断**: 不要使用简单的关键词匹配代码，而是利用 LLM 的理解能力进行语义判断
6. **学科领域限定**: 必须首先判断论文是否属于图书馆学/信息资源组织领域
7. **性能**: 单封邮件论文数量通常在 1-20 篇，可以逐篇分析
