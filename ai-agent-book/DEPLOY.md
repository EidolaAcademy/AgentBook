# 部署到 GitHub Pages

本目录已经按 GitHub Pages/Jekyll 方式准备：

- `index.md`：站点首页。
- `_config.yml`：GitHub Pages 配置。
- `.github/workflows/pages.yml`：GitHub Actions 部署 workflow。
- `chapters/`：每章独立 Markdown 文件。
- `chapters/chapter-40.md`：完整部署流程教程。

## 部署方式一：推送到已有 GitHub 仓库

在 `outputs/ai-agent-book` 目录执行：

```bash
git remote add origin https://github.com/<your-user>/<your-repo>.git
git add .
git commit -m "Add AI Agent textbook"
git push -u origin main
```

然后在 GitHub 仓库设置中启用 Pages：

```text
Settings -> Pages -> Build and deployment -> GitHub Actions
```

推送后，`.github/workflows/pages.yml` 会自动部署。

## 部署方式二：创建新仓库

如果使用 GitHub CLI：

```bash
gh repo create <your-repo> --public --source=. --remote=origin --push
```

然后在仓库 Pages 设置中选择 GitHub Actions。

## 当前还需要的信息

要由 Codex 直接推送，需要你提供：

1. 目标 GitHub 仓库 URL，或允许使用 GitHub CLI 创建新仓库。
2. 当前环境已登录 GitHub，或你授权完成登录。
3. 当前仓库可用的 git user.name 和 git user.email。
