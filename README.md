# blog

Source for <https://blog.grifflab.org/>. Built with [Hugo](https://gohugo.io/), deployed by GitHub Actions to GitHub Pages.

## How it builds and deploys

- **Workflow:** `.github/workflows/deploy.yml`.
  - **Every push and PR:** installs Hugo extended at the version pinned in `HUGO_VERSION`, then runs `hugo --gc --minify --panicOnWarning`. Any Hugo warning fails the build, so deprecations get caught in the PR.
  - **`main` only:** uploads `public/` and deploys it with `actions/deploy-pages`. PRs build but never deploy.
- **Pages setting:** repo Settings → Pages → Source must be **GitHub Actions** (`build_type: workflow`). With "Deploy from a branch", GitHub would also run its own Jekyll build, and the two would conflict.
- **Custom domain:** `blog.grifflab.org`, set in repo Settings → Pages (stored by GitHub, so there's no `CNAME` file in the repo). DNS is a `CNAME` record `blog` → `cartergriffin.github.io` at Porkbun. HTTPS is enforced in the same Pages settings once the certificate is issued. Old `cartergriffin.github.io/blog/...` links redirect to the custom domain.
- **Upgrading Hugo:** change `HUGO_VERSION` in the workflow and install the same version locally, so preview matches production.

## Local preview

```powershell
winget install Hugo.Hugo.Extended   # once
hugo server                          # http://localhost:1313/ , reloads on save
```

`hugo --gc --minify --panicOnWarning` runs the exact CI build. Output goes to `public/`, which is gitignored.

## Layout

The site uses its own small set of templates instead of a third-party theme, so there is no theme dependency to pin or update.

| Path | What it is |
|------|------------|
| `hugo.toml` | Site config: `baseURL` (must match the custom domain, `https://blog.grifflab.org/`), title, description, nav menu, permalink format, highlighting. `titleCaseStyle = "none"` stops Hugo title-casing tag names. |
| `content/posts/` | One Markdown file per post. |
| `content/about.md` | About page. |
| `layouts/baseof.html` | Page shell: `<head>` (description, canonical URL, Open Graph and Twitter cards for LinkedIn previews, RSS link, CSS), header nav, footer. |
| `layouts/home.html` | Home page: tagline and every post, newest first. |
| `layouts/single.html` | Post and page body. Posts get date, reading time and tags. |
| `layouts/list.html`, `layouts/taxonomy.html` | `/posts/`, per-tag pages, and the `/tags/` index. |
| `layouts/_partials/post-list.html` | The post list shared by the home, section and tag pages. |
| `assets/css/main.css` | Site styles. Colours are CSS variables, with light and dark sets switched by `prefers-color-scheme`. |
| `assets/css/syntax.css` | Generated code highlighting: `github` for light, `github-dark` for dark, each scoped to its colour scheme. Regenerate with the command in the file's header comment if you change styles. |

## URLs

Posts live at `/:year/:month/:day/:slug/` on `blog.grifflab.org`, the same shape as before the domain move, so the github.io redirect lands on the right page. `slug` comes from each post's front matter. Other URLs: `/posts/`, `/tags/<tag>/`, `/about/`, and the RSS feed at `/index.xml`.

## Writing a post

1. Create `content/posts/YYYY-MM-DD-short-slug.md` with front matter:

   ```yaml
   ---
   title: "Post title"
   date: 2026-11-11T09:00:00-07:00
   slug: short-slug
   tags: [powershell, group-policy]
   description: "One sentence for the home page and link previews."
   ---
   ```

2. Write Markdown below it. Without a `description`, the home page shows the start of the post.
3. Gotchas:
   - Hugo skips posts dated in the future, so a post with a later time won't appear until a build runs after that time.
   - A literal `{{<` or `{{%` in a post is read as a shortcode. Escape it as `{{</*` ... `*/>}}`.
4. Open a PR. CI builds it. Merging to `main` publishes.
