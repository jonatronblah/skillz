---
name: ableton-js
description: "Control Ableton Live from Node.js/TypeScript via the ableton-js library and the Live MIDI Remote Script API. WHEN: ableton-js, Ableton Live Node, control Live from script, Live remote API, Live MIDI Remote Script, automate Ableton, get/set tempo, start/stop playback, manipulate tracks/clips/scenes, create MIDI clips, edit notes, fire clips, device parameters, mixer volume/pan/sends, listen to Live changes, AbletonJS plugin, Live API from Node, build Ableton controller, Live setlist automation, ableton-js get set addListener."
---

# ableton-js — Control Ableton Live from Node.js

`ableton-js` is a Node.js/TypeScript library that exposes Ableton Live's [MIDI Remote Script](https://nsuspray.github.io/Live_API_Doc/11.0.0.xml) (the "Live API") over a local UDP link. You create one `Ableton` instance in a Node process, connect it to a running copy of Live, and then drive nearly everything in a Live Set programmatically: playback, tempo, tracks, clips, scenes, devices, mixer, browser, MIDI notes, and more.

Use this skill whenever code must talk to Live — build a setlist manager, a custom controller, an auto-arranger, a clip generator, a parameter LFO, or any automation that reads/changes Live state. It is **not** for real-time audio DSP, drawing into Live's native UI, or running inside Live (for in-Live extensions see the `ableton-extensions` skill; for audio DSP use Max for Live).

The full Live API object model (every property/function Ableton exposes) is documented at <https://nsuspray.github.io/Live_API_Doc/11.0.0.xml>. ableton-js mirrors that model as typed TypeScript namespaces.

## How it works in one paragraph

A Python script (`AbletonJS`) runs **inside** Live as a MIDI Remote Script control surface. It listens on a local UDP port. Your Node app (`ableton-js`) binds its own UDP port, and the two find each other via small port files in the OS temp dir. Every interaction is a JSON command (`get_prop`, `set_prop`, `add_listener`, or a function name) tagged with a UUID; the script replies with a JSON result. ableton-js wraps all of this in typed, async classes so you almost never touch raw JSON.

## Installation & first connection

Two parts must be in place: the **Python script inside Live**, and the **npm package** in your project.

**1. Install the Python Remote Script into Live.** Copy the `midi-script/` folder from the ableton-js repo into Live's Remote Scripts folder and **rename it to `AbletonJS`**:

- macOS: `~/Music/Ableton/User Library/Remote Scripts`
- Windows: `Users\<you>\Music\Ableton\User Library\Remote Scripts`

Then launch Live → **Preferences → Link/Tempo/MIDI → Control Surfaces → Add** → choose **AbletonJS**. (If you cloned the repo on macOS, `yarn ableton12:start` copies the script, launches Live, and tails the log for you.)

**2. Install the npm package.** Requires Node ≥ 20.

```sh
npm install ableton-js
```

**3. Connect.**

```typescript
import { Ableton } from "ableton-js";

const ableton = new Ableton({ logger: console });
await ableton.start(); // binds the socket and waits until Live is reachable
const tempo = await ableton.song.get("tempo");
console.log("Tempo:", tempo);
```

If `start()` hangs, the script isn't running in Live (re-add the AbletonJS control surface) or the port files in the temp dir are stale. See `references/setup-and-protocol.md`.

## The core object model

An `Ableton` instance exposes a tree of **namespaces** that mirror Live's object hierarchy. Each namespace is a typed class with `get`, `set`, `addListener`, plus its own methods.

```
ableton
├── song                         → Song  (the open Live Set)
│   ├── view                     → SongView  (selection, detail clip)
│   ├── tracks[] / return_tracks[] / master_track   → Track
│   ├── scenes[]                 → Scene
│   └── cue_points[]             → CuePoint
├── application                  → Application  (Live app version, dialog buttons)
│   ├── view                     → ApplicationView  (focus/show/zoom views)
│   └── browser                  → Browser  (instruments, samples, load/preview)
├── session                      → Session  (session ring box for control surfaces)
├── midi                         → Midi  (send/receive raw MIDI bytes)
└── internal                     → Internal  (ping, plugin version)
```

Object namespaces are reached **through** the tree. From a `Track` you get its `devices` (→ `Device[]`), each device's `parameters` (→ `DeviceParameter[]`), the track's `mixer_device` (→ `MixerDevice` with `volume`/`panning`/`sends`), its `clip_slots` (→ `ClipSlot[]`), and each slot's `clip` (→ `Clip` with MIDI notes). Don't memorize it — use TypeScript's IntelliSense to explore from `ableton.song`.

