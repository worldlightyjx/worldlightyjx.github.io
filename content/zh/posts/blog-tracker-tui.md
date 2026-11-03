+++
title = '一个中午，我用 Codex 做了个追踪博客的工具'
date = 2026-11-03
draft = false
description = '从收藏博客的痛点出发，介绍 Blog Tracker TUI 的实现思路、安装、使用和配置。'
tags = ['Codex', 'Rust', 'TUI', '博客']
+++

我偶尔喜欢在网上逛个人博客。读到写得好的文章，总想把这个博客留下来，过阵子再看看作者有没有更新。以前试过好几款 RSS 订阅工具，但用起来总觉得不够顺手；更麻烦的是，有些喜欢的博客根本没有提供 RSS。把网址塞进 Chrome 书签也不是办法——收藏的那一刻很认真，之后却很少主动点开。

于是我想做一个足够简单的工具：给它一个博客地址，它替我记住这个博客和文章链接，顺手看看有没有新内容。程序员一天里大半时间都在终端里，偶尔想摸个鱼，敲一条命令就能扫一眼关注的博客更新，岂不正好？这个念头冒出来后，我和 Codex 用一个中午把它做成了 [Blog Tracker TUI](https://github.com/worldlightyjx/blog-tracker-tui)。

## 它能做什么

Blog Tracker TUI 是一个运行在终端里的本地博客追踪器。添加博客主页或 Feed 地址后，它会检查新文章，显示未读数量，并记住哪些文章已经看过。它支持 RSS、Atom、JSON Feed，也会尝试从没有 Feed 的公开网页里提取文章链接。

我把它设计成“发现更新的入口”，没有在终端里重做一套阅读器。界面只保存和展示文章标题、链接等信息；选中文章按 `Enter`，系统默认浏览器会打开原文，成功打开后标为已读。这样读文章仍在作者的网站上，终端里只需负责追踪。

## 实现思路与架构

代码是 Rust 写的，核心流程可以概括为：

```text
博客主页或 Feed URL
    → 抓取与识别（Feed 优先，HTML 兜底）
    → 提取文章标题、URL、发布时间
    → SQLite 记录订阅、文章和已读状态
    → TUI 展示博客列表与未读文章
```

抓取层用 `reqwest` 请求页面，`feed-rs` 解析 Feed，`scraper` 处理普通 HTML。输入如果本身是 Feed，就直接解析；如果是博客主页，程序先寻找页面里的 Feed 链接；找不到时，再从文章列表的 HTML 结构中识别链接。网页识别毕竟无法覆盖所有站点，所以也可以给某个博客指定 CSS 选择器和分页规则。

数据层使用本地 SQLite。每篇文章以所在博客和文章 URL 识别，刷新时只把新 URL 加进来；标题变化不会把旧文章重新标成未读。首次成功检查时，最多只把最近三篇文章标为未读，其余作为已有历史保存，避免刚添加一个博客就看到几十条“新消息”。程序启动后先展示本地记录，检查工作在线程中运行；运行期间默认每 30 分钟检查一次，也可以随时手动刷新。检查失败时，已经保存的文章不会丢失，界面会显示错误和上次成功检查的时间。

界面层由 `ratatui` 和 `crossterm` 实现，负责博客列表、文章列表、未读视图和键盘操作。抓取结果通过消息通道送回界面，所以网络请求不会挡住日常浏览。

## 安装

目前可以在 macOS 或 Linux 上使用，需要先安装较新的 Rust 工具链。最直接的方式是从 GitHub 安装：

```bash
cargo install --git https://github.com/worldlightyjx/blog-tracker-tui.git
blog-tracker-tui
```

如果已经把仓库克隆到本地，也可以在项目目录运行 `cargo install --path .`。目前不需要先创建配置文件；不写配置时会使用默认设置。

## 怎么用

进入界面后，按 `a`，粘贴博客主页或 Feed 地址并回车。选中博客按 `Enter` 看文章，再选中文章按 `Enter` 在浏览器中打开。日常我最常用的是 `u` 看全部未读、`r` 立即检查更新，然后继续在终端里做别的事。

| 按键 | 作用 |
| --- | --- |
| `↑` / `↓` 或 `j` / `k` | 移动光标 |
| `Enter` | 进入博客；打开选中的文章 |
| `a` | 添加博客主页或 Feed |
| `u` / `s` | 查看全部未读 / 返回博客列表 |
| `r` | 立即检查所有博客 |
| `Space` | 切换当前文章的已读状态 |
| `m` | 将当前博客的文章标为已读；在未读视图中标记全部 |
| `h` | 在浏览器中打开博客主页 |
| `d` | 确认后删除博客及其本地文章历史 |
| `q` / `Esc` | 返回或退出；`?` 打开帮助 |

## 配置没有 RSS 的博客

自动识别不到文章时，博客会显示 `needs config`。这时可以在 `config.toml` 里为它指定文章列表页和 CSS 选择器。配置文件默认位置是：

- macOS：`~/Library/Application Support/blog-tracker-tui/config.toml`
- Linux：`$XDG_CONFIG_HOME/blog-tracker-tui/config.toml`；未设置 `XDG_CONFIG_HOME` 时为 `~/.config/blog-tracker-tui/config.toml`

目录和文件不存在时需要自行创建。下面是一个示意配置；把网址和选择器换成目标博客的实际值即可：

```toml
check_interval_minutes = 30 # 自动检查间隔，单位为分钟
max_pages = 3                # 默认最多检查的列表页数

[[sites]]
url = "https://example.com/blog/"          # 必须对应在 TUI 中添加的地址
list_url = "https://example.com/blog/archive" # 可选：文章列表页
article_selector = "article h2 a[href]"    # 文章链接的 CSS 选择器
# next_page_selector = "a[rel='next']"     # 可选：下一页链接
max_pages = 5                               # 可选：覆盖全局页数
```

保存后在 TUI 中按 `r`，程序会重新读取配置并检查。也可以启动时用 `--config /path/to/config.toml` 指定文件，用 `--data-dir /path/to/data` 指定 SQLite 数据目录；订阅和已读状态保存在数据目录下的 `tracker.db`，不在 TOML 配置里。更详细的选择器示例见仓库的 [CONFIG.md](https://github.com/worldlightyjx/blog-tracker-tui/blob/main/CONFIG.md)。

目前网页识别针对公开、无需登录且能从 HTML 看到文章链接的站点。需要 JavaScript 才生成文章列表的博客还不支持；很久没有刷新、而旧文章已从 Feed 或可访问的列表页消失时，也无法补回那段历史。这是一个刻意保持小巧的工具，我会继续按自己的使用体验慢慢改。

如果你也有一堆“收藏了却忘记再打开”的博客，可以试试 [Blog Tracker TUI](https://github.com/worldlightyjx/blog-tracker-tui)。觉得有用的话，欢迎给 [GitHub 仓库](https://github.com/worldlightyjx/blog-tracker-tui) 点个 Star ⭐。
