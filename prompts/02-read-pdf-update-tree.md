# Codex Prompt: Read PDFs and Update Problem Tree

请读取这些文献卡片中的 `pdf_path`，逐篇提取 PDF 证据，然后更新文献卡片和问题节点。

核心要求：

1. 不要只总结摘要。
2. 每篇卡片新增或更新 `## PDF 精读证据`。
3. 每条证据尽量标注页码。
4. 区分作者解决了什么、没有解决什么、依赖什么假设。
5. 问题节点只写综合判断，不堆长摘要。
6. 更新 `status: pdf-reviewed`、`pdf_reviewed: true`、`pdf_review_date`。
7. 如果 PDF 缺失，记录为 `pdf_missing`，不要臆测。

输出：

- 更新后的文献卡片。
- 更新后的问题节点。
- 一份精读底稿，放到 `PDF精读/`。
