# Codex Prompt: AI Literature Discovery

请围绕以下研究问题搜索新增文献候选：

```text
<research question>
```

要求：

1. 优先搜索 Crossref、OpenAlex、Semantic Scholar、arXiv、出版社页面和学科数据库。
2. 不要直接修改 Zotero 或 Obsidian。
3. 先输出候选文献清单。
4. 每条候选必须包含 title、authors、year、venue、DOI、URL、摘要要点、对应问题节点、推荐理由、风险。
5. 按 `core`、`baseline`、`adjacent` 分类。
6. 标记是否有 DOI；没有 DOI 的给出保存到 Zotero 的 URL。
7. 输出一段可直接复制到 Zotero `Add Item(s) by Identifier` 的 DOI 列表。
8. 等我确认后，再指导我导入 Zotero。

候选清单格式：

| rank | category | title | year | venue | DOI | problem node | reason | risk |
|---:|---|---|---:|---|---|---|---|---|

确认导入后，再执行：

1. 等待 Better BibTeX 自动更新 `.bib`。
2. 读取 `<vault>/Zotero/导出文件/research-library.bib`。
3. 对比已有 `Zotero/文献卡片`。
4. 只为新增 citekey 生成卡片。
5. 更新问题节点和文献矩阵。
