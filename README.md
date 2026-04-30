# blog

Blog posts for [pikemd.com](https://pikemd.com/blog).

## Structure

- `posts/*.md` — markdown files with frontmatter (`title`, `description`, `date`, `tags`, `draft`)
- `assets/<slug>/` — images referenced in posts as `/blog/<slug>/image.jpg`

## Adding a post

1. Create `posts/<slug>.md` with frontmatter.
2. Put images in `assets/<slug>/`.
3. Reference images in markdown as `/blog/<slug>/image.jpg`.
4. Push to `main` — personal-site rebuilds automatically.
