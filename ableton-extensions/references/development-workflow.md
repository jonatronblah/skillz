# Development Workflow — Ableton Extensions SDK

## Prerequisites

- **Node.js ≥ 24.14.1** (for CLI; SDK requires ≥ 22.11.0)
- **Ableton Live** — the Live Beta build that supports Extensions (from Centercode)
- **SDK distribution zip** — `extensions-sdk-<version>.zip` (from Centercode; SDK is beta, **not on npm**)
- **VS Code** recommended (gets `.vscode/launch.json` + `tasks.json` for F5 debugging)

## Scaffolding a New Extension

The SDK is distributed as a zip containing vendored `.tgz` packages and a project creator. From an empty folder:

```sh
mkdir my-ext && cd my-ext
npx file:/path/to/extracted/ableton-create-extension-1.0.0-beta.0.tgz
```

Prompts:

- **Extension name** — package name + shown in Live.
- **Author** — name or org.
- **Ableton Live application** — auto-detected; pick or enter custom path. Only Extension-capable installs are listed.
- **UI?** — yes adds a Vite webview scaffold (for modal dialogs).

The creator writes `EXTENSION_HOST_PATH` to a gitignored `.env`, runs `npm install`, and if VS Code is detected, adds `.vscode/launch.json` + `tasks.json`.

### Resulting Structure

```
├── .env              EXTENSION_HOST_PATH=… (gitignored)
├── .gitignore
├── manifest.json     name, author, entry, version, minimumApiVersion
├── package.json      scripts: start, build, package
├── build.ts          esbuild config
├── tsconfig.json
├── src/extension.ts  exports activate(activation)
├── ui/               (only if UI scaffolded)
├── vendor/           ableton-extensions-*.tgz
└── node_modules/
```

### manifest.json

```json
{
  "name": "my-extension",
  "author": "Your Name",
  "entry": "dist/extension.js",
  "version": "1.0.0",
  "minimumApiVersion": "1.0.0"
}
```

### package.json scripts

```json
{
  "type": "module",
  "scripts": {
    "build": "tsc --noEmit && tsx build.ts --production",
    "build:dev": "tsc --noEmit && tsx build.ts",
    "start": "npm run build:dev && extensions-cli run",
    "package": "npm run build && extensions-cli package"
  },
  "dependencies": {
    "@ableton-extensions/sdk": "file:vendor/ableton-extensions-sdk-*.tgz"
  },
  "devDependencies": {
    "@ableton-extensions/cli": "file:vendor/ableton-extensions-cli-*.tgz",
    "esbuild": "0.28.0",
    "tsx": "^4.19.0",
    "typescript": "^5.9.3"
  }
}
```

## Running Examples

Examples ship in `examples/` and reference vendored `.tgz` by relative path — **run them in place** (don't copy elsewhere or `npm install` breaks). They have no `.env`, so pass `--live`:

```sh
cd examples/context-menu
npm install
npm start -- --live "/Applications/Ableton Live Beta.app"          # macOS
npm start -- --live "C:\ProgramData\Ableton\Live Beta\Program\Ableton Live Beta.exe"  # Windows
```

## Build Configuration

`build.ts` uses esbuild. Output must be a **single CJS file** — Live won't resolve `node_modules` at runtime.

```ts
import * as esbuild from "esbuild";
import * as fs from "node:fs";

const manifest = JSON.parse(fs.readFileSync("manifest.json", "utf8"));
const production = process.argv.includes("--production");

await esbuild.build({
  entryPoints: ["src/extension.ts"],
  outfile: manifest.entry, // dist/extension.js
  bundle: true,
  format: "cjs",
  platform: "node",
  sourcesContent: false,
  minify: production,
  sourcemap: !production,
  // For HTML imports (webviews): add loader
  // loader: { ".html": "text" },
});
```

For HTML imports in TS, declare a module:

```ts
// html.d.ts
declare module "*.html" {
  const content: string;
  export default content;
}
```

A different bundler can replace esbuild — `extensions-cli` only cares about the `entry` in manifest.json.

## Development Cycle (No Live Restart)

1. Open Live → **Preferences → Extensions → enable Developer Mode**. (Required — without it, `npm start` can't connect.)
2. Run `npm start` in your project — builds dev bundle + launches Extension Host.
3. Edit code → re-run `npm start` to reload (no Live restart needed).

## CLI: `extensions-cli`

```sh
extensions-cli run [dir] [options]
extensions-cli package [dir] [options]
```

**run options:**

- `--live <path>` — Live path (.app/.exe/install root/`ExtensionHostNodeModule.node`). Overrides `EXTENSION_HOST_PATH`.
- `--storage-directory <path>` — overrides `context.environment.storageDirectory`.
- `--temp-directory <path>` — overrides `context.environment.tempDirectory`.
- `--inspect` — enable VS Code debugging (`--inspect-brk`).

`run` reads `EXTENSION_HOST_PATH` from env or `.env` in the extension dir.

**package options:**

- `-o, --output <path>` — custom output path.
- `-i, --include <p...>` — bundle extra assets alongside entry (files or dirs, recursive; must be relative to extension dir, can't escape it).

`package` does **not** run your build step — always build first. Output is a `.ablx` archive.

## Packaging & Distribution

1. `npm run package` → builds production + creates `.ablx`.
2. Users install by **dragging the `.ablx` into Live's Extensions settings page**.

A packaged extension = single JS file + `manifest.json` + any explicitly included assets.

## Debugging & Logging

The Extension Host writes a log file with host info, your `console.{log,error,info,warn}` output, and uncaught exception stack traces:

- **Windows**: `%APPDATA%\Ableton\Live x.x.x\Preferences\ExtensionHost.txt`
  (`\Users\<user>\AppData\Roaming\Ableton\Live x.x.x\Preferences\ExtensionHost.txt`)
- **macOS**: `~/Library/Preferences/Ableton/Live x.x.x/ExtensionHost.txt`

For step-debugging, run `npx extensions-cli run --inspect` and attach VS Code's debugger (the project creator adds a launch config).

## Troubleshooting

| Symptom                             | Fix                                                                                         |
| ----------------------------------- | ------------------------------------------------------------------------------------------- |
| `npm start` doesn't connect to Live | Enable Developer Mode in Preferences → Extensions.                                          |
| No output / not loading             | Check `.env` has valid `EXTENSION_HOST_PATH`; edit if Live moved.                           |
| Wrong API behavior                  | Ensure `initialize(activation, "<version>")` matches manifest `minimumApiVersion`.          |
| Handle throws on resolve            | Object was deleted/moved/session changed. Re-query; don't cache.                            |
| File access fails                   | Only `storageDirectory`/`tempDirectory` allowed. Use `importIntoProject` for outside files. |
| Async inside transaction fails      | `withinTransaction` is sync-only — return `Promise.all`, await outside.                     |
