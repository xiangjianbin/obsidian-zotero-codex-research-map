# Zotero 文献工作流

## Zotero 负责什么

Zotero 是文献事实来源，负责：

- DOI
- URL
- 作者、年份、期刊
- PDF 附件
- 本地附件路径
- citation key

Obsidian 不替代 Zotero，只保存“阅读后的知识结构”。

## Obsidian 文献卡片负责什么

每张文献卡片回答：

- 这篇文章解决什么问题？
- 方法核心是什么？
- 数据或实验是什么？
- 结论和局限是什么？
- 它挂到哪些问题节点？
- 它给机器学习留下什么机会？

## 推荐字段

```yaml
type: paper
citekey: ""
title: ""
authors: []
year:
venue: ""
doi: ""
url: ""
pdf_path: ""
field: ""
problems: []
methods: []
solves: []
ml_related: false
evidence_level: ""
status: unread
```

## 批量导入

1. 在 Zotero 中选中文献。
2. 用 Better BibTeX 导出 `.bib`。
3. 放入 `<vault>/Zotero/导出文件/`。
4. 让 Codex 使用 `prompts/01-import-bibtex.md`。
5. Codex 生成或更新文献卡片。
6. 再更新 `01-文献矩阵.md` 和相关问题节点。

## PDF 精读

精读不是“总结摘要”，而是补证据：

- 关键问题在哪一页提出？
- 传统方法解决到什么程度？
- 作者承认了什么假设或局限？
- 实验是合成、实测还是 benchmark？
- 是否和传统基线公平比较？

精读后建议在文献卡片中加入：

```markdown
## PDF 精读证据

- ...
```

并在 YAML 中写：

```yaml
status: pdf-reviewed
pdf_reviewed: true
pdf_review_date: "YYYY-MM-DD"
```
