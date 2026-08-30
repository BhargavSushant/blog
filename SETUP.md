# Setup, run these in order

Prerequisites: `git`, and Hugo **extended** locally if you want live preview.

```bash
# Hugo extended (Ubuntu/Debian). Match the version pinned in deploy.yml
HUGO_VERSION=0.128.0
curl -fsSL -o /tmp/hugo.deb \
  "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb"
sudo dpkg -i /tmp/hugo.deb
hugo version   # must say "extended"
```

## 1. Create the repo

On GitHub, create an **empty public repo** named `blog` (or `sushantbhargav.com`).
No README, no .gitignore, this scaffold provides them.

## 2. Wire it up locally

```bash
cd /path/to/this/scaffold
git init -b main

# Theme as a submodule. The workflow checks out submodules recursively.
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod

git add .
git commit -m "Initial site: Hugo + PaperMod + Pages deploy"
git remote add origin git@github.com:YOUR_GITHUB_USERNAME/blog.git
git push -u origin main
```

## 3. Replace the placeholders

Search and replace `YOUR_GITHUB_USERNAME` and `YOUR_LINKEDIN` in:
- `hugo.toml`
- `content/about.md`

```bash
grep -rn "YOUR_GITHUB_USERNAME\|YOUR_LINKEDIN" . --exclude-dir=themes --exclude-dir=.git
```

## 4. Enable Pages

Repo → **Settings → Pages → Build and deployment → Source: GitHub Actions**

Not "Deploy from a branch". The workflow uses the Actions deployment path.

## 5. Custom domain

Repo → **Settings → Pages → Custom domain** → `sushantbhargav.com` → Save.
Then tick **Enforce HTTPS** once the certificate provisions (takes a few minutes).

## 6. Cloudflare DNS

Add these records. **All grey-cloud (DNS only)**, proxying breaks GitHub's cert issuance.

| Type | Name | Value | Proxy |
|-------|------|--------------------------------|-------|
| A | @ | 185.199.108.153 | DNS only |
| A | @ | 185.199.109.153 | DNS only |
| A | @ | 185.199.110.153 | DNS only |
| A | @ | 185.199.111.153 | DNS only |
| CNAME | www | YOUR_GITHUB_USERNAME.github.io | DNS only |

Cloudflare SSL/TLS mode: **Full (strict)**.

## 7. Verify

```bash
# local preview, drafts included
hugo server -D

# what CI will build
hugo --minify --gc
```

Push to `main` → Actions tab → watch the deploy. First run takes ~1 minute.

## Writing

```bash
hugo new content posts/my-post.md
```

Set `draft: false` when ready. Drafts are excluded from production builds.

## Cross-posting

Publish here first. Then post to dev.to / Hashnode with a canonical URL pointing
back to `https://sushantbhargav.com/posts/<slug>/` so search engines credit the
original. Both platforms have an explicit canonical URL field.
