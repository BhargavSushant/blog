# CLAUDE.md, sushantbhargav.com

Hugo blog deployed to GitHub Pages via GitHub Actions. Read automatically by Claude Code.

## Stack

- Hugo **extended** (pinned to `0.164.0` in `.github/workflows/deploy.yml`, keep the
  local version and the CI version in sync)
- PaperMod theme as a **git submodule** at `themes/PaperMod` (the workflow checks out
  with `submodules: recursive`, don't vendor the theme instead)
- Deploy via `actions/upload-pages-artifact` + `actions/deploy-pages` (OIDC), not the
  legacy `gh-pages` branch method
- Custom domain `sushantbhargav.com`, DNS at Cloudflare, **grey-cloud / DNS only**

## Layout

```
content/posts/       # published posts, one .md per post
content/about.md     # about page, url: /about/
static/CNAME         # custom domain, do not delete, Pages needs it
hugo.toml            # site config
.github/workflows/deploy.yml
```

## Conventions

- Front matter: `title`, `date`, `draft`, `tags`, `summary`. `summary` is what shows
  in the list view, write it deliberately, it's the hook.
- Filenames are the slug. Keep them short and descriptive, hyphenated, no dates.
- Code blocks always get a language tag. Shell sessions use `console` with `#` or `$`
  prompts so the prompt isn't copied by the copy button.
- Prose voice: direct, specific, first person. Concrete failure over general advice.
  Avoid stock openers and listicles.
- Posts should be defensible, if a post claims experience, it must survive an
  interviewer asking about it.

## Editorial direction

Depth over cadence. Three or four substantial posts beat thirty tutorials. The niche
is **configuration that looks correct and does nothing**, hardening controls that
don't control, verification that verifies the wrong thing.

Post pipeline:
1. ✅ "The Linux Hardening Spec That Would Have Broken Every Server It Touched"
2. Building the toolkit, transactional state, why file hashes are the wrong drift signal
3. Terraform state as the same problem in a different domain
4. GitHub Actions CI across real VM images, and why containers can't test kernel state

## Commands

```bash
hugo server -D                 # local preview, drafts included
hugo --minify --gc             # what CI builds
hugo new content posts/x.md    # new post
```

## Gotchas

- Pages source must be **GitHub Actions**, not "Deploy from a branch".
- Cloudflare proxy (orange cloud) breaks GitHub's cert issuance. Keep records grey.
- `fetch-depth: 0` in the workflow is required, `enableGitInfo` uses git history for
  `.Lastmod`. Don't "optimise" it to a shallow clone.
- `concurrency.cancel-in-progress` is deliberately `false`. A cancelled deploy can
  leave Pages serving a partial build.

## Cross-posting

Publish here first. Then dev.to / Hashnode with `canonical_url` pointing back to
`https://sushantbhargav.com/posts/<slug>/`. Never publish elsewhere first.
