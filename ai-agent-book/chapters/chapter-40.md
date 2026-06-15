# 第 40 章：部署、文档站、GitHub Pages 与发布流程

前面我们已经把书写成了一个多章节 Markdown 项目。每章是独立文件，首页有目录，进度文件记录状态，还准备了 GitHub Pages 的配置和 workflow。

这一章要讲最后一段工程：如何把一个本地文档项目发布到 GitHub，并通过 GitHub Pages 变成可访问的网站。

这章不只适用于本书，也适用于你以后发布自己的 Agent 教程、项目文档、API 文档、课程笔记和开源手册。

你会学到：

1. 一个 Markdown 文档站需要哪些文件。
2. GitHub Pages 的基本工作方式。
3. Jekyll 在这里扮演什么角色。
4. GitHub Actions workflow 如何构建并部署 Pages。
5. 如何初始化 git 仓库、提交、添加远程仓库、推送。
6. 如何启用 GitHub Pages。
7. 如何排查部署失败。
8. 如果缺少远程仓库 URL 或 GitHub 授权，如何安全交接。
9. 如何为后续维护建立发布流程。

## 40.1 为什么最后要做部署

一本书如果只躺在本地目录里，就还只是草稿。部署以后，它才变成可以分享、阅读、维护和迭代的文档产品。

部署带来几个好处：

1. 读者可以通过浏览器阅读。
2. 每章有稳定链接。
3. Git 历史记录每次修改。
4. GitHub Issues 可以收反馈。
5. Pull Request 可以协作修订。
6. GitHub Actions 可以自动检查和发布。
7. 后续可以绑定域名。

对 Agent 教材来说，部署还有一个额外价值：它本身就是一次工程实践。你不只是写“如何搭建 Agent”，还把书稿整理成一个可以发布的工程项目。

这符合整本书的主线：AI Agent 不只是 prompt，而是工程系统。文档也一样，不只是文本，而是可以构建、部署、版本化、协作的项目。

## 40.2 当前书稿项目结构

本书现在的输出目录是：

```text
outputs/ai-agent-book/
  README.md
  index.md
  _config.yml
  DEPLOY.md
  progress.md
  00-writing-plan.md
  01-manuscript.md
  .gitignore
  .github/
    workflows/
      pages.yml
  chapters/
    00-preface.md
    chapter-01.md
    chapter-02.md
    ...
    chapter-40.md
```

每个文件的职责：

1. `README.md`：GitHub 仓库首页。别人打开仓库首先看到它。
2. `index.md`：GitHub Pages 站点首页。浏览器访问文档站时首先看到它。
3. `_config.yml`：Jekyll 配置。设置站点标题、描述、主题、Markdown 引擎。
4. `.github/workflows/pages.yml`：GitHub Actions 部署流程。
5. `chapters/`：正式章节。
6. `progress.md`：写作进度。
7. `DEPLOY.md`：部署操作说明。
8. `00-writing-plan.md`：写作计划。
9. `01-manuscript.md`：历史合并稿，后续以 chapters 为主。

这种结构很适合 GitHub Pages，因为它保持了 Markdown 原生可读性。即使不构建网站，读者也可以在 GitHub 里直接打开每一章。

## 40.3 README 和 index 的区别

新手常常分不清 `README.md` 和 `index.md`。

`README.md` 面向 GitHub 仓库页面。它解释这个仓库是什么、怎么阅读、怎么部署、怎么贡献。

`index.md` 面向网站首页。它是 GitHub Pages 生成站点时的首页。

为什么两个文件内容看起来很像？

因为本书是文档站，仓库首页和网站首页都需要目录。但它们的上下文不同。

`README.md` 可以包含更多仓库维护信息，比如：

1. 项目说明。
2. 本地预览方法。
3. 贡献方式。
4. 许可证。
5. 部署说明链接。

`index.md` 更像读者入口，应该简洁，突出阅读目录。

当前我们让两者都包含章节目录，是为了保证无论用户在 GitHub 仓库里看，还是在 Pages 网站里看，都能直接进入章节。

## 40.4 `_config.yml` 的作用

当前 `_config.yml` 内容很简洁：

```yml
title: 从零搭建 AI Agent
description: 从 Claude Code 源码到工程实战
theme: minima
markdown: kramdown
```

这些字段含义：

1. `title`：站点标题。
2. `description`：站点描述。
3. `theme`：Jekyll 主题。
4. `markdown`：Markdown 渲染引擎。

`minima` 是 GitHub Pages 常用的简洁主题。它不是最漂亮的主题，但足够稳定，适合教材类项目。

