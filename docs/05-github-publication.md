# 发布到 GitHub

## 第一次发布

在仓库根目录运行：

```powershell
git init
git add .
git commit -m "Initial open-source research-map workflow"
gh repo create xiangjianbin/obsidian-zotero-codex-research-map --public --source . --remote origin --push
```

## 后续更新

```powershell
git status --short
git add .
git commit -m "Update methodology docs"
git push
```

## 公开前检查

```powershell
rg -n "D:\\\\|C:\\\\|api[_-]?key|token|secret|zotero\\\\storage|\\.pdf|github_pat" .
```

如果输出包含真实路径、密钥或 PDF，先处理再公开。

## 推荐 README 边界声明

明确写：

- 本仓库是方法论和模板。
- 不包含 PDF。
- 不包含 Zotero 私人数据库。
- 不包含完整个人 Obsidian vault。
- 示例文献只作为公开学术引用和问题树示范。
