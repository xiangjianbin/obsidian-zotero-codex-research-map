# AI 搜索新文献并同步进 Zotero

目标：让 AI 帮你扩大文献库，但不要让 AI 不经确认就污染 Zotero 和问题树。

## 推荐原则

使用半自动闭环：

```text
AI 搜索候选 -> 人确认 -> Zotero 保存 -> BibTeX 自动同步 -> Codex 更新 Obsidian
```

AI 负责发现和初筛，人负责纳入决定，Zotero 负责正式保存，Codex 负责整理图谱。

## AI 搜索来源

优先使用可稳定引用的公开来源：

- Crossref
- OpenAlex
- Semantic Scholar
- arXiv
- publisher pages
- PubMed、IEEE、SEG、AGU、EGU 等学科数据库

不建议把自动化搜索完全依赖 Google Scholar，因为它不适合稳定自动抓取。

## 候选清单格式

AI 每次搜索后先输出候选表：

| 字段 | 说明 |
|---|---|
| title | 论文题名 |
| authors | 作者 |
| year | 年份 |
| venue | 期刊/会议 |
| DOI | 优先给 DOI |
| URL | DOI 或出版社页面 |
| abstract summary | 摘要要点 |
| problem nodes | 对应问题节点 |
| reason to include | 为什么值得纳入 |
| risk | 为什么可能不该纳入 |

## 人工确认后进入 Zotero

### 有 DOI 的文献

1. 复制 AI 给出的 DOI 列表。
2. 打开 Zotero。
3. 点击 `Add Item(s) by Identifier`。
4. 粘贴 DOI，可以一次粘贴多个。
5. Zotero 自动抓取元数据。
6. 如果能找到 PDF，保存 PDF；否则稍后用 Zotero Connector 或机构权限补 PDF。
7. 把文献加入你的研究 collection，例如 `Research Map`。

### 没有 DOI 的文献

1. 打开论文页面。
2. 用 Zotero Connector 保存到 Zotero。
3. 检查作者、年份、标题、URL 是否正确。
4. 加入研究 collection。

### 批量网页候选

如果候选来自出版社或数据库页面：

1. AI 输出 URL 列表。
2. 研究者逐个打开。
3. 用 Zotero Connector 保存。
4. Zotero 自动抓取 PDF 或网页元数据。

## 同步到 Obsidian

一旦 Zotero collection 更新：

1. Better BibTeX 自动更新 `research-library.bib`。
2. Codex 读取 `.bib`。
3. Codex 找出新增 citekey。
4. Codex 生成文献卡片。
5. Codex 把文献挂到问题节点。
6. Dataview 自动更新矩阵。

## Codex 搜文献提示词

见 `prompts/04-ai-literature-discovery.md`。

## 纳入标准建议

可以设置三类：

- `core`：直接回答你的主问题。
- `baseline`：传统方法、理论基线、经典约束。
- `adjacent`：相邻领域方法，可迁移但不是直接证据。

不要把所有 AI 搜到的论文都放入核心库。候选文献必须能解释它对应哪个问题节点。

## 质量控制

每次扩库后让 Codex 输出：

- 新增文献数量。
- 重复 DOI/citekey。
- 缺 PDF 文献。
- 只读摘要、未读全文的文献。
- 被挂到最多问题节点的文献。
- 没有问题节点的文献。