为什么不一开始就做复杂主题？

因为文档项目最重要的是内容可读、链接稳定、部署可靠。复杂主题会带来 CSS、布局、导航、搜索、移动端适配等额外问题。等内容稳定后再美化更稳。

如果以后要增强，可以考虑：

1. 添加导航栏。
2. 添加上一章/下一章链接。
3. 添加站内搜索。
4. 添加自定义 CSS。
5. 添加目录侧栏。
6. 添加代码高亮配置。

但第一版发布不需要这些。

## 40.5 GitHub Actions workflow 怎么工作

当前 workflow 文件是 `.github/workflows/pages.yml`。

它的触发条件：

```yml
on:
  push:
    branches: ["main"]
  workflow_dispatch:
```

意思是：

1. 每次推送到 `main` 分支，自动部署。
2. 也可以在 GitHub Actions 页面手动触发。

权限：

```yml
permissions:
  contents: read
  pages: write
  id-token: write
```

这些权限允许 workflow 读取仓库内容、写入 Pages，并使用 OIDC token 部署。

构建步骤：

```yml
- name: Checkout
  uses: actions/checkout@v4

- name: Setup Pages
  uses: actions/configure-pages@v5

- name: Build with Jekyll
  uses: actions/jekyll-build-pages@v1
  with:
    source: .
    destination: ./_site

- name: Upload artifact
  uses: actions/upload-pages-artifact@v3
```

部署步骤：

```yml
- name: Deploy to GitHub Pages
  id: deployment
  uses: actions/deploy-pages@v4
```

整体流程：

```text
push main
  -> GitHub Actions 启动
  -> checkout 仓库
  -> 配置 Pages
  -> 用 Jekyll 构建 Markdown
  -> 上传 _site artifact
  -> 部署到 GitHub Pages
```

你不需要自己手动生成 `_site`，workflow 会在 GitHub 的 runner 上完成。

## 40.6 本地检查部署文件

在推送前，先检查几个点。

第一，目录是否存在：

```bash
ls
ls chapters
ls .github/workflows
```

第二，首页是否包含最新章节：

```bash
grep "第 40 章" README.md
grep "第 40 章" index.md
```

第三，workflow 是否存在：

```bash
cat .github/workflows/pages.yml
```

第四，Jekyll 配置是否存在：

```bash
cat _config.yml
```

第五，Git 状态：

```bash
git status --short
```

如果看到很多 `??`，说明文件还没有 `git add`。这不是错误，只是还没提交。

## 40.7 初始化 git 仓库

如果还没有 git 仓库，执行：

```bash
git init
git branch -M main
```

本项目已经在 `outputs/ai-agent-book` 初始化过本地 git 仓库，分支是 `main`。

你可以检查：

```bash
git branch --show-current
```

应该输出：

```text
main
```

如果不是 main，可以执行：

```bash
git branch -M main
```

为什么用 main？

因为 GitHub Pages workflow 当前配置监听 `main` 分支：

```yml
branches: ["main"]
```

如果你推送到其他分支，workflow 不会自动触发，除非你修改 workflow。

## 40.8 配置 git 用户信息

提交前需要 git user name 和 email。

检查：

```bash
git config user.name
git config user.email
```

如果为空，可以在当前仓库设置：

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

注意：这只设置当前仓库，不影响全局配置。

也可以设置全局：

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

在协作或开源项目中，email 最好使用 GitHub 账户绑定的邮箱，或者 GitHub 提供的 noreply 邮箱。

如果没有配置，`git commit` 通常会失败，并提示：

```text
Author identity unknown
```

这是正常的安全保护，不是文档项目问题。

## 40.9 提交文件

确认文件后：

```bash
git add .
git commit -m "Add AI Agent textbook"
```

如果后续追加章节，可以用：

```bash
git add chapters README.md index.md progress.md
git commit -m "Add chapter 40 deployment guide"
```

提交信息建议清楚描述变化。

好的提交信息：

```text
Add chapter 40 deployment guide
Update table of contents for chapter 40
Configure GitHub Pages workflow
```

不好的提交信息：

```text
update
fix
aaa
final
```

提交信息不是给机器看的，是给未来的自己和协作者看的。

## 40.10 添加远程仓库

如果你已经有 GitHub 仓库，比如：

```text
https://github.com/your-user/ai-agent-book.git
```

添加远程：

```bash
git remote add origin https://github.com/your-user/ai-agent-book.git
```

检查：

```bash
git remote -v
```

应该看到：

```text
origin  https://github.com/your-user/ai-agent-book.git (fetch)
origin  https://github.com/your-user/ai-agent-book.git (push)
```

