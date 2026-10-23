# The jacks — `input`, `output`, and built-in actions

crow has **2 CV inputs** and **4 CV outputs**, all 16-bit over **[−5 V, +10 V]** (clipped outside).
This file covers everything about reading inputs, writing outputs, and the built-in ASL actions
(LFOs, envelopes, pulses). Custom ASL journeys are in `api-asl.md`; timing loops in `api-time.md`.

## Units & pitch convention (read this first)

- **Voltage**: plain volts. For pitch, everything is **1V/oct** ("V/8" in monome-speak):
  **0 V = middle C = 261.63 Hz**, 1.0 V = one octave up, `1/12` V = one semitone.
  (Monome and Whimsical Raps docs call 0 V "C3" — Yamaha naming; in scientific pitch notation it
  is **C4**. `hztovolts()` defaults to the same 261.63 Hz reference.)
- **Time**: **seconds** everywhere (slew, ASL, `clock.sleep`, metro, timeline).
- Voltage range **−5 … +10 V** on both sides.

---

# `input` — 2 CV inputs

`input` is a table: `input[1]`, `input[2]`.

## Queries

```lua
_ = input[n].volts        -- current voltage (read at any time)
input[n].query            -- push the current value to the host now
                          --   -> ^^stream(n, volts); also calls the .stream handler
```

## Modes

Set with `input[n].mode(...)` (or shorthand `input[n].mode = 'stream'`). `'none'` disables events.

```lua
input[n].mode('none')

input[n].mode('stream', time)                    -- report every `time` seconds
input[n].mode('change', threshold, hysteresis, direction)
    -- direction: 'rising' | 'falling' | 'both'  (state = true when high)
input[n].mode('window', windows, hysteresis)
    -- windows: table of boundary voltages, eg {-3,-1,1,3} -> 3 windows
input[n].mode('scale', notes, temperament, scaling)
    -- notes: eg {0,2,4,5,7,9,11} (semitones) or JI ratios {1/1, 9/8, 3/2, ...}
    -- temperament: notes per octave (default 12) or 'ji'
    -- scaling: volts per octave (default 1.0; 1.2 for Buchla)
input[n].mode('volume', time)                    -- RMS amplitude of audio, every `time` s
input[n].mode('peak', threshold, hysteresis)     -- audio-transient detector (RMS level)
input[1].mode('freq', time)                      -- INPUT 1 ONLY: measure connected osc frequency
input[n].mode('clock', division)                 -- drive the internal clock from pulses
    -- division: rate at which pulses arrive; eg 1/4 for 4 ticks per beat (default 1/4)
```

**Defaults**: `time = 0.1`, `threshold = 1.0`, `hysteresis = 0.1`, `direction = 'both'`,
`temperament = 12`, `scaling = 1.0`, `division = 1/4`.

### `'clock'` mode details

- Pulses are detected like `'change'` (rising-only, current threshold/hysteresis) and feed the
  **internal clock module's tempo**: `clock.sync`/`clock.sleep`/timelines follow the patched clock.
- While active, `clock.tempo` reports the **measured** tempo; assigning it does nothing.
- **Raises no Lua input event** (no `.change`/`.stream` handler fires) — it hooks the clock
  directly. `clock.start()`/`clock.stop()` switch the source back to internal.
- Mutually exclusive with `'change'` mode on the same input — to detect edges too, poll
  `input[2].volts` in a `clock.run` loop (see `community-techniques.md`).

## Event handlers

Default handlers send `^^…` events to the USB host. Assign your own to process in Lua:

```lua
input[n].stream = function(volts) ... end   -- 'stream': value in volts
input[n].change = function(state) ... end   -- 'change': boolean (true = above threshold)
input[n].window = function(win, dir) ... end -- 'window': win = 1-based window index, dir = boolean
input[n].scale  = function(s) ... end       -- 'scale': s = {index, octave, note, volts}
input[n].volume = function(level) ... end   -- 'volume': RMS level
input[n].peak   = function() ... end        -- 'peak': no args (the event IS the transient)
input[n].freq   = function(freq) ... end    -- 'freq': Hz (input 1 only)
```

`input[n].reset_events()` clears custom handlers (also done by `crow.reset()`).

