---
name: ableton-extensions
description: "Build extensions for Ableton Live using the @ableton-extensions/sdk. WHEN: Ableton extension, Ableton Live plugin, Live SDK, manipulate Live Set, create clips/tracks, context menu, webview dialog, @ableton-extensions/sdk, extensions-cli, manifest.json, activate function, .ablx packaging, warp mode, arrangement selection, render audio, import into project."
---

# Ableton Extensions SDK

Build extensions for Ableton Live with TypeScript/Node.js. Extensions run in a Node.js process alongside Live, providing programmatic access to the Live Set, custom UIs, and the npm ecosystem.

## What You Can Build

- **Manipulate the Live Set** — batch rename clips, create/delete scenes & tracks, edit MIDI notes, change device parameters.
- **Work with audio & files** — import audio, render pre-FX audio from arrangement, transform audio offline.
- **Custom UIs** — modal dialogs (webviews) built with HTML/CSS/JS, progress bars for long tasks.
- **Integrate npm** — full Node.js + npm access (fetch APIs, process files, etc.).

**Not suited for:** real-time audio processing, MIDI routing/manipulation, drawing into Live's native UI, background/persistent processes, control surface integration. Use Max for Live for these.

## Core Architecture

Every extension exports an `activate` function. Call `initialize` to get the `context` object — your gateway to all SDK functionality.

```ts
import { initialize, type ActivationContext } from "@ableton-extensions/sdk";

export function activate(activation: ActivationContext) {
  const context = initialize(activation, "1.0.0");
  const { tempo } = context.application.song;
  console.log(`Tempo: ${tempo} bpm`);
}
```

## The `context` Object (ExtensionContext)

| Service                       | Purpose                                                                          |
| ----------------------------- | -------------------------------------------------------------------------------- |
| `context.application`         | Root of model — access `application.song` (tracks, clips, scenes, tempo, scale). |
| `context.commands`            | `registerCommand(id, callback)` — named actions triggered by UI or code.         |
| `context.ui`                  | `registerContextMenuAction`, `showModalDialog`, `withinProgressDialog`.          |
| `context.resources`           | `importIntoProject(path)`, `renderPreFxAudio(track, start, end)`.                |
| `context.environment`         | `storageDirectory`, `tempDirectory`, `language` (persistent vs temp files).      |
| `context.getObjectFromHandle` | Resolve a `Handle` into a typed SDK object (e.g. `Track`, `Clip`).               |
| `context.withinTransaction`   | Group multiple mutations into one undo step (sync callback).                     |

## Key Concepts

**Handles** — Lightweight IDs (`{ id: bigint }`) referencing Live objects. Resolve via `context.getObjectFromHandle(handle, Class)`. Handles become invalid on deletion, move, or session change — don't cache long-term.

**Polymorphism** — Use base classes to write generic code, narrow with `instanceof`:

- `Track` → `AudioTrack`, `MidiTrack`
- `Clip` → `AudioClip`, `MidiClip`
- `Device` → `Simpler`, `RackDevice` → `DrumRack`
- `DataModelObject` — base for all objects.

**Transactions** — Group mutations into one undo step. Callback is **synchronous**; for async ops (create clips/tracks), return `Promise.all([...])` and await the transaction call. Cannot create-then-modify in the same transaction.

**Commands & Context Menus** — Register a command, then bind it to a right-click scope:

```ts
context.commands.registerCommand("my-ext.action", (handle) => {
  const clip = context.getObjectFromHandle(handle as Handle, Clip);
  clip.name = "Renamed";
});
context.ui.registerContextMenuAction("AudioClip", "My Action", "my-ext.action");
```

Object scopes: `AudioClip`, `MidiClip`, `AudioTrack`, `MidiTrack`, `ClipSlot`, `Scene`, `Simpler`, `Sample`, `DrumRack`.
Selection scopes: `AudioTrack.ArrangementSelection`, `MidiTrack.ArrangementSelection`, `ClipSlotSelection`.

## Filesystem (Restricted!)

Only access `context.environment.storageDirectory` (persistent) and `tempDirectory` (temp). **Never** access arbitrary paths (Documents, Desktop). Child processes & native addons must respect the same limits. To bring an outside file in, use `context.resources.importIntoProject(path)`.

## Project Structure

```
├── .env                 EXTENSION_HOST_PATH=… (gitignored)
├── manifest.json        name, author, entry, version, minimumApiVersion
├── package.json         scripts: start, build, package
├── build.ts             esbuild config (bundle → dist/extension.js)
├── src/extension.ts     exports activate(activation)
└── vendor/              .tgz SDK/CLI packages (beta — not on npm)
```

**manifest.json**: `{ "name", "author", "entry": "dist/extension.js", "version", "minimumApiVersion": "1.0.0" }`

## Development Commands

The SDK is **beta, not on npm** — distributed as a zip with vendored `.tgz` packages.

```sh
npx file:/path/to/ableton-create-extension-1.0.0-beta.0.tgz  # scaffold
npm start        # build:dev + extensions-cli run (needs Live Dev Mode ON)
npm run build    # production bundle (minified, no sourcemaps)
npm run package  # build + .ablx archive (install via drag-drop into Live)
```

CLI direct: `npx extensions-cli run --live "<Live path>" [--inspect] [--storage-directory <p>] [--temp-directory <p>]`

**Dev cycle**: Enable Developer Mode in Live's Preferences → Extensions. Run `npm start`, edit, re-run `npm start` (no Live restart needed).

**Logs**: `console.*` output goes to `ExtensionHost.txt`:

- Windows: `%APPDATA%\Ableton\Live x.x.x\Preferences\ExtensionHost.txt`
- macOS: `~/Library/Preferences/Ableton/Live x.x.x/ExtensionHost.txt`

## References (load as needed)

- `references/api-reference.md` — Full object model, class accessors/methods, enums, interfaces, ExtensionContext detail.
- `references/development-workflow.md` — Setup prerequisites, scaffolding, build config, packaging, debugging.
- `references/patterns-and-examples.md` — Working code: context menus, handles/polymorphism, transactions, progress dialogs, modal webviews, resources/filesystem.

## Critical Rules

1. Always call `initialize(activation, "<apiVersion>")` first — passing the `minimumApiVersion` from manifest.
2. Command callbacks receive `unknown` — cast: `(arg as Handle)` or `(arg as ArrangementSelection)`.
3. `withinTransaction` callback must be synchronous; return `Promise.all` for async ops.
4. Resolve handles as-needed, never cache SDK objects long-term.
5. Stay within `storageDirectory` / `tempDirectory` for filesystem access.
6. Bundle to a single JS file (`format: "cjs"`, `platform: "node"`) — Live won't resolve `node_modules` at runtime.
7. For HTML imports in TS, configure esbuild with `.html` loader (text) — see modal-dialog pattern.
