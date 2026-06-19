# Patterns & Examples — Ableton Extensions SDK

Working code patterns drawn from the SDK docs and bundled examples. Import style may vary (`import * as ableton` vs named imports) — both work.

## 1. Minimal Extension (scaffold default)

```ts
import { initialize, type ActivationContext } from "@ableton-extensions/sdk";

export function activate(activation: ActivationContext) {
  const context = initialize(activation, "1.0.0");
  const { tempo } = context.application.song;
  console.log(`Hello! Your Live Set's tempo is: ${tempo} bpm.`);
}
```

## 2. Context Menu + Command (most common pattern)

Register a command, then bind it to a right-click scope. The command receives a `Handle`.

```ts
import {
  initialize,
  Clip,
  type ActivationContext,
  type Handle,
} from "@ableton-extensions/sdk";

export function activate(activation: ActivationContext) {
  const context = initialize(activation, "1.0.0");

  context.commands.registerCommand("my-ext.rename-clip", (arg: unknown) => {
    const clip = context.getObjectFromHandle(arg as Handle, Clip);
    clip.name = "Processed Clip";
  });

  context.ui.registerContextMenuAction(
    "AudioClip", // scope
    "Rename Clip", // label
    "my-ext.rename-clip", // command id
  );
}
```

**Object scopes** (pass `Handle`): `AudioClip`, `MidiClip`, `AudioTrack`, `MidiTrack`, `ClipSlot`, `Scene`, `Simpler`, `Sample`, `DrumRack`.

Register the same command on multiple scopes:

```ts
(["MidiClip", "AudioClip"] as const).forEach((scope) =>
  context.ui.registerContextMenuAction(scope, "Rename", "my-ext.rename"),
);
```

## 3. Resolving Handles & Polymorphism

Resolve with the specific type when known, or a base class + `instanceof` when unknown.

```ts
// Specific type (command bound to AudioClip scope)
const clip = context.getObjectFromHandle(handle as Handle, AudioClip);
clip.warpMode = WarpMode.Complex;

// Base class for generic handling
const clip = context.getObjectFromHandle(handle as Handle, Clip); // audio or midi
clip.name = "Renamed";

// Most generic — then narrow
const obj = context.getObjectFromHandle(handle as Handle, DataModelObject);
if (obj instanceof ClipSlot) {
  /* ... */
} else if (obj instanceof Track) {
  /* ... */
}
```

## 4. Arrangement Selection (multi-track time range)

Selection scopes pass an `ArrangementSelection` (not a handle) containing beats and lane handles.

```ts
import {
  initialize,
  Track,
  MidiTrack,
  DataModelObject,
  TakeLane,
  type ActivationContext,
  type ArrangementSelection,
} from "@ableton-extensions/sdk";

export function activate(activation: ActivationContext) {
  const api = initialize(activation, "1.0.0");

  api.commands.registerCommand(
    "my-ext.process-selection",
    async (arg: unknown) => {
      const selection = arg as ArrangementSelection;
      const { time_selection_start, time_selection_end, selected_lanes } =
        selection;

      // Resolve lanes; filter to Tracks and TakeLanes
      const objs = selected_lanes.map((h) =>
        api.getObjectFromHandle(h, DataModelObject),
      );
      const lanes = objs.filter(
        (o): o is Track<"1.0.0"> | TakeLane<"1.0.0"> =>
          o instanceof Track || o instanceof TakeLane,
      );

      // Narrow to MIDI lanes (Track or TakeLane under a MidiTrack)
      const midiLanes = lanes.filter(
        (o) =>
          o instanceof MidiTrack ||
          (o instanceof TakeLane && o.parent instanceof MidiTrack),
      );

      // Create clips across the selected range
      const clips = await Promise.all(
        midiLanes.map((lane) =>
          lane.createMidiClip(
            time_selection_start,
            time_selection_end - time_selection_start,
          ),
        ),
      );
      clips.forEach((c, i) => (c.name = `New Clip ${i + 1}`));
    },
  );

  api.ui.registerContextMenuAction(
    "MidiTrack.ArrangementSelection",
    "Process selection",
    "my-ext.process-selection",
  );
}
```

## 5. Transactions (grouping undo steps)

`withinTransaction` is **synchronous**. Group sync mutations directly; for async ops (create tracks/clips), return `Promise.all` and await the call.

```ts
// Sync mutations — one undo step
context.withinTransaction(() => {
  context.application.song.tracks.forEach(
    (t, i) => (t.name = `Track ${i + 1}`),
  );
});

