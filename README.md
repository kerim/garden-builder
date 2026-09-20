# garden-builder

This repository is what actually gets published as Kerim's digital garden
website. It is small on purpose: it holds the exported content, the site
config, and a pointer to the builder tooling — not the tooling itself.

- `export/` — the raw public export from Logseq (`index.html`, `404.html`,
  `assets/`, `static/`). This gets replaced wholesale every time a new export
  is made.
- `site.json` — the site's title, navigation, and rendering options (URL
  style, embed style, license, etc). Edit this by hand when the site's
  structure or metadata needs to change.
- `builder/` — a git submodule tracking the `kerim-theme` branch of
  [kerim/garden](https://github.com/kerim/garden.git), the actual static-site
  generator. This repo never edits the builder directly; changes to the
  generator happen in that other repository and get pulled in here as a
  submodule update.
- `build.sh` — builds `dist/` locally from `export/` and `site.json`. This is
  also what Cloudflare Pages runs.
- `publish.sh` — the manual publish routine: build locally first (so a broken
  export never gets pushed), then commit and push `export/` and `site.json`.
- `watch/` — an optional launchd job that runs `publish.sh` automatically
  whenever `export/` changes.

## The routine

1. In Logseq, export the public pages into
   `~/Code/garden-builder/export/`, replacing what's there.
2. Either run `./publish.sh` yourself, or let the watcher (see below) do it
   automatically a little while after the export finishes.
3. Cloudflare Pages picks up the push and rebuilds the live site.

`publish.sh` builds locally before it commits anything, so a broken export
never reaches the deployed site — it just fails locally with an error to fix.

## For a collaborator

Anyone else publishing to this garden needs:

```sh
git clone --recurse-submodules <this repo's URL>
cd garden-builder
```

Then follow the same routine above: drop a fresh export into `export/`, run
`./publish.sh`. The `builder/` submodule brings in the generator automatically
on clone; `git submodule update --init --recursive` refreshes it later if the
generator changes upstream.

## Cloudflare Pages settings

Per the builder's own docs (`builder/static-garden/README.md`):

- **Build command:** `./build.sh`
- **Output directory:** `dist`
- **Framework preset:** None
- **Root directory:** repository root

The builder needs **Python 3.10+** (with venv support) and **Node.js** at
build time — Node runs the exported KaTeX bundle and the graph layout script,
neither of which needs `npm install`. If Cloudflare Pages doesn't already
have suitable versions on its default image, set these build environment
variables to pin them (adjust to whatever versions are actually available on
Cloudflare's image list):

- `PYTHON_VERSION` — e.g. `3.11`
- `NODE_VERSION` — e.g. `20`

Cloudflare needs to be able to clone the `builder` submodule, which means the
`kerim-theme` branch must exist on `https://github.com/kerim/garden.git`
(the fork) before the first Cloudflare build — see "What remains" below.

## The watcher (optional, not yet installed)

`watch/garden-builder-publish` is a small script that:

- debounces: Logseq's export writes many files over several seconds, so the
  script waits 20 seconds after being triggered, then checks whether
  `export/index.html` was modified in the last 15 seconds. If it was (export
  still in progress), it exits without publishing — a later trigger from the
  same export will catch it once things settle.
- uses a `mkdir`-based lock (`.publish.lock/`) so two triggered runs can't
  publish concurrently.
- calls `publish.sh` and logs everything, with timestamps, to
  `~/Library/Logs/garden-builder-publish.log`.

`watch/net.oxus.garden-builder-publish.plist` is the launchd job definition
that triggers this script whenever `export/` changes (`WatchPaths`). Its
`ProgramArguments[0]` points directly at the `garden-builder-publish` script
(not `bash` or `sh`), so macOS shows a meaningful job name in Login Items &
Extensions rather than a generic interpreter name.

This job is **not currently loaded**. To install it:

```sh
launchctl bootstrap gui/$(id -u) ~/Code/garden-builder/watch/net.oxus.garden-builder-publish.plist
```

To remove it later:

```sh
launchctl bootout gui/$(id -u) ~/Code/garden-builder/watch/net.oxus.garden-builder-publish.plist
```

Logs land at `~/Library/Logs/garden-builder-publish.log`.
