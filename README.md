# bm — Bookmarks, pass-style

`bm` is a simple command-line bookmark manager inspired by [`pass`](https://www.passwordstore.org/). It comes with a OpenClaw integation, easy to save your bookmarks in git.

Instead of storing bookmarks in a database, **each bookmark is stored as a small Markdown file in a normal directory tree**.

This makes the bookmark collection:

* easy to read and edit by hand
* easy to search with standard Unix tools
* easy to back up
* easy to synchronize with Git
* easy to use from scripts
* easy to integrate with OpenClaw
* independent of a browser or cloud service

The main idea is simple:

> **One bookmark = one text file.**

This also means finding an old link does not depend on an LLM remembering it. Bookmark searches are deterministic and operate directly on the bookmark store.

---

## Components

| Component                          | Purpose                  |
| ---------------------------------- | ------------------------ |
| `scripts/bookmarks/bm`             | Command-line interface   |
| `scripts/bookmarks/_bm.py`         | Python library           |
| `mcp-servers/bookmarks-mcp-server` | OpenClaw/MCP integration |

**Execution:** on demand
**LLM required:** no

---

## Bookmark store

By default, bookmarks are stored in:

```text
~/.bookmarks
```

The location can be changed using `BOOKMARK_DIR`.

For example:

```bash
export BOOKMARK_DIR="$HOME/.bookmarks"
```

Bookmarks are organized using normal directories:

```text
~/.bookmarks/
├── cooking/
│   ├── carbonara.md
│   └── pizza-dough.md
├── linux/
│   ├── systemd.md
│   └── nftables.md
└── networking/
    └── sonic/
        └── dell-documentation.md
```

### Bookmark format

Each bookmark is a small Markdown/text file:

```yaml
url: https://example.com/carbonara
title: Pasta carbonara
tags: recept, italiaans
added: 2026-09-20

Free-form notes can be written after the first blank line.
```

The filename and directory provide the bookmark name and hierarchy.

For example:

```text
cooking/pasta/carbonara.md
```

becomes:

```text
cooking/pasta/carbonara
```

---

## Store location resolution

`bm` determines the bookmark directory in the following order:

1. `$BOOKMARK_DIR`
2. `$BM_STORE`
3. an `export BOOKMARK_DIR=...` line found in `~/.bashrc` or `~/.profile`
4. `~/.bookmarks`

You can see which store is currently being used with:

```bash
bm path
```

For more detail:

```bash
bm path --why
```

### Why read `.bashrc` and `.profile`?

Services such as OpenClaw, cron jobs and systemd units do not necessarily load your interactive shell configuration.

In particular, `.bashrc` commonly exits early when Bash is not running interactively.

Without this handling, you could accidentally end up with:

```text
Terminal  → /some/custom/bookmark/store
OpenClaw → ~/.bookmarks
```

`bm` therefore reads the `export BOOKMARK_DIR=...` definition directly as text.

The parser understands:

* `$HOME`
* `~`
* quoted paths
* comments

Command substitutions are deliberately ignored.

---

# CLI

Run:

```bash
bm -h
```

to see the available commands.

## Show the bookmark tree

```bash
bm
```

This displays the bookmark directory tree.

---

## Create or edit a bookmark

```bash
bm -m NAME
```

or:

```bash
bm -m NAME URL
```

This is equivalent to:

```bash
bm edit NAME
```

For example:

```bash
bm -m cooking/carbonara https://example.com/carbonara
```

`bm` opens `$EDITOR` with a template:

```yaml
url: https://example.com/carbonara
title:
tags:
added: 2026-09-20

# Notes can be added here.
```

Lines beginning with `#` are ignored.

If the title is left empty, `bm` attempts to retrieve the page title automatically.

If you close the editor without changing the template, nothing is saved.

Invalid URLs cause the editor to be opened again so the bookmark can be corrected.

If the URL already exists in the bookmark store, `bm` warns about the duplicate.

---

## Add a bookmark directly

```bash
bm add URL [NAME]
```

Example:

```bash
bm add https://example.com/carbonara cooking/carbonara
```

Optional metadata can be supplied directly:

```bash
bm add https://example.com/carbonara cooking/carbonara \
    -t "recept,italiaans" \
    -T "Pasta carbonara" \
    -n "Good traditional carbonara recipe."
```

Useful options:

```text
-t    tags
-T    title
-n    notes
```

To prevent `bm` from fetching information from the webpage:

```bash
bm add URL NAME --no-fetch
```

---

## Search bookmarks

```bash
bm find WORDS...
```

Example:

```bash
bm find carbonara
```

Multiple words can be supplied:

```bash
bm find pasta carbonara
```

All supplied words must match somewhere in the bookmark.

The search includes:

* bookmark name
* URL
* title
* tags
* notes

Title and tag matches receive the highest ranking.

### Search by tag

```bash
bm find pasta -t recept
```

### Search inside a folder

```bash
bm find linux -f documentation
```

### JSON output

```bash
bm find carbonara --json
```

This is useful for scripts and integrations.

---

## Open a bookmark

```bash
bm open NAME
```

or search and open directly:

```bash
bm open WORDS...
```

Example:

```bash
bm open carbonara
```

If several bookmarks match, `bm` presents a picker.

The browser is selected using:

```text
$BROWSER
```

and falls back to:

```text
xdg-open
```

### Print instead of opening

```bash
bm open carbonara -p
```

When no graphical session is available, for example over SSH, `bm` prints the URL instead of silently failing to launch a browser.

---

## Show a bookmark

```bash
bm show NAME
```

Useful variants include:

```bash
bm show NAME -u
bm show NAME -o
bm show NAME --json
```

---

## Search using a regular expression

```bash
bm grep REGEX
```

Example:

```bash
bm grep 'github\.com'
```

---

## List tags

```bash
bm tags
```

---

## Move or rename bookmarks

```bash
bm mv OLD NEW
```

Example:

```bash
bm mv recipes/carbonara cooking/italian/carbonara
```

---

## Remove a bookmark

```bash
bm rm NAME
```

---

# Import browser bookmarks

`bm` can import standard browser `bookmarks.html` files:

```bash
bm import bookmarks.html
```

This works with exported bookmarks from browsers such as:

* Chrome
* Chromium
* Firefox
* Edge
* Safari

During import, `bm`:

* preserves the browser's folder structure
* preserves original bookmark dates where available
* skips duplicate URLs
* skips unsupported links

Only HTTP and HTTPS bookmarks are imported.

Links such as the following are ignored:

```text
javascript:
```

---

# Safe bookmark names

Bookmark names are sanitized before being used as filesystem paths.

Names cannot contain constructs that could escape from the bookmark store.

For example, names using:

```text
..
```

or leading dots and path traversal tricks are rejected.

This ensures that bookmark operations stay inside the configured bookmark directory.

---

# Git integration

Because every bookmark is an ordinary text file, the entire bookmark store can be managed with Git.

Initialize the store:

```bash
bm git init
```

After initialization, changes made through `bm` are automatically committed.

You can run normal Git commands through:

```bash
bm git ARGS...
```

For example:

```bash
bm git status
```

or:

```bash
bm git log --oneline
```

## Automatic push

If the repository has a remote named:

```text
origin
```

new commits are automatically pushed in the background.

Automatic pushing can be disabled with:

```bash
bm git config bm.autopush false
```

This keeps normal bookmark operations fast while still synchronizing changes to the remote repository.

---

## Git credentials

Authentication credentials are deliberately **not stored in the Git remote URL or Git configuration**.

Instead, a repository-local credential helper reads the Forgejo token file when a push is performed.

This avoids configurations such as:

```text
https://username:TOKEN@git.example.org/user/bookmarks.git
```

and keeps credentials separate from the repository configuration.

The current production bookmark store uses a private Forgejo repository:

```text
benjamin/bookmarks
```

---

# OpenClaw integration

`bm` includes an MCP server so OpenClaw can work with the bookmark store directly.

The MCP tool is called:

```text
bookmarks
```

It provides the following operations:

```text
find
show
list
add
tags
```

Deletion and moving bookmarks are deliberately **not exposed through MCP**.

Destructive operations remain CLI-only.

---

## Adding bookmarks through OpenClaw

The MCP `add` operation follows two important rules:

1. Existing bookmarks are never overwritten.
2. The same URL is never stored twice.

If OpenClaw attempts to add a URL that already exists, new tags are merged into the existing bookmark instead.

For example, through the fastlane interface:

```text
bewaar https://example.com
tags: linux, networking
naam: docs/example
notitie: Useful documentation
```

---

## Searching through OpenClaw

Examples:

```text
zoek in mijn bookmarks naar systemd
```

or:

```text
welke bookmarks heb ik met tag linux
```

The search is performed by `bm` itself.

No AI or semantic memory is required to locate the bookmark.

---

# Design philosophy

`bm` intentionally avoids a database.

The filesystem **is the database**.

A bookmark collection remains useful even without the `bm` command:

```bash
find ~/.bookmarks -type f
```

```bash
grep -R "carbonara" ~/.bookmarks
```

```bash
git -C ~/.bookmarks log
```

Bookmarks can also be edited using any text editor.

This keeps the system:

* transparent
* portable
* scriptable
* Git-friendly
* easy to recover
* independent of a particular application

---

# Testing

The CLI has been tested end-to-end using temporary bookmark stores.

Tests include:

* editor sessions using a fake `$EDITOR`
* adding bookmarks
* editing bookmarks
* searching
* regular-expression searches
* tag handling
* moving bookmarks
* removing bookmarks
* nested folder imports
* invalid bookmark names
* invalid URLs
* duplicate detection
* Git initialization
* automatic Git commits
* background Git pushes
* MCP JSON-RPC communication
* Dutch OpenClaw commands
* English OpenClaw commands

Background pushing has been tested against both:

* a local bare Git repository
* the production Forgejo repository

### Not tested

Launching a real graphical browser has not been tested because development and testing were performed over SSH without a graphical display.

The SSH fallback that prints the URL instead has been tested.

---

# Quick start

Set the bookmark directory:

```bash
export BOOKMARK_DIR="$HOME/.bookmarks"
```

Create your first bookmark:

```bash
bm -m linux/systemd https://systemd.io/
```

Search for it:

```bash
bm find systemd
```

Show it:

```bash
bm show linux/systemd
```

Open it:

```bash
bm open systemd
```

Initialize Git:

```bash
bm git init
```

Your bookmark collection is now simply:

```text
~/.bookmarks/
```

A directory full of small, readable, version-controlled files.