// Async ops — return Promise.all, await outside
const newTracks = await context.withinTransaction(() =>
  Promise.all([
    context.application.song.createAudioTrack(),
    context.application.song.createMidiTrack(),
  ]),
);

// CANNOT create-then-modify in same transaction (need the instance first).
// Chain as separate transactions:
//   1. await withinTransaction(() => Promise.all([create...]))
//   2. withinTransaction(() => { modify created... })
```

## 6. Progress Dialog (long-running tasks)

Shows Live's standard progress bar. User is blocked from UI while open. `update(msg, pct)` is async.

```ts
context.commands.registerCommand("my-ext.long-task", () => {
  void context.ui.withinProgressDialog(
    "Doing work",
    {},
    async (update, signal) => {
      await update("Starting", 0);
      for (let i = 0; i < 100; i++) {
        await delay(100);
        await update(`Progress ${i}%`, i);
        signal.throwIfAborted(); // respects cancel
      }
      await update("Done", 100);
    },
  );
});
```

**Combine progress + transaction** (common for analyze-then-modify): do async work in the progress callback, wrap final mutations in a transaction, and `await Promise.all` the transaction's promises before the dialog closes:

```ts
await context.ui.withinProgressDialog("Analyzing...", {}, async (update) => {
  const ranges = await analyzeAudio();
  await update("Applying", 90);
  const mods = context.withinTransaction(() =>
    ranges.map((r) => track.clearClipsInRange(r.start, r.end)),
  );
  await Promise.all(mods); // ensure done before dialog closes
});
```

## 7. Modal Dialog (Webview)

Show custom HTML UI; receive result via `postMessage`. Pass HTML as a Data URL.

```ts
import modalInterface from "./interface.html"; // esbuild inlines as text

context.commands.registerCommand("my-ext.dialog", () => {
  void (async () => {
    const url = `data:text/html,${encodeURIComponent(modalInterface)}`;
    const result = await context.ui.showModalDialog(url, 360, 240);
    const data = JSON.parse(result); // whatever you sent via close_and_send
    console.log(data.name);
  })();
});
```

### HTML boilerplate (platform-aware messaging)

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <script>
      function sendMessage(message) {
        if (window.webkit?.messageHandlers?.live) {
          window.webkit.messageHandlers.live.postMessage(message); // macOS
        } else if (window.chrome?.webview) {
          window.chrome.webview.postMessage(message); // Windows
        }
      }
      function closeWithResult(result) {
        sendMessage({
          method: "close_and_send",
          params: [JSON.stringify(result)],
        });
      }
    </script>
    <style>
      :root {
        --ableton-bg: #383838;
        --ableton-panel: #4e4e4e;
        --ableton-button: #ffa500;
        --ableton-text: #ffffff;
        --ableton-border: #2c2c2c;
        --ableton-input-bg: #2c2c2c;
      }
      body {
        background: var(--ableton-panel);
        color: var(--ableton-text);
        font-family: sans-serif;
        margin: 0;
        height: 100vh;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
      }
      button {
        background: var(--ableton-button);
        border: none;
        padding: 8px 16px;
        cursor: pointer;
        color: #000;
      }
    </style>
  </head>
  <body>
    <h1>Extension UI</h1>
    <button onclick="closeWithResult({ status: 'success' })">
      Close & Return
    </button>
  </body>
</html>
```