## The Namespace pattern (read this once)

Every Live object class (`Song`, `Track`, `Clip`, `Device`, ...) extends a generic `Namespace` base class parameterized by **four interfaces**:

| Interface               | Used by             | Meaning                                                                                                                                                                                                                    |
| ----------------------- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GettableProperties`    | `get(prop)`         | Props you can read. Returns the _raw_ type.                                                                                                                                                                                |
| `TransformedProperties` | `get(prop)` (auto)  | Subset of gettable props whose raw value is auto-wrapped into a richer object (e.g. a track list becomes `Track[]`, a color int becomes a `Color`). You get the _transformed_ type automatically — never wrap it yourself. |
| `SettableProperties`    | `set(prop, value)`  | Props you can write.                                                                                                                                                                                                       |
| `ObservableProperties`  | `addListener(prop)` | Props Live will push change events for.                                                                                                                                                                                    |

So the four operations on any object are always:

```ts
const value = await obj.get("prop"); // read
await obj.set("prop", value); // write
const off = await obj.addListener("prop", (v) => {}); // observe (returns an unsubscribe fn)
const data = await obj.sendCommand("function_name", { arg: 1 }); // call a Live function
```

**`get` is the right starting point for everything.** Namespaces with a `.raw` field (`Track`, `Clip`, `Scene`, `Device`, `DeviceParameter`, `MixerDevice`, `CuePoint`, `ClipSlot`, `Chain`, `DrumPad`, `BrowserItem`) are concrete objects that hold a stable `.raw.id` identifying that exact object in Live. Pass them around by value — the `.id` is what the protocol keys on.

## The most useful namespaces at a glance

- **`Song`** — playback (`startPlaying`, `stopPlaying`, `is_playing`), `tempo`, `signature_numerator/denominator`, `loop`, create/delete/duplicate tracks & scenes, `undo`/`redo`, `beginUndoStep`/`endUndoStep`, `getData`/`setData` (free-form key/value persisted in the Set).
- **`Track`** — `name`, `color`, `mute`, `solo`, `arm`, `volume`/`pan` via `mixer_device`, `devices[]`, `clip_slots[]`, `arrangement_clips[]`, routing, `createMidiClip`/`createAudioClip`/`duplicateClipToArrangement`.
- **`Clip`** (Session/Arrangement) — `name`, `color`, `is_playing`, `fire`/`stop`, loop points, warp mode, and the full **MIDI note** API: `getNotesExtended`, `setNotes`, `addNewNotes`/`replaceSelectedNotes`, `removeNotesExtended`, `quantize`, `applyNoteModifications`.
- **`Scene`** — `name`, `tempo`, `fire()` (launches the whole scene row).
- **`ClipSlot`** — `has_clip`, `is_playing`, `createClip`, `duplicateClipTo`, `deleteClip`.
- **`Device` + `DeviceParameter`** — `parameters[]`, each parameter has `value`/`min`/``max`, `is_enabled`, `automation_state`. Set `.value` to move knobs/sliders.
- **`MixerDevice`** — per-track `volume`, `panning`, `sends[]`, `track_activator`, `crossfader`, `cue_volume`.
- **`Song.view` / `Application.view`** — selected track/scene/clip, focus/show/zoom views, `selectDevice`.
- **`Browser`** — `instruments`/`samples`/`drums`/… → `BrowserItem[]`; `loadItem`/`previewItem`.

For the complete property/method list per namespace, see `references/api-reference.md`.

## Events: connection & property changes

The `Ableton` instance is an `EventEmitter`. Connection lifecycle events are the backbone of a robust app:

```ts
ableton.on("connect", (type) => {
  /* "start" | "realtime" | "heartbeat" */
});
ableton.on("disconnect", (type) => {
  /* "realtime" | "heartbeat" */
});
ableton.on("ping", (ms) => console.log("latency", ms));
```

Property listeners push a value **every time Live's value changes** — for some props that's 20–30× per second. Add one with `obj.addListener`:

```ts
const off = await ableton.song.addListener("is_playing", (p) => console.log(p));
// later:
off(); // unsubscribe
```

**Re-attach listeners on reconnect.** When you load a new Set, Live tears down and restarts the script. ableton-js fires `disconnect`, clears all listeners/pending commands, then `connect`. Any listeners you added are gone — re-add them inside your `connect` handler. Likewise re-fetch any cached props. This is the single most important robustness rule.

## Common patterns

**Observe & log:**

```ts
ableton.song.addListener("tempo", (t) => console.log("tempo", t));
ableton.song.addListener("is_playing", (p) => console.log("playing", p));
```

