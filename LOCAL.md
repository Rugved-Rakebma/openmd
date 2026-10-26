# LOCAL.md — divergence from upstream

Fork of [`RufusLin/openmd`](https://github.com/RufusLin/openmd). Base: **v1.6.1** (`54dd9d9`).

`origin` = `Rugved-Rakebma/openmd` · `upstream` = RufusLin, **push disabled**
(`git remote set-url --push upstream DISABLED`).

Read this before pulling from upstream. Every entry below is a change to a file upstream also
edits, so every entry is a potential conflict.

## Why this fork exists

openmd has **two rendering surfaces**: the markdown pane is HTML in a `QWebEngineView` and is
fully themeable via CSS; everything else — sidebar panel, TOC tree, META/HELP buttons, swatch
strip, tab bar — is **native Qt widgets styled from hardcoded constants**. Upstream reads the active theme in exactly two places, both inside the swatch bar,
so no theme can ever reach the chrome.

The result out of the box is a window split in two: a themed document beside a bright green
sidebar (`#264b32`, buttons `#4caf50 → #2e7d32`) under a native light-mode tab bar.

That is not fixable by configuration. Hence the fork.

## The structural rule this fork follows

**One CSS theme drives the whole application.** Chrome colours are *derived* from the same
`body.theme-*` block the document uses — they are never written down twice.

Consequence: all future **design** work happens in `src/openmd-default.css`, in **zero** Python.
(Change 5 moved that file from `~/.openmd.css` into the repo; Change 2 originally said `$HOME`.)

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

### 4. Durable config split from runtime state — `src/openmd.py`

Upstream writes settings **and** state to `~/.openmd.config`: `theme` and `show_meta` sit alongside
`[history] last_file` and `[update] last_check`, which rewrite on every launch and every update
poll. A settings file that rewrites itself cannot be version-controlled — so the theme pin could
not be backed up without leaving the dotfile repo permanently dirty, which trains you to ignore
the one signal that catches real drift.

Third fenced block, entirely inside `_load_config` / `_save_config`:

| Path | Holds |
|---|---|
| `~/.openmd.config` | everything **not** in `_STATE_SECTIONS` — i.e. settings |
| `~/.local/state/openmd/state.ini` | `[history]`, `[update]` |

- **The bias is deliberate:** sections are excluded by name, so a future *settings* section lands
  in the tracked file by default and only known-churning ones are split out.
- **Self-migrating** — an existing config carrying those sections is split on the next save.
- **No call site changed.** Every reader and writer already funnelled through these two functions,
  and both still see one merged `ConfigParser`.

Verified: `~/.openmd.config` is byte-identical across repeated opens; `state.ini` absorbs the churn.

**On upstream pull:** if upstream adds a new settings section it lands in the config file
automatically. If it adds a new *churning* one, add its name to `_STATE_SECTIONS`.

### 5. The checkout is the stylesheet — `src/openmd.py`, `src/openmd-default.css`

`_load_user_css()` auto-seeded `$HOME/.openmd.css` from the bundled default on first run and then
preferred that copy forever. Under an **editable** install that inverts the point of the fork:
the file that actually renders is an untracked copy in `$HOME`, so editing the checkout appears
to do nothing, and every design change lives outside git.

- **Seeding removed**, along with the `$HOME` lookup that only existed to read what it wrote.
- **cwd lookup removed** — `os.getcwd()/.openmd.css` was tried *first*, and openmd's working
  directory is wherever it was launched from. Any directory containing a `.openmd.css` could
  silently restyle the app. Same class of hazard as resolving an executable from the cwd.
- Remaining chain is fallback-only: install-dir `.openmd.css` (explicit override) →
  `src/openmd-default.css` → packaged data-file → `DEFAULT_THEME_CSS`.
- `DEFAULT_THEME_CSS` is now labelled last-resort and **deliberately not synced** — it stays
  upstream-vintage so a broken install still renders.
- `_get_saved_theme` falls back to `DEFAULT_THEME = 'obsidian'` instead of `''`, so a missing
  `~/.openmd.config` yields the right theme rather than no theme.
- The duplicate `.openmd.css` at the repo root was deleted — packaged by nothing, read by
  nothing once the cwd lookup was gone. (Same shape as Change 1.)

**Consequence, superseding Change 2's wording:** design work happens in
`src/openmd-default.css`, in the repo, in zero Python.

### 6. Obsidian theme, and the chrome that was fighting it — `src/openmd-default.css`, `src/openmd.py`

A port of Obsidian 1.12.7's default dark theme, values taken from its shipped `app.css`.
Headings use the body font at `inherit` colour; tables have no zebra and no header tint; accent
purple appears only on links and the blockquote bar.

Removed alongside it, because each one contradicted the theme:

| Removed | Why |
|---|---|
| Google Fonts `@import` (11 families) | fetched on every render; none installed locally; offline the typography collapsed. The Obsidian stack resolves to SF Pro on macOS — no webfont at all |
| KaTeX + Mermaid CDN scripts | 4 more requests per file, for features this install does not use. **The render path now makes no network calls** |
| Document dim + `#58a6ff` inset border on focus | a hardcoded GitHub blue matching no theme, and a rule that washed the page out whenever you touched the sidebar |
| Sidebar `QGraphicsOpacityEffect` (0.6 on blur) | faded the TOC almost permanently, its one job being legibility |
| `QTreeWidget:focus` accent ring | a saturated box, the brightest object on screen |
| Accent-filled TOC selection and META/HELP hover | Obsidian's active nav item is a grey wash with text colour unchanged; both are grey steps now |
| Sidebar `border-right`, lightened panel background | the splitter dot is boundary enough |
| Search box, Find button, `Cmd+F`, `_SearchLineEdit`, `focusSearchRequested` | two permanent rows of chrome; Qt's `findText` highlight is also the one thing on the page that cannot be themed |
| Window title `openmd (x.y.z) by RufusLin` | now `Open MD` |

Front-matter went from a hidden, `<br>`-joined `yaml.dump` blob to a **properties table**
(`_properties_html`) — 9em muted label column, list values as pills, `tags` as filled accent
pills, URLs linkified — and `show_meta` now defaults **true**. Obsidian's three truncation rules
(row `overflow:hidden`, key ellipsis, 3-line clamp) were ported and then deliberately reversed:
they work there because clicking a value expands it, and this view is read-only. A filename
**inline title** sits above it.

Two traps worth knowing before editing:

- **The unscoped `#meta` rule** in `openmd.py`'s `CSS` constant styles the panel as a monospace
  `pre-wrap` box. Overriding background and border is not enough — `font-family` and
  `white-space` still inherit, so the theme resets them explicitly.
- **`_openmd_shade_even_rows()` writes inline styles**, so CSS cannot override it. It early-returns
  on `theme-obsidian` instead.

## Not changed, deliberately

- `SIDEBAR_CSS` / `VIEW_CSS` constants — kept verbatim as the fallback path
- HELP dialog (mint-green SVG blobs) and the update popup — both transient, not worth the diff
- HELP dialog window title still reads `openmd – Quick Help`
- The 16 legacy themes — still present and still swatch-selectable, though their display fonts
  went with the Google Fonts import, so they now fall back to system faces
- `_parse_themes` — untouched, so the swatch bar keeps working exactly as upstream intends
- The mermaid `<pre class="mermaid">` pre-processing in `_render_markdown_core` — harmless with
  the script gone (blocks render as plain code), and removing it would widen the diff

## Install

Editable, so edits here take effect on the next launch with no rebuild:

```bash
uv tool install --editable ~/agent-earth/projects/local-apps/openmd
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
