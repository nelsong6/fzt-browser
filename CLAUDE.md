# fzt-browser

Browser renderer for fzt. Compiles the Go engine to WASM and bundles the JavaScript/CSS shim that talks to it. Consumed by `my-homepage` and `fzt-showcase`.

## Contents

- `main.go` — Go-side WASM entry point (`//go:build js && wasm`). Imports `fzt/core` and `fzt-terminal/tui` for headless session rendering. Exposes a `fzt` global on window.
- `fzt-terminal.js` — Core browser terminal: ANSI parser, grid renderer (styled HTML spans), font metrics, keyboard forwarding, resize observer. Factory: `createFztTerminal(container, options)`.
- `fzt-web.js` — Higher-level wrapper with Catppuccin Mocha defaults (palette, Perfect DOS VGA 437 font, Nerd Font icons). Factory: `createFztWeb(container, options)`.
- `fzt-dom-renderer.js` — Alternative DOM renderer using structured data API (`getVisibleRows`, `getPromptState`, `getUIState`) instead of ANSI parsing. Renders native DOM elements with CSS classes.
- `fzt-terminal.css` — CRT terminal styles: scanline overlay, vignette, cursor blink, custom properties (`--fzt-*`).

## WASM API

The Go code exposes a global `fzt` object:

- `fzt.loadYAML(yaml)` — parse YAML items
- `fzt.setFrontend({name, version})` — register frontend identity
- `fzt.addCommands([{name, description, action}])` — register frontend commands for `:` palette
- `fzt.setLabel(text)` — set the top-left border label
- `fzt.init(cols, rows)` — create session, returns `{ansi, cursorX, cursorY}`
- `fzt.handleKey(key, ctrl, shift)` — process keyboard event, returns frame + action
- `fzt.clickRow(row)` — process mouse click on visual row
- `fzt.resize(cols, rows)` — resize terminal
- `fzt.getVisibleRows()` / `fzt.getPromptState()` / `fzt.getUIState()` — structured data API for the DOM renderer

`EnvTags: ["wasm", "browser"]` is set on the session config so browser-inappropriate commands (e.g. the terminal-only `update`) are filtered out of the palette.

## Dependencies

- `github.com/nelsong6/fzt` — engine (state, scoring, tree logic, YAML parsing, render abstractions)
- `github.com/nelsong6/fzt-terminal` — headless session + palette injection via tui (which in turn pulls in fzt-frontend)
- `github.com/gdamore/tcell/v2` — terminal screen library (used by the headless session)

## CI

Three workflows, split by responsibility:

- **`.github/workflows/dispatch.yml`** (fires on `repository_dispatch: [fzt-updated]`) — upstream cascade. Calls `go-dependency-update-template`: bumps `github.com/nelsong6/fzt` + `github.com/nelsong6/fzt-terminal` in go.mod, commits, pushes, then delegates to `release-and-dispatch-template` to cut the release and dispatch downstream.
- **`.github/workflows/release.yml`** (fires on `push: branches: [main]` with `paths-ignore: [go.mod, go.sum]`, plus `workflow_dispatch`) — direct-push release path for code changes. Calls `release-and-dispatch-template` directly. Shared `release-tag` concurrency group with `dispatch.yml`.
- **`.github/workflows/build.yml`** (fires on `workflow_dispatch` from the release template with the new tag as input) — compiles `fzt.wasm` via `GOOS=js GOARCH=wasm` (injecting `render.Version` and `frontend.EngineVersion` via ldflags), packages the four JS/CSS files, and `gh release upload`s them to the already-created release. Tag is passed by value so no sha-drift race.

Downstream consumers notified via `repository_dispatch`: `fzt-showcase` and `my-homepage` (both download the release assets at deploy time). GitHub App token from Azure Key Vault. Template lives at `nelsong6/pipeline-templates`.

## History

Extracted from `nelsong6/fzt-terminal` on 2026-04-16 as part of the fzt repo split. Was `fzt-terminal/web/` + `fzt-terminal/cmd/wasm/`. The grammar goal: each fzt repo names exactly one thing.
