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
content/notes/       # shorter notes, one .md per note, pasted images in images/
content/.obsidian/   # Obsidian vault settings (content/ is the vault)
content/_templates/  # Obsidian note templates, ignored by Hugo
content/training/    # one folder per course; notes inside a course list on its page
content/about.md     # about page, url: /about/
content/search.md    # search page (PaperMod, uses the home JSON output)
layouts/_markup/     # callouts, math, and diagram render hooks
.vscode/             # snippets, image paste, and "Blog:" tasks
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
hugo new content notes/x.md    # new note
hugo new content training/computer-networks/x.md   # new course note
```

The same commands are VS Code tasks: Ctrl+Shift+P → "Tasks: Run Task" → "Blog: …".

## Notes

Notes are the low-bar counterpart to posts: shorter, less polished, `summary`
optional. They don't appear on the home page (`mainSections` is posts only).

- Each note is one `.md` file. The file name becomes the URL, spaces allowed
  (`Subnetting Cheatsheet.md` → `/notes/subnetting-cheatsheet/`).
- Images go in an `images/` folder beside the note. Pasting in Obsidian or VS Code
  saves them there. `layouts/_markup/render-image.html` resolves them through the
  section's resources, and warns at build time if an image is missing.
- Images in `images/` are published even if the note using them is still a draft.
- A course note lives in its course folder (`content/training/<course>/x.md`) and
  is listed on the course page automatically, under the hand-written lab links.
- Links between pages are plain Markdown links to the `.md` file, relative to the
  current file (`[x](../posts/y.md#heading)`). `render-link.html` turns them into
  page URLs and warns on a link to a page that doesn't exist.
- New notes start as `draft: true`. Set `draft: false` to publish.
- `content/notes/Writing Guide.md` is a permanent draft: every workflow step and
  formatting feature as "you type / you get", rendered by the site itself. Keep it
  in sync when a feature changes.

### Obsidian

`content/` is an Obsidian vault (Obsidian → Open folder as vault → `content`).
The shared settings in `content/.obsidian/` make Obsidian write what Hugo reads:
Markdown links, not `[[wiki links]]`; relative paths; pasted images into `./images`.
Per-user files (workspace layout, cache) are gitignored.

- Note templates are in `content/_templates/` (Note, Course note, Post). Alt+T
  inserts one. Hugo skips that folder via `ignoreFiles` in `hugo.toml`.
- Dates in templates carry the UTC offset. A date without one is read as UTC, so a
  note dated "today" can count as future-dated, and `buildFuture = false` hides it.
- Not supported on the site: `%% comments %%` (use `<!-- -->`), inline `#tags`
  (use the tags property), `![[embeds]]`.

Markdown features (snippet prefix in brackets, type it then press Tab):
- Callouts: `> [!NOTE]`, `TIP`, `IMPORTANT`, `WARNING`, `CAUTION` [`note`, `tip`, …],
  plus Obsidian's types (`info`, `todo`, `question`, `example`, `quote`, `danger`, …).
  `> [!NOTE] Title` sets a custom title. `> [!NOTE]-` is collapsed until clicked
  (good for answers) [`fold`].
- `==highlight==` renders as highlighted text.
- Math: `$x^2$` inline, `$$ … $$` block [`math`], rendered at build time by KaTeX.
  A literal dollar sign in prose must be written `\$`. A bad formula fails the build
  with the file and line.
- Diagrams: a ` ```mermaid ` code block [`mermaid`]. Mermaid loads only on pages
  that have one, and redraws when the light/dark toggle changes.
- "Updated <date>" shows under the title when git says the page changed more than a
  day after `date`. If a commit touched a file without really changing it (e.g. a
  history rewrite), pin it with `lastmod:` in front matter.

## C Language section (notebooks)

Lives at `/training/c-language/`. Each topic is a page bundle: a folder holding
`index.md` and one `.ipynb`. The page renders the notebook itself and shows
**Open in Colab** and **Download .ipynb** buttons.

```
content/training/c-language/
  _index.md                       # index page intro; topics list below it automatically
  install-c-compiler-windows.md   # topic 0, plain page (no notebook), weight 10
  hello-world/                    # topic 1, weight 20
    index.md                      # front matter + {{< notebook >}}
    hello-world.ipynb
layouts/_shortcodes/notebook.html # renders the notebook, buttons, prev/next links
assets/css/extended/notebook.css  # styles for the above
archetypes/c-topic/               # template used by `hugo new --kind c-topic`
```

**Add a topic**

```console
$ hugo new content --kind c-topic training/c-language/pointers
```

Then in the new folder:
1. Edit `index.md`: set `title` (keep the `N.` number prefix), `summary`, and `weight`.
   Weight sets the order: use steps of 10 (topic 2 = 30, topic 3 = 40) so a topic
   can be slotted in between later.
2. Rename `notebook.ipynb` if you like (the file name appears in the Colab URL) and
   write the lesson in it. Any `.ipynb` in the folder is picked up.
3. Set `draft: false` when it's ready. `hugo server -D` previews drafts.

**Notebook format.** Colab runs Python, so C runs through two kinds of cell:
- `%%writefile name.c` on the first line saves the cell as a file. On the site it
  shows as a C code block labelled with the file name.
- Lines starting with `!` are shell commands, e.g. `!gcc -Wall -o name name.c && ./name`.
  On the site they show as a terminal block with `$` prompts.
- A `# Title` line at the top of the first markdown cell is hidden on the site,
  because the page title replaces it. Use `##` for section headings.

**Showing outputs on the site.** The page shows whatever outputs are saved in the
`.ipynb`. To fill them: Open in Colab (or VS Code with gcc), Runtime → Run all,
File → Download .ipynb, and replace the file in the topic folder.

**Edit a topic**: edit its `.ipynb` (VS Code opens it as a notebook) and commit.
The page updates on the next deploy. Never paste notebook content into `index.md`.

**Gotchas**
- Colab links load the notebook from GitHub `main`
  (`notebookRepo` / `notebookBranch` in `hugo.toml`). A new topic's Colab button
  gives a 404 until the change is pushed to `main`. The repo must stay public.
- Moving or renaming a topic folder changes its Colab URL and page URL.
- On the Training page, PaperMod lists sections after regular pages regardless of
  weight. Every course is a folder (section) now, so `weight` orders them; keep new
  courses as folders with `_index.md`, not single `.md` files.

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
