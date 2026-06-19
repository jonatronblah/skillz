# ableton-extensions Skill

An agent skill for building extensions for **Ableton Live** using the
[`@ableton-extensions/sdk`](https://ableton.github.io/extensions-sdk).

## What This Skill Does

Gives an AI coding agent the knowledge needed to create, run, and package
Ableton Live extensions written in TypeScript/Node.js. It covers:

- The extension lifecycle (`activate` → `initialize` → `context`)
- Manipulating the Live Set (tracks, clips, devices, scenes, MIDI)
- Working with audio — importing files, rendering pre-FX audio
- Custom UIs — modal webview dialogs and progress bars
- Context menus, commands, handles, and transactions (undo grouping)
- The restricted filesystem permission model
- Scaffolding, building (esbuild), and packaging (`.ablx`) via `extensions-cli`

**Out of scope:** real-time audio processing, MIDI routing, control-surface
integration — these belong to Max for Live, not this SDK.

## Structure

This skill follows the [progressive disclosure](https://en.wikipedia.org/wiki/Progressive_disclosure)
pattern: a compact entry point plus detailed references loaded on demand.

```
ableton-extensions/
├── SKILL.md                              Agent-facing instructions (entry point)
├── README.md                             This file (human-facing)
└── references/
    ├── api-reference.md                  Object model, classes, enums, interfaces
    ├── development-workflow.md           Setup, scaffolding, build, packaging, debugging
    └── patterns-and-examples.md          Working code patterns from the SDK examples
```

| File              | Audience | When it's loaded                             |
| ----------------- | -------- | -------------------------------------------- |
| `SKILL.md`        | Agent    | When the skill triggers (always, ~120 lines) |
| `references/*.md` | Agent    | On demand, when deeper detail is needed      |
| `README.md`       | Human    | When browsing the skill in the repo          |

## Skill Metadata

```yaml
name: ableton-extensions
description: Build extensions for Ableton Live using the @ableton-extensions/sdk.
```

The `description` controls when the agent invokes this skill. Trigger terms
include: Ableton extension, Live SDK, `@ableton-extensions/sdk`, context menu,
webview dialog, `manifest.json`, `.ablx`, warp mode, arrangement selection,
render audio, import into project.

## Source Material

This skill was synthesized from the SDK distribution zip contents:

- `ableton-extensions-sdk-*` — the TypeScript SDK package
- `ableton-extensions-cli-*` — the `extensions-cli` (run / package)
- `ableton-create-extension-*` — the project scaffolder
- `docs/` — the rendered HTML documentation
- `examples/` — working example extensions (context-menu, modal-dialog,
  progress-dialog, audio-clips, warpMode, arrangementselection, strip-silence)
- `api/` — the generated TypeDoc API reference

## Maintenance Notes

- The SDK is in **beta** (`1.0.0-beta.0`) and **not published to npm** — it is
  distributed as a zip from Centercode with vendored `.tgz` packages. Update the
  version strings in `SKILL.md` and `references/development-workflow.md` when a
  new beta is released.
- `SKILL.md` must stay under **200 lines** (currently ~120). Push detail into
  `references/` rather than growing the entry point.
- Reference files should be kept under ~250 lines where practical; split further
  if they grow.
- When the upstream docs or API change, regenerate the affected reference file
  from the corresponding `docs/**/*.html` (convert to markdown) or `api/` page.
