# Time & scheduling — `metro`, `clock`, `delay`, `timeline`

crow has four time systems. Choosing the right one is the difference between a script that feels
musical and one that drifts or fights you.

| System | Use it for | Tempo-aware? | Notes |
|---|---|---|---|
| `metro` | simple repeating timers at a fixed period (LFO updates, polling loops, blinkenlights) | no (seconds only) | 8 hardware-backed timers, sub-ms resolution |
| `clock` | anything rhythmic or beat-relative: sequencers, arpeggios, envelopes, ii pulses | **yes** (BPM, `clock.sync`) | coroutines; the backbone of musical timing |
| `delay` | one-shot (optionally repeated) deferred calls | no (seconds) | sugar over `clock`; returns a cancelable id |
| `timeline` | rhythmic loops, scores, arrangement sections, event sequences | **yes** (loop/score in beats; real in seconds) | declarative time/event tables, launch quantization |

---

## 1. `metro` — 8 independent timers

```lua
metro[1].event = function(c) print(c) end   -- c = 1-based stage number
metro[1].time  = 1.0                        -- seconds between ticks
metro[1]:start()
metro[1]:stop()
```

### Named initialization

```lua
mycounter = metro.init{ event = count_event
                      , time  = 2.0
                      , count = -1          -- -1 = forever (default)
                      }
mycounter:start()
mycounter:stop()

-- positional form: metro.init(fn, time, count) — defaults time=1, count=-1
-- start overrides: mycounter:start(time, count, init_stage)  -- init_stage = 1-based
```

- **8 metros total** (`metro[1..8]`). `metro.init` on an exhausted pool prints
  `Metro.init: nothing available` and returns nil — free unused ones with `metro.free(id)` /
  `metro.free_all()`.
- **`count = N` yields N+1 ticks** (the C stop rule tests the post-increment counter), and the
  `event` callback receives the 1-based stage number — useful for bar counting
  (`count = 15` + the stage arg = a **16**-tick bar; generally `count = steps − 1`).
- Params update live: `metro[1].event = f`, `mycounter.time = 0.1`. **Source gotcha: a time change
  applies from the next wakeup**, not immediately.
- Minimum period is clamped to **0.0005 s** (crash guard).
- The `event` callback receives the current **stage number** (1-based), not a boolean — useful for
  counting (`metro[1].count = 15` + a stage counter = a 16-tick bar counter).
- **Gotcha (official docs):** if you use `delay` and `metro` in the same script, initialize metros
  with `metro.init` and **do not** index `metro[n]` directly.
- `crow.reset()` frees all metros.

### Typical uses

```lua
-- polling loop
poller = metro.init{ event = function() ii.txi.get('param', 1) end, time = 0.1, count = -1 }

-- metro as a free-running LFO trigger (see api-input-output.md for ASL actions instead)
metro[2].event = function() output[1]() end
metro[2].time  = 0.5
metro[2]:start()
```

---

## 2. `clock` — tempo-synced coroutines

The clock system runs Lua functions as **coroutines** scheduled against a shared BPM tempo. This is
where musical time lives. (Running from boot; `clock.start`/`clock.stop` control the *transport*,
not `clock.sleep`.)

```lua
coro_id = clock.run(func [, args...])   -- run func as a coroutine; args are passed to func
clock.cancel(coro_id)
clock.sleep(seconds)                    -- yield for N seconds (inside a clock coroutine only)
clock.sync(beats)                       -- yield until the next multiple of `beats` (eg 1/4 = 16ths)
clock.cleanup()                         -- kill ALL running clock coroutines
```

### Tempo & beat position

```lua
clock.tempo = 120            -- set BPM (genuine property, NOT a function)
_ = clock.tempo              -- get BPM
_ = clock.get_beats()        -- beats since clock started — FUNCTION, call with ()
_ = clock.get_beat_sec()     -- seconds per beat — FUNCTION, call with ()
_ = clock.get_beat_sec(4)    -- optional: n beats in seconds
```

**NB**: `clock.get_beats` and `clock.get_beat_sec` are **functions** — `clock.get_beats` used bare in
arithmetic raises `attempt to perform arithmetic on a function value` (verified on hardware and in
`lua/clock.lua`). The official reference writes them property-style; that is shorthand.

### Sync, beat-wake & helpers

```lua
clock.sync(beats [, offset])     -- sleep to the next multiple of `beats`; the optional `offset`
                                 --   shifts the wake point (added Dec 2025, firmware #498)
clock.beat(beats)                -- wake at an ABSOLUTE beat position (not a multiple of now)
clock.resume(id, ...)            -- resume a created coroutine manually
id = clock.create(f)             -- create a coroutine WITHOUT starting it (clock.run = create+resume)
clock.time_since_last_input()    -- seconds since the last external (input 'clock' mode) tick;
                                 --   returns -1 until one arrives
```