### The `scale` handler table

```lua
input[1].mode('scale', {0,2,4,5,7,9,11})
input[1].scale = function(s)
  -- s.index  = index into the notes table (1-based)
  -- s.octave = octave offset
  -- s.note   = the quantized note value (semitones; with 'ji' a ratio)
  -- s.volts  = the quantized voltage — pitch-ready, JI already applied
  output[1].volts = s.volts
end
```

With `'ji'`, convert ratios yourself: `justvolts(s.note)` → V/oct, `just12(s.note)` → semitones.
(The scale table entries are plain Lua numbers/ratios evaluated at load.)

## Table-call syntax

Named-argument form; can set the handler in the same call:

```lua
input[1]{ mode   = 'stream'
        , time   = 0.2
        , stream = function(v) print(v) end
        }
input[1]{ mode = 'change', direction = 'rising', change = my_fn }
```

## Input-side gotchas

- **Call `input[n].mode(...)` with a DOT, never a colon** — `mode` is a closure already bound to
  the input; `input[n]:mode('stream', 0.1)` mis-binds the arguments. (`output[n]:clock(...)` is
  the exact opposite — see below.)
- `input[n]()` (calling the input itself) returns the current volts.
- Reading `input[n].mode` back returns the *function*, not the mode string.
- The `'change'` direction string is matched on its first character: `'r…'` → rising, `'f…'` →
  falling, anything else → both.
- **Naming your handler arg `volts` shadows `input[n].volts`** — use `v`.
- Registering a handler **replaces** the default host echo — re-add `tell('stream', n, v)` if you
  still want it (see `api-environment.md`).
- One handler per event type per input; re-assignment replaces silently.
- Sampling rate is 1.5 kHz — fine for CV; `stream` updates land ~2–3 ms behind real time.

---

# `output` — 4 CV outputs

`output` is a table: `output[1..4]`.

## Setting CV

```lua
output[n].volts = 3.0     -- set instantly (slew 0); also restarts any running ASL action
_ = output[n].volts       -- instantaneous DAC voltage (READ-BACK LAGS one io cycle!)
output[n].slew  = 0.1     -- slew time in seconds (default 0.0)
output[n].shape = 'expo'  -- non-linear path to the destination (default 'linear')
```

### Shapes

Documented set (the full list): `'linear'`, `'sine'`, `'logarithmic'`, `'exponential'`, `'now'`
(go instantly, then wait), `'wait'` (wait, then go), `'over'` (overshoot then land), `'under'`
(move away then ramp up), `'rebound'` (bouncing-ball).

Parser reality (`lib/slopes.c`): case-insensitive, matched on the **first character** — `s`→sine,
`e`→expo, `n`→now, `w`→wait, `o`→over, `u`→under, `r`→rebound, `l`→log **only when followed by
`o`** (`'log'`, `'logarithmic'`), anything else → linear **silently**. So `'expo'`, `'exp'`,
`'exponential'` are all valid; `'lin'`, `''`, and typos become linear.

### Read-back gotcha

`.volts` reads the instantaneous DAC value, which **lags an assignment by one io cycle**:
`output[n].volts = 4.0; print(output[n].volts)` prints the **old** value (this is not slew —
default slew is 0). Read again in a later call to confirm. Verified on hardware.

## Quantizing to scales

Applied after slew/actions; makes every output pitch-safe:

```lua
output[n].scale({0,2,3,5,7,9,10})                        -- semitone scale (any table)
output[n].scale({}, 12, 1.0)                             -- {} = chromatic; temperament; V/oct
output[n].scale({1/1, 9/8, 5/4, 4/3, 3/2, 11/8}, 'ji')   -- just intonation ratios
output[n].scale('none')                                  -- deactivate
output[n].scale()                                        -- no args = CHROMATIC quantizer ON
                                                         --   (NOT "off" — only 'none' disables)
output[n].scale = {0,7,2,9}                              -- modify table WITHOUT reactivating;
                                                         --   out-of-order tables = arpeggios
```

- Tempered notes are semitones relative to 0 V = middle C; JI tables are ratios. Max **24** scale
  entries.
