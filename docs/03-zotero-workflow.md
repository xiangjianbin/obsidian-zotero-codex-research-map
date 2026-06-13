# Zotero 文献工作流

## Zotero 负责什么

Zotero 是文献事实来源，负责：

- DOI
- URL
- 作者、年份、期刊
- PDF 附件
- 本地附件路径
- citation key
- 原始 PDF

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

## Better BibTeX 自动同步

推荐把 `.bib` 设置成自动更新，而不是每次手动导出。

操作：

1. Zotero 中新建一个 collection，例如 `Gravity-Magnetic Inversion`。
2. 把要进入知识图谱的论文都放进这个 collection。
3. 右键 collection，选择 `Export Collection...`。
4. Format 选择 `Better BibTeX`。
5. 勾选 `Keep updated`。
6. 保存到 `<vault>/Zotero/导出文件/research-library.bib`。
7. 以后 Zotero collection 变大时，这个 `.bib` 会自动同步。

同步后的 Codex 处理逻辑：

1. 读取 `research-library.bib`。
2. 提取全部 citekey。
3. 对比 `Zotero/文献卡片/` 中已有卡片。
4. 只为新增 citekey 创建卡片。
5. 旧卡片只补空字段，不覆盖 `PDF 精读证据` 和人工笔记。
6. 更新 `01-文献矩阵.md`、问题节点和 Dataview 字段。

这个流程让仓库可以跟着 Zotero 文献库持续变大。

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

## Zotero 与 Obsidian 的职责边界

- PDF 永远优先保存在 Zotero。
- Obsidian 文献卡片只保存路径、证据和问题树关系。
- `.bib` 是 Zotero 到 Obsidian/Codex 的同步接口。
- 不建议直接修改 Zotero `storage` 目录。
- 不建议把 Zotero 数据库或 PDF 上传到 GitHub。