### Transport

```lua
clock.start([beat])                      -- start from `beat` (default 0); switches source to internal
clock.stop()
clock.transport.start = start_handler    -- called when the clock starts
clock.transport.stop  = stop_handler     -- called when the clock stops
```

- `clock.start()`/`clock.stop()` do **not** affect `clock.sleep` calls in running coroutines.
- Undocumented but real (`lua/clock.lua`): `clock.handlers` is literally **the same table as
  `clock.transport`** — `clock.handlers.tempo_change = fn(tempo)` and
  `clock.transport.tempo_change` are one handler, fired when the beat duration changes by
  ≥ 1e-5 s (Dec 2025 `#498`); also `clock.time_since_last_input()`, `clock.beat()`,
  `clock.resume()`, `clock.create(f)`.
- **`clock.cleanup()` also resets `clock.tempo` to 120 and clears transport/tempo_change handlers.**

### External clock — `input[n].mode('clock', division)`

```lua
input[1].mode('clock', 1/4)   -- external clock pulses arrive at 4 ticks per beat
```

- Drives the **internal clock module's tempo** from the input (same edge detection as `'change'`
  mode, internally rising-only with the current threshold/hysteresis — defaults 1.0 / 0.1).
- While active, `clock.tempo` reports the **measured** tempo and assigning it has no effect.
- **Raises no Lua input event** — it hooks the clock directly; there is no `input.clock` handler.
- `clock.start()`/`clock.stop()` switch the source back to internal.

### Canonical patterns

```lua
-- endless pattern: ALWAYS clock.sync (or sleep) inside the loop or the watchdog kills you (~2 s)
function init()
  clock.run(forever)
end

function forever()
  while true do
    x = x + 1
    clock.sleep(0.1)
  end
end
```

```lua
-- one-shot envelope helper
function init()
  output[2].action = adsr()
  dur = 0.6
end

function note_on()
  clock.run(oneshot, dur)
end

function oneshot(seconds)
  output[2](true)
  clock.sleep(seconds)
  output[2](false)
end
```

```lua
-- beat-locked sequencer
function sequencer()
  while true do
    clock.sync(1/4)                  -- 16th notes
    output[1](pulse())               -- trigger
  end
end
```

```lua
-- clock division table (bowery idiom)
divisions = {1, 2, 3, 4, 5, 7, 8, 16}
function div_loop(n)
  while true do
    clock.sync(4 / divisions[n])     -- n divisions per bar
    output[n]()
  end
end
```

---

## 3. `delay` — deferred calls

```lua
delay(action, time [, repeats])   -- run `action` after `time` seconds; optionally repeat
```

Exact firmware semantics (from `lib/l_crowlib.c`) — the details the docs omit:

```lua
function delay(action, time, repeats)
  local r = repeats or 0
  return clock.run(function()
    for i = 1, 1 + r do clock.sleep(time); action(i) end
  end)
end
```

- Total executions = **1 + repeats** (with no `repeats`: exactly once).
- `action` receives the **1-based iteration index** as its argument.
- **Returns the coroutine id** — cancel with `clock.cancel(id)`.
- Built on `clock`, so it follows the same scheduler (and dies with `clock.cleanup()`).
- Mixing with `metro`? Initialize metros via `metro.init` (see §1 gotcha).

```lua
id = delay(function(i) print('tick ' .. i) end, 0.5, 3)   -- prints tick 1..4 at 0.5s spacing
```

---

## 4. `timeline` — rhythmic loops, scores, sequences

Declarative `{time, event, time, event, ...}` tables. Built on `clock` (beat flavors follow the
tempo). Three flavors:

| Flavor | Time meaning | Runs |
|---|---|---|
| `timeline.loop{...}` | **durations** between events (beats) | forever (unless stopped) |
| `timeline.score{...}` | **beat timestamps** of events | once (or looping via `'reset'`) |
| `timeline.real{...}` | **second timestamps** (tempo-independent) | once (or `'reset'`) |

All timelines are **launch-quantized**: they wait for the next tick before the first element /
zero timestamp (see `timeline.launch`, `timeline.launch_default`).