如果已经有 origin，要修改：

```bash
git remote set-url origin https://github.com/your-user/ai-agent-book.git
```

推送：

```bash
git push -u origin main
```

这里的 `-u` 会设置 upstream，以后可以直接：

```bash
git push
```

## 40.11 创建新 GitHub 仓库

有两种方式。

第一种，在 GitHub 网页创建。

步骤：

1. 打开 GitHub。
2. 点击 New repository。
3. 输入仓库名，比如 `ai-agent-book`。
4. 选择 Public 或 Private。
5. 不要勾选初始化 README，因为本地已经有 README。
6. 创建仓库。
7. 复制远程 URL。
8. 本地执行 remote add 和 push。

第二种，使用 GitHub CLI。

```bash
gh repo create ai-agent-book --public --source=. --remote=origin --push
```

这个命令会：

1. 创建 GitHub 仓库。
2. 设置 origin。
3. 推送当前代码。

但前提是：

1. 本机安装了 `gh`。
2. 已经 `gh auth login`。
3. 当前环境允许访问网络。

在受限环境里，GitHub CLI 可能无法登录或推送。这时应该把本地项目准备好，然后由用户在自己的 GitHub 环境中执行推送。

## 40.12 启用 GitHub Pages

代码推送后，进入 GitHub 仓库页面：

```text
Settings -> Pages
```

在 Build and deployment 里选择：

```text
Source: GitHub Actions
```

然后回到 Actions 页面，查看 `Deploy GitHub Pages` workflow。

如果 workflow 成功，会显示绿色勾。

Pages 地址通常类似：

```text
https://your-user.github.io/ai-agent-book/
```

如果是组织仓库：

```text
https://your-org.github.io/ai-agent-book/
```

如果仓库名是特殊的：

```text
your-user.github.io
```

那么站点可能是：

```text
https://your-user.github.io/
```

## 40.13 第一次部署可能遇到的延迟

GitHub Pages 第一次启用后，访问地址可能不会立刻可用。

常见表现：

1. 404。
2. Actions 已成功但页面还没刷新。
3. 浏览器缓存旧页面。
4. Pages 设置刚保存，还在生效。

一般等待 1 到 5 分钟，再刷新。

如果仍然 404，检查：

1. Pages 是否选择 GitHub Actions。
2. workflow 是否成功。
3. 仓库是否推送到 main。
4. `index.md` 是否在仓库根目录。
5. `_site` artifact 是否上传成功。
6. 仓库是否 private 且 Pages 权限受限。

## 40.14 部署失败排查：Jekyll build error

如果 Actions 中 `Build with Jekyll` 失败，常见原因：

1. Markdown frontmatter 格式错误。
2. `_config.yml` 语法错误。
3. 文件名或路径包含 Jekyll 不喜欢的字符。
4. 代码块中出现未闭合反引号。
5. Liquid 模板语法冲突。

特别注意：Jekyll 会处理双花括号语法。如果 Markdown 里有类似：

```text
{{ something }}
```

可能被当成 Liquid 模板。解决方法是用 raw 包裹，或改写示例。

如果只是普通教材，尽量避免无必要的 Liquid 语法。

排查方式：

1. 打开 Actions 失败日志。
2. 找到第一个 error。
3. 看提示文件和行号。
4. 修复 Markdown。
5. 重新 push。

## 40.15 部署失败排查：权限问题

如果 deploy 步骤失败，可能是权限设置问题。

检查 workflow 是否有：

```yml
permissions:
  contents: read
  pages: write
  id-token: write
```

检查仓库 Settings：

```text
Settings -> Actions -> General -> Workflow permissions
```

一般需要允许 GitHub Actions 读取和写入 Pages 所需权限。

如果组织有更严格策略，可能需要组织管理员允许 Pages 部署。

## 40.16 部署失败排查：分支不匹配

workflow 监听：

```yml
branches: ["main"]
```

如果你推送到 `master`、`docs`、`develop`，不会触发。

解决方式一：推送到 main。

```bash
git branch -M main
git push -u origin main
```

解决方式二：修改 workflow：

```yml
on:
  push:
    branches: ["master"]
```

建议统一用 main，减少混乱。

## 40.17 部署失败排查：路径和链接问题

Markdown 链接要使用相对路径。

正确：

```md
[第 1 章](chapters/chapter-01.md)
```

错误：

```md
[第 1 章](/Users/name/project/chapters/chapter-01.md)
```

本地绝对路径在 GitHub Pages 上不可用。

