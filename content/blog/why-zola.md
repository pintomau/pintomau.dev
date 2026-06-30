+++
title = "Why I Chose Zola for My Blog"
date = 2026-06-30
description = "A comparison of static site generators and why Zola won for this blog — speed, simplicity, and Rust under the hood."
[taxonomies]
tags = ["web", "tooling", "rust"]
[extra]
draft = false
+++

Lorem Ipsum - When I decided to start this blog, the first question was: which static site generator? There are dozens to choose from, each with its own trade-offs. Here's how I landed on [Zola](https://www.getzola.org/).

## The Contenders

| Feature | Hugo | Jekyll | Eleventy | Zola |
|---------|------|--------|----------|------|
| Language | Go | Ruby | JavaScript | Rust |
| Build Speed | ⚡ Fast | 🐢 Slow | ✅ Good | ⚡ Fast |
| Templating | Go templates | Liquid | Multiple | Tera (Jinja-like) |
| Dependencies | Single binary | Ruby + gems | Node + npm | Single binary |
| Theme Ecosystem | Huge | Huge | Growing | Small but growing |
| Markdown | Goldmark | Kramdown | Configurable | Pulldown-cmark |

## Why Zola Won

### Single Binary, Zero Dependencies

Zola is a single binary. No Ruby, no Node, no package managers. Download it, put it in your PATH, done. This makes CI/CD trivial — the GitHub Actions workflow just downloads Zola and runs `zola build`.

```bash
# That's it. No bundle install, no npm ci.
zola build
```

### Tera Templating

Zola uses [Tera](https://tera.netlify.app/), a Jinja2/Django-inspired template engine. If you've used Jinja2, Twig, or Nunjucks, you'll feel at home:

```html
{% raw %}{% for post in section.pages %}
  <h2>{{ post.title }}</h2>
{% endfor %}{% endraw %}
```

### Built-in Everything

Syntax highlighting, Sass compilation, image processing, RSS feeds, taxonomies — all built in. No plugins required. No configuration headaches.

```toml
# zola.toml
[markdown]
highlight_code = true
compile_sass = true
generate_feeds = true
```

### It's Fast

Zola builds this entire site in under 100ms. Hugo is similarly fast, but Zola's speed is more than adequate. When you're iterating on templates with `zola serve`, the hot-reload is instant.

## What About the Theme Ecosystem?

Zola's theme ecosystem is smaller than Hugo's or Jekyll's. That's a fair criticism. But quality matters more than quantity, and the [Terminus](https://github.com/ebkalderon/terminus) theme checked all my boxes:

- Catppuccin color palette (Mocha dark mode)
- Blog-first layout with clean typography
- Accessible, retro terminal aesthetic
- Active maintenance

## The Trade-offs

No tool is perfect. Here's what I'm accepting with Zola:

- **Smaller community** — Fewer tutorials, fewer Stack Overflow answers. But the docs are solid and the Discord is helpful.
- **Fewer themes** — I had to customize a theme more than I would with Hugo. But that's also a feature: less bloat, more control.
- **No plugins** — Everything is built-in, which means you can't add arbitrary functionality via plugins. So far, I haven't needed to.

## Conclusion

Zola is a pragmatic choice for a developer blog. It's fast, dependency-free, and does exactly what I need without getting in the way. If you're considering a static site generator and want something simple and modern, give it a try.

{{ newsletter() }}


