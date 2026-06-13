# Obsidian 插件设置

这个工作流示例中安装过 8 个插件，其中 6 个是核心启用插件，2 个是可选 AI 插件。

| 插件 ID | 插件名 | 示例版本 | 角色 |
|---|---|---:|---|
| `terminal` | Terminal | 3.26.0 | 在 Obsidian 内运行命令 |
| `dataview` | Dataview | 0.5.68 | 自动聚合文献和问题节点 |
| `obsidian-zotero-desktop-connector` | Zotero Integration | 3.2.1 | Zotero 文献卡片和 PDF 注释导入 |
| `advanced-canvas` | Advanced Canvas | 6.2.1 | 科研问题树 Canvas |
| `obsidian-excalidraw-plugin` | Excalidraw | 2.23.12 | 方法图和草图 |
| `mermaid-tools` | Mermaid Tools | 1.4.1 | Mermaid 图编辑 |
| `copilot` | Copilot | 3.3.3 | 可选 AI 问答 |
| `obsidian-textgenerator-plugin` | Text Generator | 0.8.7 | 可选 AI 生成 |

启用清单示例见 `configs/obsidian/community-plugins.example.json`。AI 插件的 `data.json` 可能含 API key，不要公开。

## 安装顺序

1. 先安装并启用 Dataview。
2. 安装 Zotero Integration，并确认 Zotero Desktop 正在运行。
3. 安装 Mermaid Tools 和 Advanced Canvas。
4. 安装 Excalidraw。
5. 安装 Terminal。
6. 可选安装 Copilot 和 Text Generator。

## Dataview

用途：

- 自动聚合问题节点。
- 自动生成文献矩阵。
- 按 `status`、`ml_opportunity`、`evidence_level` 排序。

示例：

```dataview
TABLE status AS "状态", ml_opportunity AS "ML机会", domains AS "适用域"
FROM "重磁反演知识图谱/问题节点"
WHERE type = "problem"
SORT ml_opportunity DESC
```

文献卡片聚合：

```dataview
TABLE year, methods, solves, evidence_level, status
FROM "Zotero/文献卡片"
WHERE contains(["paper", "literature"], type)
SORT year ASC
```

## Zotero Integration

用途：

- 从 Zotero 导入文献元数据。
- 从 PDF 注释导入图片和标注。
- 用统一模板生成文献卡片。

推荐配置：

- `noteImportFolder`: `Zotero`
- `outputPathTemplate`: `文献卡片/{{citekey}}.md`
- `imageOutputPathTemplate`: `附件图片/{{citekey}}/`
- `templatePath`: `Zotero/模板/Zotero文献卡片模板.md`
- `openNoteAfterImport`: `true`

逐步配置：

1. 打开 Obsidian `Settings -> Community plugins -> Zotero Integration`。
2. `Database` 选择 `Zotero`。
3. `Note Import Folder` 写 `Zotero`。
4. 新建一个 import format。
5. `Output Path Template` 写 `文献卡片/{{citekey}}.md`。
6. `Image Output Path Template` 写 `附件图片/{{citekey}}/`。
7. `Template Path` 写 `Zotero/模板/Zotero文献卡片模板.md`。
8. Citation style 选择 `china-national-standard-gb-t-7714-2015-author-date` 或你需要的 CSL。
9. 打开 `Open note after import`。
10. 用一篇 Zotero 文献测试导入，确认卡片进入 `Zotero/文献卡片/`。

常见问题：

- 如果找不到 Zotero 条目，确认 Zotero Desktop 正在运行。
- 如果 citekey 为空，确认 Zotero 已安装 Better BibTeX。
- 如果图片没有导出，检查 `附件图片/{{citekey}}/` 是否存在。

## Mermaid Tools

用途：

- 快速维护 `flowchart`、`mindmap`、`timeline`。
- 在总览页展示问题树。

适合放在：

- 研究总览。
- 科学挑战地图。
- 方法路线图。

## Advanced Canvas

用途：

- 把主干问题节点拖拽成空间图谱。
- 按“问题层级”分组。
- 用入口卡片连接问题树、文献矩阵和 PDF 底稿。

建议：

- Canvas 只放主干，不放长证据。
- 证据和页码写回 Markdown 卡片。

逐步配置：

1. 安装并启用 Advanced Canvas。
2. 新建 `重磁反演科学问题树.canvas`。
3. 中央放总览文件。
4. 左侧放问题节点，右侧放文献矩阵、机器学习机会、PDF 精读底稿。
5. 使用 group 框分层：固有病态、联合耦合、模型可信度、ML 迁移。

## Excalidraw

用途：

- 画概念图、论文图示草稿、方法框架图。
- 与 Canvas 互补：Canvas 管链接，Excalidraw 管视觉草图。

## Terminal

用途：

- 在 Obsidian 内运行 `git`、`rg`、`python`。
- 快速检查文件和链接。

常用命令：

```powershell
git status --short
rg -n "TODO|待补|pdf_path: \"\"" .
rg -n "type: problem" "重磁反演知识图谱/问题节点"
```

建议：

- Terminal 插件只作为便利入口。
- 涉及删除、移动、公开发布时，先让 Codex 或自己检查路径。

逐步配置：

1. 安装并启用 Terminal。
2. 默认 shell 建议使用 PowerShell。
3. 在 vault 根目录运行 `git status --short` 测试。
4. 常用命令保存到 `04-Codex维护指令.md` 或 `prompts/`。

## Copilot / Text Generator

可选。适合单篇笔记的初稿生成。

注意：

- 不要公开它们的 `data.json`。
- 不要把 API key 写入模板。
- 生成内容需要回到 PDF 和问题节点中核查。

建议用法：

- Copilot：问单篇笔记、局部总结、快速解释。
- Text Generator：生成段落初稿。
- Codex：负责跨文件批量修改、读 PDF、更新图谱、GitHub 发布。
