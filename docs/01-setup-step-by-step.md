# 从零搭建：Obsidian + Zotero + Codex

## 1. 安装软件

需要：

- Obsidian Desktop
- Zotero Desktop
- Better BibTeX for Zotero
- Git
- GitHub CLI
- Codex Desktop

可选：

- Python，用于批量解析 BibTeX 或 PDF 文本。
- VS Code，用于查看仓库。

## 2. 创建 Obsidian 文件夹

推荐结构：

```text
<vault>/
├─ 重磁反演知识图谱/
│  ├─ 00 重磁反演总览.md
│  ├─ 00-问题树.md
│  ├─ 01-文献矩阵.md
│  ├─ 02-机器学习可切入问题.md
│  ├─ 03-方法论文.md
│  ├─ 04-Codex维护指令.md
│  ├─ 问题节点/
│  ├─ 模板/
│  └─ PDF精读/
└─ Zotero/
   ├─ 文献卡片/
   ├─ 模板/
   ├─ 附件图片/
   └─ 导出文件/
```

通用领域可把 `重磁反演知识图谱` 改成你的主题名。

## 3. 安装 Obsidian 插件

在 Obsidian 中：

1. 打开 `Settings`。
2. 关闭 `Restricted mode`。
3. 进入 `Community plugins`。
4. 安装并启用：
   - Dataview
   - Zotero Integration
   - Mermaid Tools
   - Advanced Canvas
   - Excalidraw
   - Terminal

可选 AI 插件：

- Copilot
- Text Generator

注意：AI 插件可能保存 API key，不建议公开 `.obsidian/plugins/*/data.json`。

## 4. 配置 Zotero

1. 安装 Better BibTeX for Zotero。
2. 在 Zotero 中为文献生成稳定 citekey。
3. 每篇文献保留 DOI、URL、PDF、本地附件。
4. 需要批量导出时，把 `.bib` 导出到 `<vault>/Zotero/导出文件/`。

## 5. 配置 Zotero Integration

推荐设置见 `configs/obsidian/zotero-integration.example.json`。

关键字段：

- note import folder: `Zotero`
- output path template: `文献卡片/{{citekey}}.md`
- image output path template: `附件图片/{{citekey}}/`
- template path: `Zotero/模板/Zotero文献卡片模板.md`
- citation style: `china-national-standard-gb-t-7714-2015-author-date`

## 6. 配置 Codex

把 Codex Desktop 的工作区指向你的 Obsidian vault 或单独的开源仓库目录。

推荐让 Codex 做：

- 批量读取 `.bib` 并生成文献卡片。
- 读取 PDF，给文献卡片补 `PDF 精读证据`。
- 更新问题节点。
- 检查链接、重复卡片和公开前隐私风险。
- 提交并发布到 GitHub。

不建议让 Codex 自动决定：

- 论文是否真正创新。
- 实测结果是否可靠。
- 哪些私人笔记可以公开。

这些需要研究者最终检查。
