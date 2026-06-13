# Codex Prompt: Import BibTeX

读取 `<vault>/Zotero/导出文件/<file>.bib`，为每个 BibTeX 条目在 `<vault>/Zotero/文献卡片` 下生成或更新一张文献卡片。

要求：

1. 使用 `templates/zotero-literature-note-template.md` 的字段。
2. 从 title、abstract、keywords 判断 `methods`、`solves`、`ml_related` 和 `evidence_level`。
3. 每篇文献最多挂到 1-3 个问题节点。
4. 不覆盖已有人工精读内容。
5. 生成后更新 `01-文献矩阵.md` 和相关问题节点。
6. 不复制 PDF，不上传 Zotero storage。
7. 如果发现重复 citekey/title，先报告再合并。
