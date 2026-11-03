+++
title = 'I Built a Blog Tracking TUI with Codex in One Lunch Break'
date = 2026-11-03
draft = false
description = 'Why I built Blog Tracker TUI, how it works, and how to install, use, and configure it.'
tags = ['Codex', 'Rust', 'TUI', 'Blogging']
+++

I like wandering through personal blogs. When I find a good one, I want to keep it around and see what the author writes next. I tried several RSS readers, but none quite fit the way I wanted to browse. Some blogs I wanted to follow did not offer RSS at all. Browser bookmarks were no better: I would save a site with the best intentions, then rarely open it again.

So I wanted a small tool that would remember blogs for me and show me when they had new posts. I already spend much of my working day in a terminal. When I feel like taking a quick break, checking my favorite blogs from there feels natural. One lunch break and a session with Codex later, that idea became [Blog Tracker TUI](https://github.com/worldlightyjx/blog-tracker-tui).

## What it does

Blog Tracker TUI is a local terminal app for following blogs. Give it a homepage or feed URL and it tracks article links, shows unread counts, and remembers what you have opened. It supports RSS, Atom, and JSON Feed, and it can also find posts on public blog pages without a feed.

I wanted it to be a place to *discover* updates, not a full reading app. The terminal shows titles and links; pressing `Enter` on an article opens the original post in your default browser and marks it read if the browser opens successfully. Reading still happens on the author's site.

## How it works

The app is written in Rust. Its main path looks like this:

```text
Blog homepage or feed URL
    → Fetch and detect (feed first, HTML fallback)
    → Extract article titles, URLs, and dates
    → Store subscriptions, articles, and read state in SQLite
    → Show blogs and unread posts in the TUI
```

The fetcher uses `reqwest` for HTTP, `feed-rs` for feeds, and `scraper` for HTML. If the URL points to a feed, it parses it directly. For a homepage, it looks for a linked feed before trying to recognize article links in the page. HTML layouts vary, so a per-site CSS selector and pagination rule can help when automatic detection falls short.

SQLite keeps the data locally. The app identifies an article by its URL within a blog, so a changed title does not make the same post unread again. On the first successful check, it marks at most the three newest existing posts unread and saves older ones as read. After that, newly discovered URLs are unread. The interface loads saved data immediately while a worker thread checks for updates. It checks every 30 minutes by default while running, and a failed check preserves saved articles and shows the error and last successful check time. `ratatui` and `crossterm` handle the interface and keyboard input; the worker sends results back through a channel so fetching does not block navigation.

## Install

You need a recent Rust toolchain on macOS or Linux. Install directly from GitHub, then launch the app:

```bash
cargo install --git https://github.com/worldlightyjx/blog-tracker-tui.git
blog-tracker-tui
```

If you have cloned the repository, run `cargo install --path .` from its directory instead. A config file is optional; the app uses defaults when none exists.

## Use it

Press `a` to add a blog homepage or feed URL. Select a blog and press `Enter` to see its posts; press `Enter` again on a post to open it in your browser. My usual routine is `u` for the combined unread list and `r` when I want to check everything right away.

| Key | Action |
| --- | --- |
| `↑` / `↓` or `j` / `k` | Move the selection |
| `Enter` | Open a blog's list or the selected article |
| `a` | Add a blog or feed URL |
| `u` / `s` | Show all unread posts / return to blogs |
| `r` | Check all blogs now |
| `Space` | Toggle an article's read status |
| `m` | Mark the current blog read, or all posts in the unread view |
| `h` | Open the blog homepage in your browser |
| `d` | Delete a blog and its local history after confirmation |
| `q` / `Esc` | Go back or quit; `?` shows help |

## Configure blogs without feeds

If automatic HTML detection finds no posts, the blog shows `needs config`. Add a rule to `config.toml` with the article list page and a CSS selector for post links. The default file location is:

- macOS: `~/Library/Application Support/blog-tracker-tui/config.toml`
- Linux: `$XDG_CONFIG_HOME/blog-tracker-tui/config.toml`, or `~/.config/blog-tracker-tui/config.toml` if `XDG_CONFIG_HOME` is unset

Create the directory and file if they do not exist. This is an illustrative rule; replace the URLs and selector with those of the blog you follow:

```toml
check_interval_minutes = 30 # Minutes between automatic checks
max_pages = 3                # Default number of list pages to check

[[sites]]
url = "https://example.com/blog/"           # Match the URL added in the TUI
list_url = "https://example.com/blog/archive" # Optional article list page
article_selector = "article h2 a[href]"     # CSS selector for post links
# next_page_selector = "a[rel='next']"      # Optional next-page link
max_pages = 5                                # Optional per-site limit
```

Save the file and press `r` to reload the rule and check again. You can also start the app with `--config /path/to/config.toml` or choose a SQLite data directory with `--data-dir /path/to/data`. Subscriptions and read state live in `tracker.db` under that data directory, separate from the TOML file. The repository's [CONFIG.md](https://github.com/worldlightyjx/blog-tracker-tui/blob/main/CONFIG.md) has more selector examples.

HTML detection currently works with public pages whose article links are present in the HTML. It cannot handle sites that require a login or generate their entire article list with JavaScript. If an old post has disappeared from both the feed and reachable list pages before the next check, the app cannot recover it. I am keeping this tool small and improving it as I use it.

If you also have a pile of blogs you bookmarked and forgot to revisit, give [Blog Tracker TUI](https://github.com/worldlightyjx/blog-tracker-tui) a try. If it helps, please give the [GitHub repo](https://github.com/worldlightyjx/blog-tracker-tui) a Star ⭐.
