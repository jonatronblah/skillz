# crow troubleshooting & gotchas — consolidated pitfall reference

Every known way a crow script (or its ii ecosystem) fails, organized so an agent can find the one
it's hitting. Entries are symptom → cause → fix wherever possible, with exact error strings in
**bold** so they can be grepped. Sourced from the monome lines forum threads
[crow: druid scripts](https://llllllll.co/t/crow-druid-scripts/25974) and
[crow strategies: just friends](https://llllllll.co/t/crow-strategies-just-friends/25937), official
monome docs, the crow firmware source, and hardware testing of a serial-protocol crow client.

**Legend** — entries ending in *(author, [post](url))* are community findings; entries marked
**[v]** are verified from official docs, crow firmware source, or hardware testing. Community code
is copied **verbatim** from the thread (including its original typos, which are flagged where they
matter). Ground facts used throughout: pitch `0.0` = middle C = 261.63 Hz (the docs call this "C3");
I/O range is **-5 V to +10 V**; `time()` returns **milliseconds**; 1 V = 1 octave.

See also: [api-time.md](api-time.md), [ii-overview.md](ii-overview.md),
[ii-just-friends.md](ii-just-friends.md), [ii-wslash.md](ii-wslash.md),
[druid-and-protocol.md](druid-and-protocol.md), [technical.md](technical.md),
[community-techniques.md](community-techniques.md).

---

## 1. Execution model & Lua pitfalls

### 1.1 Top-level code runs once at load, BEFORE `init()`

Anything outside a function executes top-to-bottom the moment the script is evaluated — before
`init()` is called, and before any input/metro event exists. A variable assigned inside `init()` or
inside a handler is still `nil` at that point.

- **`attempt to perform arithmetic on a nil value`** at load — a free-floating statement like
  `final_pitch = offset + pitch` evaluates while `offset`/`pitch` are nil. "Since this assignment is
  outside of any function, it executes before even your `init()` function is even run." Fix:
  initialize the variables (`pitch = 0`, `offset = 0`) at file scope and do the math inside a
  handler *(csboling, [25974/93](https://llllllll.co/t/crow-druid-scripts/25974/93))*.
- Load-order proof, verbatim *(dani_derks, [25974/241](https://llllllll.co/t/crow-druid-scripts/25974/241))*:

  ```lua
  -- _test flow_
  -- how does a script get parsed upon load?

  print("hello from the top of the script")
  ----------------------------------------------------------------
  --					init function
  ----------------------------------------------------------------
  function init()
  	print("hello from init")
  	another_function()
  end
  ----------------------------------------------------------------
  --					do something else
  ----------------------------------------------------------------
  function another_function()
  	print("hello from another function!")
  end

  -- typically, where 'nil' gets returned:
  print("hello from below the init")
  ```

  Print order: `hello from the top of the script` → `hello from below the init` →
  `Running: ...` → `hello from init` → `hello from another function!`
- Deeper explanation: `dofile` evaluates the whole file, then C code calls `init()`, then events
  drive everything. "crow has no way of 'looking ahead'" — `a = b` above `b = 5` yields nil
  *(alanza, [25974/244](https://llllllll.co/t/crow-druid-scripts/25974/244))*.

### 1.2 `init()` must be spelled exactly

Rename `function init()` to `function int()` and nothing in it runs — no input modes, no metros,
nothing. There is no error; the script just does nothing *(dani_derks,
[25974/204](https://llllllll.co/t/crow-druid-scripts/25974/204); vinc,
[25974/203](https://llllllll.co/t/crow-druid-scripts/25974/203))* **[v]** (crow calls the global
named `init`, nothing else). Likewise `init(clocked_sh)` is wrong: extra args are unused, it must be
`init()` *(Galapagoose,
[25974/139](https://llllllll.co/t/crow-druid-scripts/25974/139))*.

### 1.3 Handlers are registered callbacks — assign the function, don't call it

`input[1].change = function() ... end` *registers* a callback. Three traps:

- **`attempt to call a nil value (field 'change')`** on every trigger — you wrote
  `input[1].change = ansiblePlay(1, math.random()*5)`, which *executes now* and assigns the return
  value (nil) as the handler. Wrap it:
  `input[1].change = function() ansiblePlay(1, math.random()*5) end`
  *(brick, [25974/265](https://llllllll.co/t/crow-druid-scripts/25974/265))* **[v]**. Same trap with
  outputs: `input[1].change = output[1]( ar(...) )` runs the envelope once at load and stores nil.
- **Assigning to the handler's argument does nothing** — `function(x) x = math.min(input[1].volts, 5) end`
  overwrites the local `x` (which is discarded when the function returns) and never touches an
  output. "You will need to do something with `x` other than assigning to it" *(Galapagoose,
  [25974/200](https://llllllll.co/t/crow-druid-scripts/25974/200))*.
- **One handler per input per event type** — assigning `input[1].stream` twice silently replaces the
  first. "Each input can only use one handler function at a time" *(csboling,
  [25974/93](https://llllllll.co/t/crow-druid-scripts/25974/93))* **[v]**. Same for `.change`,
  `.window`, `.scale` — one registered function each; re-assigning replaces silently.

### 1.4 Naming a handler arg `volts` shadows `input[n].volts` / `output[n].volts`

The parameter name masks the `.volts` key on the input/output tables — `output[n].volts = volts`
then indexes the output table with the *voltage number* instead of the string. Use `v` (or any name
other than `volts`). Verbatim fix *(Galapagoose,
[25974/35](https://llllllll.co/t/crow-druid-scripts/25974/35))* **[v]**:

```lua
input[1].stream = function(v)
  for n=1,4 do
    output[n].volts = v -- because v == input[1].volts
  end
end
```

### 1.5 `for i,o in output do` is not valid Lua

It looks Python-ish and errors or misbehaves. Use `for i=1,4 do ... output[i] ...` (or
`for k,v in ipairs(output) do`). There is no inherent meaning to the letter `o` — `output[o]` with
`o` unbound is a nil-index error *(Galapagoose,
[25974/250](https://llllllll.co/t/crow-druid-scripts/25974/250), [25974/252](https://llllllll.co/t/crow-druid-scripts/25974/252))* **[v]**.

### 1.6 Multi-returns auto-truncate on assignment

`unique_id()` returns 3 values, but `local f = unique_id()` silently keeps the first. "In lua, if a
function returns multiple values, but the consumer of that function only uses 1, it will just take
the first element and discard the rest for you automatically" *(Galapagoose,
[25974/218](https://llllllll.co/t/crow-druid-scripts/25974/218))* — no need for
`local f, _, _ = unique_id()` *(vinc,
[25974/217](https://llllllll.co/t/crow-druid-scripts/25974/217))* **[v]**.

### 1.7 Overriding `input[n].stream` kills druid's auto-echo

By default druid echoes input voltages; redefining `input[n].stream` replaces that default action.
Re-add it if you still want the echo *(Galapagoose,
[25974/37](https://llllllll.co/t/crow-druid-scripts/25974/37))* **[v]**:

```lua
input[1].stream = function(v)
  crow.tell('stream',1,v) -- sends the voltage to druid
  for n=1,4 do
    output[n].volts = v -- because v == input[1].volts
  end
end
```

Alternative — leave the input event untouched and drive from a metro (verbatim, same post):

```lua
function set_outs()
  for n=1,4 do
    output[n].volts = input[1].volts -- need to query the input lib again
  end
end

function init()
  input[1].mode('stream', 0.01)
  input[2].mode('stream', 0.01)
  m = metro.init(set_outs, 0.01)
  m:start()
end
```

### 1.8 `n2v` returns a *function*, not a value (ASL helper)

**`attempt to perform arithmetic on a function value (global offset)`** — you used `n2v(...)` to
convert semitones to volts in plain Lua. `n2v` is an ASL builder that calculates at ASL runtime;
for direct V/oct math just divide by 12 ("12 semitones = 1 octave maps to 1 volt – not really any
point to having a dedicated function for this") *(csboling,
[25974/95](https://llllllll.co/t/crow-druid-scripts/25974/95), [25974/97](https://llllllll.co/t/crow-druid-scripts/25974/97))*.
**Flagged:** `n2v` has been missing from some firmware versions — old scripts that quantized with it
broke ("quantization was broken when `n2v` was removed") *(joshhh,
[25974/231](https://llllllll.co/t/crow-druid-scripts/25974/231))* — verify presence on your firmware
(`print(n2v)`) or use `.scale` quantization (§5.13).

### 1.9 `output[n].scale(...)` is setup, not assignment; values set at init are static

`output[4].scale({}, 12, 1.0) = r` is invalid — `.scale` is a setup function, not a voltage setter.
And `r = math.random() * 10.0 - 5.0` at file scope is computed **once** at load. Verbatim fix
*(Galapagoose, [25974/135](https://llllllll.co/t/crow-druid-scripts/25974/135))*:

```lua
function init()
  output[4].scale( {}, 12, 1.0 )
end

input[1].change = function()
  output[4].volts = r() -- call the random function again
end
```

with `function r() return math.random() * 10.0 - 5.0 end` defined for fresh values each call.

### 1.10 Spelling & call-style cheat sheet

| Wrong | Right | Note |
|---|---|---|
| `function int()` | `function init()` | §1.2 |
| `Metro.init(...)` | `metro.init(...)` | lowercase **[v]**; *(Galapagoose, [25937/51](https://llllllll.co/t/crow-strategies-just-friends/25937/51))* |
| `input[1].mode = 'change'` | `input[1].mode('change', 1, 0.1, 'rising')` | `mode` is a function, not a field *(Galapagoose, [25974/139](https://llllllll.co/t/crow-druid-scripts/25974/139))* |
| `Counter1[1]:start()` | `Counter1:start()` | metro object, not array *(koerby, [25974/253](https://llllllll.co/t/crow-druid-scripts/25974/253))* |
| `ar(0.5, 0.2, 5.0, linear)` | `ar(0.5, 0.2, 5.0, 'linear')` | shape must be a quoted string (§5.7) |
| `input[1].change = foo(1)` | `input[1].change = function() foo(1) end` | §1.3 |
| smart quotes `'change'` | plain `'` quotes | Word-processor quotes are a syntax error *(license, [25974/104](https://llllllll.co/t/crow-druid-scripts/25974/104))* |
| `math.rand()` | `math.random()` | standard Lua name **[v]** |
| `True` | `true` | Lua is case-sensitive *(Galapagoose, [25937/103](https://llllllll.co/t/crow-strategies-just-friends/25937/103))* |

### 1.11 Mental model: live code vs event handlers

"Whenever you type a line in druid or load a script into crow, each line gets executed right there…
there are also events that occur outside of your code… Think of `input[1].stream = ...` as similar to
filling in the screen for script 1 in Teletype" *(3-foot-1,
[25974/202](https://llllllll.co/t/crow-druid-scripts/25974/202))*; callbacks/event-handler framing
*(Galapagoose, [25974/200](https://llllllll.co/t/crow-druid-scripts/25974/200))*.

### 1.12 No "is a cable plugged in?" test

`input[n].stream == nil` tests for a *handler*, not hardware. There is no jack-detect; an unplugged
input reads ≈ 0 V. Approximate with a threshold like `v <= 0.01` *(vinc,
[25974/194](https://llllllll.co/t/crow-druid-scripts/25974/194); postsolarpunk,
[25974/195](https://llllllll.co/t/crow-druid-scripts/25974/195))* **[v]**.

---

## 2. Time, clocks, metro & timeline

Details of the timing APIs live in [api-time.md](api-time.md); this section is what breaks.

### 2.1 `clock.get_beats()` / `clock.get_beat_sec()` are FUNCTIONS

Bare use in arithmetic — `clock.sync(1/4) ... x = beats * 2` where `beats = clock.get_beats` —
raises **`attempt to perform arithmetic on a function value`**. Call them: `clock.get_beats()`,
`clock.get_beat_sec()`. `clock.get_beat_sec(n)` returns *n beats* in seconds (no arg = 1 beat).
`clock.tempo`, by contrast, **is** a property: `clock.tempo = 120` *(Justmat,
[25974/223](https://llllllll.co/t/crow-druid-scripts/25974/223))* **[v]**.

### 2.2 `clock.sync` drifts forever with additive swing offsets

Symptom: swung clock gets progressively unsteady. Cause: "clock.sync looks for multiples of the sync
value relative to the origin beat. So simple ratios work fine… Meanwhile `1/4 + 0.00001`… will shift
further and further forward in time every beat." Fix — sync the unswung beats, *sleep* the swung
ones (verbatim, *(Galapagoose,
[25974/187](https://llllllll.co/t/crow-druid-scripts/25974/187), answering ktnsft
[25974/186](https://llllllll.co/t/crow-druid-scripts/25974/186)*)* **[v]**:

```lua
clock.sleep(clock.get_beat_sec()/4 + swing_in_seconds)
```

i.e. `clock.sync(1/4)` on even beats, `clock.sleep(clock.get_beat_sec()/4 + swing)` on odd beats.

### 2.3 `clock.cleanup()` is a sledgehammer (and `timeline.cleanup()` is the same function)

**[v]** `clock.cleanup()` kills **ALL** clock coroutines, resets `clock.tempo` to 120, and clears
`transport` / `tempo_change` handlers. `timeline.cleanup()` is literally `clock.cleanup()`. To stop
one routine, use `clock.cancel(coro_id)` on the handle returned by `clock.run` *(Galapagoose,
[25974/144](https://llllllll.co/t/crow-druid-scripts/25974/144))*.
**Flagged:** some community scripts comment `clock.cleanup(playbass)` expecting single-coroutine
stop *(onoma, [25937/164](https://llllllll.co/t/crow-strategies-just-friends/25937/164))* — treat
cleanup as global and use `clock.cancel` for one.

### 2.4 `timeline` reference-page typos & vestigial methods **[v]**

- `timeline.queue` is a **function**: `timeline.queue():loop{ ... }` — the paren-less form in the
  reference page is a typo.
- `timeline.launch_default` defaults to `1`.
- `:once()` appears in the method table but **errors** when called (vestigial) — use `:loop`/`:count`.
- String timeline events: only `'reset'` is implemented; anything else is ignored.
- Nested timelines aren't hot-swappable.
- `score` `'reset'` re-zeroes to the *current* zero + the reset event's timestamp (it does not rewind
  to wall-clock 0).

### 2.5 metro: 8 slots, lowercase `init`, pool exhaustion

- **`Metro.init: nothing available`** — all 8 metro slots are taken. Typical cause: a home-rolled
  `delay()` helper that allocates a metro per call and leaks them *(cosmicsoundexplorer,
  [25937/49](https://llllllll.co/t/crow-strategies-just-friends/25937/49))*; also seen when the name
  is miscased (`Metro`/`Merto`) — the API is lowercase `metro.init` *(Galapagoose,
  [25937/51](https://llllllll.co/t/crow-strategies-just-friends/25937/51))* **[v]**. Fix: free metros
  (`metro.free(m.id)`) after use, share metros via counters, or use the builtin `delay` (§2.6).
  More timing duties beyond 8: use `clock` coroutines instead *(juhbuba,
  [25974/182](https://llllllll.co/t/crow-druid-scripts/25974/182))*.
- Minimum metro period: 0.0005 s **[v]**.
- **Time changes apply at next wakeup only** — quoting `metro.lua`: "NB: metro time isn't applied
  until the next wakeup. this is true even if you are setting time from the metro event" *(capleniz,
  [25974/77](https://llllllll.co/t/crow-druid-scripts/25974/77))* **[v]**. On older firmware, setting
  `metro.time` at all could retrigger the metro into running flat-out; fixed mid-2020 *(junklight,
  [25974/79](https://llllllll.co/t/crow-druid-scripts/25974/79))* — see §8.5.
- If using `delay` and `metro` together, initialize with `metro.init`, never direct `metro[n]`
  indexing **[v]**.

### 2.6 `delay(action, time [, repeats])` — action FIRST **[v]**

The builtin takes the *function* first (it receives the iteration index as its argument), then the
time in seconds, then optional repeats (total runs = 1 + repeats). It returns a cancelable coro id.
**Trap:** passing a *call result* as the first arg — `phase1 = output[1]()` then `delay(phase1, 0.5)` —
means no delay happens and the pulse fires immediately: "the first argument to `delay` needs to
itself be a function (currently it's the result of a function call)" *(Galapagoose,
[25974/225](https://llllllll.co/t/crow-druid-scripts/25974/225), fixing Neither-cloud
[25974/224](https://llllllll.co/t/crow-druid-scripts/25974/224))*. Working fix, verbatim *(Neither-cloud,
[25974/227](https://llllllll.co/t/crow-druid-scripts/25974/227))*:

```lua
function init()
  input[1].mode('change', 1.0, 0.1, 'rising')
  output[1].action = pulse(0.1, 5, 1)
end

input[1].change = function()
  delay(delayed_pulse, 0.5 )
end

function delayed_pulse()
  output[1]()

end
```

**Argument-order warning:** older community scripts define their own `delay(time, action)` wrapper —
the *opposite* order. Verbatim pre-builtin shim *(Galapagoose,
[25937/48](https://llllllll.co/t/crow-strategies-just-friends/25937/48))* (also the source of the
metro-leak `Metro.init: nothing available` reports):

```lua
-- add this function to your script
-- (to be included in crow firmware in future)
function delay( time, action )
    local function devent()
        action() -- this line executes the delayed function
        metro.free(dmetro.id)
    end
    local dmetro = metro.init( devent, time, 1 )
end

-- delay the call that reads the input voltage by 3ms
-- the old line is 'wrapped' in a function
-- this stops it from being executed immediately
input[1].change = function(s)
    delay( 0.003, function() ii.jf.play_note(input[2].volts,5) end )
end

function init()
    ii.pullup(true)
    ii.jf.mode(1)
    input[1].mode('change', 1, 0.1, 'rising')
end
```

If a script defines this shim, its `delay(time, action)` calls are *not* the builtin's signature —
check which one you're calling.

### 2.7 `input[n].mode('clock', …)` raises no event and excludes `'change'` **[v]**

- While an input is in `'clock'` mode there is **no** Lua input event — nothing like
  `input[n].change = ...` will ever fire for it. `'clock'` and `'change'` modes are mutually
  exclusive on the same input ("you can't use `clock` and `change` mode at the same time")
  *(Galapagoose, [25974/146](https://llllllll.co/t/crow-druid-scripts/25974/146), answering koerby
  [25974/253](https://llllllll.co/t/crow-druid-scripts/25974/253))* **[v]**. Want both? Poll
  `input[n].volts` from a clock routine (threshold trick), or use `'change'` mode and derive tempo
  from edge timing (§2.10).
- While `'clock'` mode is active, `clock.tempo` is *measured from the input* and assigning it does
  nothing; `clock.start()` / `clock.stop()` revert the source to internal **[v]**.
- Also don't do `input[2].change = clock.run(...)` — `clock.run` returns a handle, and in clock mode
  there is no change event anyway *(Galapagoose,
  [25974/146](https://llllllll.co/t/crow-druid-scripts/25974/146))*.
- Syntax (undocumented in older refs): `input[1].mode('clock')`, or customized
  `input[1].mode('clock', 1.0, 0.2)` = 1 V threshold, 0.2 V hysteresis *(Galapagoose,
  [25974/155](https://llllllll.co/t/crow-druid-scripts/25974/155))* — exists since firmware v3, and
  `input[2].mode('clock', 1, 'rising')` is a valid mode mirroring norns' crow-clock handling, though
  it was missing from the docs *(Galapagoose,
  [25974/148](https://llllllll.co/t/crow-druid-scripts/25974/148))*.

### 2.8 Sampling speed, latency, and why Lua loops can't win **[v]**

- Input sampling is 1.5 kHz; fastest sane stream period ≈ 1.5 ms *(community-measured minimum stream
  interval 0.0014583 s, [25974/182](https://llllllll.co/t/crow-druid-scripts/25974/182))*.
- `input[n].stream` events arrive ~2–3 ms after the fact: "a quick test using `time()` on
  `input.stream`… was always 2-3ms later than expected" *(license,
  [25974/112](https://llllllll.co/t/crow-druid-scripts/25974/112))*.
- Metros & input streams run on **hardware timers** with sub-microsecond accuracy: "Both Metros & the
  input streams run on hardware timers on the chip. They have sub-microsecond accuracy." Lua
  coroutines are cooperative, not parallel — you cannot out-time the hardware *(Galapagoose,
  [25974/111](https://llllllll.co/t/crow-druid-scripts/25974/111))*; coroutines are for breaking up
  long work, not for precision *(synthetivv,
  [25974/113](https://llllllll.co/t/crow-druid-scripts/25974/113))*.

### 2.9 Multiple independent metros drift apart on tempo changes

Symptom: polyrhythmic metronome jitter; phase "goes way out of whack" when BPM changes *(license,
[25974/112](https://llllllll.co/t/crow-druid-scripts/25974/112))*; "I tried using multiple dynamic
metros at different speeds… some of them ended up with delays I didn't want to handle" *(capleniz,
[25974/87](https://llllllll.co/t/crow-druid-scripts/25974/87))*. Cause: time changes land at each
metro's next wakeup (§2.5), so per-metro rounding accumulates differently. Fix: one fast metro (the
greatest common divisor rate) plus phase/threshold counters *(synthetivv,
[25974/113](https://llllllll.co/t/crow-druid-scripts/25974/113); Galapagoose,
[25974/114](https://llllllll.co/t/crow-druid-scripts/25974/114))* — "Always off-course, always
correcting" *(license,
[25974/115](https://llllllll.co/t/crow-druid-scripts/25974/115))*.

### 2.10 Deriving tempo from an external clock: `time()` deltas

`time()` is **milliseconds** **[v]**. Verbatim tempo follower *(forrest,
[25974/48](https://llllllll.co/t/crow-druid-scripts/25974/48))*:

```lua
last_tick = time()
bpm = 0

function init()
  input[1].mode('change',3.0,0.1,'rising')
  metro[1].time = 1 
  metro[1]:start()
  metro[1].event = function(c) print(string.format("metro[1] bpm: %d", math.floor(bpm))) end
end

input[1].change = function(state)
  local elapsed = time() - last_tick
  bpm = 60000/elapsed
  metro[1].time = elapsed * 0.001
  last_tick = time()
end
```

**Gotchas:** `lfo()` and ASL times are **seconds** — convert ms→s (÷1000) *(Galapagoose,
[25974/139](https://llllllll.co/t/crow-druid-scripts/25974/139))*; and after a long idle,
`time() - last_tick` is huge — guard the first/after-gap measurement *(vinc,
[25974/257](https://llllllll.co/t/crow-druid-scripts/25974/257))*. Averaging the last N intervals
smooths jitter *(voidstar's modes.lua approach, referenced in
[25974/48](https://llllllll.co/t/crow-druid-scripts/25974/48))*.

### 2.11 `sequins` + `timeline`: nested modifiers don't bubble; triplets drift

`:all():every(4)` inside a doubly-nested sequins is ignored. Alternative, verbatim *(Galapagoose,
[25974/212](https://llllllll.co/t/crow-druid-scripts/25974/212))*:

```lua
sequins{ sequins{1}:count(15), sequins{1/3}:count(3) }
```

"because the timing in `timeline` is not synchronized after it launches, it's best to use a fraction
for triplets to avoid drift across many repetitions" *(same post)* **[v]**.

### 2.12 Timeout / watchdog-to-idle pattern

Verbatim reusable timeout (cancel + respawn on every input event) *(Galapagoose,
[25974/144](https://llllllll.co/t/crow-druid-scripts/25974/144))*:

```lua
function init()
  input[2].mode('clock', 1, 'rising')
  timeout = clock.run(function() end) -- empty clock routine to avoid clock.cancel error on first call
  input[2].change = function()
    -- first do your "on change" stuff here
    clock.cancel(timeout) -- stop the current timeout
    timeout = clock.run(function() -- start the new timeout
        clock.sleep(1) -- set timeout in seconds
        clock.cleanup()
      end)
  end
end
```

`clock.cancel` on an already-dead handle errors — hence the empty starter routine. (A generalized
`set_timeout(coro, fn)` variant appears in the same post; its second block has a stray `)` in
`clock.cancel(coro))` — delete one when pasting.)

---

## 3. Watchdog & CPU load

### 3.1 **`CPU timed out.`** — the 2-second watchdog **[v]**

A Lua loop that never yields for ~2 s trips the CPU watchdog: the script errors out and prints
`CPU timed out.` Any `while true do` **must** contain `clock.sleep(...)` or `clock.sync(...)`.
Busy-waiting for a value, audio-rate while-loops, or a huge computation (see §3.2) all hit this.
Related: "Has anyone succeeded using infinite loops? … I crashed my crow yesterday and had to put it
in bootloader mode to recover it" *(license,
[25974/108](https://llllllll.co/t/crow-druid-scripts/25974/108))* — recovery in §6.

### 3.2 **`event queue full!`** — handler too expensive for its rate

Symptom prints `event queue full!` in druid. Cause: event handlers queue faster than they drain —
e.g. heavy math (`math.atan2`) on two 1 ms streams: "Each millisecond, two calls… are put on the
event queue… If crow can't get through both calls in less than one millisecond… the queue will
slowly grow, until it is full" *(3-foot-1,
[25974/206](https://llllllll.co/t/crow-druid-scripts/25974/206), explaining vinc
[25974/205](https://llllllll.co/t/crow-druid-scripts/25974/205))* **[v]**. Fixes (all from that post):

- slow the stream (1 ms → 10 ms is usually fine),
- register the handler on **one** input only (if it reads both jacks anyway),
- drive from a 10 ms `metro` instead of streams,
- early-return when values haven't changed, or use `window` mode to fire only on change.

Verified summary: "`event queue full!` = handler too expensive for its rate — slow the stream, use
one input, or drive from a 10ms metro" **[v]**.

### 3.3 `cputime()` **[v]**

Returns main-loop iterations per DSP block — **higher = more idle**. It is not a percentage and not
seconds; use it only for relative load comparisons.

### 3.4 crow is not a computer: audio-rate ambitions fail

"crow isn't intended to be audio-rate" — clocking scripts at extreme rates can lock them up until
firmware/script updates *(crim,
[25974/2](https://llllllll.co/t/crow-druid-scripts/25974/2))*; continuous PLL-style tracking exceeds
what 1.5 kHz sampling allows *(TheProblemBelief,
[25974/291](https://llllllll.co/t/crow-druid-scripts/25974/291))*.

---

## 4. ASL & outputs

### 4.1 Resource limits: **`ERROR: no stages left!`** / `no sequences left!` / `no dynamic slots remain` **[v]**

Per output ASL: max **16** `to` stages, **8** sequences, **8** stages per sequence, **40** dynamic
slots. Exceeding any prints the corresponding line in druid. **Functions inside an ASL** error with
**"Do you have a function in your ASL? Replace it with dyn."** — pass runtime-varying values as
`dyn{name = initial}` and mutate them (§4.3), not as Lua functions/closures. ASL resource errors
appear as **plain lines** in druid output (no stack trace) — scroll for them before assuming a Lua
bug **[v]**.

### 4.2 **`casl_setdynamic (number expected, got nil)`** — dyn used before any ASL exists

Setting `output[n].dyn.X = v` before the output has ever run an ASL action fails
(`bad argument … to 'casl_setdynamic' (number expected, got nil)`) because the dyn slots are created
by the action. "The error arises when trying to set an `output[n].dyn._` element when it hasn't been
initialized by an ASL constructor" *(Galapagoose,
[25974/256](https://llllllll.co/t/crow-druid-scripts/25974/256))* **[v]**. Fix: run the action once
in `init()` before touching dyn — verbatim pattern (note: the `to(...)` line in the original post is
missing one closing paren — add it when pasting) *(Galapagoose,
[25974/252](https://llllllll.co/t/crow-druid-scripts/25974/252))*:

```lua
-- this value is a "time" not a "rate", hence the name
slew_time = 1 -- start with an initial value

function rand_volts()
  for i=1,4 do
    --define an ASL and execute it immediately
    output[i](to(math.random(-5,5), dyn{slew=slew_time})
  end
end

function set_slew(volts)
  slew_time = volts/5
  for i=1,4 do
    -- set the slew time of currently-in-motion outputs
    output[i].dyn.slew = slew_time
  end
end

function init()
  input[1].change = rand_volts
  input[2].stream = set_slew

  rand_volts() -- create initial ASL state to avoid errors with dyn

  input[1].mode{mode = "change", direction = "rising"} -- only trigger on rising edge
  input[2].mode("stream", 0.01) -- scan every 10ms
end
```

A `dyn` assigned a **nil** at load (e.g. `output[1].dyn.time = voltage` where `voltage` isn't set
yet) fails the same way — `bad argument #3 to 'casl_setdynamic' (number expected, got nil)`. Give
the variable a starting value first *(dani_derks,
[25974/238](https://llllllll.co/t/crow-druid-scripts/25974/238))* **[v]**.

### 4.3 **`casl_getdynamic (number expected, got nil)`** — handler args passed into actions

`bad argument #2 to 'casl_getdynamic' (number expected, got nil)` with a stack trace through
`field 'change'` — caused by `input[1].change = function(s) output[1](s) end`, i.e. passing the
change-event direction flag into the ASL bang. Fix: `output[1]()` — actions take no event args
*(Galapagoose, [25974/236](https://llllllll.co/t/crow-druid-scripts/25974/236), fixing koerby
[25974/234](https://llllllll.co/t/crow-druid-scripts/25974/234))* **[v]**. Rule: **never pass handler
args into `output[n](...)`**.

### 4.4 dyn values snapshot at stage start — mid-stage changes are ignored

Symptom: changing `output[1].dyn.time` mid-ramp doesn't change the ramp's speed; a very slow LFO
takes seconds to respond to a frequency increase. Cause: "the start of each ASL segment snapshots the
value" *(dessertplanet,
[25974/281](https://llllllll.co/t/crow-druid-scripts/25974/281))*; "the asl `to()` stage has to
complete first then it'll update" *(postsolarpunk,
[25974/283](https://llllllll.co/t/crow-druid-scripts/25974/283))* **[v]**. Workaround: break the
motion into many short `to()` micro-stages via a stepped `dyn` — verbatim ramp oscillator
*(EvanEverfield, [25974/284](https://llllllll.co/t/crow-druid-scripts/25974/284))*:

```lua
output[1](loop{
    to(dyn{pos=hi}:step(dyn{dir = -1} * step_size):wrap(lo, hi), (dyn{t = t} / steps))
})
```

("the jump from one edge to other is fast enough at medium speeds" but visible at very slow rates;
stepping an integer counter × step_size avoids float accumulation error — same post.)

### 4.5 `times()` doesn't retrigger inside `loop{}` **[v]**

The `times(n)` ASL construct relies on a counter that isn't reset per loop iteration. Rebuild it with
`asl._while` — verbatim *(postsolarpunk,
[25974/284](https://llllllll.co/t/crow-druid-scripts/25974/284))*:

```lua
output[1](loop{
    asl._while( dyn{counter = steps+1}:step(-1):wrap(0, steps+1), {
        to(dyn{pos=hi}:step(-1 * step_size):wrap(lo, hi-step_size), dyn{t=(t/steps)})
    }),
    -- do stuff after each cycle
})
```

### 4.6 Zero-time stages can lock the ASL **[v]**

A `loop{}` whose stages all take 0 s spins forever. A pause implemented via `asl._if` needs a
matching tiny-time branch: "The second `_if` seems necessary to prevent lockup from an infinitely
looping sequence with zero time passing each loop" *(postsolarpunk,
[25974/284](https://llllllll.co/t/crow-druid-scripts/25974/284))* — verbatim pause/resume:

```lua
lo = -5.0
hi = 5.0
t=0.5
step_size = 0.01
steps = math.floor((hi - lo) / step_size) - 1

output[1](loop{
    asl._if(dyn{run=1},
        {to(dyn{pos=hi}:step(dyn{dir = -1} * step_size):wrap(lo, hi), (dyn{t = t} / steps))}
    ),
    asl._if((1 - dyn{run=1}),
        {to(dyn{pos=hi}, 0.001)}
    )
})

-- pause the oscillator
output[1].dyn.run = 0
-- resume the oscillator
output[1].dyn.run = 1
```

### 4.7 Shape arguments must be STRINGS, matched on the first character **[v]**

- `ar(0.5, 0.2, 5.0, 'linear')` — unquoted `linear` is a nil global → syntax/runtime error
  *(license, [25974/104](https://llllllll.co/t/crow-druid-scripts/25974/104))*; smart/curly quotes
  are a hard syntax error.
- Matching is **case-insensitive on the first character only**: `'log'` needs at least `'lo'`;
  `'expo'`, `'exp'`, `'exponential'` all resolve. **Unknown strings silently become `'linear'`** —
  a typo changes your envelope shape with no error **[v]**.

### 4.8 `output[n].volts = x` interrupts any running ASL **[v]**

Setting `.volts` cancels the current action (and slews/shape-izes that jump). To **freeze** an output
mid-action, set destination = current jack voltage (the readback is the *jack*, not the target):

```lua
output[n].volts = output[n].volts -- set destination to current voltage
```

verbatim *(Galapagoose,
[25974/74](https://llllllll.co/t/crow-druid-scripts/25974/74))* — "This works because
`output[n].volts` returns the current voltage at the output jack (not the destination)."

### 4.9 Read-back race: `print(output[n].volts)` shows the OLD value **[v]**

Immediately after `output[n].volts = 4.0`, a read returns the previous jack voltage — reads lag one
io cycle (this lag is *not* the slew time). Read again a moment later (or trust your assignment).

### 4.10 Set `.action` once in `init()`, bang with `output[n]()` **[v]**

Reassigning `output[n].action = ar(...)` inside every trigger is wasteful and (combined with §4.2)
the source of dyn errors. "I don't think the output action setup needs to be ran more than once…
put the `output[n].action = ar()` setup stuff in the init function and then just call `output[1]()`
in your change function" *(Justmat,
[25974/105](https://llllllll.co/t/crow-druid-scripts/25974/105))*; *(license,
[25974/104](https://llllllll.co/t/crow-druid-scripts/25974/104))*.

### 4.11 `slew` is a time in seconds, applied at assignment **[v]**

The field is `output[n].slew = seconds` (not `.slewrate`); slew and volts only take effect when
`.volts` is assigned, so setting slew continuously from a stream does nothing until the next
assignment. For slew that changes *while* voltage is in motion, use an ASL `to(v, dyn{slew=…})` as
in §4.2 *(Galapagoose,
[25974/250](https://llllllll.co/t/crow-druid-scripts/25974/250), [25974/252](https://llllllll.co/t/crow-druid-scripts/25974/252))*.

### 4.12 `lfo()` helper

`output[n]( lfo( cycle_time, level, shape ) )` — time in **seconds**, shape optional (default
`'sine'` since firmware 1.0.3) *(Galapagoose,
[25974/71](https://llllllll.co/t/crow-druid-scripts/25974/71))* **[v]**. LFOs of related rates stay
synced by re-banging all at once — verbatim *(same post)*:

```lua
function init()
  for n=1,4 do
    output[n]( lfo( 1.0 / n ) ) -- results in 1s, 500ms, 333ms, 250ms
  end
end

function reset() -- to synchronize lfos
  for n=1,4 do
    output[n]()
  end
end
```

**Flagged:** one doc example passes `'now'` as the shape *(Galapagoose,
[25974/65](https://llllllll.co/t/crow-druid-scripts/25974/65))* — unexplained; probably a typo for
`'saw'`. Don't rely on it.

### 4.13 V/oct arithmetic: 1 V = 1 octave, not 12

`output[3].volts = input[2].volts - 12.0` drops **12 octaves**, not one. "the `.volts` signals are
all direct voltage levels where 1V = 1 octave. the dividing by 12 thing from many other contexts
isnt needed" *(Galapagoose,
[25974/267](https://llllllll.co/t/crow-druid-scripts/25974/267))* **[v]**. Semitones are `/12`
(3 semitones = `3/12`), and 0 V = middle C = 261.63 Hz (docs say "C3").

### 4.14 Self-integrating outputs (runaway / FM-like garbage)

`output[1].volts = output[1].volts + pitch` inside a stream **integrates** — every 5 ms it adds
again: "This looks to me like you are basically integrating the value of output[1]… You might want to
keep a separate variable for storing the sum… then use a single stream handler or a separate metro"
*(csboling, [25974/91](https://llllllll.co/t/crow-druid-scripts/25974/91))* — the reported symptom is
"weird fm like pitch output instead of a clean melody" *(Wisdom.Water,
[25974/90](https://llllllll.co/t/crow-druid-scripts/25974/90))*.

### 4.15 Quantize with `.scale`, and JI gotchas with `s.index`

`input[n].mode('scale', {…}, temperament, scaling)` / `output[n].scale(…)` (since firmware 2.0) do
the quantizing for you *(csboling,
[25974/91](https://llllllll.co/t/crow-druid-scripts/25974/91))* **[v]**. With just-intonation scales
(`'ji'` temperament): `s.volts` is the correct voltage; `s.index` is the raw table index and
`myscale[s.index]` is just the ratio **float** (`9/8 == 1.125`) — "Lua doesn't have type-level
support for ratios". Convert with `just12(ratio)` or `justvolts(ratio)` *(Galapagoose,
[25974/184](https://llllllll.co/t/crow-druid-scripts/25974/184))* **[v]**.

### 4.16 Calibration (outputs out of tune)

Run `cali.lua` from the bowery repo; calibration is saved to flash after a `---- PASS ----` and is
factory-set on most units. Expect roughly ±5 cents over 7.5 V; negative voltages show worse
non-linearity (two-point extrapolation from 0 V & 2.5 V) *(Galapagoose,
[25974/270](https://llllllll.co/t/crow-druid-scripts/25974/270))* **[v]**.

---

## 5. ii / i2c

Full command semantics: [ii-overview.md](ii-overview.md), [ii-just-friends.md](ii-just-friends.md),
[ii-wslash.md](ii-wslash.md). This section is what goes wrong on the wire.

### 5.1 No per-device handles — index at call time **[v]**

`ii.mod[n]`, `ii.jf[2]`, `ii.wtape[2]` all re-point a **shared** table and return the same object.
`local t = ii.jf[2]` then `t.play_note(...)` later silently sends to whichever address was set
*last*. Always write `ii.jf[2].command(...)` at the moment of the call. Same class of bug: randomly
choosing a device inside a loop produces flickering behavior — "The 'flick' is when both random
choices select the same device but the random value for `.play` or `.record` is different. You
probably just want to do `ii.wtape[n].record`" *(Galapagoose,
[25974/218](https://llllllll.co/t/crow-druid-scripts/25974/218))*.

### 5.2 Getters never return values — replies arrive in `.event` **[v]**

`Fader1 = ii.faders.get(1)` is **always nil**. Getters fire an async ii query; the reply is delivered
to the device's `.event` handler, and **with no handler installed current firmware does not forward
replies at all** — queries vanish silently. Pattern, verbatim *(desolationjones,
[25937/88](https://llllllll.co/t/crow-strategies-just-friends/25937/88))*:

```lua
ii.faders.event = function (e, data, id)
	ii.jf.play_note(data, 10)
	--crow.tell('fb', e, data, id)
end
```

("The getter function is triggered the associated event, and pass the gotten data along to it"
*(desolationjones, [25937/88](https://llllllll.co/t/crow-strategies-just-friends/25937/88))*; the
same conclusion after a `nil` hunt: "the get function does not return a value" *(vinc,
[25974/240](https://llllllll.co/t/crow-druid-scripts/25974/240))*.) Handler signature:
`function(e, value)` with `e.name`, `e.arg`, `e.device` — see
[ii-just-friends.md](ii-just-friends.md) §7.

### 5.3 **`getter not found for 'x'`** is a local typo guard **[v]**

An unknown getter *name* raises this Lua error **even with no device attached** (the name is checked
against crow's device definition). Silence, by contrast, just means no reply arrived (absent device,
no handler, or busy bus).

### 5.4 Writes to absent devices are silently ignored **[v]**

No ACK check — `ii.jf.play_note(...)` with no JF on the bus fails without any error. Presence probe:
install a printing `.event` handler, issue one cheap `get()`, wait — reply = present, silence = absent
*(from hardware testing; see also [ii-just-friends.md](ii-just-friends.md) §7)*.

### 5.5 Pull-ups, fast mode, and bus leaders **[v]**

- `ii.pullup(true)` in `init()` when crow is the first/only bus device (most community scripts do
  this).
- `ii.fastmode(true)` ≈ 4× bus speed.
- Only **one leader should transmit at a time**: two crows (+ Teletype) on one bus collide if they
  talk simultaneously. "using 2 leaders on the same i2c bus is gonna give you all sorts of headaches.
  it might work sometimes, but mostly wont" *(Justmat,
  [25974/259](https://llllllll.co/t/crow-druid-scripts/25974/259))*; "as long as you aren't sending
  i2c stuff at the same time it works as expected" *(Justmat,
  [25974/261](https://llllllll.co/t/crow-druid-scripts/25974/261))* — stagger traffic or use two
  buses. A second crow transmitting can also hit the "wrong" JF when per-device addressing isn't
  used (§5.1) *(Duksos,
  [25974/260](https://llllllll.co/t/crow-druid-scripts/25974/260))* — target `ii.jf[2]` explicitly
  *(Justmat, [25974/263](https://llllllll.co/t/crow-druid-scripts/25974/263))*.

### 5.6 i2c saturation: throttle sweeps and polls **[v]**

Continuous parameter sweeps (glissandi, feedback sweeps) can saturate the bus. Throttle to ~15 fps
with the `fnl()` framed-slew helper from
[Maps S2E3](https://gist.github.com/trentgill/ec9fcf2d2583bb75934b7130baf231bb#file-maps_s2e3-lua-L50)
*(Galapagoose, [25937/105](https://llllllll.co/t/crow-strategies-just-friends/25937/105))*; pace
getter polling to ~1.5 ms between requests. Recipe collection: [community-techniques.md](community-techniques.md).

### 5.7 Getter polling safety

JF panel queries: values lag by ~two triggers, and polling at only ~5 Hz crashed a JF in one report
*(ObscBySds, [25937/119](https://llllllll.co/t/crow-strategies-just-friends/25937/119))* — treat
panel getters as occasional, not per-note. Also noted there: JF crashing mid-performance when
triggers arrive fast while polling getters.

### 5.8 Teletype-era integer scaling (Geode args) — resolved from source

JF's ii protocol was written for Teletype's integers ("super large integer scaling system") *(voidstar,
[25937/32](https://llllllll.co/t/crow-strategies-just-friends/25937/32))*. **Resolved against
`lua/ii/jf.lua` + `lib/ii.c`**: crow types `play_note`/`play_voice` args as **`s16V`** and always
multiplies floats by **1638.3** (`II_TT_VOLT`) before sending, while JF's Geode engine reads the
**raw wire integers** as `divs`/`repeats` (Teletype `JF.NOTE 4 3` sends wire ints). So Geode args
must be **divided by 1638.3** on the way out of crow — the hack is current, not historical:

```lua
ii.jf.play_note(4/1638.3, -1/1638.3)   -- divs=4, repeats=-1 (forever)
```

**Why the `retune` experiments failed**: `ii.jf.retune` args are **`s8` raw integers 0–127 — no
scaling at all**. `ii.jf.retune(2, 2/1638, 1/1638)` *(dani_derks,
[25937/43](https://llllllll.co/t/crow-strategies-just-friends/25937/43))* truncates to
`retune(2, 0, 0)` — hence "no response from JF at all" *(iiii,
[25937/44](https://llllllll.co/t/crow-strategies-just-friends/25937/44))*. Never divide `s8`
args (retune, tick, trigger channels, quantize, mode, …). Full treatment:
[ii-just-friends.md](ii-just-friends.md) §5.

---

## 6. ii / i2c — Just Friends

Full JF command reference: [ii-just-friends.md](ii-just-friends.md).

### 6.1 **`mode(1)` latches** — "JF stops tracking pitch CV" is almost always this

`ii.jf.mode(1)` puts JF into Synthesis/Geode and it **stays there** until `mode(0)` or a power cycle
— meanwhile JF ignores its front-panel V/oct. Symptom reports: "JF is stuck in some sort of i2c
limbo… its not tracking 1v/oct via Ansible" *(ludvista29,
[25937/62](https://llllllll.co/t/crow-strategies-just-friends/25937/62), [25937/65](https://llllllll.co/t/crow-strategies-just-friends/25937/65))*;
"JF continues to run in the same mode" after switching crow scripts *(vagueintl,
[25937/107](https://llllllll.co/t/crow-strategies-just-friends/25937/107))*.

- **Cause**: any script (crow, norns, Teletype, Ansible) that sent `mode(1)` and never sent `mode(0)`.
  "It's probably a norns script telling JF to enter synth mode when it boots up" *(Galapagoose,
  [25937/67](https://llllllll.co/t/crow-strategies-just-friends/25937/67))*; "the issue is not that JF
  is receiving i2c messages, it's that it has been sent a `crow.ii.jf.mode(1)` message, then norns
  has gone off to do something else, but hasn't told JF to return to normal operation" *(Galapagoose,
  [25937/75](https://llllllll.co/t/crow-strategies-just-friends/25937/75))*.
- **Fix**: send `ii.jf.mode(0)` (or `crow.ii.jf.mode(0)` from norns/maiden). norns scripts should do
  it in `cleanup()` — verbatim *(Galapagoose,
  [25937/70](https://llllllll.co/t/crow-strategies-just-friends/25937/70))*:

  ```lua
  function cleanup()
    if params:get("output") == 5 then
      crow.ii.jf.mode(0)
    end
  end
  ```

  Better: also send it when the *output param changes away* from JF *(crim,
  [25937/81](https://llllllll.co/t/crow-strategies-just-friends/25937/81))*.
- **Emergency**: pull the ii cable and power-cycle JF — "Disconnecting i2c worked!" *(ludvista29,
  [25937/69](https://llllllll.co/t/crow-strategies-just-friends/25937/69))*; power cycling JF alone
  resets to normal *(hghrlearning,
  [25937/110](https://llllllll.co/t/crow-strategies-just-friends/25937/110))*. There is **no**
  front-panel reset incantation *(3-foot-1,
  [25937/156](https://llllllll.co/t/crow-strategies-just-friends/25937/156))*.
- Ansible also re-latches JF at startup if it was set to JF output when powered down *(3-foot-1,
  [25937/111](https://llllllll.co/t/crow-strategies-just-friends/25937/111))*; unexpected mode
  changes from multiple drivers discussed *(SimonKirby,
  [25937/112](https://llllllll.co/t/crow-strategies-just-friends/25937/112))*.
- **Prevention**: put `ii.jf.mode(1)` in `init()` of scripts that need it (so state is deterministic)
  and `mode(0)` in anything that hands JF back. Note `JF.NOTE` on Teletype does *not* auto-switch
  modes if `JF.MODE 1` was never sent *(Justmat,
  [25937/113](https://llllllll.co/t/crow-strategies-just-friends/25937/113))*.

### 6.2 Dropped notes on simultaneous `play_note` (older firmware)

Symptom: 6-note chords lose pitch data / drop voices in Synthesis mode. Cause: JF "loses track of
pitch data when it receives simultaneous messages over i2c" — worked around by spacing messages
~10 ms *(dani_derks,
[25937/1](https://llllllll.co/t/crow-strategies-just-friends/25937/1))*. norns workaround, verbatim
(same post):

```lua
jf_note_spacer = metro.init()
jf_note_spacer.time = 0.01
jf_note_spacer.count = 1
```

with `jf_note_spacer.event = jf_spacer`, `jf_note_spacer:stop()` at startup, first voice sent
immediately, second voice sent from the 10 ms metro. A smarter Max-side queue (instant first note,
10 ms stagger only within 10 ms of the last) *(Rodrigo,
[25937/15](https://llllllll.co/t/crow-strategies-just-friends/25937/15))*.
**Firmware note:** proper polyphony was fixed in crow's ii layer — "This will be fixed in the next
Crow firmware release. I've got proper polyphony working already" *(Galapagoose,
[25937/39](https://llllllll.co/t/crow-strategies-just-friends/25937/39))* — see §7.4.

### 6.3 `get('trigger')` semantics (voice-free detection is a myth)

- Returns `1`/`0`, not `true`/`false` *(Galapagoose,
  [25937/103](https://llllllll.co/t/crow-strategies-just-friends/25937/103))*.
- In Synthesis mode it **stays 1 even when the voice is silent**: "it's always 1 when in synthesis
  mode… `trigger` returns whether the channel is in motion – … when the oscillator goes silent it is
  still oscillating (just being muted)" *(Galapagoose,
  [25937/105](https://llllllll.co/t/crow-strategies-just-friends/25937/105))* — you cannot use it as
  a free-voice detector; use `play_note`'s allocator or roll your own with `play_voice` *(same post)*.
- `get('trigger', 0)` returns a **count** of active channels, not a boolean
  ([ii-just-friends.md](ii-just-friends.md) §10).

### 6.4 JF freezes on first ii command — crow 4.x + JF 4.2 + PSU interaction

Symptom: "JF doesn't seem to be reacting to anything… If I then run `ii.jf.get('curve')`, All of the
lights stop running / JF is frozen until I restart my case" *(Bigtime_Snacks,
[25937/139](https://llllllll.co/t/crow-strategies-just-friends/25937/139))*; "after a couple of notes
sent from Ableton" *(lleicamun,
[25937/91](https://llllllll.co/t/crow-strategies-just-friends/25937/91))*; getters return nil and
`jf.mode` appears to do nothing *(zbs,
[25937/92](https://llllllll.co/t/crow-strategies-just-friends/25937/92))*.

- **Cause** (maintainer): "certain power supplies cause the i2c driver on JF to get in a bad state
  before the module even starts running – then when you send the first message it gives up and
  crashes" *(Galapagoose,
  [25937/148](https://llllllll.co/t/crow-strategies-just-friends/25937/148))*; suspected trigger is
  "the fast trigger feature of JF 4.2 and the new clock features in crow 4.0" *(3-foot-1,
  [25937/145](https://llllllll.co/t/crow-strategies-just-friends/25937/145))*; reported with
  Intellijel cases/power *(JD800,
  [25937/149](https://llllllll.co/t/crow-strategies-just-friends/25937/149); mreid,
  [25937/150](https://llllllll.co/t/crow-strategies-just-friends/25937/150))*.
- **Fix**: revert crow to **v3.0.1** *(3-foot-1,
  [25937/142](https://llllllll.co/t/crow-strategies-just-friends/25937/142))* — "ooh, that does appear
  to have fixed the issue" *(Bigtime_Snacks,
  [25937/143](https://llllllll.co/t/crow-strategies-just-friends/25937/143))*; and/or JF **4.1.2**
  (no fast-trigger); and/or JF calibration/factory reset per the
  [JF latest-version page](https://www.whimsicalraps.com/pages/jf-latest-version) *(maaark,
  [25937/93](https://llllllll.co/t/crow-strategies-just-friends/25937/93))* — "After that calibration
  it's at least temporarily working" *(zbs,
  [25937/95](https://llllllll.co/t/crow-strategies-just-friends/25937/95))*. Reinstalling JF firmware
  also temporarily helps *(kev,
  [25937/140](https://llllllll.co/t/crow-strategies-just-friends/25937/140))* — but note "after I
  power down my case and turn it back on JF reverts back to freezing" *(Bigtime_Snacks,
  [25937/141](https://llllllll.co/t/crow-strategies-just-friends/25937/141))*.
- Sanity check when debugging: `ii.jf.trigger(1, 1)` / `ii.jf.trigger(1, 0)` should blink the trig
  LED *(kev, [25937/140](https://llllllll.co/t/crow-strategies-just-friends/25937/140))*.

### 6.5 `ii.jf.retune` descriptor bug (older crow)

`ii.jf.retune(1, x, 1)` **silenced the channel** instead of retuning (Teletype's `JF.TUNE 1 x 1`
worked); `retune(0,0,0)` reset fine; scaled args didn't help *(dani_derks,
[25937/56](https://llllllll.co/t/crow-strategies-just-friends/25937/56); iiii,
[25937/42](https://llllllll.co/t/crow-strategies-just-friends/25937/42))*. Cause: "the i2c descriptor
is wrong for Just Friends on crow. I corrected it on my end now, so this command will be fixed in the
next crow update" *(Galapagoose,
[25937/57](https://llllllll.co/t/crow-strategies-just-friends/25937/57))* — fixed in later crow
(§7.3). Independent limit: **retune numerator/denominator max 127** *(foldedworlds,
[25937/167](https://llllllll.co/t/crow-strategies-just-friends/25937/167))*.

### 6.6 Ratio-based pitch without `retune`

`retune` affects INTONE-relative ratios, not per-note pitch. For ratio tunings in Synthesis mode use
`justvolts` *(Galapagoose,
[25937/117](https://llllllll.co/t/crow-strategies-just-friends/25937/117))*:

```lua
ii.jf.play_note( justvolts(3/2), 3)
```

Pitch/level units: pitch is V/oct relative to JF's base, `0.0` = middle C = 261.63 Hz (docs "C3");
level in volts, `5.0` = 5 Vpp *(voidstar,
[25937/37](https://llllllll.co/t/crow-strategies-just-friends/25937/37); csboling,
[25937/86](https://llllllll.co/t/crow-strategies-just-friends/25937/86))* **[v]**.

### 6.7 CV sampling races when crow → JF plays external sequencers

Symptom: "it doesn't always sample the correct cv… I feel like it may be sampling too soon" — the
gate edge arrives before the pitch CV has settled *(cosmicsoundexplorer,
[25937/46](https://llllllll.co/t/crow-strategies-just-friends/25937/46))*. Fix: delay the read by a
few ms so the source settles — see the `delay` shim in §2.6 ("delay the call that reads the input
voltage by 3ms") *(Galapagoose,
[25937/48](https://llllllll.co/t/crow-strategies-just-friends/25937/48))* **[v]**.

### 6.8 What ii *can't* do with JF

- JF does not transmit its output signals over ii; at most 2 of its 7 outs are capturable via crow's
  inputs *(desolationjones,
  [25937/59](https://llllllll.co/t/crow-strategies-just-friends/25937/59); Galapagoose,
  [25937/60](https://llllllll.co/t/crow-strategies-just-friends/25937/60))*.
- Panel knobs/switches are **read-only** over ii (getters); you can't set TIME/INTONE/etc. remotely
  *(PiecesOfQuiet,
  [25937/132](https://llllllll.co/t/crow-strategies-just-friends/25937/132))*.
- Gliss mid-envelope has no built-in support ("JF is already at the limit of processing power in
  synthesis mode") *(Galapagoose,
  [25937/105](https://llllllll.co/t/crow-strategies-just-friends/25937/105))*.
- Sustain/voice-targeting via the M4L device was unstable — "my tests with sustain were all stable up
  to a point, but then modules would hang" *(dani_derks,
  [25937/84](https://llllllll.co/t/crow-strategies-just-friends/25937/84))*.

---

## 7. ii / i2c — W/ & W/Del

Command reference: [ii-wslash.md](ii-wslash.md).

### 7.1 W/Del hardware never answers `get()` **[v]**

Silence from `ii.wdel.get(...)` / `ii.wtape.get(...)` does **not** mean the device is absent — the
hardware simply doesn't respond to getters. Don't poll for presence here.

### 7.2 **`ii.wtape.get(...)` can hang crow permanently**

"I executed `ii.wtape[1].get( 'play' )` once with no issue, but cannot execute the same command
again, it didn't response. I need to turn off the power (because `^^reset` would no longer work too).
If I executed `ii.wtape[2].get( 'play' )`, it never return, and would get stuck too. But I can do
things like `ii.wtape[2].record(1)`" *(vinc,
[25974/220](https://llllllll.co/t/crow-druid-scripts/25974/220))* — **avoid repeated wtape getters;
a power cycle may be the only recovery** **[v]** (matches one hardware-testing report).

### 7.3 `loop_active` getter on wtape — version-dependent

"after I read the wtape git page… it seems there's no getter for `loop_active`" *(vinc,
[25974/217](https://llllllll.co/t/crow-druid-scripts/25974/217))* — true **at the time**: the getter
was **added to crow 2025-01-22** (commit `2cff9f59`, "add missing wtape loop_active getter"). On
firmware from that date `get('loop_active')` works; on older firmware it returns nil. If you must
support both, track loop state in Lua rather than gating on the getter.

### 7.4 Randomized device targeting causes flick

Sending `play`/`record` to `ii.wtape[math.random(2)]` inside a per-device loop double-sends to one
unit with conflicting values ("a flick (play light blink once and unlit)"). Use the loop index:
`ii.wtape[n].record(1)` *(Galapagoose,
[25974/218](https://llllllll.co/t/crow-druid-scripts/25974/218))* **[v]**.

---

## 8. druid, upload, USB & recovery

Protocol details: [druid-and-protocol.md](druid-and-protocol.md).

### 8.1 One process owns the serial port **[v]**

druid, Max `[crow]`, norns, or any MCP/serial client — only one can hold the port. Close druid before
running anything else that talks to crow.

### 8.2 `r` vs `u` vs `p` **[v]**

- `r file.lua` = **run from RAM** — instant, but lost on power cycle. Best for iteration (faster and
  no flash wear) *(Galapagoose,
  [25937/97](https://llllllll.co/t/crow-strategies-just-friends/25937/97))*.
- `u file.lua` (or `u` with no name = current script) = **upload to flash** — survives power cycling.
  The classic "my patch is gone after I turned the case off" is running `r` and never `u` *(vagueintl,
  [25937/96](https://llllllll.co/t/crow-strategies-just-friends/25937/96); Galapagoose,
  [25937/97](https://llllllll.co/t/crow-strategies-just-friends/25937/97))*.
- `p` / `^^p` prints the **flash** copy — not whatever you last `r`-an. Don't use it to debug a RAM
  run **[v]**.
- After a failed `r`: wait and retry, then `^^kill` to clear the running script, then `r` again **[v]**.

### 8.3 **`!chunk too long!`** — REPL line limits **[v]**

Pasting into druid's REPL: chunks must stay under **2048 bytes** (keep < 2000 to be safe). Larger
pastes must go through `u`/`r` as files. Total user script size: **16380 bytes** (~700 lines) **[v]**
(some docs say 8 kB — older community posts mention "the 8kb limit for script upload" *(damon,
[25974/32](https://llllllll.co/t/crow-druid-scripts/25974/32))*; treat **16 kB** as the working limit;
"Script length is an arbitrary 16k characters (out of a possible 64k held in RAM)" *(juhbuba,
[25974/182](https://llllllll.co/t/crow-druid-scripts/25974/182))*). Large uploads also get flakier —
"uploading to crow seems to be getting more unstable the bigger the file gets" *(crim,
[25974/25](https://llllllll.co/t/crow-druid-scripts/25974/25))*.

### 8.4 **`<eof> expected`** — missing trailing newline (older firmware)

Symptom on upload: `[string "eval"]:13: 'end' expected (to close 'function' at line 11) near <eof>`
or `<eof> expected near 'end'` + `User script evaluation failed.` Cause: file lacks a final
newline/blank line *(damon,
[25974/32](https://llllllll.co/t/crow-druid-scripts/25974/32); Oootini,
[25974/36](https://llllllll.co/t/crow-druid-scripts/25974/36))*; tabs vs spaces and CRLF vs LF are
**irrelevant** *(csboling,
[25974/34](https://llllllll.co/t/crow-druid-scripts/25974/34); Galapagoose,
[25974/35](https://llllllll.co/t/crow-druid-scripts/25974/35))*. Fixed in later crow+druid; on old
versions add a trailing newline and, if a script still won't load after a failed attempt, run `^^c`
first *(Oootini,
[25974/36](https://llllllll.co/t/crow-druid-scripts/25974/36))*.

### 8.5 USB framing: 64-byte writes need a trailing `\n` **[v]**

If a USB write's length is an **exact multiple of 64 bytes** and doesn't end with `\n`, the link can
die (macOS/Windows). Always terminate host→crow chunks with a newline. Full protocol notes:
[druid-and-protocol.md](druid-and-protocol.md).

### 8.6 `^^r`, `^^c`, `^^F`, and bootloader recovery **[v]**

- `^^r` (reboot) re-enumerates USB — the host must reconnect after ~2 s.
- `^^c` clears a crashing script so you can upload again.
- `^^F` / `^^First` reloads the built-in First demo script *(csboling,
  [25974/60](https://llllllll.co/t/crow-druid-scripts/25974/60))* (it's compiled into the firmware,
  not in the Bowery *(csboling,
  [25974/58](https://llllllll.co/t/crow-druid-scripts/25974/58))*).
- If druid can't find crow at all: force bootloader (ii header centre pin to GND while powering on)
  and run the erase-userscript script, or flash directly:
  `dfu-util -a 0 -s 0x08020000 -R -D crow.bin -d ,0483:df11`.
- A crashed crow (watchdog, bad loop) may need `^^kill`, `^^r`, or power cycle — one report needed
  full power-off after a wtape getter hang (§7.2).

### 8.7 OS / host quirks **[v]**

Windows 7 needs the Zadig CDC driver; on Linux your user must be in the `dialout` group; iPadOS
cannot run druid (Apple doesn't expose the serial device) *(dani_derks,
[25974/177](https://llllllll.co/t/crow-druid-scripts/25974/177))*.

### 8.8 druid's working directory matters

`r script.lua` resolves paths relative to where druid was **launched** — `cd` to your scripts folder
first or autocomplete will list the wrong directory and uploads fail with syntax/file errors
*(csboling,
[25974/124](https://llllllll.co/t/crow-druid-scripts/25974/124), [25974/126](https://llllllll.co/t/crow-druid-scripts/25974/126))*:

```
$ cd /path/to/my/bowery
$ druid
r boids.lua
```

CLI automation works too: `druid download` / `druid upload file.lua` *(driedstr,
[25974/285](https://llllllll.co/t/crow-druid-scripts/25974/285))*.

### 8.9 Live-coding via the REPL is first-class

Anything you type executes immediately — `output[3].volts = 3/12`, re-scale inputs
(`input[1].mode('scale', maj)`), or call pre-baked functions for complex changes. Design scripts with
small callable functions so the REPL can drive them *(dani_derks,
[25974/175](https://llllllll.co/t/crow-druid-scripts/25974/175), [25974/179](https://llllllll.co/t/crow-druid-scripts/25974/179))*;
`public` variables exist mainly for norns as remote UI — plain globals + REPL suffice with druid
*(same)*. norns users can also drive crow from maiden: `crow.send("...")` *(jlmitch5,
[25937/71](https://llllllll.co/t/crow-strategies-just-friends/25937/71))*.

---

## 9. Firmware version issues

Check your versions first: `^^v` in druid. Cross-reference: [technical.md](technical.md).

| Issue | Versions | Status / fix |
|---|---|---|
| **i2c broken for many devices** | crow 4.0.1 | Fixed in 4.0.2+ (core i2c rewrite; betas on the [crow discord](https://discord.gg/wSQekv5Ssf)) *(dani_derks, [25937/135](https://llllllll.co/t/crow-strategies-just-friends/25937/135))* |
| JF freezes on first ii command | crow 4.x + JF 4.2 "fast trigger" + some PSUs (Intellijel reported) | Revert crow to v3.0.1 and/or JF 4.1.2; JF recalibrate — §6.4 |
| `ii.jf.retune(n, x, 1)` silences channel | older crow (bad JF i2c descriptor) | Fixed after 25937/57; update crow — §6.5 *(Galapagoose, [25937/57](https://llllllll.co/t/crow-strategies-just-friends/25937/57))* |
| JF drops notes on simultaneous `play_note` | crow pre-fix / older JF | Fixed in crow ii layer — §6.2 *(Galapagoose, [25937/39](https://llllllll.co/t/crow-strategies-just-friends/25937/39))* |
| `metro.time` changes retrigger the metro | firmware before commit `f491781` (mid-2020) | Fixed; update — §2.5 *(junklight, [25974/79](https://llllllll.co/t/crow-druid-scripts/25974/79); capleniz, [25974/85](https://llllllll.co/t/crow-druid-scripts/25974/85))* |
| `math.random` repeats the same sequence | firmware before the 2020+ update | Restart the script as a workaround; fixed in later firmware *(ringsandrocks, [25974/141](https://llllllll.co/t/crow-druid-scripts/25974/141); Galapagoose, [25974/142](https://llllllll.co/t/crow-druid-scripts/25974/142))* |
| Missing trailing newline → `<eof> expected` | old crow/druid | Fixed in later crow+druid — §8.4 *(Galapagoose, [25974/35](https://llllllll.co/t/crow-druid-scripts/25974/35))* |
| `n2v` missing / renamed | some versions after ~2021 | Old scripts (e.g. Ornithopters) lose quantization — §1.8 *(joshhh, [25974/231](https://llllllll.co/t/crow-druid-scripts/25974/231))* |
| Bowery scripts bit-rot against new APIs | e.g. quantizer.lua vs `.scale` (crow v2+) | Use the merged `main` branch fixes *(brick, [25974/174](https://llllllll.co/t/crow-druid-scripts/25974/174); dani_derks, [25974/175](https://llllllll.co/t/crow-druid-scripts/25974/175))*; script-vs-firmware mismatch also produced stack tracebacks in Ornithopters on 2.0.0 *(j450n, [25974/137](https://llllllll.co/t/crow-druid-scripts/25974/137))* |
| `input[n].mode('clock')` undocumented | since firmware v3 | Works; documented later — §2.7 *(Galapagoose, [25974/155](https://llllllll.co/t/crow-druid-scripts/25974/155))* |
| Dynamic metro time, `sequins` v2, `timeline` | crow v4 features | Not available when JF forces a v3 rollback — v3 loses these libraries (works on norns regardless) *(ObscBySds, [25937/144](https://llllllll.co/t/crow-strategies-just-friends/25937/144))* |

---

## 10. Quick symptom index

| Symptom / exact error string | Go to |
|---|---|
| `attempt to call a nil value (field 'change')` | §1.3 (called instead of passed) |
| `attempt to call a nil value` (other field) | §1.10 spelling table; §1.3 |
| `attempt to perform arithmetic on a function value` | §2.1 (`clock.get_beats`), §1.8 (`n2v`) |
| `attempt to perform arithmetic on a nil value` | §1.1 (top-level code before init) |
| `bad argument #2 to 'casl_getdynamic' (number expected, got nil)` | §4.3 |
| `casl_setdynamic (number expected, got nil)` / `bad argument #3 to 'casl_setdynamic'` | §4.2 |
| `CPU timed out.` | §3.1 |
| `event queue full!` | §3.2 |
| `ERROR: no stages left!` / `no sequences left!` / `no dynamic slots remain` | §4.1 |
| `Do you have a function in your ASL? Replace it with dyn.` | §4.1 |
| `Metro.init: nothing available` | §2.5 |
| `!chunk too long!` | §8.3 |
| `<eof> expected` / `'end' expected (to close 'function' …) near <eof>` | §8.4 |
| `getter not found for 'x'` | §5.3 |
| `User script evaluation failed.` | §1.1, §8.4 |
| `nil` where a value was expected | §1.1, §1.3, §5.2 (ii getters) |
| Handler never fires (mode set, no events) | §1.2 (`init` typo), §2.7 (clock mode has no events) |
| Handler fires but output doesn't move | §1.3 (assigned to arg), §4.10 (forgot `output[n]()`), §4.8 |
| Script gone after power cycle | §8.2 (`r` vs `u`) |
| `p` shows a different script than expected | §8.2 (flash vs RAM) |
| Upload syntax error on a file that looks fine | §8.4 (trailing newline), §1.10 (smart quotes), §8.8 (cwd) |
| druid can't find crow | §8.6 (bootloader/dfu), §8.1 (port busy), §8.7 (drivers) |
| Link dies mid-upload | §8.5 (64-byte writes), §8.3 (chunk size) |
| i2c command reaches the wrong device | §5.1 (cached handle) |
| ii getter always nil / silence | §5.2 (async `.event`), §5.4 (absent device), §7.1 (W/ never answers) |
| ii write does nothing, no error | §5.4 (absent device), §6.1 (JF not in mode 1), §5.5 (bus collision) |
| i2c works then wedges / bus stops | §5.5 (two leaders), §5.6 (saturation), §7.2 (wtape getter hang) |
| `ii.wtape.get(...)` freezes crow entirely | §7.2 |
| JF ignores front-panel pitch (V/oct dead) | §6.1 (`mode(1)` latch) |
| JF freezes on first ii command | §6.4 (crow 4.x + JF 4.2 + PSU) |
| JF drops notes / wrong pitches on chords | §6.2 (10 ms stagger / firmware) |
| `ii.jf.get('trigger')` always 1 | §6.3 |
| `ii.jf.retune(x, n, d)` silences the channel | §6.5 (firmware descriptor bug), §6.5 (num/den ≤ 127) |
| JF knobs lag / JF crashes while polling getters | §5.7 |
| `ii.jf.get('curve')` etc. freezes JF lights | §6.4 |
| Swing clock drifts unsteadily | §2.2 |
| Metros drift apart when tempo changes | §2.9 |
| Metro ignores new `.time` | §2.5 |
| `delay` does nothing / fires immediately | §2.6 (arg order; function vs call) |
| `Metro.init: nothing available` after using a delay shim | §2.5 + §2.6 (leaked metro slots) |
| dyn change ignored while ASL runs | §4.4 |
| `times()` never retriggers inside `loop{}` | §4.5 |
| ASL locks up / crow freezes on a looping action | §4.6 (zero-time stages) |
| Envelope shape wrong, no error | §4.7 (unknown shape string → linear) |
| `output[n].volts = x` kills my envelope | §4.8 |
| `print(output[n].volts)` prints stale value | §4.9 |
| Output slews unexpectedly on jump | §4.8 (`.volts` applies slew/shape) |
| Slew not changing mid-motion | §4.11 |
| Pitch wildly off (octaves wrong) | §4.13 (1 V = 1 octave) |
| Quantized output wrong with JI ratios | §4.15 (`s.index` vs `s.volts`) |
| Oscillator/sequencer out of tune | §4.16 (calibration) |
| Output integrates / runaway "FM" pitch | §4.14 |
| External clock sampling reads wrong CV | §6.7 (settle delay), §2.10 (ms vs s) |
| `lfo()` rate wrong by 1000× | §2.10 (`time()` is ms; `lfo` wants seconds) |
| Unplugged input can't be detected | §1.12 |
| Crow locks up at high clock rates | §3.4, §3.1 |
| `math.random` repeats the same sequence | §9 (firmware bug; restart) |
| Random device flickers on W/ | §7.4 |
| JF stuck in synth mode with no laptop | §6.1 (power cycle / unplug ii) |
| Two crows fight over one bus | §5.5 |
| `unique_id()` unpack error worries | §1.6 (auto-truncation) |

---

## 11. Flagged / unresolved claims

Carry these as caveats; don't treat as settled:

- **`ii.wtape` `loop_active` getter**: **resolved** — it did not exist at the time of the thread
  report *(vinc, [25974/217](https://llllllll.co/t/crow-druid-scripts/25974/217))* but was **added
  2025-01-22** (crow commit `2cff9f59`, "add missing wtape loop_active getter"). On firmware from
  that date `get('loop_active')` works; older firmware/`ii_help.txt` captures return nil. Track
  loop state locally if you must support old firmware.
- **`Metro.init: nothing available` casing** — the firmware message starts with `Metro.` even though
  the API is lowercase `metro`; user reports also show the typo `Merto.init`. Search for the string
  without relying on casing *(cosmicsoundexplorer,
  [25937/49](https://llllllll.co/t/crow-strategies-just-friends/25937/49))*.
- **`clock.cleanup(handle)`** — community comments imply per-coroutine cleanup *(onoma,
  [25937/164](https://llllllll.co/t/crow-strategies-just-friends/25937/164))*; verified behavior is
  global cleanup. Use `clock.cancel` for one coroutine until proven otherwise.
- **`n2v`** — described as a live ASL helper returning a function *(csboling,
  [25974/95](https://llllllll.co/t/crow-druid-scripts/25974/95))* yet reported removed from later
  firmware *(joshhh, [25974/231](https://llllllll.co/t/crow-druid-scripts/25974/231))* — probe your
  firmware before using it.
- **`lfo(0.1, 5, 'now')`** in a maintainer example *(Galapagoose,
  [25974/65](https://llllllll.co/t/crow-druid-scripts/25974/65))* — **actually valid**: `'now'` is a
  documented shape (jump instantly, then wait). (If you go shape-hunting: the working set is
  `linear sine log expo now wait over under rebound` — there is no `'saw'`/`'tri'`/`'square'`;
  unrecognized strings silently become linear.)
- **Stream latency 2–3 ms** *(license,
  [25974/112](https://llllllll.co/t/crow-druid-scripts/25974/112))* vs 1.5 kHz sampling **[v]** —
  both true but different layers (event delivery vs ADC rate); expect ~1.5–3 ms end-to-end.
- **Script size limit** — **resolved**: 16380 bytes (PR #396); the 8 kB figure in older docs/posts
  is stale. REPL chunk limit remains 2048 bytes.
- **Geode `/1638.4` scaling** — **resolved from firmware source** (see §5.8): `play_note`/
  `play_voice` args are `s16V` (always ×1638.3), Geode reads raw wire ints, so **divide by 1638.3**;
  the failed retune experiments used `s8` args which don't scale at all.
- **Galapagoose `set_timeout` variant** *(25974/144)* contains a stray `)` — see §2.12 note.
- **Galapagoose `rand_volts` snippet** *(25974/252)* is missing a closing paren on the `to(...)` line
  — see §4.2 note. Copied verbatim per policy.

---

## 12. Sources

- crow: druid scripts (thread dump): https://llllllll.co/t/crow-druid-scripts/25974
- crow strategies: just friends (thread dump): https://llllllll.co/t/crow-strategies-just-friends/25937
- Individual posts cited inline with author + `thread/N` permalinks.
- Official monome docs (reference, scripting, druid, technical, first, teletype), crow firmware
  source (`lua/input.lua`, `lua/output.lua`, `lua/metro.lua`, `lua/asl.lua`, `lua/ii/*.lua`), Just
  Type / Just-Friends docs, and hardware testing notes from a serial-protocol crow MCP client.
- Companion references: [api-time.md](api-time.md), [ii-overview.md](ii-overview.md),
  [ii-just-friends.md](ii-just-friends.md), [ii-wslash.md](ii-wslash.md),
  [druid-and-protocol.md](druid-and-protocol.md), [technical.md](technical.md),
  [community-techniques.md](community-techniques.md).
