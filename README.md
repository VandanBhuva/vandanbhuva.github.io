# vandanbhuva.github.io

Personal portfolio site, built with [Hugo](https://gohugo.io) using a small custom theme (no external theme dependency) and deployed to GitHub Pages by GitHub Actions.

## Run locally

```sh
# Install Hugo extended 0.139+ (e.g. `brew install hugo`), then:
hugo server
# open http://localhost:1313
```

## Where things live

| What | File |
| --- | --- |
| Name, tagline, email/GitHub/LinkedIn/resume links | `hugo.toml` → `[params]` |
| Home / about text | `content/_index.md` |
| Experience & education | `content/experience.md` (uses the `role` shortcode) |
| Projects | `content/projects/*.md` (one file per project) |
| CTF / vuln writeups | `content/writeups/*.md` |
| Blog posts | `content/blog/*.md` |
| Resume | put the PDF at `static/resume.pdf`, set `resume = "/resume.pdf"` |
| Contact | `content/contact.md` |
| Styles | `assets/css/main.css` (colour tokens at the top) |
| Templates | `layouts/` |

Placeholder content is marked with the `{{< todo >}}…{{< /todo >}}` shortcode, which renders as a yellow TODO box. Search for `todo` to find everything that still needs real content.

### New post or writeup

```sh
hugo new content writeups/my-ctf-challenge.md
```

Front matter: `title`, `date`, `tags`, `description`. Set `draft: true` to keep it off the live site.

## Deploying

Pushes to `main` run `.github/workflows/hugo.yml`, which builds the site and publishes it. One-time setup: **Settings → Pages → Build and deployment → Source: GitHub Actions**.
