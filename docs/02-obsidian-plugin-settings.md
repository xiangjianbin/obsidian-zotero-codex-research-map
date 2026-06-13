# Obsidian 插件设置

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

## Copilot / Text Generator

可选。适合单篇笔记的初稿生成。

注意：

- 不要公开它们的 `data.json`。
- 不要把 API key 写入模板。
- 生成内容需要回到 PDF 和问题节点中核查。
