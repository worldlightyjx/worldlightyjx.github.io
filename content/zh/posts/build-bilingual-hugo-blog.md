+++
title = '从零搭建中英文 Hugo 技术博客'
date = 2026-10-08
draft = false
description = '用 Hugo、PaperMod 和 GitHub Actions 搭建默认中文、英文独立管理的技术博客。'
tags = ['Hugo', 'PaperMod', 'GitHub Pages', '多语言']
+++

这篇文章记录一个可复用的搭建过程：用 Hugo 生成静态页面，用 PaperMod 提供博客界面，将中文和英文内容分开放置，再由 GitHub Actions 自动部署到 GitHub Pages。配置示例使用通用占位符，实际使用时替换成自己的信息即可。

## 1. 创建 Hugo 站点并安装 PaperMod

先按照 [Hugo 官方安装说明](https://gohugo.io/installation/)安装 Hugo，并确认 `hugo version` 能正常运行。接着创建项目，将 PaperMod 作为 Git 子模块加入：

~~~bash
hugo new site bilingual-blog
cd bilingual-blog
git init -b main
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
~~~

子模块记录的是主题版本。以后在其他电脑或 GitHub Actions 中检出项目时，需要递归检出子模块。Hugo 的[快速入门](https://gohugo.io/getting-started/quick-start/)也使用这种方式安装主题。

## 2. 配置中文默认、英文独立

修改项目根目录的 `hugo.toml`，核心配置如下。`YOUR_USERNAME` 是占位符；如果使用 GitHub 用户站点仓库，仓库名通常是 `YOUR_USERNAME.github.io`。

~~~toml
baseURL = 'https://YOUR_USERNAME.github.io/'
theme = 'PaperMod'
defaultContentLanguage = 'zh'
defaultContentLanguageInSubdir = false

[params]
  mainSections = ['posts']

[languages.zh]
  contentDir = 'content/zh'
  label = '中文'
  locale = 'zh-CN'
  title = '我的技术博客'
  weight = 1
  hasCJKLanguage = true

[languages.en]
  contentDir = 'content/en'
  label = 'English'
  locale = 'en-US'
  title = 'My Tech Blog'
  weight = 2
~~~

这样，中文首页位于站点根路径 `/`，英文首页位于 `/en/`。两种语言的文章分别写在 `content/zh/posts/` 和 `content/en/posts/`，不必为每篇文章提供译文。在没有手动设置翻译关联键时，两边相同的相对路径和文件名会让 Hugo 自动关联语言版本；本文的中英文版本就是一个有意关联的例子。详情见 [Hugo 多语言文档](https://gohugo.io/content-management/multilingual/)。

内容目录可以从下面这个结构开始：

~~~text
content/
├── zh/
│   ├── _index.md
│   ├── about.md
│   ├── archives.md
│   ├── tags/_index.md
│   └── posts/
│       ├── _index.md
│       └── build-bilingual-hugo-blog.md
└── en/
    ├── _index.md
    ├── about.md
    ├── archives.md
    ├── tags/_index.md
    └── posts/
        ├── _index.md
        └── build-bilingual-hugo-blog.md
~~~

`_index.md` 设置首页或栏目页的标题；`about.md` 保存自我介绍。导航菜单可以在 `hugo.toml` 的 `languages.zh.menus.main` 和 `languages.en.menus.main` 中分别配置，链接到首页、文章、标签、归档和关于页。

## 3. 写文章、标签和归档

可以直接在对应目录新建 Markdown 文件，也可以运行：

~~~bash
hugo new content content/zh/posts/first-zh-post.md
hugo new content content/en/posts/first-en-post.md
~~~

这里使用不同文件名，表示两篇独立文章。Hugo 会根据 `archetypes/default.md` 生成标题、日期和草稿状态。

Hugo 的[文章元数据](https://gohugo.io/content-management/front-matter/)写在 Markdown 文件顶部。例如：

~~~toml
+++
title = '我的第一篇文章'
date = 2026-10-08
draft = true
tags = ['Hugo', 'GitHub Pages']
+++
~~~

正文接在第二个 `+++` 后面。`tags` 是 Hugo 默认支持的分类法，不用提前建标签清单；Hugo 会生成标签列表和每个标签对应的文章页。写作时保持标签拼写一致，读者就能从标签页找到同主题文章。[Hugo 标签配置说明](https://gohugo.io/configuration/taxonomies/)

归档由 PaperMod 自动生成。在两种语言的 `archives.md` 中分别设置 `layout = 'archives'`，并在站点配置中将 `mainSections` 设为 `['posts']`。这样，已发布的文章会按日期进入各自语言的归档；标签不决定归档位置。PaperMod 的[功能说明](https://github.com/adityatelange/hugo-PaperMod/wiki/Features)介绍了归档布局。

## 4. 本地预览与发布

写作时保留 `draft = true`，运行 `hugo server -D` 预览草稿。确认内容后改为 `draft = false`，再运行 `hugo build --gc --minify` 检查正式构建。Hugo 默认不会发布草稿或未来日期的文章。[Hugo 基本用法](https://gohugo.io/getting-started/usage/)

在 GitHub 创建站点仓库后，到 **Settings → Pages** 将来源设为 **GitHub Actions**。项目中的 `.github/workflows/hugo.yaml` 应完成四件事：递归检出含 PaperMod 的仓库、安装 Hugo、构建并上传 `public/`、使用 Pages 部署。按 [Hugo 官方 GitHub Pages 教程](https://gohugo.io/host-and-deploy/deploy-to-github-pages/)建立工作流，再提交并推送到 `main`：

~~~bash
git add .
git commit -m "Create bilingual Hugo blog"
git remote add origin https://github.com/YOUR_USERNAME/YOUR_USERNAME.github.io.git
git push -u origin main
~~~

以后发布新文章，只需将文章设为非草稿、提交并推送。Actions 会重新构建和部署；`public/` 是生成结果，不需要手工编辑。如果启用 PaperMod 的头像首页模式，首页会显示个人简介而不是文章列表，此时保留“文章 / Posts”导航，读者就能找到最新文章。

至此，中文、英文、标签、归档和自动部署形成了一条完整的写作流程。下一篇文章可以只写中文，也可以只写英文；两种语言各自独立发布。
