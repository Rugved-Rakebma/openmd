# openmd - by Rufus Lin

**Markdown viewer for humans (not AI models).**

I got tired of reading raw Markdown with `less` or opening VS Code/Cursor just to see it nicely rendered. So I built **openmd**.

Run `openmd *.md` (or any Markdown file) from the shell and a window pops up instantly. Independent Qt window, ESC closes it. Think of it as "less" but for markdown files.

![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue) ![License: MIT](https://img.shields.io/badge/license-MIT-green) ![Platform: macOS / Linux](https://img.shields.io/badge/platform-macOS%20%7C%20Linux-lightgrey) ![PyPI version](https://img.shields.io/pypi/v/openmd)

**GitHub:** [RufusLin/openmd](https://github.com/RufusLin/openmd)

**Warning - Lazy Maintainer:** Yeah, not a fan of reading PRs, but will pay attention to issues to fix bugs. Feel free to fork, but remember to give credit, please.🙏🏻

X: @rufuslinjapan

---

## What it looks like

<video src="https://github.com/user-attachments/assets/42f0b30d-5d7a-4e61-98c6-c3354ad10e16" width="85%" controls autoplay loop muted playsinline></video>
<sub><i>Mobile app users: <a href="https://github.com/user-attachments/assets/42f0b30d-5d7a-4e61-98c6-c3354ad10e16">Tap here to watch the demo video</a></i></sub>

###openmd renders markdown
<img src="https://raw.githubusercontent.com/RufusLin/openmd/main/pix/1.jpg" width="85%" alt="openmd with multiple tabs and TOC sidebar">

###CSS themes (customizable)
<img src="https://raw.githubusercontent.com/RufusLin/openmd/main/pix/2.png" width="85%" alt="CSS themes (customizable)">

###Mermaid, KaTeX too!
<img src="https://raw.githubusercontent.com/RufusLin/openmd/main/pix/3.png" width="85%" alt="Mermaid diagram and KaTeX math rendered">

---

## Usage

```
# Open a single file (extension optional!)
openmd README
openmd README.md

# Open most recently viewed file
openmd -r

# Open multiple files (each in its own tab, up to 12)
openmd doc1 doc2.markdown doc3

# No arguments — interactive picker (choose from .md files in current directory)
openmd

# Glob expansion (works on Windows & POSIX shells)
openmd docs/*.md

# Check CLI version or help
openmd -v
openmd --help

# Pipe stdio
tail -100 x.md | openmd

# on MacOS, render selected markdown
Select text, right click, Services, "Open in openmd"
```

### Remote preview via SSH (optional)

```
remotemd() {
    local remote_path="$1"
    local filename=$(basename "$remote_path")
    local tmp_file="/tmp/remote_preview_${filename}.md"
    scp "home:$remote_path" "$tmp_file" && openmd "$tmp_file"
}
```

---

## Features

- **Auto `.md` Extension & Path Resolution** — run `openmd filename` without `.md` and openmd automatically resolves `.md`, `.markdown`, `.mdown`, or `.mkd` files (with case-insensitive fallback)
- **Local & Remote Link Navigation** — click relative links (`[Doc](./doc.md)`) or remote Markdown URLs (`https://.../file.md`) and openmd opens them directly in new tabs
- **Re-open Last Viewed File** — run `openmd -r` (or `--recent` / `--last`) to instantly reopen your most recently viewed document
- **Closable Multi-file Tabs (up to 12)** — open up to 12 files at once; tabs feature close buttons and `Cmd+W` / `Ctrl+W` closing shortcuts
- **Native CLI Flags** — fast terminal stdout output for `openmd -v` / `--version` and `openmd -h` / `--help` without launching GUI windows
- **Cross-Platform Glob Expansion** — wildcards like `openmd docs/*.md` work on Windows `cmd.exe`/PowerShell as well as POSIX shells
- **Quick Clipboard Shortcuts** — `Cmd+Shift+C` copies absolute file path, `Cmd+Alt+C` copies document text
- **Meta panel** — shows YAML front‑matter in a hidden-by-default div; toggle via the **META** button or the **M** shortcut key
- **Quick Help** — access a concise help dialog with navigation and shortcuts via the **HELP** button or the **H** shortcut key
- **Instant launch** — the shell prompt returns immediately; openmd runs as a fully detached GUI app (no `&` needed, no blocking)
- **Obsidian dark theme by default** — a port of Obsidian's own default dark, with YAML front matter rendered as a properties table
- **17 built-in themes** — switch instantly via the swatch bar at the bottom of the sidebar; fully customizable via `src/openmd-default.css`
- **No network calls** — no webfonts, no CDN scripts; renders identically offline
- **Live reload** — the display pane updates instantly when the file is saved; no manual refresh needed
- **Mermaid diagrams** — *disabled in this fork* (fenced mermaid blocks render as plain code)
- **KaTeX math** — *disabled in this fork*
- **Sidebar TOC** — hierarchical (H1 → H2 → H3); click or press Return to jump to any heading. The sidebar takes up 20% of your screen, and you can easily jump between the sidebar and display using the left/right arrow keys.
- **Dynamic Pane Focus** — unselected panes automatically dim to 60% opacity so you always know exactly where your keyboard focus is.
- **Unix pipes** - accepts markdown from stdin, e.g. ```echo 'markdown string here' | openmd```
- **Interactive file picker** — run with no arguments and choose from `.md` files in the current directory via a curses-based picker
- **Remote image caching** — remote images in your Markdown are downloaded to a local temp cache so they render correctly in the Qt WebEngine view
- **External link handling** — clicking general `http`/`https` web links opens them in your default browser; the display window never navigates away
- **Update notifications** — on startup, openmd quietly checks PyPI (at most once every 6 hours) and shows a non-intrusive popup if a newer version is available
- **Version in title bar** — the window title shows the running version for quick reference
- **Keyboard shortcuts** — `Esc` closes the window; Up/Down arrows and Return navigate the sidebar, Left/Right switch panes, Cmd+Left/Right switch tabs, Cmd+W closes tabs, and Cmd+Shift+C copies paths

---

## Theming with `.openmd.css`

**Edit `src/openmd-default.css`.** That file is the stylesheet — there is no copy to keep in sync.

> **This fork differs from upstream here.** Upstream auto-seeded `$HOME/.openmd.css` on first
> run and then preferred that copy forever, so editing the checkout appeared to do nothing
> while the file that actually rendered sat untracked in your home directory. The seeding, the
> home-directory lookup, and the working-directory lookup have all been removed. The working
> directory one was also a hazard: openmd's cwd is wherever it was launched from, so any
> directory containing a `.openmd.css` could silently restyle the app.

**Lookup order** (first match wins):

| Priority | Location |
|----------|----------|
| 1 | `.openmd.css` in the install directory — explicit opt-in override |
| 2 | **`src/openmd-default.css` — the real one** |
| 3 | `<sys.prefix>/share/openmd/openmd-default.css` — packaged data-file |
| 4 | `DEFAULT_THEME_CSS` embedded in `openmd.py` — broken-install fallback |

**Active theme: `obsidian`** — a port of Obsidian's default dark theme, at the bottom of the
file. It is also the default baked into `_get_saved_theme()`, so no `~/.openmd.config` is
needed. It uses no webfonts: the stack resolves to SF Pro on macOS, which is what Obsidian
itself renders with. The Google Fonts `@import` that pulled eleven families on every render is
gone, as are the KaTeX and Mermaid CDN scripts — the render path makes no network calls.

The sixteen legacy themes are still there and still selectable from the swatch bar, but their
display fonts came from that `@import`, so they now fall back to system faces.

To add a theme, write a `body.theme-yourname { ... }` block. The swatch bar discovers the first
16 automatically. Put `color:` before `background-color:` in the block — `_theme_palette()`
scrapes both with an unanchored regex to build the Qt chrome, and the wrong order makes it
match the tail of `background-color` and silently fall back.

---

## Requirements

- macOS or Linux (POSIX)
- **Python 3.10+**
- [PySide6](https://pypi.org/project/PySide6/) + PySide6-WebEngine
- [Markdown](https://pypi.org/project/Markdown/)
- [BeautifulSoup4](https://pypi.org/project/beautifulsoup4/)
- [PyYAML](https://pypi.org/project/PyYAML/)

**Note:** Mermaid and KaTeX require an internet connection to load from CDN.

---

## Installation

The easiest way to install openmd (recommended, just two steps):

1. Install **uv** (the fastest Python tool manager):
   ```
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```
*(Windows users: use the same curl command in PowerShell or run winget install astral-sh.uv)*

2. Install openmd: ```uv tool install openmd```

That’s it! You can now run openmd like this:
```openmd <filename(s)>```

###From source
```
git clone https://github.com/RufusLin/openmd.git
cd openmd
pip install -e .
```

After this, the openmd command will be available in your shell.


---

## Keyboard shortcuts

| Key | Action |
|-----|--------|
| `Esc` | Close the preview window |
| `↑` / `↓` | Navigate the sidebar TOC |
| `←` / `→` | Move focus between sidebar and display |
| `Cmd + ← / →` | Navigate among tabs |
| `Cmd + W` | Close current active tab |
| `Cmd + Shift + C` | Copy active file path to clipboard |
| `Cmd + Alt + C` | Copy active document text to clipboard |
| `Cmd + Shift + < / >` | Change font size |
| `Return` | Jump to selected heading |
| `M` | Toggle Meta Panel (YAML front-matter) |
| `H` | Show Quick Help dialog |

---

## License

MIT

---

### Troubleshooting

**If you still see an old version after upgrading**  
This happens if you previously installed with `pip install -e .` or plain `pip`.  
Run this once:

```bash
rm -f ~/Library/Python/*/bin/openmd          # remove old pip executable (macOS)
uv tool uninstall openmd
uv tool install openmd
```
