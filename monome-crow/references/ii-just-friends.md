# Just Friends (`ii.jf`) — complete ii reference

Just Friends (JF) is a 6-channel function generator / envelope generator / LFO / rhythm / synthesis
module by Whimsical Raps. Over ii it becomes a full 6-voice synth, a polymetric rhythm engine, and a
remotely retunable quantized oscillator bank. This file covers everything needed to drive it from
crow at expert level.

**Sources:** [Just-Type.md](https://github.com/whimsicalraps/Just-Friends/blob/main/Just-Type.md) (the
canonical Just Type / crow reference), crow firmware device definition
[`lua/ii/jf.lua`](https://github.com/monome/crow/blob/main/lua/ii/jf.lua), hardware testing notes from
the crow-mcp project.

---

## 1. What Just Type is

"Just Type" is a set of **invisible patch cords** between crow (or Teletype) and Just Friends:

- remote **TRIGGERS** and **RUN** control (digital + velocity),
- **transpose** of the whole module,
- **retuneable INTONE** (per-channel frequency ratios, savable to flash),
- two full modal personalities unlocked by `ii.jf.mode(1)`:
  - **Synthesis** — 6-voice polyphonic synthesizer with dynamic voice allocation,
  - **Geode** — clocked polymetric rhythm engine (division-based event streams).

Just Type ships on all JF modules since late 2017; a major feature update (Geode, god mode,
quantize, getters) landed July 2020. If a JF misbehaves on ii, update its firmware first.

### Enabling Just Type

```lua
ii.jf.mode(1)   -- enter Synthesis or Geode (see below)
ii.jf.mode(0)   -- back to standard JF behaviour
```

- **Which personality you get depends on the physical SPEED switch** (labeled sound/shape):
  switch at **sound** → Synthesis; switch at **shape** → Geode.
- `mode(1)` persists until power-cycle or `mode(0)` is called. The docs recommend calling
  `ii.jf.mode(x)` in your crow script's `init()` so the state is deterministic on every run.
- Standard mode (mode off): JF behaves normally as function generators; ii triggers/RUN/transpose
  still work.

---

## 2. Addressing

- ii addresses: `{0x70, 0x75}` — index **1** (default) and index **2**.
- Two JFs on one bus: set the second unit's index with `ii.jf.address(2)`:

  ```lua
  ii.jf[1].play_note(0, 5)   -- first JF (always index at call time!)
  ii.jf[2].play_note(0, 5)   -- second JF
  ```

  **Procedure** (from the docs): power down → disconnect JF #1 → power up →
  `ii.jf.address(2)` on the second unit → verify with `ii.jf[2].trigger(1, 1)` →
  power down → reconnect JF #1 → power up.
- **NB**: `ii.jf[n]` re-points the shared i2c address and returns the same table — never cache the
  handle (`local t = ii.jf[2]` then calling `t.x(...)` silently sends to whichever address was set
  last). Always write `ii.jf[2].command(...)` at call time. See `ii-overview.md`.

---

## 3. Command reference — standard (all modes)

| Command | Args | Semantics |
|---|---|---|
| `ii.jf.trigger(channel, state)` | int, int | Virtual TRIGGER. `channel` 1–6 (1=IDENTITY … 6=6N); **0 = all six channels** (hardware norm ignored). `state` 1 (any non-zero) = high 5V; 0 = low 0V. **Only sustain mode cares about the low state** (see §6). |
| `ii.jf.run_mode(mode)` | int | Virtual RUN enable: 1 (non-zero) activates, 0 deactivates. A cable in the physical RUN jack still forces RUN high. *(The Just-Type doc has a heading typo `ii.crow.run_mode` — the command is `ii.jf.run_mode`.)* |
| `ii.jf.run(volts)` | float | Virtual voltage **added to** the physical RUN state. Range **-5 to +5**. **Requires `run_mode(1)` first.** In Geode this drives amplitude cycles (§6); in Synthesis it feeds the FM ratio behaviour (§7). |
| `ii.jf.transpose(pitch)` | float | Shift transposition regardless of speed switch. **1.0 = one octave** (doubles frequency in sound mode / rate in shape mode); `1/12` = one semitone; fractional/microtonal values fine. |
| `ii.jf.vtrigger(channel, level)` | int, float | Trigger with velocity. `level` = amplitude in volts (`5.0` ≈ 5V trigger = standard modular level). `level` 0 puts the channel in the low state and **does not change the saved velocity**. Channel 0 = all. **Channels remember the latest velocity** and apply it to later digital or CV triggers. |
| `ii.jf.retune(channel, num, denom)` | int, int, int | Redefine the INTONE ratio `num/denom` for channel 1–6 (default = harmonic series: 1/1, 2/1, 3/1, 4/1, 5/1, 6/1). **`retune(0,0,0)` resets all channels to defaults. `retune(-1,0,0)` saves the current custom tuning to flash** (survives power-cycle, auto-recalled). |
| `ii.jf.address(index)` | int | Set this unit's ii index to 1 or 2 (dual-JF setups, §2). |
| `ii.jf.mode(state)` | int | 1 (non-zero) = enter Synthesis (sound) or Geode (shape) per the physical switch; 0 = standard functionality. |

---

## 4. Synthesis personality (SPEED switch = **sound**, `ii.jf.mode(1)`)

6-voice polyphonic synthesizer. Pitch is **1V/oct** (V/8): **`0.0` = middle C = 261.63 Hz** (the
Just-Type doc calls this "C3" — Yamaha naming; in scientific pitch notation it is C4. It is the same
0 V reference as crow's own pitch scale). `1.0` = one octave up. Level is in volts:
**`5.0` = 5V peak-to-peak** (standard modular level).

| Command | Args | Semantics |
|---|---|---|
| `ii.jf.play_note(pitch, level)` | float, float | Play a note with **dynamic voice allocation**. Free voices are prioritized; if all voices are sustaining, the **oldest note is stolen**. |
| `ii.jf.play_voice(channel, pitch, level)` | int, float, float | Same on an explicit channel 1–6. `channel` 0 = all channels. |
| `ii.jf.pitch(channel, pitch)` | float | Set pitch **without triggering the envelope** (portamento). Channel 0 = all. Use for: pitch changes during decay (transient), glides over non-retriggered cycles (cycle), or scale selection over CV-triggered channels. |
| `ii.jf.god_mode(state)` | int | 1 = retune C3 base so A=432Hz ("God note"); 0 = A=440Hz (default). |

Envelope shape per voice is governed by the panel: **TIME** knob+CV and **INTONE** knob+CV
(attack/release character), and the **MODE switch** (transient/sustain/cycle — §6). Notes can be
excited digitally (`play_note` / `play_voice` / `vtrigger` / `trigger`) or by hardware triggers
(which fire the envelope using the voice's *current* pitch and saved velocity).

**Level memory:** `vtrigger(ch, 0)` = low state without forgetting the velocity; a later
`trigger(ch, 1)` reuses the remembered velocity.

---

## 5. Geode personality (SPEED switch = **shape**, `ii.jf.mode(1)`)

Clocked polymetric rhythm engine. Each channel emits a stream of envelope events at a musical
division of a shared measure (**measure = 4 beats**).

| Command | Args | Semantics |
|---|---|---|
| `ii.jf.play_note(divs, repeats)` | float, float | Start a rhythmic stream, **round-robin channel allocation**. |
| `ii.jf.play_voice(channel, divs, repeats)` | int, float, float | Same on explicit channel 1–6 (0 = all channels). |
| `ii.jf.tick(clock_or_bpm)` | int | Dual semantics — see below. |
| `ii.jf.quantize(divisions)` | int | Queue/delay all events to a subdivision of the timebase. 1–32 = activate; 0 = off. |

### divs / repeats semantics

- **`divs`**: divides the measure into this many segments. `4` = quarter notes, `8` = eighths,
  `15` = "weird" (the docs' word — prime divisions produce syncopated polymeters).
- **`repeats`**: how many retriggers of the stream:
  - `-1` = repeat indefinitely,
  - `0` = initial trigger only (1 event),
  - `1` = trigger + 1 repeat (**2 events total**),
  - `n` = `n+1` events total.

### ⚠ Crow wire-scaling for Geode arguments (resolved from source + community)

`ii.jf.play_note`/`play_voice` args are typed **`s16V`** in crow's device table
(`lua/ii/jf.lua`) — crow **multiplies float args by 1638.3** (the Teletype `V` unit) before
sending. Synthesis pitch is defined in those V-units (so passing volts is correct there), but the
JF's **Geode engine interprets the raw wire integers as `divs`/`repeats` directly** (Teletype
users pass plain ints: `JF.NOTE 4 3`). So from crow, **divide Geode arguments by 1638.3** to land
on the intended integers:

```lua
-- want divs=4 (quarter notes), repeats=-1 (forever):
ii.jf.play_note(4/1638.3, -1/1638.3)
```

Community reports (voidstar, lines thread 25937/27–32) describe exactly this with the Max/Teletype
wire format. Caveats: encoding truncates toward zero — if a division sounds one segment short,
nudge the numerator up slightly (e.g. `(4 + 0.5)/1638.3`); verify on hardware. The
`(divs, repeats)` **argument order** is the wiki/firmware order (some thread posts label them
reversed).

### `tick()` — timebase

- `tick(1..48)` = **ticks per measure** (measure = 4 beats): `tick(4)` = one tick per beat, so call
  it from a metro/clock every beat (the docs' example: `tick(4)` once per second at 60 BPM).
- `tick(49..255)` = **static BPM** for the internal timebase (free-run).
- `tick(0)` = **reset to the start of the measure** (both forms).
- `ii.jf.get('tick')` returns the current timebase as BPM.

### `quantize()`

Events queue to the nearest subdivision of the timebase (1–32; 0 = off). Doc recipe for instant
groove: **odd `divs` (5/7/11/13/15) + even quantize (8 or 16)** locks syncopation to the grid —
"quantize brings everything back to the grid, and only a truly select few survive".
Approximate Euclidean rhythms by choosing `divs`/`repeats` pairs (true rotation requires
timing the `play_voice` calls).

---

## 6. Transient / Sustain / Cycle (the MODE switch, `tsc`)

The panel MODE switch selects the envelope's amplitude behaviour, queryable via `ii.jf.get('tsc')`.
**Known discrepancy:** the Just-Type doc body says `1=transient, 2=sustain, 3=cycle`; crow's
`lua/ii/jf.lua` documents `0=transient, 1=sustain, 2=cycle`. Verify on hardware before relying on
the numeric value; write code that maps the value through a constant you can flip.

### In Synthesis mode

- **Transient**: classic pluck — envelope fires and decays; `pitch()` glides mid-decay.
- **Sustain**: envelope holds; **the trigger's low state matters here** (releasing a sustain
  channel brings it down).
- **Cycle**: continuous retriggering behaviour; `pitch()` for non-retriggered cycles.

### In Geode mode — the three temporal layers

Docs describe three layers of time: **TIME/INTONE** = base envelope rate, **`divs`** = rhythm,
**RUN voltage** = amplitude cycle. RUN fluidly modifies amplitude cycles relative to `repeats` in
all three `tsc` modes:

- **Transient** — each repeat at full (or `vtrigger`) velocity; **RUN creates velocity cycles every
  n repeats**: 0V = all emphasized (static); slight + = every 2nd emphasized, up to cycles of 10
  with sawtooth decay; negative = reversed (drop then rise). Doc tip: *"8 repeats, triggered every
  8 clocks, choose a rhythm with RUN."*
- **Sustain** — repeats decay to silence over the `repeats` events: 0V = exactly `repeats` to fade;
  +V speeds the decay with reflective bounce (~1V = triangle bounce, more = wavefolding); −V slows
  it; **−5V ≈ uniform amplitudes**. "Pseudo-delay envelopes."
- **Cycle** — continuous repeats-sensitive amplitude cycling: +RUN emphasizes every 2nd/3rd/4th
  event (all in-between fractions available — subtle groove via volume); negative RUN →
  every-fraction-of-a-beat emphasis, increasingly random. "Endless subtle movement; works extra
  well with quantize active."

---

## 7. Getters and the event handler

Every getter goes through `ii.jf.get(name [, arg])`; **the reply arrives only through the Lua
`ii.jf.event` handler** (current crow firmware does not forward ii replies to the USB host
otherwise — queries vanish silently with no handler installed).

```lua
ii.jf.event = function(e, value)
  -- e.name  = getter name ('trigger', 'ramp', ...)
  -- e.arg   = first argument of the get (e.g. channel)
  -- e.device = unit index (1/2 for dual JF)
  if e.name == 'trigger' then
    print('ch ' .. tostring(e.arg) .. ' moving = ' .. tostring(value))
  end
end
```

| Getter | Args | Returns |
|---|---|---|
| `get('trigger')` | channel | 1 if that channel is moving; with channel 0 → **count of active channels** |
| `get('run_mode')` | — | RUN mode state |
| `get('run')` | — | current RUN voltage |
| `get('transpose')` | — | current transpose |
| `get('mode')` | — | 1 if Synthesis/Geode active |
| `get('tick')` | — | current Geode tempo in BPM |
| `get('god_mode')` | — | 1 if god mode active |
| `get('quantize')` | — | quantize divisions |
| `get('speed')` | — | SPEED switch: **0 = shape, 1 = sound** |
| `get('tsc')` | — | MODE switch (see the discrepancy note in §6) |
| `get('ramp')` | — | RAMP knob position (-5..5) |
| `get('curve')` | — | CURVE knob position (-5..5) |
| `get('fm')` | — | FM knob position (-5..5), knob only (no CV) |
| `get('time')` | — | TIME knob **+ CV** (-5..5) |
| `get('intone')` | — | INTONE knob **+ CV** (0 = C3, 1V/oct scaled) |

**Hardware notes (crow-mcp testing):** an unknown getter *name* raises a local Lua error
`getter not found for 'x'` even with no device attached — a useful typo guard. An *unanswered*
query (device absent) simply produces nothing. Presence probe: install a printing `.event` handler,
issue one cheap `get()`, wait briefly — reply = present, silence = absent (or crashed/busy).

---

## 8. Tuning: INTONE, retune, god_mode

- JF's INTONE knob+CV sets the base character; `retune` redefines the **frequency ratio** of a
  channel relative to it. Defaults are the harmonic series (`1/1, 2/1, ... 6/1`).
- Musical framing from the docs: *"Think a guitar with open-G tuning — it lends itself to an
  entirely different style of play."* Retune to scale ratios (e.g. `5/4`, `3/2`) for chord
  banks, or to inharmonic ratios (`7/5`) for noise/percussion timbres.
- `retune(channel, n, d)`: `retune(0,0,0)` = reset; `retune(-1,0,0)` = **save to flash** (recalled
  automatically on power-up).
- `god_mode(1)` moves the reference from A=440 to A=432Hz (retunes the C3 base).
- **INTONE getter scaling:** `get('intone')` returns in the same V/oct space (0 = C3) — the docs
  suggest using panel values as CV sources elsewhere: `get('time')` can drive an external
  oscillator's pitch; the FM knob could select retune ratios; the SPEED switch could switch
  arpeggio banks.

---

## 9. Musical techniques and recipes

All from the Just-Type doc unless noted.

- **FM timbre cycling** (Synthesis): FM knob CCW + a fixed cyclic sequence → six timbre levels
  cycling; with 5- or 7-step sequences over simple arpeggios → long phasing timbral patterns.
- **FM index character**: knob CCW = INTONE-style modulation, CW = uniform. **The RUN jack sets
  the modulator:carrier ratio** in synthesis FM: uncabled 1:1, +5V → 2:1, −5V → 1:2 ("many
  grumbles" in between — i.e. it sweeps ratios continuously).
- **RUN as a performance controller** (Geode): shape the amplitude cycle live with RUN CV while
  `divs` keeps the rhythm — three independent time layers (TIME/INTONE, divs, RUN).
- **Prime divisions + even quantize** for syncopated percussion; **sustain + RUN** for
  pseudo-delay envelope clouds; **cycle + quantize** for "endless subtle movement";
  **negative-RUN cycle zone** for controlled unpredictability.
- **Geode exploration tip**: *"Start with TIME set very fast, then dial it back to hear how the
  repeats entangle."*
- **VOX+NOTE interaction** (Synthesis): interesting mono/poly splits — alternate
  `play_voice` (fixed voice) with `play_note` (allocation) for lead + accompaniment.
- **crow-side patterns** (see `patterns-musical.md`): chord strum (staggered `play_note` calls),
  velocity as articulation (crow-side sequins of levels into `vtrigger`), JF tempo lock
  (`ii.jf.tick(clock.tempo)` when clock.tempo is an integer 49–255, or tick-from-metro for beat
  sync), and Geode `divs` sequences for polyrhythm.

### Minimal working script

```lua
function init()
  ii.jf.mode(1)          -- put it in init() so state is deterministic
  ii.jf.run_mode(1)
  ii.jf.transpose(0)     -- 1.0 = octave
  notes = sequins{0, 4, 7, 11, 14}/12   -- V/8 (0 = C3)
  clock.run(arpeggio)
end

function arpeggio()
  while true do
    clock.sync(1/4)
    ii.jf.play_note(notes(), 5)   -- pitch in V/8, level 5V
  end
end
```

Geode starter (per the docs' "8 repeats every 8 clocks" idea):

```lua
function init()
  ii.jf.mode(1)             -- SPEED switch at 'shape'
  ii.jf.tick(120)           -- 49..255 = static BPM (or tick(4) per beat)
  ii.jf.quantize(8)
  ii.jf.run_mode(1)
  ii.jf.run(0)              -- 0V = flat amplitude cycle; vary for grooves
  for ch = 1, 6 do
    ii.jf.play_voice(ch, ({4, 5, 7, 8, 11, 13})[ch], -1)  -- divs, indefinite repeats
  end
end
```

---

## 10. Gotchas & troubleshooting

- **`mode(1)` before anything modal.** `play_note`/`tick`/`quantize` only exist meaningfully in
  the modal personalities; without `mode(1)` the ii commands are accepted but the module stays in
  standard function-generator behaviour.
- **SPEED switch selects the personality**, not `mode()`. If you get rhythms where you expect
  synths, check the physical switch (`get('speed')`).
- **`run()` needs `run_mode(1)` first** — verified in both doc and firmware comments.
- **Getter replies need the `.event` handler** — no handler, no reply (this is silent).
- **Channel 0 = all** for `trigger`/`vtrigger`/`pitch`/`play_voice` — handy for full-chord
  resets (`ii.jf.trigger(0, 0)`).
- **Velocity is sticky** — a `vtrigger` level persists for later `trigger`s on that channel.
- **`tick` integer ranges matter**: 1–48 is division semantics, 49–255 is BPM. There is no
  fractional BPM form; for tempo-lock to crow's `clock.tempo`, send `tick(math.floor(clock.tempo + 0.5))`
  (only valid at tempos 49–255) or derive ticks-per-beat from the clock.
- **Two JFs need `address(2)`** set on one unit before `ii.jf[2]` reaches anything; unacknowledged
  writes are silently ignored (no error).
- **`get('trigger', 0)` returns a count**, not a boolean.
- **tsc value mapping is ambiguous** (§6) — wrap it in a named constant.

---

## 11. Verified claims ledger

| Claim | Status |
|---|---|
| `ii.jf.run(volts)` requires `run_mode(1)` | Confirmed (doc + firmware comment) |
| Pitch `0.0` = C3, 1V/oct ("V/8") | Confirmed |
| Level `5.0` = 5V peak-to-peak | Confirmed |
| `tick(divs)` 1–48, `tick(bpm)` 49–255, `0` = measure reset | Confirmed |
| `play_note(pitch, level)` Synthesis vs `play_note(divs, repeats)` Geode — same ii command, dual semantics (likewise `play_voice`) | Confirmed (cmd 8/9) |
| `repeats`: −1 forever, 0 trigger-only, 1 = 2 events | Confirmed |
| Doc heading `ii.crow.run_mode` | Typo — real name `ii.jf.run_mode` |
| `get('tsc')` values | **Unresolved**: doc 1/2/3 vs firmware comment 0/1/2 |
| Geode `divs`/`repeats` via crow | **Resolved**: args are `s16V` (×1638.3), Geode reads wire ints → pass `divs/1638.3`; order is `(divs, repeats)` (jf.lua) |
| `retune` args | `s8` raw 0–127, **not** scaled — never divide by 1638 |

---

## 12. Sources

- Just Type / crow reference: https://github.com/whimsicalraps/Just-Friends/blob/main/Just-Type.md
- crow firmware JF ii definition (addresses `{0x70,0x75}`, command arities):
  https://github.com/monome/crow/blob/main/lua/ii/jf.lua
- Firmware update / releases: linked from Just-Type.md (Whimsical Raps GitHub)
