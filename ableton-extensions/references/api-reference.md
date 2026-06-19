# API Reference — Ableton Extensions SDK

Package: `@ableton-extensions/sdk` (beta, v1.0.0-beta.0, not on npm). Requires Node.js ≥ 22.11.0.

## Object Model Hierarchy

All classes extend `DataModelObject` (which has `.handle: Handle` and static `.className`).

```
DataModelObject
├── Application              (root — access via context.application)
├── Song                     (the current Live Set)
├── Clip
│   ├── MidiClip
│   └── AudioClip
├── Track
│   ├── MidiTrack
│   └── AudioTrack
├── ClipSlot
├── CuePoint
├── TakeLane
├── Chain
│   └── DrumChain
├── ChainMixer
├── Device
│   ├── RackDevice
│   │   └── DrumRack
│   └── Simpler
├── DeviceParameter
├── TrackMixer
├── Sample
└── Scene
```

**Polymorphism**: Resolve with a base class, narrow with `instanceof`. `getObjectFromHandle` throws if the object's type doesn't match the requested class (or if deleted).

## ExtensionContext (returned by `initialize`)

```ts
interface ExtensionContext<Version> {
  application: Application<Version>;
  commands: Commands<Version>;
  environment: Environment<Version>;
  resources: Resources<Version>;
  ui: Ui<Version>;
  getObjectFromHandle<T>(handle: Handle, type: new (...args) => T): T;
  withinTransaction<T>(fn: () => T): T;
}
```

- `getObjectFromHandle` — Objects cached by handle ID (same Live object → same SDK instance). Throws on deleted/wrong-type/unknown.
- `withinTransaction` — Groups mutations into one undo step. Nested transactions collapse into the outermost. **Sync only** — return `Promise.all([...])` for async ops.

### Application

Access root: `context.application.song` → `Song`.

### Environment

- `storageDirectory: string` — persistent across sessions (config, presets, credentials).
- `tempDirectory: string` — temp files (analysis, downloads); may be cleaned between sessions.
- `language: string | undefined` — Live's UI language, uppercase ISO 639-1 (`"EN"`, `"DE"`, `"JA"`).

### Resources

- `importIntoProject(filePath: string): Promise<string>` — copies file into Live project; returns the imported path. Use this returned path (not the original) for clip creation. Runs host-side, so it CAN access files outside the sandbox.
- `renderPreFxAudio(track, startTime, endTime): Promise<string>` — renders pre-effects audio to a WAV in temp dir; returns path.

### Commands

- `registerCommand(id: string, callback: (arg: unknown) => void | Promise<void>): void`

### Ui

- `registerContextMenuAction(scope, label, commandId): Promise<() => Promise<void>>` — returns unregister function.
- `showModalDialog(url: string, width: number, height: number): Promise<string>` — opens webview modal; resolves with the string passed via `close_and_send`.
- `withinProgressDialog(title: string, options: {}, callback: (update, signal) => Promise<void>): Promise<void>`
  - `update(message: string, progress?: number): Promise<void>` — progress is 0–100.
  - `signal: AbortSignal` — check `signal.aborted` or `signal.throwIfAborted()`.

## Song (the Live Set)

Access via `context.application.song`.

**Accessors (get/set)**: `tempo` (number), `name`.

**Accessors (get only)**:

- `tracks: Track[]` — regular tracks (excludes return & main tracks).
- `returnTracks: Track[]`
- `mainTrack: Track`
- `scenes: Scene[]`
- `cuePoints: CuePoint[]`
- `gridQuantization: GridQuantization`, `gridIsTriplet: boolean`
- `rootNote: number` (0–11), `scaleName: string`, `scaleMode: boolean`, `scaleIntervals: number[]`

**Methods** (all async, return Promise):

- `createAudioTrack()`, `createMidiTrack()` — inserted after last selected track, or appended.
- `createScene(index: number)` — 0-based; pass `-1` to append.
- `createCuePoint(time: number)` — time in beats.
- `deleteTrack(track)`, `deleteScene(scene)`, `deleteCuePoint(cuePoint)` — await to ensure processed.
- `duplicateTrack(track)`, `duplicateScene(scene)`.

## Track (base for AudioTrack, MidiTrack)

