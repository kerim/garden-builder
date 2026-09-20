# garden-builder

This is the operating manual for publishing Kerim's Digital Garden. It covers
the everyday routine, first-time setup, how to change the site's look and
behavior, hosting, local preview, and troubleshooting.

## 1. What this is

Kerim writes in Logseq, in a database ("DB") graph called "Kerim's Digital
Garden." Logseq's **File → Export public pages** feature dumps the pages
marked public as static HTML into this repo's `export/` folder. The
**builder** — a Git submodule pointing at branch `kerim-theme` of
[kerim/garden](https://github.com/kerim/garden) (Kerim's personal fork of
[Arney1/garden](https://github.com/Arney1/garden)) — turns that export into a
finished website (`dist/`). A GitHub Actions workflow builds the site and
deploys it to the Cloudflare Pages project `garden`, publishing it at
<https://garden.oxus.net> (also reachable at
<https://garden-77y.pages.dev>).

In short: **Logseq → export/ → builder → GitHub Actions → Cloudflare Pages →
garden.oxus.net.**

## 2. Publishing a change (the everyday routine)

1. Edit and mark pages public as usual in Logseq.
2. In Logseq: **File → Export public pages**, and point it at
   `~/Code/garden-builder/export` (replacing what's there).
3. Wait about 1–2 minutes.

That's it in the normal case, because the **watcher** — a background job
(`launchd` job `garden-builder-publish`, when installed — see section 3) —
notices the change to `export/`, builds the site, and commits and pushes it
automatically. You don't have to run anything yourself.

To check that it worked:

- Log file: `~/Library/Logs/garden-builder-publish.log` — records the
  watcher's own actions (triggered, debounced, published, or failed).
- GitHub Actions: the repo's Actions tab shows whether the deploy to
  Cloudflare Pages succeeded.

**Manual alternative.** If you don't want to wait for the watcher, or it
isn't installed, run `./publish.sh` yourself from `~/Code/garden-builder`. It
builds locally first, so a broken export fails on your machine instead of
reaching the live site.

**If the browser still shows the old page:** hard-reload it (e.g.
`Cmd+Shift+R` in most browsers) — this is ordinary browser caching, not a
publish failure.

**Logseq export gotcha:** if the export fails with a `copyfile ENOENT`
error, it's a stale Logseq temp file (only purged after the app has been
running for several days). Quit and relaunch Logseq, then export again.

## 3. First-time setup on a new Mac / for a collaborator

Requirements: `git`, `python3` 3.10 or newer, and `node`.

Clone with submodules, since the builder is a submodule and won't come along
otherwise:

```sh
git clone --recurse-submodules <this repo's URL>
cd garden-builder
```

Point Logseq's "Export public pages" at `<clone>/export` (i.e. the `export/`
folder inside wherever you cloned this repo).

Then either:

- Run `./publish.sh` by hand each time you export, or
- Install the watcher so it happens automatically:

  ```sh
  launchctl bootstrap gui/$(id -u) ~/Code/garden-builder/watch/net.oxus.garden-builder-publish.plist
  ```

  If your clone is **not** at `~/Code/garden-builder`, first edit the paths
  in `watch/net.oxus.garden-builder-publish.plist` (the `ProgramArguments`
  and `WatchPaths` entries) and `watch/garden-builder-publish` (the
  `REPO_DIR` variable) to match your clone's location.

  To remove the watcher later:

  ```sh
  launchctl bootout gui/$(id -u) ~/Code/garden-builder/watch/net.oxus.garden-builder-publish.plist
  ```

A collaborator needs **write access to the GitHub repo** (so `publish.sh` can
push). They do **not** need any Cloudflare access — deployment happens
automatically from GitHub Actions using secrets already stored in the repo.

## 4. Changing how the site looks or behaves

Most changes are **configuration**, edited in this repo's `site.json`; a
smaller set are **code**, changed in the builder fork.

### Configuration: `site.json`

| Key | What it controls |
| --- | --- |
| `title` | The site's title. |
| `home_page` | Which page is the homepage. |
| `description` | Site description (meta tags, etc). |
| `url` | The canonical site URL. |
| `language` | Site language code. |
| `navigation` | The list of top-level section pages shown in the sidebar/menu. |
| `navigation_label` | Optional label override for a navigation entry. |
| `url_style` | `"sections"` groups page URLs under whichever navigation entry links to them (e.g. `/technology/page/`); the default `"uuid"` gives every page a flat `/page/<slug>--<uuid>/` URL. |
| `embed_titles` | Whether embedded pages/blocks show a title link above them (only matters when `embed_style` is `"boxed"`). |
| `embed_style` | `"boxed"` (default) draws a box with a title around embedded pages/blocks; `"inline"` renders them as plain sub-blocks with no box, title, or bullet. |
| `author` | Shown in the sidebar license note. |
| `license` | License name shown in the footer/sidebar; known names like `"CC BY 4.0"` are auto-linked. |

Edit `site.json` directly in this repo, commit, and push (or let the next
`publish.sh` run pick it up) — no submodule update needed.

### Code: the builder fork

Anything beyond what `site.json` can control — new rendering behavior, CSS,
new features — is a change to the generator itself, made in
`~/Code/garden` (Kerim's fork, on branch `kerim-theme`), not in this repo.

`kerim-theme` is stacked on top of two upstream pull requests that live on
their own branches in the same fork: `db-embeds` (Arney1/garden #2, embeds)
and `video-embeds` (Arney1/garden #3, video). Changes to those features
should go on the relevant branch; everything else goes on `kerim-theme`
directly.

Once a change lands upstream in `kerim-theme`, pull it into this repo's
`builder/` submodule and record the new pointer:

```sh
cd builder
git pull fork kerim-theme
cd ..
git add builder
git commit -m "Update builder submodule"
git push
```

## 5. Hosting

The site is a **Cloudflare Pages** project named `garden`, on Cloudflare's
free tier. Deployment is handled by `.github/workflows/deploy.yml`, which
runs on every push to `main` (or manually via `workflow_dispatch`): it builds
the site and uploads `dist/` with `wrangler pages deploy`. Cloudflare's own
Git integration is not used.

This needs two GitHub repo secrets:

- `CLOUDFLARE_API_TOKEN` — a token created with the "Edit Cloudflare
  Workers" template, scoped to zone `oxus.net` (this template also grants
  Pages edit access).
- `CLOUDFLARE_ACCOUNT_ID` — the 32-character code visible in the Cloudflare
  dashboard's URL.

`.github/workflows/domain.yml` attaches a custom domain to the Pages
project, but it **cannot** edit DNS — the API token has no DNS permission,
so the domain's DNS record has to be created by hand in the Cloudflare
dashboard:

- **CNAME** `garden` → `garden-77y.pages.dev`, proxied (orange cloud on).

The old Netlify/Eleventy site (repo `kerim/mydatagarden`) is superseded by
this setup and can be deleted.

## 6. Local preview

From this repo:

```sh
./build.sh
python3 -m http.server 8001 --directory dist
```

Then open <http://localhost:8001> in a browser. Note: Python's preview
server doesn't apply the security headers Cloudflare adds in production, so
some behavior (e.g. CSP-dependent features) may differ slightly from the
live site.

## 7. Troubleshooting

**The watcher didn't fire.**
Check the log first:

```sh
tail -50 ~/Library/Logs/garden-builder-publish.log
```

Then check whether the job is actually loaded:

```sh
launchctl print gui/$(id -u)/net.oxus.garden-builder-publish
```

If it's not loaded, install it (see section 3). If it's loaded but silent,
re-export from Logseq to re-trigger it, or just run `./publish.sh` manually.

**Build warnings.**
After a build, check `dist-report.json` (next to `dist/`, not deployed with
the site) — it records page/attachment counts, sizes, and any warnings from
the build.

**A GitHub Actions run failed.**
From this repo, with the `gh` CLI:

```sh
gh run list
gh run view --log-failed
```

**The domain still resolves to the old Netlify site.**
This is almost always DNS or caching, not the Pages deploy. Confirm the
`garden` CNAME record in Cloudflare DNS points to `garden-77y.pages.dev`
and is proxied (see section 5); if it was recently changed, allow time for
DNS and browser/OS DNS caches to catch up, or test with a hard-reload or a
different network.
