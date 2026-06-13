# 实操手册：让 AI 搜文献并同步 Zotero + Obsidian

这是一份可以直接照着做的 SOP。推荐采用“安全半自动”流程：AI 负责发现候选文献，研究者负责确认纳入，Zotero 负责保存正式条目和 PDF，Better BibTeX 负责同步 `.bib`，Codex 负责更新 Obsidian 问题树。

## 一次性设置

### 1. Zotero 中建立研究 collection

建议命名：

```text
Research Map - Gravity Magnetic Inversion
```

所有要进入 Obsidian 问题树的论文，都放进这个 collection。

### 2. 设置 Better BibTeX 自动导出

在 Zotero 中：

1. 右键 collection。
2. 选择 `Export Collection...`。
3. Format 选择 `Better BibTeX`。
4. 勾选 `Keep updated`。
5. 保存到：

```text
<vault>/Zotero/导出文件/research-library.bib
```

之后 Zotero collection 变大时，`.bib` 会自动更新。

### 3. Obsidian 中确认 Zotero Integration

推荐设置：

```text
Note Import Folder: Zotero
Output Path Template: 文献卡片/{{citekey}}.md
Image Output Path Template: 附件图片/{{citekey}}/
Template Path: Zotero/模板/Zotero文献卡片模板.md
```

### 4. Codex 工作区

让 Codex 打开你的 Obsidian vault，或者打开一个能访问 vault 的工作目录。

Codex 需要能读：

```text
<vault>/Zotero/导出文件/research-library.bib
<vault>/Zotero/文献卡片/
<vault>/研究主题知识图谱/问题节点/
```

## 日常扩库流程

### Step 1：让 AI 搜候选文献

在 Codex 里使用这个提示：

```text
请围绕“重力磁力联合反演中的学习式软结构耦合、非同源解耦、合成到实测迁移、生成式后验近似”搜索 20 篇候选文献。

要求：
1. 优先 Crossref、OpenAlex、Semantic Scholar、arXiv、出版社页面。
2. 不要直接修改 Zotero 或 Obsidian。
3. 输出候选表：title、authors、year、venue、DOI、URL、摘要要点、对应问题节点、推荐理由、风险。
4. 按 core / baseline / adjacent 分类。
5. 最后输出一段可复制到 Zotero Add Item(s) by Identifier 的 DOI 列表。
```

也可以直接使用：

```text
prompts/04-ai-literature-discovery.md
```

### Step 2：人工确认纳入

不要把 AI 搜到的所有论文都放进库里。每篇候选至少要回答：

- 它对应哪个问题节点？
- 是核心文献、传统基线，还是相邻方法？
- 有 DOI 吗？
- 是否可能重复？
- 是否有 PDF 或开放版本？

推荐只纳入：

- `core`：直接支撑当前科学问题树。
- `baseline`：传统方法、经典理论、重要对照。
- `adjacent`：相邻领域可迁移方法，但需要标明不是直接证据。

### Step 3：导入 Zotero

有 DOI 的文献：

1. 复制 DOI 列表。
2. 打开 Zotero。
3. 点击魔杖图标 `Add Item(s) by Identifier`。
4. 粘贴 DOI，可以一次粘贴多行。
5. Zotero 自动抓取元数据。
6. 把新增条目拖进研究 collection。
7. 尽量补 PDF。

没有 DOI 的文献：

1. 打开论文网页。
2. 用 Zotero Connector 保存。
3. 检查标题、作者、年份、URL。
4. 加入研究 collection。

### Step 4：等待 Better BibTeX 同步

确认这个文件更新时间变化：

```text
<vault>/Zotero/导出文件/research-library.bib
```

如果没有变化：

- 检查文献是否加入了正确 collection。
- 检查 Better BibTeX 导出是否勾选 `Keep updated`。
- 手动右键 collection 重新导出一次。

### Step 5：让 Codex 增量更新 Obsidian

在 Codex 里输入：

```text
读取 <vault>/Zotero/导出文件/research-library.bib。
对比 <vault>/Zotero/文献卡片/ 里的已有 citekey。
只处理新增 citekey，不覆盖已有卡片中的 PDF 精读证据和人工笔记。

请完成：
1. 为新增文献生成文献卡片。
2. 根据 title、abstract、keywords 初步挂到 1-3 个问题节点。
3. 如果 pdf_path 可用，标记为可精读；如果缺 PDF，列入缺 PDF 清单。
4. 更新 01-文献矩阵.md。
5. 更新相关问题节点的“代表文献”或“新增文献线索”。
6. 输出新增、疑似重复、缺 PDF、待精读四类清单。
```

也可以直接使用：

```text
prompts/01-import-bibtex.md
```

### Step 6：PDF 精读并更新问题树

对核心文献再运行：

```text
读取新增核心文献卡片中的 pdf_path。
逐篇阅读 PDF，给卡片补充“PDF 精读证据”，尽量标页码。
然后更新对应问题节点的综合判断。
不要只总结摘要，不要覆盖已有人工内容。
```

对应提示：

```text
prompts/02-read-pdf-update-tree.md
```

## 推荐质量检查

每次扩库后，让 Codex 检查：

```text
请检查：
1. 是否有重复 DOI 或重复 title。
2. 是否有 citekey 冲突。
3. 是否有 pdf_path 为空。
4. 是否有文献没有挂到任何问题节点。
5. 是否有 evidence_level 为空。
6. 是否有 status 仍为 unread 但已进入文献矩阵。
7. 是否有 AI 搜到但未确认的候选混进正式库。
```

## 三种自动化等级

### Level 1：推荐安全版

AI 搜候选，人工确认，Zotero UI 导入，Codex 更新 Obsidian。

优点：可靠、可控、不污染 Zotero。

### Level 2：半自动版

AI 输出 DOI 列表，研究者批量粘贴 DOI 到 Zotero，其他同步自动完成。

优点：效率高，仍有人类确认。

### Level 3：全自动版

用 Zotero Web API 或本地 API 自动创建 Zotero 条目。

不推荐作为默认方案，因为需要处理：

- Zotero API key。
- 重复条目。
- PDF 下载版权。
- 错误元数据。
- 低质量文献误入库。

如果以后要做全自动，建议先做一个 `candidate_literature.csv` 审核队列，而不是直接写 Zotero。

## 最终闭环

```text
AI 搜索候选
-> 人确认 DOI/URL
-> Zotero 保存条目和 PDF
-> Better BibTeX 自动同步 .bib
-> Codex 增量生成 Obsidian 文献卡片
-> Codex 精读 PDF 并更新问题节点
-> Dataview/Canvas 自动呈现最新研究图谱
```

这个流程的关键是：Zotero 是事实库，Obsidian 是问题图谱，Codex 是同步和整理代理。
