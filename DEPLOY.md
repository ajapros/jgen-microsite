# Deploying jgen.chenile.org

This is a GitHub Pages Jekyll site served at the custom domain in `CNAME` (`jgen.chenile.org`).

## First-time publish

1. Create a GitHub repo (e.g. `rajakolluru/jgen-welcome-page` or `jgen.chenile.org`).
2. From this folder:

   ```bash
   git init
   git add .
   git commit -m "JGen documentation microsite"
   git branch -M main
   git remote add origin git@github.com:rajakolluru/<repo>.git
   git push -u origin main
   ```

3. In the repo's **Settings → Pages**, set the source to the `main` branch (root). GitHub builds the Jekyll
   site automatically.
4. **DNS:** add a CNAME record for `jgen` → `rajakolluru.github.io` (or an `ALIAS`/`A` set for an apex). The
   `CNAME` file in this repo already pins `jgen.chenile.org`; change it if you pick another domain.

## Updates

```bash
git add -A && git commit -m "…" && git push
```

GitHub Pages rebuilds on push. For a local preview: `bundle install && bundle exec jekyll serve`.
