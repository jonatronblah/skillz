# W/ (W/Tape, W/Del, W/Syn) — complete ii reference

The Whimsical Raps **W/** ("wslash") is one eurorack module with **three engines**: a virtual tape
deck (**W/Tape**), a tape echo / BBD delay (**W/Del**), and a 2-op FM polysynth with vactrol LPG
(**W/Syn**). Only one engine runs at a time, switchable at runtime; all three have their own ii
feature set, and — importantly for crow — **each engine has its own ii address**, so crow addresses
them as three distinct devices: `ii.wtape`, `ii.wdel`, `ii.wsyn`.

**Sources:** the [wslash wiki](https://github.com/whimsicalraps/wslash/wiki/) (pages: Home, Tape,
Del, Syn, SD Card — all ii docs live on the per-mode pages, there are no ii subpages), crow firmware
device definitions [`lua/ii/wtape.lua`](https://github.com/monome/crow/blob/main/lua/ii/wtape.lua),
[`wdel.lua`](https://github.com/monome/crow/blob/main/lua/ii/wdel.lua),
[`wsyn.lua`](https://github.com/monome/crow/blob/main/lua/ii/wsyn.lua), hardware testing notes from
the crow-mcp project.

---

## 1. Platform facts (all modes)

- **Engine switcher ("launcher")**: hold record + play + loop **in order**. Settings and presets
  persist across power cycles *and* mode changes.
- **Each engine has a dedicated ii address**, so one module per engine can coexist on the bus
  (and a crow can lead any combination simultaneously):

  | Engine | crow namespace | i2c addresses (primary, alternate) |
  |---|---|---|
  | W/Tape | `ii.wtape` | `0x71`, `0x72` |
  | W/Syn | `ii.wsyn` | `0x76`, `0x77` |
  | W/Del | `ii.wdel` | `0x78`, `0x79` |

- **Second module running the same engine** (alternate address): power off the case →
  **hold Play+Down** → power on (no visual confirmation). Return to normal address: power off →
  **hold Play+Up** → power on. ("Down = confirm, Up = cancel" in W/ UX language.)
- crow syntax for two same-engine modules: index after the module name —
  `ii.wsyn[1].play_note(0.0, 5.0)` / `ii.wsyn[2].play_note(2.0, 5.0)`. **Index at call time**
  (the shared-table caveat in `ii-overview.md` applies).
- Teletype uses abbreviated names (`W/T.REC`, `W/D.FBK`, `W/S.VEL`); crow uses the full names in
  this file.
- SD card (all modes): FAT32, Class 10, 4–8 GB. See wiki page SD-Card for replacement/formatting.

### Value conventions

- Musical pitch arguments (`freq`) are **V/oct** — `0.0` = middle C = 261.63 Hz (the wiki calls it
  "C3"; scientific pitch notation: C4), same scale as crow's `output.volts`.
- Control values (`-5..5`, `0..1`) match the module's panel CV ranges.
- Time arguments (`timestamp`, `seek`, `time`) are **seconds**.

---

## 2. W/Tape (`ii.wtape`) — virtual tape deck

~3 hours of recording per tape (3 tapes as of firmware 2.1), one giant continuous loop with no
start/end. Vari-speed works **while recording**. Record blends continuously between **overdub**
(`erase_strength` 0) and **overwrite** (`erase_strength` 1).

### Setters

| Command | Args | Semantics |
|---|---|---|
| `record(is_recording)` | int | 1 = record on, 0 = off. |
| `play(is_playing)` | int | **1 = forward @1×, 0 = stop, −1 = reverse @1×**. |
| `reverse()` | — | Flip playback direction (no args). |
| `speed(rate [, denominator])` | float[, float] | Speed as rate or ratio; **negative = reverse**. Ratiometric form (`speed(3, 2)` = 3:2) yields just-intoned harmonies / polyrhythmic relationships between two modules. |
| `freq(v8)` | float | Speed as V/oct musical value. **Maintains the current direction** (reverse stays reverse). `freq(0)` ≡ `speed(1)`. |
| `erase_strength(level)` | float | 0 = overdub, 1 = overwrite, continuous between ("opposite of feedback"). |
| `monitor_level(gain)` | float | Dry IN→OUT gain. **Goes through-zero up to 2× gain**; negative inverts the signal. |
| `rec_level(gain)` | float | Input gain of material recorded to tape. |
| `echo_mode(active)` | int | 1 = play back **before** erasing (echo/delay style); 0 (default) = erase first. Irrelevant at pure overdub (no erase); in blended mode sets the 1st echo at full volume vs erase-head level. |
| `loop_start()` | — | Mark the current location as loop start. |
| `loop_end()` | — | Mark loop end; jumps to start. |
| `loop_active(is_active)` | int | 1/0 activate/deactivate. **Deactivating does not erase the loop points** — old loops remain recoverable. |
| `loop_scale(scale)` | int | **+n = multiply loop length by n, −n = divide by n, 0 = reset** to the original window. |
| `loop_next(direction)` | int | + = jump forward one loop length, − = backward one length, **0 = jump to loop start (retrigger)**. |
| `timestamp(seconds)` | float/int | Move the playhead to an absolute position (seconds). |
| `seek(seconds)` | float/int | Move the playhead relative to the current position (seconds). |

**Undocumented (firmware only):** `ii.wtape.WARNING_clear_tape()` — erases all audio on the tape,
unrecoverable. Never call it casually.

### Getters

Valid getter names: `record`, `play`, `speed` (→ rate), `freq`, `erase_strength`,
`monitor_level`, `rec_level`, `echo_mode`, `loop_start` (→ timestamp s), `loop_end` (→ timestamp s),
`loop_active` (added to crow firmware 2025-01-22, commit `2cff9f59` — older `ii_help.txt`
captures lack it), `loop_scale`, `timestamp` (→ timestamp s).

Set-only (no getter — `get('reverse')` etc. raise `getter not found`): `reverse`, `loop_next`, `seek`.

### Musical notes (from the wiki)

- `timestamp` + local storage (e.g. a crow table of cue points) replicates the v1 "cue" system.
- `loop_scale` / `loop_next` = dynamic rhythmic looping: nested slicing of a recorded phrase at
  musical ratios, jumping between slices live.
- Use `freq` for **musical pitch relationships** (tape as oscillator/sampler), `speed` ratios for
  **polyrhythmic and just-intoned playback** between two tapes.
- ii reaches destinations the panel doesn't expose, e.g. direct `rec_level` automation.
- CV notes (context for emulating gestures from crow): record CV 0→5V raises rec level + erase
  strength; negative voltages cross-fade toward the opposite erase setting; 5V+ gate = momentary
  record gate.

### Example: live looper

```lua
function init()
  ii.wtape.play(1)
  ii.wtape.record(1)
end

input[1].mode('change', 1, 0.1, 'rising')
input[1].change = function()
  recording = not recording
  ii.wtape.record(recording and 1 or 0)
end

input[2].mode('change', 1, 0.1, 'rising')
input[2].change = function()
  if not looping then
    ii.wtape.loop_start()
    looping = true
  else
    ii.wtape.loop_end()
    ii.wtape.record(0)
  end
end
```

---

## 3. W/Del (`ii.wdel`) — vari-speed tape echo / delay

Delay from 4 s down to 1 ms at 1× rate; slowing the rate stretches the 4 s loop up to ~1 minute.
Buffer manipulation (length/position/cut) makes it a live-looper as much as a delay.

### Setters

| Command | Args | Semantics |
|---|---|---|
| `feedback(level)` | float | Read-head→write-head feedback (−5 = single repeat … +5 and above = infinite / sound-on-sound). |
| `mix(fade)` | float | 0 = dry, 1 = wet. |
| `filter(cutoff)` | float | Centre frequency of the filter in the feedback loop (lowpass character, −5..5 panel scale). |
| `freeze(is_active)` | int | 1 = record head off, buffer frozen (infinite sustain); 0 = normal. |
| `time(seconds)` | float | Delay buffer length in seconds **measured at 1× rate**. |
| `length(num, den)` | int, int | Buffer loop size as a **fraction of the buffer time**. |
| `position(num, den)` | int, int | Loop location as a fraction of the buffer time. |
| `cut(num, den)` | int, int | Jump to a loop location as a fraction of the buffer time. |
| `rate(multiplier)` | float | Direct tape-speed multiplier. |
| `freq(volts)` | float | Tape speed as a V/oct musical value. |
| `clock()` | — | Receive a clock pulse for synchronization (the wiki says "sends" — the firmware means **receive**). |
| `clock_ratio(mul, div)` | int, int | Clock pulses per buffer time. (Panel note: currently 4 clocks per 1 delay-time by default.) |
| `pluck(volume)` | float | Pluck the delay line with noise — **Karplus-Strong string synthesis**. |
| `mod_rate(rate)` | float | Modulation rate multiplier (tape-mod character). |
| `mod_amount(amount)` | float | Modulation depth. |

### Getters — read carefully

The wiki says getters are **"NONE"**. crow's device table marks `get` names for `feedback`, `mix`,
`filter`, `freeze`, `time`, `rate`, `freq`, `mod_rate`, `mod_amount` (plus `length`, `position`,
`clock_ratio` names being valid in the table). **Hardware testing (crow-mcp) shows the W/Del
firmware does not answer ii queries at all** — a `get()` to W/Del returns nothing, ever. So:

- valid getter *names* don't raise the local `getter not found` error,
- but silence does **not** mean the device is absent (it does for other devices),
- keep state crow-side instead of polling W/Del.

Setter-only names on W/Del (`get('pluck')`, `get('cut')`, `get('echo_mode')`, …) raise
`getter not found for 'x'` locally.

### The time/rate model (four approaches, from the wiki's notes)

1. **mlr-style chopping**: `length`/`position`/`cut` + `freq` — slice and re-time loops like a
   monome grid looper.
2. **Fixed rhythmic delays**: `time` + `rate` — traditional delay, tuned to tempo with ratios.
3. **Clock-synced**: `clock()` pulses + `clock_ratio(mul, div)` (or `rate`/`freq` derivation) —
   tempo-locked echoes; from crow, send pulses from a `clock.sync` loop.
4. **Karplus-Strong pitches**: `time(0)` + `freq(pitch) + pluck(volume)` — the buffer becomes a
   tuned string; freeze for sustain.

Panel extras worth emulating from crow: tap-tempo / direct loop-time set; recursive ×2 buffer zoom
(`play`+`record` combo); 'strike' noise injection for string/percussion synthesis (that's what
`pluck` is); freeze-gate punch-ins.

### Example: clock-synced delay

```lua
function init()
  ii.wdel.time(0.5)       -- 500ms at 1x
  ii.wdel.mix(0.7)
  ii.wdel.feedback(3.5)
  clock.run(send_clock)
end

function send_clock()
  while true do
    clock.sync(1/4)
    ii.wdel.clock()       -- one pulse per beat-quarter
  end
end
```

---

## 4. W/Syn (`ii.wsyn`) — 2-op FM polysynth with vactrol LPG

**4 voices** (per the wiki). Pitch-capture is gate-aware: with a gate high, pitch changes are
captured (enables poly arpeggios with slow envelopes). The IN jack is **AC-coupled** (audio-rate
only). Pitch CV 0V = middle C (261.63 Hz; the wiki's "C3").

### Setters

| Command | Args | Semantics |
|---|---|---|
| `velocity(voice, velocity)` | int, float | Strike the voice's vactrol at velocity (volts). |
| `pitch(voice, pitch)` | int, float | Set voice pitch (V/oct), no strike. |
| `play_voice(voice, pitch, velocity)` | int, float, float | Set pitch + strike. |
| `play_note(pitch, velocity)` | float, float | **Dynamic voice assignment** + play (4-voice allocation). |
| `ar_mode(is_ar)` | int | 1 = attack-release: all notes "plucked", no release needed; 0 = off (gates sustain). |
| `curve(curve)` | float | Waveform cross-fade: −5 = square, 0 = triangle, +5 = sine. |
| `ramp(ramp)` | float | Symmetry: −5 = ramp-wave, 0 = triangle, +5 = sawtooth. Affects FM tone; with square `curve` = PWM. |
| `fm_index(index)` | float | FM amount: −5 negative, 0 minimum, +5 maximum. |
| `fm_env(amount)` | float | Vactrol envelope applied to FM index (bipolar −5..5; can counteract `fm_index`). |
| `fm_ratio(num, den)` | float, float | Modulator:carrier ratio. **Float values up to 20.0 supported** (the panel quantizes to integers; ii allows floats — e.g. `fm_ratio(1.5, 1)`). |
| `lpg_time(time)` | float | Vactrol time constant: **−5 = drones, 0 = vtl5c3 (natural), +5 = blits**. |
| `lpg_symmetry(symmetry)` | float | Vactrol attack/release ratio: −5 = fastest attack, +5 = long swells. |
| `voices(count)` | int | Polyphonic voice count; **0 = unison**. |
| `patch(jack, param)` | int, int | Route a jack to a parameter for preset automation: jack 1 = THIS, 2 = THAT. param: 1 = ramp, 2 = curve, 3 = fm_env, 4 = fm_index, 5 = lpg_time, 6 = lpg_symmetry, 7 = gate (note at current pitch), 8 = pitch (V/oct, only while gate high), 9 = fm_ratio numerator, 10 = fm_ratio denominator. |

### Getters

- Wiki documents only `get('fm_ratio')`.
- crow's device table additionally marks getters for `ar_mode`, `curve`, `ramp`, `fm_index`,
  `fm_env`, `lpg_time`, `lpg_symmetry`, `patch` (with jack arg), `voices`.
- Unlike W/Del, W/Syn hardware **does** answer queries (install the `.event` handler first).

### Musical notes

- **FM index + env in opposing directions** (e.g. `fm_index(3)`, `fm_env(-2)`) is the wiki's
  recommended timbre trick: bright attacks decaying to a softer core.
- `ar_mode(1)` + fast `lpg_time` = plucked-string/EP tones; `ar_mode(0)` + `lpg_time(-5)` = drones.
- `voices(0)` unison stacks all four voices for thick leads; `voices(4)` for polyphony.
- Float FM ratios (up to 20.0) unlock non-integer spectra the panel can't reach.
- `pitch()` without `velocity` = portamento / scale slides under sustained notes.

### Example: FM drone

```lua
function init()
  ii.wsyn.fm_ratio(3, 1)     -- 3:1 FM
  ii.wsyn.fm_index(3)
  ii.wsyn.fm_env(-2)         -- opposing env: bright attack, soft core
  ii.wsyn.lpg_time(-2)       -- long vactrol
  ii.wsyn.voices(4)
  ii.wsyn.play_note(0, 5)    -- C3 at 5V
end
```

---

## 5. Polling multiple W/ getters from crow

W/ devices answer at most one query at a time and i2c traffic should be paced; the bowery example
(`twigs/w_slash_stats.lua`) polls like this:

```lua
ii.wdel.event = function(e, value)
  print(e.name .. ' = ' .. value)   -- NOTE: W/Del hardware never replies; this works on W/Tape & W/Syn
end

function poll()
  while true do
    for _, name in ipairs{'feedback', 'mix', 'time', 'rate'} do
      ii.wdel.get(name)
      clock.sleep(0.0015)    -- pace requests to avoid i2c congestion
    end
    clock.sleep(0.3)         -- full sweep every 300ms
  end
end
```

Two-arg getters work the same way (e.g. `ii.wsyn.get('patch', 1)`).

---

## 6. Gotchas & troubleshooting

- **Silent failures**: ii writes to an address with no responding device are silently ignored —
  no error, no ACK check. Verify your engine/namespace pairing (`ii.wtape` while the module is in
  Syn mode reaches the *W/Tape* address `0x71`, not the currently-running engine).
- **Only one engine runs at a time** — `ii.wtape`/`ii.wdel`/`ii.wsyn` name *engines* (distinct
  addresses), not the currently-active sound engine. On a single W/ module, commands to a
  non-active engine will not be audible until you switch engines (launcher gesture or power-up).
  Multiple W/ modules can coexist precisely because each engine's address is distinct.
- **W/Del never answers `get()`** (§3) — don't build presence detection on W/Del queries.
- **Getter typos raise local Lua errors** (`getter not found for 'x'`) — that's crow validating the
  name, not the device answering.
- **`play(−1)` is reverse** — `play(0)` is stop; don't assume boolean semantics.
- **`loop_active(0)` keeps loop points** — you can bank and recall loops by toggling rather than
  re-marking.
- **`speed` sign flips direction; `freq` doesn't.** Choose deliberately when reversing.
- **Dual same-engine modules** need the Play+Down boot gesture for the second unit (§1) or
  `ii.wtape[2]` writes land on the first unit's address.
- **`WARNING_clear_tape()`** is irreversible tape erasure — never use it in example code.

---

## 7. Sources

- W/ wiki home: https://github.com/whimsicalraps/wslash/wiki/
- W/Tape ii: https://github.com/whimsicalraps/wslash/wiki/Tape
- W/Del ii: https://github.com/whimsicalraps/wslash/wiki/Del
- W/Syn ii: https://github.com/whimsicalraps/wslash/wiki/Syn
- SD card: https://github.com/whimsicalraps/wslash/wiki/SD-Card
- crow firmware device definitions (addresses, arity, scaling):
  https://github.com/monome/crow/tree/main/lua/ii — `wtape.lua`, `wdel.lua`, `wsyn.lua`