在最终发布的 Markdown 文件里，章节链接应该是相对路径。当前 `README.md` 和 `index.md` 使用的是：

```md
chapters/chapter-XX.md
```

这是正确的。

注意：我在聊天回复里给你的本地文件链接会使用绝对路径，因为 Codex 桌面环境需要这样才能打开本机文件。但书稿内部链接必须保持相对路径。

## 40.18 部署失败排查：中文路径和编码

本书文件名使用英文路径，比如：

```text
chapter-40.md
```

章节标题是中文，但文件名是 ASCII。这是比较稳妥的方式。

为什么？

1. URL 更稳定。
2. GitHub Pages 处理更少问题。
3. 跨平台命令更方便。
4. 自动脚本更容易写。

内容本身可以是中文，Markdown 文件用 UTF-8 保存即可。

如果你用中文文件名，也不是不行，但要注意 URL 编码、链接复制和某些脚本兼容。

## 40.19 给文档站加上一章和下一章链接

当前目录已经能阅读，但每章之间没有上一章/下一章链接。如果要改善阅读体验，可以在每章末尾添加：

```md
---

上一章：[第 39 章：从 mini-agent 到生产级 Agent 平台的完整路线图](chapter-39.md)

下一章：[附录 A：术语表](appendix-a.md)
```

这个工作可以自动化。

思路：

1. 读取 chapters 目录。
2. 按文件名排序。
3. 给每个文件末尾插入导航。
4. 避免重复插入。

但自动修改 40 多个文件有风险，第一版可以先不做。保持内容稳定优先。

## 40.20 给文档站加搜索

GitHub Pages 默认不带搜索。

可选方案：

1. 使用 Jekyll 插件。
2. 使用静态 JSON 索引和前端搜索库。
3. 使用浏览器自带页面搜索。
4. 使用 GitHub 仓库搜索。

第一版不建议立刻加搜索。原因：

1. GitHub Pages 对插件有限制。
2. 搜索索引会增加构建复杂度。
3. 当前目录已经足够清晰。

如果以后要加，推荐生成一个简单 `search.json`，包含标题、路径、摘要，再用前端脚本搜索。

## 40.21 给文档站加版本

当书继续迭代时，可以考虑版本。

简单方式：

1. 使用 git tag。
2. 在 README 写当前版本。
3. 每次大更新打 tag。

命令：

```bash
git tag v1.0.0
git push origin v1.0.0
```

更复杂方式：

1. `docs/v1/`
2. `docs/v2/`
3. 首页选择版本。

对于教材项目，第一版用 git tag 就够了。

## 40.22 发布前检查清单

发布前执行这张清单。

内容检查：

1. README 目录是否完整。
2. index 目录是否完整。
3. progress 是否更新。
4. 每章标题是否正确。
5. 是否有明显重复章节。
6. 是否有未闭合代码块。
7. 是否有本地绝对路径写进书稿链接。

工程检查：

1. `_config.yml` 存在。
2. Pages workflow 存在。
3. 分支是 main。
4. git status 清楚。
5. commit 信息清楚。
6. remote origin 正确。
7. GitHub Pages 选择 GitHub Actions。

安全检查：

1. 没有 API key。
2. 没有私有 token。
3. 没有 `.env`。
4. 没有用户敏感路径。
5. 没有不该发布的临时文件。

## 40.23 当前项目的部署状态

当前本地已经具备：

1. 章节文件。
2. README。
3. index。
4. Jekyll 配置。
5. GitHub Pages workflow。
6. 部署说明。
7. 本地 git 仓库。

当前还缺：

1. 目标 GitHub 仓库 URL。
2. 可用的 GitHub 认证。
3. git 提交身份配置。
4. 网络推送权限。

这意味着：本地项目已经可以交付，但还不能由我直接完成远程 push，除非你提供仓库 URL 并允许使用当前环境的 GitHub 认证，或者你自己在本机执行 DEPLOY.md 中的命令。

这是一个重要边界。部署不是写文件那么简单，它涉及你的 GitHub 账户和远程权限。没有授权时，Agent 不应该猜测仓库地址，也不应该尝试把内容推到不确定的位置。

## 40.24 如果由用户手动部署

你可以在本机进入目录：

```bash
cd /Users/yanfangfeng/Documents/Codex/2026-06-15/yasasbanukaofficial-claude-code-https-github-com/outputs/ai-agent-book
```

设置身份：