```lua
t1 = timeline.loop{1, kick, 2, snare, 1, kick, 4, rest}

t2 = timeline.loop{1, kick, 2, snare}
     :unless(function() return input[1].volts > 2 end)   -- stop when predicate true

t3 = timeline.loop{0.55, hihat, 0.45, hihat}:times(16)   -- exactly 16 loops

t4 = timeline.score{0, intro, 32, verse}                 -- at beat 0, at beat 32
t5 = timeline.score{0, intro, 32, verse, 64, 'reset'}    -- loops: 'reset' re-zeroes the score
t6 = timeline.score{ 0, intro
                   , 32, verse
                   , 64, function() if math.random() > 0.5 then return 'reset' end end
                   }                                     -- probabilistic loop

t7 = timeline.real{0, note_1, 0.33, note_2, 0.5, note_3} -- seconds
t8 = timeline.real{0, note_1, 1.2, 'reset'}
```

### Events

- Any **function** (called with no args), or a **function table** `{fn, arg1, arg2, ...}`
  (applied like a lisp `apply`).
- The string `'reset'` (single or double quotes) re-zeros the timeline. It may be the last event
  **or any event**, and may be **returned from a function** (probabilistic repetition).
- Times **and** function-table arguments may be **sequins** — each execution advances them:

  ```lua
  tt = timeline.loop{1, {print, "boop!"}}
  tt = timeline.loop{sequins{3,3,2}, {ii.jf.play_note, sequins{0,4,7,11}/12, 2}}
  ```

### Pre-methods (chain before `:loop`/`:score`/`:real`)

```lua
tt = timeline.queue():loop{2, kick, 2, snare}   -- create QUEUED: does not auto-start
tt:play()                                        -- begin when you say so
tt:stop()
tt:play()                                        -- restart

tt = timeline.launch(8):loop{2, kick, 2, snare}  -- quantize launch to the next multiple of 8 beats
timeline.launch(0):real{...}                     -- 0 = no quantization, start immediately
timeline.launch(16):queue():loop{...}            -- chain as many as you need
```

**Parens on `queue` are required** — `queue` is a *function* returning the pre-queued object:
`timeline.queue():loop{...}`. (The older reference page shows `timeline.queue:loop{...}` without
parens — that is a known documentation typo; the timeline page and `lua/timeline.lua` are
authoritative.) Source also offers the undocumented shortcut `timeline.qloop{...}` =
`queue():loop{...}`.

```lua
timeline.launch_default = 4   -- global default launch quantization for NEW timelines (default 1)
timeline.launch_default = 0   -- disable quantization globally
```

### Post-methods (after the timeline exists)

```lua
tt:play()   -- begin (queued) or restart (running) — re-applies launch quantization & :times count
tt:stop()   -- halt; :play() restarts from the beginning
tt:iter()   -- completed loop iterations (loop only; undocumented)
```

Loop-only (docs: "as of now, only `loop` functions can have post-methods"):

```lua
:unless(pred)   -- pred = value or fn returning a value; true => stop. `:unless(true)` = run once
:times(n)       -- run exactly n times (n >= 1; :play() refreshes the count)
```

### Panic button

```lua
timeline.cleanup()   -- stops ALL timelines... and is clock.cleanup(): kills all clock coroutines,
                     -- resets tempo to 120, clears transport handlers. Use with care.
```

### Gotchas

- **`'reset'` for `score`** re-zeroes to `current zero + the reset event's timestamp` (so
  `{0, a, 32, b, 64, 'reset'}` repeats a 64-beat cycle).
- Nested timelines are **not** hotswappable.
- String events are "keywords" — only `'reset'` is implemented.
- `:once()` appears in the method table but is **undefined** (calling it errors — vestigial).
- `timeline.hotswap(old, new)` exists and is what the `hotswap` library uses (see
  `api-environment.md`).

---

## 5. Choosing between them (musical guidance)

- **BPM-following rhythmic content** → `clock.sync` loops or `timeline.loop` with `sequins` args.
  Timeline wins when the pattern has *uneven* durations or section changes (`score` with `'reset'`
  = a chorus form).
- **Envelope/one-shot gestures** → `output` ASL actions (`ar`, `adsr`) or `delay`/`clock.run`.
- **Modulation at audio-ish rates** → `output` ASL `oscillate`/`lfo` (runs in the DSP engine),
  not a metro — metros bottom out at 0.5 ms and cost Lua time.
- **Polling ii devices or inputs** → `metro` (fixed period) or `clock` (beat-relative).
- **Arrangements** (intro/verse/break) → `timeline.score` with function-returning-`'reset'` for
  probabilistic forms; `:unless()` for ending conditions.
- **Tempo changes**: `clock.tempo = t` moves everything clock-based together — including
  timelines and `clock.sync` loops. Metros won't follow (seconds are seconds).
