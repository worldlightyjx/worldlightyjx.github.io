+++
title = 'Build a Bilingual Hugo Blog from Scratch'
date = 2026-10-08
draft = false
description = 'Build a Chinese-first technical blog with independent English content, PaperMod, and GitHub Pages deployment.'
tags = ['Hugo', 'PaperMod', 'GitHub Pages', 'Multilingual']
+++

This post documents a reusable setup: Hugo generates the static site, PaperMod provides the blog layout, Chinese and English content live in separate directories, and GitHub Actions deploys the result to GitHub Pages. All account names in the examples are placeholders.

## 1. Create the site and add PaperMod

Install Hugo using its [official installation guide](https://gohugo.io/installation/) and check that `hugo version` works. Then create a project and add PaperMod as a Git submodule:

~~~bash
hugo new site bilingual-blog
cd bilingual-blog
git init -b main
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
~~~

The submodule records the theme version. Other machines and GitHub Actions must check out submodules recursively. Hugo's [quick start](https://gohugo.io/getting-started/quick-start/) uses the same approach for themes.

## 2. Make Chinese the default language

Edit the root `hugo.toml` with the core settings below. Replace `YOUR_USERNAME` with your own GitHub username. A GitHub user-site repository is normally named `YOUR_USERNAME.github.io`.

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

The Chinese home page is served at `/` and the English one at `/en/`. Write posts under `content/zh/posts/` or `content/en/posts/`. A post does not need a counterpart in the other language. Without an explicit translation key, matching relative paths and file names make Hugo link two pages as translations. This post deliberately has a matching Chinese counterpart; future posts can use different names when they are independent. See [Hugo's multilingual documentation](https://gohugo.io/content-management/multilingual/).

A useful starting structure is:

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

An `_index.md` file sets the title of a home or section page, while `about.md` contains an introduction. Configure navigation separately in `languages.zh.menus.main` and `languages.en.menus.main`, linking to Home, Posts, Tags, Archives, and About.

## 3. Write posts with tags and an archive

Create Markdown files in the appropriate content directory, or use Hugo's command:

~~~bash
hugo new content content/zh/posts/first-zh-post.md
hugo new content content/en/posts/first-en-post.md
~~~

The different file names indicate independent posts. Hugo uses `archetypes/default.md` to generate an initial title, date, and draft status.

Each Markdown post starts with [front matter](https://gohugo.io/content-management/front-matter/):

~~~toml
+++
title = 'My First Post'
date = 2026-10-08
draft = true
tags = ['Hugo', 'GitHub Pages']
+++
~~~

The body follows the second `+++`. Tags are a default Hugo taxonomy: you do not have to register each tag. Hugo builds a tag index and a page for each tag, so consistent spelling helps readers find related posts. See the [taxonomy configuration guide](https://gohugo.io/configuration/taxonomies/).

PaperMod generates the archive automatically. Create an `archives.md` file in each language with `layout = 'archives'`, and set `mainSections = ['posts']` in the site configuration. Published posts then appear in their own language's archive, grouped by date. Tags do not control the archive. PaperMod documents its [archive layout here](https://github.com/adityatelange/hugo-PaperMod/wiki/Features).

## 4. Preview and deploy

Keep `draft = true` while writing and preview with `hugo server -D`. When ready, change it to `draft = false` and check a production build with `hugo build --gc --minify`. Hugo excludes drafts and future-dated posts from normal builds. See [Hugo's basic usage guide](https://gohugo.io/getting-started/usage/).

Create an empty GitHub user-site repository, then select **GitHub Actions** under **Settings → Pages**. The `.github/workflows/hugo.yaml` workflow should check out submodules, install Hugo, build and upload `public/`, and deploy through GitHub Pages. Follow [Hugo's official GitHub Pages guide](https://gohugo.io/host-and-deploy/deploy-to-github-pages/) for the full workflow. Commit and push the project to `main`:

~~~bash
git add .
git commit -m "Create bilingual Hugo blog"
git remote add origin https://github.com/YOUR_USERNAME/YOUR_USERNAME.github.io.git
git push -u origin main
~~~

For each later post, mark it as published, commit, and push. GitHub Actions rebuilds the site; `public/` is generated output and does not need manual editing. If you enable PaperMod's avatar-style profile home page, keep a Posts menu item because the home page shows the profile instead of the post list.

The result is a writing workflow with separate Chinese and English content, tags, automatic archives, and automatic deployment. Future posts may be published in either language without creating a translation.