**Passing data INTO the webview**: inline it into the HTML string before encoding, or append query params to the Data URL and consume in-page.

### esbuild config for HTML imports

```ts
// build.ts — add loader + a html.d.ts module declaration
await esbuild.build({
  // ... standard options
  loader: { ".html": "text" },
});
// html.d.ts: declare module "*.html" { const content: string; export default content; }
```

## 8. Resources & Filesystem

### Import a file into the project (required before creating audio clips)

```ts
const imported =
  await context.resources.importIntoProject("/path/to/audio.wav");
await clipSlot.createAudioClip(imported, false);
// OR object form with loop settings:
await clipSlot.createAudioClip({
  filePath: imported,
  isWarped: true,
  loopSettings: {
    looping: true,
    startMarker: 1,
    endMarker: 5,
    loopStart: 1,
    loopEnd: 5,
  },
});
```

### Download from API → temp → import → clip

```ts
const res = await fetch("https://api.example.com/audio");
const buf = Buffer.from(await res.arrayBuffer());
const tempPath = path.join(context.environment.tempDirectory, "dl.wav");
await fs.writeFile(tempPath, buf);
const imported = await context.resources.importIntoProject(tempPath);
await track.createAudioClip({
  filePath: imported,
  startTime: 0,
  duration: 4,
  isWarped: false,
});
```

### Render pre-FX audio from arrangement

```ts
const wavPath = await context.resources.renderPreFxAudio(
  track,
  startBeat,
  endBeat,
);
const data = await fs.readFile(wavPath); // WAV in temp dir
```

### Persist config across sessions

```ts
const configPath = path.join(
  context.environment.storageDirectory,
  "config.json",
);
await fs.writeFile(configPath, JSON.stringify({ apiKey: "..." }));
const config = JSON.parse(await fs.readFile(configPath, "utf-8"));
```

## 9. Warp Mode Toggle

```ts
import {
  AudioClip,
  Clip,
  WarpMode,
  initialize,
  type Handle,
} from "@ableton-extensions/sdk";

context.commands.registerCommand("my-ext.cycle-warp", (arg: unknown) => {
  const clip = context.getObjectFromHandle(arg as Handle, Clip);
  if (!(clip instanceof AudioClip)) {
    console.error("Not an AudioClip.");
    return;
  }
  const modes = [
    WarpMode.Beats,
    WarpMode.Tones,
    WarpMode.Texture,
    WarpMode.Repitch,
    WarpMode.Complex,
    WarpMode.ComplexPro,
  ];
  clip.warpMode = modes[(modes.indexOf(clip.warpMode) + 1) % modes.length];
});
context.ui.registerContextMenuAction(
  "AudioClip",
  "Cycle Warp Mode",
  "my-ext.cycle-warp",
);
```

## 10. Error Handling in Async Commands

Command callbacks that do async work should catch and log — uncaught rejections go to the host log.

```ts
context.commands.registerCommand(
  "my-ext.async",
  (arg: unknown) =>
    void (async () => {
      try {
        // ... async work
      } catch (e) {
        console.error(e);
      }
    })(arg as Handle),
);
```

## Design Notes (for webview UIs)

- Use visual hierarchy for sonic importance; prominent controls for high-impact params.
- Group related params; vary control types to avoid mechanical feel.
- Match Ableton conventions: yellow/blue for interactivity, "Cancel" consistently labeled.
- Buttons: verb labels, single words best, same width within a group. Use `checkbox`/`toggle` for on/off (not buttons).
- ≤5 options + see-at-once → `radio` group; ≥4 or long labels → `select`.
- Sliders for ranges/relative position; avoid for 2-step discrete (use radio).
- Meet WCAG contrast for custom colors. Prefer `:disabled` over hiding controls.
- Extensions can't process real-time audio — use visualization to preview output.
