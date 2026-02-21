# CLAUDE.md

Guide for AI assistants working on the Gridmapper codebase.

## Project Overview

Gridmapper is a browser-based dungeon mapping tool for tabletop RPGs. Users open `gridmapper.svg` directly in a browser to create dungeon maps using keyboard-driven input. The project emphasizes minimalism — there is no build step, no framework, and no package manager.

**License:** Public domain (CC0 1.0 Universal)
**Author:** Alex Schroeder

## Repository Structure

```
gridmapper.svg          # The entire application: SVG + CSS + JavaScript (≈4,785 lines)
gridmapper-server.pl    # WebSocket collaboration server (Perl/Mojolicious, 173 lines)
megadungeon.pl          # Procedural dungeon generator (Perl/Mojolicious, 638 lines)
Makefile                # Deployment only: `make upload` rsyncs to production
README.md               # User-facing documentation
LICENSE.md              # CC0 public domain license
images/                 # Screenshots for documentation
```

## Architecture

### gridmapper.svg (Main Application)

A single self-contained SVG file with embedded `<style>` and `<script>` blocks. No external dependencies, no modules, no bundling.

**Key global objects:**

| Object | Purpose |
|--------|---------|
| `Map` | Central state: grid dimensions (30x30), tile width (20px), SVG element references, level management, WebSocket state |
| `Pen` | Cursor/pointer state, mouse/touch input tracking, wall-mode coordinates |
| `Commands` | Undo/redo stack with `push()`, `do()`, `undo()`, `redo()`, `merge()` |
| `Demo` | Tutorial/demo playback system |
| `Tiles` | Sparse 2D data structure (`get(x,y)` returns a `Tile`) |
| `Tile` | Per-cell data: arcs, floor type, walls array, label |

**Key function categories:**

- **Drawing:** `draw()`, `createTile()`, `wallDraw()`, `wallMode()`, `wallPlacement()`
- **Input:** `keyPressed()`, `keyHandling()`, `getChar()`, `move()`
- **Tile manipulation:** `rotateTile()`, `removeWall()`, `getUnusedAngle()`
- **Labels:** `setLabel()`, `showLabelField()`, `hideLabel()`, `deleteLabel()`
- **File I/O:** `download()`, `textExport()`, `textImport()`, `link()`
- **Collaboration:** `join()`, `rejoin()`, `leave()`, `save()`, `load()`
- **Scripting:** `interpret()`, `interpretCont()`, `recreateModel()`

**Variant system:** Tiles cycle through related types via the `Map.variants` object (a linked-list-style mapping). For example, pressing `v` on a `door` cycles through `secret` → `concealed` → `gate` → `archway` → `one-way-door` → `door`.

**Symbol mapping:** `Map.symbolInit()` builds a reverse lookup from keyboard keys to tile types (e.g., `d` → door, `s` → stair, `f` → empty/floor).

### gridmapper-server.pl (Collaboration Server)

- **Framework:** Mojolicious::Lite (Perl)
- **Protocol:** WebSocket on port 8082
- **Deployment:** Hypnotoad (single worker required — all state is in-memory)
- **Message framing:** Uses ASCII control characters (Ctrl-A through Ctrl-E)
- **Routes:** `GET /` (map listing), `websocket /join/:map` (collaboration)

### megadungeon.pl (Dungeon Generator)

- **Framework:** Mojolicious::Lite (Perl)
- **Dependencies:** GD (graphics), MIME::Base64, List::Util
- **Output:** Gridmapper-compatible URLs and PNG step-by-step images
- **Grid:** 30x30 rooms, 30x30 cells, up to 10 levels

## Technology Stack

- **Frontend:** Vanilla JavaScript (ES5-era style), inline SVG, inline CSS
- **Backend:** Perl 5, Mojolicious::Lite
- **No:** npm, Node.js, TypeScript, React, build tools, linters, formatters, CI/CD

## Code Style

The project references the [Google JavaScript Style Guide](https://google.github.io/styleguide/jsguide.html) in a comment at the top of the script section. In practice:

- **ES5 conventions:** `var` declarations, `function` keyword, prototype-style objects (object literals with methods)
- **No modules:** Everything is in the global scope within a single `<script>` block
- **Naming:** camelCase for functions and variables, PascalCase for the main objects (`Map`, `Pen`, `Commands`, `Demo`)
- **Comments:** JSDoc-style `/** */` for major functions, inline `//` comments for implementation notes
- **Quoting:** Single quotes preferred for strings in JavaScript, double quotes in SVG attributes
- **Indentation:** 2 spaces in JavaScript, mixed in SVG/HTML sections
- **Perl style:** `use Modern::Perl` where present; Mojolicious conventions

## Development Workflow

### No Build Step

The SVG file is the application. Edit it directly and open in a browser to test.

### Deployment

```bash
make upload    # rsync gridmapper.svg to production server
```

### Testing

There is no automated test suite. The `.gitignore` references Selenium artifacts from a previous testing setup that has been removed. Testing is manual: open the SVG in a browser and verify behavior.

### Adding a New Tile

1. Add the SVG definition (with `id` and `width` attributes) in the `<defs>` or tile section of `gridmapper.svg`
2. Register variant cycles in `Map.variants` (each variant points to the next in the cycle)
3. Add symbol-to-key mapping in `Map.symbolInit()` if it needs a keyboard shortcut
4. Add documentation entry in the help/legend section at the bottom of the SVG (search for existing `<use xlink:href="#..."/>` patterns)

### Adding Keyboard Shortcuts

Keyboard handling is in `keyHandling()` (line ~2823) via a `switch(key)` block. Key mappings:
- Arrow keys / `h`, `j`, `k`, `l`: Movement (vim-style)
- `u` / `r`: Undo / redo
- `v`: Cycle tile variant
- `?`: Toggle help screen
- Letters map to tile types via `Map.symbolInit()`

## Git Conventions

- Commit messages are short, imperative sentences (e.g., "Fix label deletion", "Fix PNG export")
- No conventional commits prefix (no `fix:`, `feat:`, etc.)
- No branch protection or PR workflow apparent in history
- Commits tend to be small and focused on a single change

## Important Notes

- The main SVG file mixes markup and code. When editing, be careful with XML/SVG validity — unclosed tags or mismatched CDATA sections will break the entire application.
- All JavaScript runs inside a `<script type="application/javascript"><![CDATA[ ... ]]></script>` block.
- All CSS is inside a `<style><![CDATA[ ... ]]></style>` block.
- The application checks `window.location.hostname` to conditionally show wiki save/load features (only on `campaignwiki.org`).
- The collaboration server must run as a single Hypnotoad worker since all client state is held in Perl process memory.