```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

提交：

```bash
git add .
git commit -m "Add AI Agent textbook"
```

添加远程：

```bash
git remote add origin https://github.com/<your-user>/<your-repo>.git
```

推送：

```bash
git push -u origin main
```

然后去 GitHub：

```text
Settings -> Pages -> Build and deployment -> GitHub Actions
```

等待 Actions 成功。

## 40.25 如果授权 Codex 继续部署

如果你希望我继续执行部署，需要提供其中一种方式：

方式一：提供已有仓库 URL。

例如：

```text
https://github.com/your-user/ai-agent-book.git
```

并确认当前环境可以推送。

方式二：允许使用 GitHub CLI 创建仓库。

需要当前环境已登录：

```bash
gh auth status
```

如果未登录，需要你完成登录。

方式三：你先创建空仓库，然后把 URL 发给我。

这是最稳的方式，因为仓库所有权和可见性由你控制。

我拿到远程 URL 后，可以执行：

```bash
git remote add origin <repo-url>
git add .
git commit -m "Add AI Agent textbook"
git push -u origin main
```

如果 git identity 未配置，需要先设置当前仓库的 user.name 和 user.email。

## 40.26 发布后维护流程

发布不是结束，而是开始。

后续维护建议：

1. 每新增章节，更新 README、index、progress。
2. 每次发布前检查链接。
3. 每次大改打 tag。
4. 收集读者 issue。
5. 修正错别字和代码示例。
6. 增加附录和术语表。
7. 给章节添加导航。
8. 定期更新源码分析对应版本。

一个典型更新流程：

```bash
git pull
# 编辑章节
git status --short
git add chapters/chapter-41.md README.md index.md progress.md
git commit -m "Add appendix glossary"
git push
```

GitHub Actions 会自动重新部署。

## 40.27 如何处理源码版本漂移

本书基于 `yasasbanukaofficial/claude-code` 仓库分析。但开源仓库可能继续变化。

为了避免读者混淆，建议在 README 中记录：

1. 分析基于哪个仓库。
2. 分析日期。
3. 如有 commit hash，记录 commit hash。
4. 哪些章节是源码映射，哪些章节是通用工程教学。

如果以后更新源码分析，可以新增：

```text
源码基准版本：<commit hash>
分析日期：2026-06-15
```

这样读者知道书中描述和当前 upstream 可能有差异。

## 40.28 文档许可证

开源文档最好明确许可证。

常见选择：

1. CC BY 4.0：适合教材内容，允许传播和改编，但要求署名。
2. CC BY-SA 4.0：要求改编作品使用相同协议。
3. MIT：更常用于代码，也可用于文档，但对教材表达不如 CC 明确。
4. All rights reserved：不开放授权。

如果你希望别人学习、引用、改编，推荐 CC BY 4.0 或 CC BY-SA 4.0。

可以添加 `LICENSE` 文件，并在 README 说明。

本次我没有擅自添加许可证，因为许可证是作者权利选择，需要你确认。

## 40.29 文档站的长期演进

第一版发布后，可以继续优化：

1. 添加附录 A：术语表。
2. 添加附录 B：代码清单。
3. 添加附录 C：常见错误排查。
4. 添加附录 D：Claude Code 源码文件索引。
5. 添加附录 E：mini-agent 完整实现。
6. 添加每章练习答案。
7. 添加项目模板。
8. 添加读者路线图。
9. 添加讲师版课件。
10. 添加 PDF 或 EPUB 版本。

如果要生成 PDF，可以后续使用 Pandoc 或专门的文档工具。但 PDF 排版会带来新的工作：分页、目录、代码换行、字体、中文断行、封面。

第一阶段先让 GitHub Pages 可读，是性价比最高的发布方式。

## 40.30 本章小结

这一章我们完成了发布与部署流程的工程化说明。

你需要记住：

1. `README.md` 是仓库首页。
2. `index.md` 是 Pages 首页。
3. `_config.yml` 配置 Jekyll。
4. `.github/workflows/pages.yml` 负责自动部署。
5. 推送到 main 后，GitHub Actions 会构建并部署 Pages。
6. Pages 设置里要选择 GitHub Actions。
7. 部署失败时先看 Actions 日志。
8. 链接要用相对路径，不要把本地绝对路径写进书稿。
9. 没有远程仓库 URL 和 GitHub 授权时，不能直接推送。
10. 发布后要维护版本、链接、章节目录和读者反馈。

到这里，整本书已经从“分析源码”走到了“形成可发布文档站”。这也是 Agent 工程学习的一个完整闭环：理解系统、拆解系统、重建系统、评估系统、发布系统。

后续如果继续扩展，可以进入附录部分：术语表、源码索引、mini-agent 完整代码清单、常见问题、练习答案和项目模板。
