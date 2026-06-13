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

## 文献库变大时的增量迭代

当 Zotero 的自动同步 `.bib` 变大时，Codex 应该按增量方式处理：

1. 读取同步 BibTeX。
2. 提取全部 citekey。
3. 与 `Zotero/文献卡片/` 中已有卡片比对。
4. 只处理新增 citekey。
5. 对已存在卡片，只补缺失字段，不覆盖人工精读内容。
6. 输出新增、重复、缺 PDF、待精读四类清单。

推荐提示词见 `prompts/01-import-bibtex.md`。

## AI 搜索文献入口

如果你希望 AI 帮忙扩大文献数量，推荐把 Codex 用作“发现候选文献”的入口，而不是直接写入 Zotero：

1. Codex 根据研究问题在线搜索候选文献。
2. Codex 输出 DOI/URL 清单和纳入理由。
3. 研究者确认。
4. 研究者用 Zotero DOI 批量导入或 Zotero Connector 保存。
5. Better BibTeX 自动同步 `.bib`。
6. Codex 执行增量导入。

这样能避免 AI 把低质量或无关文献直接放进正式知识库。

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

## 推荐提示词索引

- `prompts/01-import-bibtex.md`：从同步 BibTeX 增量生成文献卡片。
- `prompts/02-read-pdf-update-tree.md`：读取 PDF 并更新问题节点。
- `prompts/03-maintain-vault.md`：检查冗余、断链和公开风险。
- `prompts/04-ai-literature-discovery.md`：AI 搜索候选文献并生成 Zotero DOI 导入清单。
