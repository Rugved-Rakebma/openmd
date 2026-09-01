# LOCAL.md — divergence from upstream

Fork of [`RufusLin/openmd`](https://github.com/RufusLin/openmd). Base: **v1.6.1** (`54dd9d9`).

`origin` = `Rugved-Rakebma/openmd` · `upstream` = RufusLin, **push disabled**
(`git remote set-url --push upstream DISABLED`).

Read this before pulling from upstream. Every entry below is a change to a file upstream also
edits, so every entry is a potential conflict.

## Why this fork exists

openmd has **two rendering surfaces**: the markdown pane is HTML in a `QWebEngineView` and is
fully themeable via `~/.openmd.css`; everything else — sidebar panel, TOC tree, search box,
META/HELP buttons, swatch strip, tab bar — is **native Qt widgets styled from hardcoded
constants**. Upstream reads the active theme in exactly two places, both inside the swatch bar,
so no theme can ever reach the chrome.

The result out of the box is a window split in two: a themed document beside a bright green
sidebar (`#264b32`, buttons `#4caf50 → #2e7d32`) under a native light-mode tab bar.

That is not fixable by configuration. Hence the fork.

## The structural rule this fork follows

**One CSS theme drives the whole application.** Chrome colours are *derived* from the same
`body.theme-*` block the document uses — they are never written down twice.

Consequence: all future **design** work happens in `~/.openmd.css`, in **zero** Python. Change 2
below is meant to be the last styling change ever made here.

Change 3 (single instance) is a separate axis — behaviour, not appearance — and could not be
config either, since upstream ships no IPC at all. Each new Python change should have to justify
itself the same way: *not expressible as CSS or config, and confined to its own fenced block.*

## Changes

### 1. Removed the duplicated source file — commit `openmd` (root)

Upstream ships `openmd` and `src/openmd.py` as **byte-identical blobs** (same git sha).
`pyproject.toml` packages `src/` only (`package-dir = {"" = "src"}`), so the root copy is dead
weight that silently drifts from the file actually being edited.

**On upstream pull:** if it comes back, delete it again. Never edit it.

### 2. Theme-derived Qt chrome — `src/openmd.py`

Additive block between `_parse_themes` and `_load_config`, fenced by
`── LOCAL PATCH` / `── end LOCAL PATCH` comments. Nothing above it is modified.

| Added | Purpose |
|---|---|
| `_hex_rgb` `_hex_str` `_blend` `_shift` `_is_dark` `_decl` | small colour helpers |
| `_theme_palette(css, name)` | pull `background-color` / `color` from the theme block, plus an accent from its `a` or `h1` rule. **Returns `None` on anything non-hex** |
| `_chrome_css(p)` | sidebar stylesheet — **returns upstream's `SIDEBAR_CSS` unchanged when `p` is `None`** |
| `_swatchbar_css(p)` | swatch strip container |
| `_tabbar_css(p)` | tab bar — upstream has no stylesheet here at all |

Four call sites changed:

| Site | Change |
|---|---|
| `sidebar_col.setStyleSheet(...)` | `SIDEBAR_CSS` → `_chrome_css(self._chrome_palette())`; widget kept on `self._sidebar_col` |
| `_build_swatch_bar` | hardcoded `#0d1117` → `_swatchbar_css(...)`; widget kept on `self._swatch_bar_widget` |
| `_apply_theme` | added `self._restyle_chrome()` so a swatch click repaints the Qt widgets too |
| before `setTabsClosable` | added tab-bar stylesheet from the saved theme |

New methods: `_chrome_palette()`, `_restyle_chrome()`.

**Failure policy: cosmetic code must never break the app.** Every path is wrapped and degrades to
upstream's static chrome. `SIDEBAR_CSS`, `VIEW_CSS` and `_parse_themes` are left intact and
unmodified precisely so that fallback stays truthful.

**On upstream pull:** conflicts will land in the four call sites, not the added block. If upstream
restyles the sidebar itself, re-evaluate whether this patch is still needed.

### 3. Single instance — new files open as tabs, not new windows — `src/openmd.py`

Upstream has **no IPC of any kind**: every `openmd <file>` starts a fresh process and a fresh
window. `MDPreviewWindow.open_tab()` already did the right thing (dedupes to an existing tab, caps
at 12, replaces the active tab past that) — it was simply unreachable from a second process.

Second additive block, again fenced by `── LOCAL PATCH` / `── end LOCAL PATCH`.

| Added | Purpose |
|---|---|
| `_IPC_PATH` | `/tmp/openmd-<uid>.sock`, created mode `0600` |
| `_single_instance_enabled(cfg)` | reads `[display] single_instance`, default **true** |
| `_send_to_running_instance(paths)` | client — plain `AF_UNIX` write, 1 s timeout |
| `_InstanceServer(QObject)` | server — daemon accept thread, `fileReceived` Signal |

Two call sites changed:

| Site | Change |
|---|---|
| before the detached re-exec in `main()` | try the handoff first; `os._exit(0)` if delivered |
| after `window = MDPreviewWindow(...)` | start `_InstanceServer`, wire `fileReceived` → `open_tab` + raise, close on `aboutToQuit` |

**The handoff runs in the parent, before openmd's re-exec** — so a second open costs one socket
write instead of a full Qt/Chromium startup. Measured: **0.18 s vs ~7 s**.

Design notes worth keeping:

- `fileReceived` is emitted from the accept thread; because the server object lives on the GUI
  thread, Qt delivers it **queued**. `open_tab` is never called off-thread.
- Stale-socket cleanup catches **only** `ConnectionRefusedError` / `FileNotFoundError`. A broader
  `except` would delete the socket of an instance that was merely slow to answer.
- **stdin mode is deliberately excluded** — it owns a temp file the receiving instance could not
  clean up.
- Imports are local to the block and QtCore is re-imported under an alias, so upstream's import
  lines are untouched and cannot conflict.
- Escape hatch: `single_instance = false` in `~/.openmd.config` restores upstream behaviour.

Verified: handoff, tab dedupe on reopen, and recovery from a `SIGKILL`-orphaned socket.

**On upstream pull:** if upstream ever adds its own single-instance handling, drop this entirely.

## Not changed, deliberately

- `SIDEBAR_CSS` / `VIEW_CSS` constants — kept verbatim as the fallback path
- HELP dialog (mint-green SVG blobs) and the update popup — both transient, not worth the diff
- Mermaid `theme:'dark'` pin — revisit only if a light document theme is adopted
- `_parse_themes` — untouched, so the swatch bar keeps working exactly as upstream intends

## Install

Editable, so edits here take effect on the next launch with no rebuild:

```bash
uv tool install --editable ~/code/openmd
python -c "import openmd; print(openmd.__file__)"   # must print this checkout
```

## Updating from upstream

```bash
git fetch upstream --tags
git log --oneline HEAD..upstream/main -- src/openmd.py    # read before merging
git merge upstream/main
```

Then re-check: does the app still launch, does a swatch click restyle the sidebar, and did the
root `openmd` duplicate come back?
