# Security and Open-Source Checklist

Before publishing an Obsidian/Zotero research vault, remove or avoid:

- PDF files and Zotero `storage` folders.
- Raw `.bib`, `.ris`, `.enl` exports if they expose local file paths.
- `.obsidian/plugins/*/data.json` for AI plugins, because they may contain API keys or model provider settings.
- GitHub tokens, OpenAI keys, Zotero API keys, and local machine paths.
- Private notes, unpublished paper drafts, reviewer comments, and personal annotations.

Safe to publish:

- Markdown templates.
- Sanitized plugin configuration examples.
- Problem-tree methodology.
- Literature-matrix schema.
- Example prompts for Codex.
- Public bibliographic citations without PDF content.

Recommended command before publishing:

```powershell
rg -n "D:\\\\|C:\\\\|api[_-]?key|token|secret|zotero\\\\storage|\\.pdf|github_pat" .
```
