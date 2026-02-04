---
description: "生成 Google Scholar Alerts 日报,自动过滤相关论文"
allowed-tools: "Skill"
argument-hint: "[date]"
---

# Google Scholar Alerts 日报生成

生成指定日期的 Scholar Alert 日报，自动过滤相关论文。

## 参数

- `date`: 可选，指定日期 (格式: YYYY-MM-DD，默认: 今天)

## 使用示例

```
/scholar-daily              # 今天的日报
/scholar-daily 2026-02-03   # 指定日期
```

## 执行

调用 `scholar-daily-skill` 执行完整的日报生成流程。