- **Gotcha**: temperament/scaling are C statics **shared across all outputs** —
  `output[1].scale(list, 5, 1.2)` silently changes the defaults inherited by outputs 2–4. Pass
  them explicitly per call if you mix temperaments.

## Clock mode

An output can emit a tempo-synced clock (call with **colon**):

```lua
output[n]:clock(division)     -- division in beats, same units as clock.sync (default 1)
output[n]:clock('none')       -- disable ('off' also works)
output[n].clock_div = 1/4     -- change division live WITHOUT re-syncing
```

Internally this is `pulse()` on that output's ASL plus a coroutine doing
`clock.sync(clock_div); action()`. The default action is `pulse` — you can reassign
`output[n].action` to any other shape after enabling.

```lua
-- zero-script divider bank (community classic):
clock.tempo = 120
for i = 1, 4 do output[i]:clock(i/4) end
-- periods 1/4, 2/4, 3/4, 4/4 beats = every 1st/2nd/3rd/4th 16th-note:
-- out1 = 16ths ("straight"), out2 = 8ths, out3 = dotted-8ths, out4 = quarters.
-- For divisors of the *beat* (1,2,3,4 beats per pulse) use output[i]:clock(i).
```

## Actions (ASL)

Outputs can run **actions** — ASL journeys (see `api-asl.md` for custom ones):

```lua
output[n].action = lfo()     -- set the action
output[n]()                  -- start it (restarts from the top if already running)
output[n](lfo(2, 3))         -- set AND start in one call
output[n].done = function() print('done!') end   -- fired when the action completes
```

### Built-in actions (`asllib.lua`)

| action | signature | defaults |
|---|---|---|
| `lfo` | `lfo(time, level, shape)` | time 1, level 5, shape `'sine'` |
| `pulse` | `pulse(time, level, polarity)` | time 0.01, level 5, polarity 1 |
| `ramp` | `ramp(time, skew, level)` | time 1, skew 0.25, level 5 |
| `ar` | `ar(attack, release, level, shape)` | 0.05, 0.5, 7, `'log'` |
| `adsr` | `adsr(attack, decay, sustain, release, shape)` | 0.05, 0.3, 2, 2, `'linear'` |
| `oscillate` | `oscillate(freq, level, shape)` | docs say 1, 5, `'sine'` — **source: `oscillate = lfo(1/freq, …)`; calling `oscillate()` without freq errors** |

Source facts worth knowing: `adsr`'s attack ramps to a **hardcoded 7.0 V** then drops to the
sustain level (inside `held{}`); `pulse` is `to(level, time, 'now')`; `ramp` skews the two halves
via `time*(1.996*skew+0.001)` / `time*(1.998−1.996*skew)`; `note(mn, dur)` exists but is
**deprecated**.

### ADSR directives

```lua
output[1].action = adsr()
output[1](true)    -- start attack, pause at sustain (re-bang = re-attack from current spot)
output[1](false)   -- enter release
output[1]()        -- nil arg = restart
-- 'unlock' releases a lock{} construct (api-asl.md)
```

### `.action` as a string

`.action` may also be assigned a **string of ASL Lua** (norns-style):
`output[1].action = "{ to(0,0), to(5,0.1) }"`. Handy for storing journeys in data tables
(community "Eckelopes" pattern).

## Undocumented but real (from `lua/output.lua`)

```lua
output[n].query()             -- send current voltage to host -> ^^output(ch, volts)
output[n].receive = fn        -- handler for query results
output[n].dyn                 -- the ASL dyn table (see api-asl.md)
output[n].execute             -- alias for calling the action with directives
output[n].reset_events()      -- clear done/receive handlers
```

## Output-side gotchas

- Assigning `.volts` **kills the running action** — that's also the freeze trick:
  `output[n].volts = output[n].volts`.
- `output[n](s)` passing a change-handler arg errors in ASL (`casl_setdynamic … got nil`) —
  bang with `output[n]()`.
- Shape args must be **strings** (`ar(0.5, 0.2, 5.0, 'linear')`, not bare `linear`).
- Set actions in `init`, bang on events — reassigning `.action` per trigger is wasteful.
- ASL resource limits (16 `to` stages / 8 sequences / 8 stages / 40 dyns per output) — see
  `api-asl.md` and `troubleshooting.md`.