**Batch create tracks + clips:**

```ts
const track = await ableton.song.createMidiTrack(0); // index -1 = end
const slot = (await track.get("clip_slots"))[0];
const clip = await slot.createClip(4); // 4-beat clip
await clip.set("name", "Loop 1");
await clip.setNotes([
  { pitch: 60, time: 0, duration: 1, velocity: 100, muted: false },
]);
```

**Drive a device parameter:**

```ts
const track = (await ableton.song.get("tracks"))[0];
const device = (await track.get("devices"))[0];
const params = await device.get("parameters");
await params[0].set("value", 0.75);
params[0].addListener("value", (v) => console.log("param moved", v));
```

**Mixer volume / pan / sends:**

```ts
const mixer = await track.get("mixer_device");
const vol = await mixer.get("volume"); // DeviceParameter
await vol.set("value", 0.8);
```

More recipes (undo grouping, MIDI note editing, browser loading, reconnection wrapper) are in `references/patterns-and-examples.md`.

## Gotchas & constraints

- **Listener quirks in Live itself.** `output_meter_level` can hang Live every few hundred ms — listen to `output_meter_left`/`output_meter_right` instead. `ClipSlot.playing_status` listener never fires in Live (see ableton-js issues #4 and #25). Work around these; they are Live bugs, not library bugs.
- **High-frequency listeners are expensive.** Meter & position props fire dozens of times per second. Throttle/debounce in your handler and avoid doing heavy work synchronously.
- **`get` is async & round-trips over UDP.** Walking a deep tree with one `get` per prop is slow. For snapshots, read the arrays once (they're cached on the client) and reuse the returned objects. ableton-js caches large lists (tracks, scenes, clip_slots, devices, parameters) with an ETag so repeated reads are cheap.
- **Objects are invalidated by edits.** A `Track`/`Clip`/`Device` `id` stops being valid after the object is deleted, moved, or (sometimes) after a big structural change. Re-fetch from `song` rather than caching long-term.
- **Version mismatch.** The npm library warns if the installed `AbletonJS` plugin is older than the JS version. Keep both at the same version (`ableton.internal.isPluginUpToDate()`).
- **MIDI clip vs audio clip.** Note APIs (`getNotesExtended`, `setNotes`, ...) only work on MIDI clips; warp/sample-time APIs only on audio clips. Check `clip.get("is_midi_clip")` / `"is_audio_clip"` first.
- **Temp file port handshake.** `start()` reads `ableton-js-server.port` from the OS temp dir. If a previous run left a stale file pointing at a dead port, connection can hang. Delete the port files in the temp dir when debugging. See `references/setup-and-protocol.md`.

## References (load as needed)

- **`references/api-reference.md`** — Complete typed surface of every namespace: gettable/settable/observable properties, methods, and enums (Song, Track, Clip, Scene, ClipSlot, Device, DeviceParameter, MixerDevice, CuePoint, Browser, Application, ApplicationView, SongView, Midi, Session, Color, Note). Read this when you need exact property/method names and types.
- **`references/setup-and-protocol.md`** — Deep dive on installing the Remote Script across OSes, the control-surface add step, `AbletonOptions`, the connection lifecycle (heartbeat, port files), the events map, and the raw UDP/JSON/gzip/chunk/ETag protocol. Read when debugging connection issues or writing low-level code.
- **`references/patterns-and-examples.md`** — Working recipes: robust reconnect wrapper, group edits into one undo step, full MIDI clip generation, bulk parameter sweeps, browser loading, session ring control, sending raw MIDI.

## Critical rules

1. **Always `await ableton.start()` before using any namespace.** Operations before connection resolves will time out.
2. **Re-attach listeners and re-read state on `connect`.** A new Set clears everything; the `disconnect`→`connect` cycle is normal.
3. **Trust the transforms — don't double-wrap.** `track.get("color")` already returns a `Color`; `song.get("tracks")` already returns `Track[]`. Only `set("color", …)` accepts a raw number or `Color`.
4. **Pass object namespaces by their value, not by re-fetching.** A `Track`/`Clip` carries its Live `id`; reuse it. But don't hold it past a deletion.
5. **Match API calls to clip/track type.** MIDI note calls need MIDI clips; audio warp calls need audio clips. Guard with `is_midi_clip`/`is_audio_clip`.
6. **Keep the `AbletonJS` plugin and the npm package at the same version.** Check with `ableton.internal.isPluginUpToDate()`.
7. **For anything you can't find in the TS types, consult the Live API XML** at <https://nsuspray.github.io/Live_API_Doc/11.0.0.xml> — it is the source of truth for every property and function Live exposes.
