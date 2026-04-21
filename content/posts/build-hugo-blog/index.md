---
title: "Build Hugo Blog"
date: 2026-04-22T04:06:35+08:00
lastmod: 2026-04-22T04:06:35+08:00
draft: true
description: ""
summary: ""
tags: []
categories: []
---

## 为什么是 Hugo

这是我的第一篇博客，打算记录一下自己从零把这个站搭起来的过程。之所以选 Hugo，理由很简单：

- **够快**。Go 写的静态生成器，几百篇文章也就一两秒构建完
- **Markdown 写作**。纯文本源文件，将来哪天想搬家成本几乎为零
- **GitHub Pages 免费托管**。不用买服务器，不用备案，第一版直接能上线
- **内容永久可控**。源码和图片都在自己的 Git 仓库里，不依赖任何第三方图床、第三方托管

技术栈最终定为：

| 环节 | 选择 |
| --- | --- |
| 生成器 | Hugo Extended |
| 主题 | [PaperMod](https://github.com/adityatelange/hugo-PaperMod) |
| 托管 | GitHub Pages |
| 部署 | GitHub Actions |
| 评论 | Giscus（后续接入） |
| 写作 | VS Code + Markdown |

## 环境准备（macOS）

先确认 Homebrew 装好了。没装的话：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Apple Silicon 的 Mac 装完后需要手动把 brew 加到 PATH：

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

然后一把梭装好需要的工具：

```bash
brew install hugo git
brew install --cask visual-studio-code
```

Homebrew 装的 Hugo 默认就是 extended 版（很多主题依赖其中的 SCSS 功能），验证：

```bash
hugo version
# hugo v0.160.1+extended darwin/arm64 ...
```

看到 `extended` 就对了。

## 初始化站点

```bash
cd ~
hugo new site myblog
cd myblog
git init
```

目录结构：

```
myblog/
├── archetypes/    # 文章模板
├── content/       # 文章正文
├── layouts/       # 自定义模板（覆盖主题）
├── static/        # 静态资源
├── themes/        # 主题
└── hugo.toml      # 配置
```

## 引入 PaperMod 主题

用 git submodule 方式，方便以后同步上游更新：

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

然后把 `hugo.toml` 整个覆盖成：

```toml
baseURL = "https://example.github.io/"
languageCode = "zh-cn"
title = "我的博客"
theme = "PaperMod"
defaultContentLanguage = "zh-cn"
enableRobotsTXT = true
enableGitInfo = true

[outputs]
  home = ["HTML", "RSS", "JSON"]

[params]
  env = "production"
  DateFormat = "2006-01-02"
  ShowReadingTime = true
  ShowCodeCopyButtons = true
  ShowToc = true
  ShowBreadCrumbs = true
  ShowPostNavLinks = true
  ShowWordCount = true

[params.homeInfoParams]
  Title = "欢迎 👋"
  Content = "记录学习、考研、生活。"

[[menu.main]]
  name = "文章"
  url = "/posts/"
  weight = 1
[[menu.main]]
  name = "归档"
  url = "/archives/"
  weight = 2
[[menu.main]]
  name = "关于"
  url = "/about/"
  weight = 3
```

`env = "production"` 这行很关键，PaperMod 只在生产模式下才会完整输出 SEO 所需的 meta 标签。

## 本地预览

```bash
hugo new content posts/hello-world.md
```

把生成的 Markdown 里的 `draft: true` 改成 `false`，随便写几行正文保存，然后：

```bash
hugo server -D
```

浏览器打开 `http://localhost:1313` 看到首页和第一篇文章，本地就跑通了。

## 推到 GitHub

先在 GitHub 网页建一个仓库，名字按约定填 `你的用户名.github.io`，设为 Public，**不要**勾 Add README。

然后本地：

```bash
echo "public/" > .gitignore
echo "resources/" >> .gitignore
echo ".hugo_build.lock" >> .gitignore

git add .
git commit -m "init: hugo blog with PaperMod"
git branch -M main
git remote add origin git@github.com:你的用户名/你的用户名.github.io.git
git push -u origin main
```

推荐用 SSH key 认证（比 Personal Access Token 省心）：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
pbcopy < ~/.ssh/id_ed25519.pub
```

然后把公钥粘到 GitHub → Settings → SSH and GPG keys。

## 配 GitHub Actions

在仓库根目录建 `.github/workflows/hugo.yml`：

```yaml
name: Deploy Hugo to Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.160.1
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i hugo.deb
      - uses: actions/checkout@v6
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Build with Hugo
        env:
          HUGO_ENVIRONMENT: production
        run: hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

**两个容易踩坑的点**：

1. `HUGO_VERSION` 一定要跟本地 `hugo version` 看到的一致，否则一边能跑一边不能跑
2. 开启 GitHub Pages：仓库 Settings → Pages → Source 选 **GitHub Actions**（不是 Deploy from a branch）

推送之后去 Actions 标签页看 workflow 跑的情况。成功绿勾就说明上线了。

## 踩坑记录

理想很丰满，现实是我第一次部署就炸了。下面是真实遇到的问题。

### 坑一：Node.js 20 deprecation 警告

第一次 push 后 Actions 构建日志一堆黄色警告：

```
Node.js 20 actions are deprecated. The following actions are running on Node.js 20 and may not work as expected: actions/checkout@v4, actions/configure-pages@v5.
```

一开始以为是报错，其实**只是警告**，不影响构建——GitHub 在提前通知大家 2026 年 6 月起会强制切到 Node 24。

查了下，目前只有 `actions/checkout` 出了 v6 支持 Node 24，其他几个 Pages 系列 action 还没放新版。所以只把 checkout 升一下：

```yaml
- uses: actions/checkout@v6
```

其他 action 继续用旧版本，等官方更新。

### 坑二：`partial "google_analytics.html" not found`

本地 `hugo` 构建时突然开始报：

```
ERROR render of "page" failed: ... executing "partials/head.html" at
<partial "google_analytics.html" .>: error calling partial:
partial "google_analytics.html" not found
```

每一页都炸，站点根本构建不出来。

一开始以为是 PaperMod 的 bug，打算在 `layouts/partials/` 下建个空文件覆盖。但同步主题到最新版后，日志里多了一行关键信息：

```
WARN  Module "PaperMod" is not compatible with this Hugo version: Min 0.146.0
ERROR => hugo v0.146.0 or greater is required for hugo-PaperMod to build
```

真正原因找到了：**PaperMod 最新版要求 Hugo ≥ 0.146.0，但我本地和 Actions 里都是 0.134.0**，版本太老导致新主题里某些 partial 的引用没法正确解析，连锁反应表现为「partial 找不到」。

解决办法两步走：

本地升级：

```bash
brew upgrade hugo
hugo version
# hugo v0.160.1+extended darwin/arm64 ...
```

Actions 里也改：

```yaml
env:
  HUGO_VERSION: 0.160.1
```

再 push，绿勾。

### 坑三：在 GitHub 网页改了文件，本地没同步

Actions 工作流我是直接在 GitHub 网页新建的，后来想本地改一下配置，发现本地根本没这个文件。

解决也简单：

```bash
git pull
```

这里学到一个习惯：**每次开工前先 pull，收工前再 push**。尤其是多端写作、或者会在 GitHub 网页上直接改文件的情况，不 pull 直接改会产生冲突。

## 写作流程

搭完后日常写一篇文章就三步：

```bash
# 1. 新建文章（用 page bundle 方式，图片跟文章放一起）
hugo new content posts/article-name/index.md

# 2. 本地预览
hugo server -D

# 3. 发布
git add .
git commit -m "post: 文章标题"
git push
```

push 之后约 1-2 分钟 Actions 跑完，线上就更新了。

`archetypes/default.md` 我改成了这样，每次 `hugo new` 自动带上 SEO 所需字段：

```yaml
---
title: "{{ replace .Name "-" " " | title }}"
date: {{ .Date }}
lastmod: {{ .Date }}
draft: true
description: ""
summary: ""
tags: []
categories: []
---
```

每篇认真填 `description`——它会变成搜索结果里的摘要，是 SEO 最关键的一个字段。

## 小结

整个搭建流程看似步骤多，其实本质只有三件事：

1. 本地把 Hugo + 主题跑通（能看到首页）
2. 推到 GitHub，配 Actions 自动部署
3. 打开 Pages 开关

真正花时间的不是搭建，是**排查版本不匹配**这种隐性问题。总结下来几条经验：

- **本地和 CI 的 Hugo 版本保持一致**，尤其是升级主题后
- **看报错要抓关键信息**，`google_analytics.html not found` 是表象，真正的线索是日志里的 `requires hugo v0.146.0 or greater`
- **Git 同步是基本功**，pull 在前 push 在后，不吃版本冲突的亏

下一步的计划：

- 买个自己的域名绑定上去（`*.github.io` 这种子域名不够「永久」）
- 提交 Google Search Console 做搜索引擎收录
- 接入 Giscus 评论
- 写第二篇文章 🚀

那就先这样。