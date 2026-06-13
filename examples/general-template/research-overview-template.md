---
type: research-map
topic: ""
status: active
---

# 研究主题总览

## 入口

- [[00-问题树]]
- [[01-文献矩阵]]
- [[02-机器学习可切入问题]]
- [[Zotero/文献卡片/README]]

## 科学挑战地图

```mermaid
flowchart LR
  A[固有困难] --> A1[问题1]
  A --> A2[问题2]
  B[传统方法] --> B1[基线1]
  B --> B2[基线2]
  C[新方法机会] --> C1[机器学习机会1]
  C --> C2[机器学习机会2]
  A --> B --> C
```

## Dataview 自动问题节点表

```dataview
TABLE status AS "状态", ml_opportunity AS "ML机会", domains AS "适用域"
FROM "问题节点"
WHERE type = "problem"
SORT ml_opportunity DESC
```
