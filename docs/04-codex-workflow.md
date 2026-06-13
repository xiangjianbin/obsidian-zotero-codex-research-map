# Codex 协作工作流

Codex 适合做本地文献工程助手。它可以在你的 Obsidian vault 中读写 Markdown、检查链接、批量更新模板、运行 `git` 和脚本。

## 推荐让 Codex 做的事

- 批量解析 BibTeX。
- 根据题录和摘要生成初始文献卡片。
- 从 PDF 提取文本并补充“PDF 精读证据”。
- 把文献挂到问题节点。
- 检查重复卡片、断链、空字段。
- 整理 GitHub 开源文档。

## 不建议完全交给 Codex 的事

- 判断某篇论文是否真正可靠。
- 判断一个实测案例是否地质合理。
- 决定是否公开私人笔记或 PDF。
- 给出最终科研结论。

## 一次完整迭代

1. 新文献进入 Zotero。
2. Better BibTeX 导出 `.bib`。
3. Codex 读取 `.bib`，生成文献卡片。
4. Codex 检查 `pdf_path`，读取 PDF。
5. Codex 写入 `PDF 精读证据`。
6. Codex 更新问题节点。
7. 研究者检查是否误判。
8. Codex 更新文献矩阵和 Canvas。
9. Git 提交。

## 适合给 Codex 的约束

```text
不要只总结论文摘要。
每篇文献必须挂到 1-3 个问题节点。
结论必须区分 theoretical / synthetic / field-case / benchmark / review。
机器学习机会必须说明输入、输出、物理约束和风险。
不要上传 PDF、Zotero storage、API key 或私人路径。
```

## 建议维护文件

在你的 vault 或仓库中保留：

- `04-Codex维护指令.md`
- `AGENTS.md`
- `prompts/`

其中 `AGENTS.md` 适合写长期规则，`prompts/` 适合保存一次性任务提示词。
