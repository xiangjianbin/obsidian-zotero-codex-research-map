# Repository Instructions for Codex

This repository is documentation-first. Keep edits focused, readable, and safe to publish.

## Do

- Write documentation in Chinese by default.
- Keep examples generic unless they are under `examples/gravity-magnetic/`.
- Use placeholder paths such as `<vault>/Zotero/文献卡片`, not personal absolute paths.
- Keep templates small and copyable.
- Preserve the two-version structure: gravity-magnetic example and general template.
- Add security warnings when a workflow touches PDFs, Zotero storage, API keys, or GitHub tokens.

## Do Not

- Do not commit copyrighted PDFs, Zotero `storage`, raw personal BibTeX exports, or full private vaults.
- Do not include API keys, GitHub tokens, Copilot settings, Text Generator keys, or local account identifiers.
- Do not present machine-learning outputs as verified science unless the workflow includes PDF evidence and physical residual checks.

## Preferred Verification

- Check Markdown links and headings with `rg`.
- Run `git status --short` before final handoff.
- If changing JSON examples, parse them with PowerShell `ConvertFrom-Json` or Python `json.loads`.
