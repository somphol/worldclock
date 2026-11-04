# Deployment

Hosted on **GitHub Pages**, served straight from the repository. There is no build
artifact and no CI workflow: what is committed on `main` is what is served.

- Live URL: <https://somphol.github.io/worldclock/>
- Repository: <https://github.com/somphol/worldclock>
- Source branch: `main`, folder `/` (repository root)
- Entry file: `index.html`

## 1. Deploy a change

```bash
cd ~/Projects/worldclock

# 1. verify locally
python3 -m http.server 8000        # http://localhost:8000/
#    run the manual matrix in docs/TESTING.md

# 2. commit and push
git add -A
git commit -m "…"
git push origin main
```

Pages rebuilds on push, usually within a minute. Confirm:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://somphol.github.io/worldclock/
```

## 2. Repository settings that matter

| Setting | Value | Why |
| --- | --- | --- |
| Pages source | Deploy from a branch → `main` → `/ (root)` | the app is one file at the root |
| Custom domain | none | default `*.github.io` host |
| Enforce HTTPS | on | default certificate |
| Actions | none required | no workflow exists in `.github/workflows/` |

`.nojekyll` is committed. Keep it: it stops Jekyll from touching files and keeps the
deploy path a plain static copy.

## 3. Files that are published

Everything in the repository root is public, including documentation and images:

- `index.html` — the application
- `preview.png`, `docs/social-card.png` — README/social images
- `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`,
  `LICENSE`, `AGENTS.md`, `docs/*.md`
- `robots.txt`, `sitemap.xml` — crawler hints

There is no secrets file and no environment variable. Nothing in the app reads
configuration at runtime.

## 4. Rollback

Pages serves the current `main`, so rolling back is a normal history operation:

```bash
git revert <bad-commit>          # preferred: keeps history
git push origin main
```

or, for an immediate local reset:

```bash
git reset --hard <good-commit>
git push --force-with-lease origin main
```

Force-pushing rewrites the published history; prefer `git revert` unless the bad commit
contains something that must not remain public.

## 5. Caching

GitHub Pages serves static files with short-lived caching and ETags. After a deploy,
hard-refresh (Shift-Reload) or check in a private window if an old `index.html` appears.
The app has no service worker and no offline cache to invalidate.

## 6. Local preview versus production

`index.html` also works from `file://`, so a quick check needs no server. Use the HTTP
server when testing `localStorage` persistence or sharing a link: `file://` origins are
opaque in some browsers and storage behaviour can differ.
