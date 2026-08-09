# tungnhitran.github.io

Jekyll site with a custom homepage (profile card + contents + auto category sidebar).

## Edit these
- `_config.yml` → your name, bio description, and social links (github/linkedin/devto)
- `index.md` → the bio paragraphs shown under your name
- `assets/img/avatar.svg` → replace with your photo (name it avatar.jpg and update
  the path in `_layouts/home.html`, or just overwrite avatar.svg)

## How it works
- `_layouts/home.html` → homepage: profile, Contents list, Categories sidebar
  (counts auto-generated from each post's `tags`)
- `_layouts/post.html` / `default.html` → article + page shell
- `assets/css/style.css` → all styling (edit colors at the top)
- Posts in `_posts/` as `YYYY-MM-DD-slug.md` with front matter

Source = "Deploy from a branch" (main / root). No Actions workflow needed —
delete any `.github/workflows/*.yml` to avoid conflicts.
