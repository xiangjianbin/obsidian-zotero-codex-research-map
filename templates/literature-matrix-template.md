---
type: evidence-matrix
field: ""
status: active
---

# 文献矩阵

## 核心证据矩阵

- [[文献卡片]]
  - 对应问题：
  - 解决程度：
  - 留下空白：
  - ML 机会：

## Dataview 自动视图

```dataview
TABLE year, methods, solves, evidence_level, status
FROM "Zotero/文献卡片"
WHERE contains(["paper", "literature"], type)
SORT year ASC
```