**Accessors (get/set)**: `name`, `mute`, `solo`, `arm` (all boolean).

**Accessors (get only)**:

- `arrangementClips: Clip[]`
- `clipSlots: ClipSlot[]`
- `devices: Device[]`
- `takeLanes: TakeLane[]`
- `mixer: TrackMixer`
- `groupTrack: Track | null`
- `mutedViaSolo: boolean`

**Methods** (async):

- `clearClipsInRange(startTime, endTime)` — beats; overlapping clips truncated to range edge.
- `createTakeLane()`, `deleteClip(clip)`, `deleteDevice(device)`, `duplicateDevice(device)`.
- `insertDevice(deviceName, index)` — built-in Live devices only (e.g. `"Reverb"`, `"Auto Filter"`).

### AudioTrack adds:

- `createAudioClip(...)` — accepts positional args `(filePath, isWarped?)` OR object form `{ filePath, startTime, duration, isWarped, loopSettings }`.

### MidiTrack adds:

- `createMidiClip(startTime, duration)` — both in beats.

## Clip (base for AudioClip, MidiClip)

**Accessors (get/set)**: `name` (string), `color` (number), `muted` (boolean), `looping` (boolean — enabling on unwarped audio auto-enables warping).

**Accessors (get only)**: `startTime`, `endTime`, `duration`, `startMarker`, `endMarker`, `loopStart`, `loopEnd` (all numbers, in beats).

### AudioClip adds:

- `warpMode: WarpMode` (get/set)
- `gain: number` (get/set)

### MidiClip adds:

- MIDI note access (see API docs for note editing methods).

## ClipSlot

- `createAudioClip(filePath | {filePath, isWarped, loopSettings}, isWarped?): Promise<AudioClip>` — positional or object form.
- `createMidiClip(): Promise<MidiClip>`
- `deleteClip(): Promise<void>`
- `hasClip: boolean` (get)
- `clip: Clip | null` (get)

## Device (base for Simpler, RackDevice)

- `name: string`, `parent` (Track/Chain).
- `parameters: DeviceParameter[]` — control device params.

### RackDevice adds: `chains: Chain[]`. DrumRack extends RackDevice.

### Simpler adds: `sample: Sample`.

## Scene

- `name: string` (get/set)
- `color: number` (get/set)

## Enums

### WarpMode

`Beats | Tones | Texture | Repitch | Complex | ComplexPro`

### GridQuantization

Grid quantization values for the arrangement (see API docs for full list).

## Key Interfaces

### Handle

```ts
interface Handle {
  id: bigint;
}
```

### ArrangementSelection

Passed to commands registered on `AudioTrack.ArrangementSelection` / `MidiTrack.ArrangementSelection` scopes.

```ts
interface ArrangementSelection {
  time_selection_start: number; // beats
  time_selection_end: number; // beats
  selected_lanes: Handle[]; // Track or TakeLane handles
}
```

### ClipSlotSelection

Passed to commands on `ClipSlotSelection` scope.

```ts
interface ClipSlotSelection {
  selected_clip_slots: Handle[];
}
```

### ClipLoopSettings

```ts
interface ClipLoopSettings {
  looping: boolean;
  startMarker: number;
  endMarker: number;
  loopStart: number;
  loopEnd: number;
}
```

## Context Menu Scopes

**Object scopes** (pass `Handle` to command):
`AudioClip`, `MidiClip`, `AudioTrack`, `MidiTrack`, `ClipSlot`, `Scene`, `Simpler`, `Sample`, `DrumRack`.

**Selection scopes** (pass selection object):
`AudioTrack.ArrangementSelection`, `MidiTrack.ArrangementSelection`, `ClipSlotSelection`.

## SDK Entry Point

```ts
import { initialize, type ActivationContext } from "@ableton-extensions/sdk";

export function activate(activation: ActivationContext) {
  const context = initialize(activation, "1.0.0"); // version = manifest minimumApiVersion
  // ... register commands, context menus, etc.
}
```

`initialize` validates host compatibility and returns a typed `ExtensionContext`. The version string must match your manifest's `minimumApiVersion`. Classes are generic over `Version` (e.g. `Track<"1.0.0">`) — usually inferred.
