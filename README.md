# blog

Source for <https://cartergriffin.github.io/blog/>.

## How it's built and published

- **Engine:** GitHub Pages' built-in Jekyll build ("Deploy from a branch", `main`, root `/`). There is no Actions workflow in this repo; GitHub builds and publishes the site on every push to `main`. Build status shows under the repo's Actions tab as "pages build and deployment".
- **Theme:** [minima](https://github.com/jekyll/minima), set by `theme:` in `_config.yml`. It is one of the themes GitHub Pages supports natively, so no Gemfile is needed.
- **Plugins:** `jekyll-feed` (Atom feed at `/blog/feed.xml`) and `jekyll-seo-tag` (meta and Open Graph tags, used for LinkedIn link previews). Both are on GitHub Pages' allow-list.
- **URL settings:** `url` and `baseurl` in `_config.yml` must stay `https://cartergriffin.github.io` and `/blog`, because the site is a project page under `/blog/`. If a custom domain is added later, change both.

## Layout

| Path | What it is |
|------|------------|
| `_config.yml` | Site title, description, theme, plugins, permalink format (`/:year/:month/:day/:title/`), header nav pages |
| `index.md` | Home page; minima's `home` layout lists every post, newest first, with excerpts |
| `about.md` | About page, linked from the header via `header_pages` |
| `_posts/` | One Markdown file per post |
| `assets/` | Images and other files referenced from posts, if any |

## Writing a post

1. Create `_posts/YYYY-MM-DD-short-slug.md`. The date in the filename sets the URL and sort order.
2. Start it with front matter:

   ```yaml
   ---
   layout: post
   title: "Post title"
   date: 2026-11-11 09:00:00 -0700
   tags: [powershell, group-policy]
   description: "One sentence used for link previews."
   ---
   ```

3. Write Markdown below it. The first paragraph is the excerpt on the home page.
4. Gotcha: Jekyll runs Liquid over every post, so a literal `{{` or `{%` in a code block gets interpreted. Wrap such blocks in `{% raw %}` and `{% endraw %}`.
5. Merge to `main`. The site updates a minute or two later.

## Previewing locally (optional)

This needs Ruby. Run `gem install bundler github-pages`, then `jekyll serve --baseurl /blog`, and open <http://localhost:4000/blog/>.
