# The environment — script lifecycle, host I/O, `public`, `hotswap`, `cal`, randomness, globals

Everything around the jacks and the clock: how scripts load, how to talk to the USB host (druid /
crow-mcp / norns), how to expose live parameters (`public`), live-code safely (`hotswap`),
self-calibrate (`cal`), and the full globals list.

---

## 1. Script lifecycle

```lua
-- top-level code runs ONCE at load, BEFORE init() (see troubleshooting.md)
myvar = 0

function init()      -- called after the environment is ready: do setup here
  input[1].mode('stream', 0.1)
  output[1].slew = 0.01
end

function cleanup()   -- optional: called before a new script replaces this one
end
```

- crow stores **one** script in flash and auto-runs it at power-on (see `technical.md`).
- `init()` must be spelled exactly; extra args are ignored/invalid.
- **`crow.reset()`** — deactivate input modes, zero outputs & slews, free all metros,
  `ii.reset_events`, reset ii-follow slots, `public.clear()`, `clock.cleanup()`,
  `hotswap.cleanup()`, restore default handlers. (It also suppresses the running user script.)
- After `init()` completes crow sends **`^^ready()`** to the host.

## 2. Talking to the host

**Nothing is reported automatically** — the host only sees what you send:

```lua
print(x)                 -- plain line -> druid/crow-mcp output
tell(event, <args>)      -- custom event -> ^^event(arg1, ...) ; event name + up to 4 values
                         --   (more raises: too many args to tell.)
quote(table)             -- string representation of a table, for tell()ing it
```

### Built-in `^^` events (sent by default handlers)

```
^^stream(ch, volts)   ^^change(ch, state)   ^^window(ch, win, dir)
^^scale(ch, note)     ^^volume(ch, val)     ^^peak(ch, val)   ^^freq(ch, hz)
^^output(ch, volts)   (from output[n].query())
^^ready()             ^^version('x.y.z')    ^^identity('0x…')
^^pub … / ^^pupdate … (public sync)
```

Custom handlers that replace defaults can re-echo manually: `tell('stream', 1, v)`.

On **norns**, crow events surface through the norns API (below). Full wire details:
`druid-and-protocol.md`.

## 3. `public` — variables synced with the USB host

Expose live-tweakable state (norns gets a free param UI; druid/crow-mcp can read/write):

```lua
public{name = init_value}     -- declare (== public.name = init_value)
public.name = 'jin'           -- set
_ = public.name               -- get
```

**Supported types**: numbers, strings, flat tables, sequins (first layer only).
**Not supported**: functions, coroutines, nested tables.

### Metadata (chainable; each returns the var for chaining)

**NB**: `:range` must be called with **parentheses** — `:range(-5, 10)`. The docs' brace form
`:range{0.001, 10}` passes a table as `min` and breaks clamping (source: the chain fn is
`function(p, m, x) p.min, p.max = m, x`).

```lua
public{volts = 0}:range(-5, 10)                     -- clamp range (20 -> 10)
public{seconds = 1}:range(0.001, 10):type('exp')    -- exponential scaling hint for hosts
public{count = 0}:type('int')                       -- integer steps
public{readonly = 'hello'}:type('@')                -- host read-only
public{readint = 22}:type('@int')                   -- read-only int
public{myslide = 0}:range{-5, 10}:type('slider')    -- slider UI hint
public{myopt = '+'}:options{'+', '-', '*', '/'}     -- enumerated options (invalid sets IGNORED)
public{speed = 1.0}:action(update_speed)            -- fn(value) on remote (and local) update
public.xrange(min, max)                             -- undocumented: range + 'exp' in one
```

### View (throttled jack-state streaming to the host)

```lua
public.view.all([state])        -- all inputs & outputs (no arg = on)
public.view.input[n]([state])   -- n = 1,2
public.view.output[n]([state])  -- n = 1..4
```

### Discovery & remote sync (host-side calls)

```lua
public.discover()                          -- prints all declared vars as lua chunks,
                                           --   terminated by key '_end'
public.update(name, value [, subkey])      -- remote write; triggers :action; subkey writes
                                           --   public.name[subkey]
public.clear()                             -- clear all + tell host '_clear' (undocumented)
```

Sequins stored in `public` broadcast playhead moves to the host as index updates. Clamping applies
to table elements too. From norns, prefix host-side calls with `norns.crow.`.

## 4. `hotswap` — live-coding state preservation

Re-assigning a `sequins` or `timeline` through `hotswap` **keeps its playhead/position**:

```lua
hotswap.seq = sequins{1, 2, 3}
hotswap.seq()          --> 1
hotswap.seq()          --> 2
hotswap.seq = sequins{4, 5, 6, 7}   -- playhead preserved
hotswap.seq()          --> 6        -- picks up mid-sequence

hotswap.tt = timeline.loop{sequins{3,3,2}, {ii.jf.play_note, sequins{0,4,7,11}/12, 2}}
hotswap.tt = timeline.loop{sequins{3,2,2,1}, {ii.jf.play_note, sequins{0,4,7,10}/12, 2}}
```

- Only **sequins** and **timeline** values are understood; anything else prints
  `hotswap does not know this type`.
- Implementation: sequins → `:settable()`; timeline → `:hotswap()`. State of previously existing
  elements is preserved (element-type swaps included).
- `hotswap.cleanup()` clears the registry (called by `crow.reset()`).
- Idiomatic aliases: `s = sequins; tl = timeline; hs = hotswap`.

## 5. `cal` — self-calibration

Normally done at the workshop; the official script is
[bowery/cali.lua](https://github.com/monome/bowery/blob/main/cali.lua) (patch out1→in2 when
prompted; PASS/FAIL printed; ±5 cents over 7.5 V expected; saved on PASS).

```lua
cal.save()                 -- save current calibration to flash
cal.source(chan)           -- configure the output->input multiplexer;
                           --   chan accepts {1,2,3,4,'gnd','2v5'}

_ = cal.input[n].offset    -- get/set input trim
cal.input[n].offset = _
_ = cal.input[n].scale
cal.input[n].scale = _
_ = cal.output[n].offset   -- get/set output trim
cal.output[n].offset = _
_ = cal.output[n].scale
cal.output[n].scale = _
```

## 6. Randomness

```lua
math.random()              -- TRUE hardware random (on-chip TRNG): 0.0 .. 1.0
math.random(max)           -- integer 1 .. max inclusive
math.random(min, max)      -- integer min .. max inclusive

math.srandomseed(seed)     -- seed the pseudo-RNG (alias of math.randomseed)
math.srandomseed(math.random() * 2^31)  -- typical
math.srandomseed(unique_id())           -- per-hardware-unit reproducible sequence
math.srandom()             -- pseudo-random 0.0 .. 1.0 (deterministic after seeding)
math.srandom(5)            -- integer 1 .. 5  (one doc says 0..5 — firmware semantics say 1..max)
math.srandom(3, 10)
```

`math.random` is genuine entropy — every call is independent. `math.srandom*` is stock Lua
`math.random` behind a seed — use it for reproducible generative pieces.

## 7. Globals reference

```lua
crow.reset()                    -- full reset (see §1)

tell(event, <args>)             -- custom host event, max 4 args (crow.tell = tell)
quote(...)                      -- lua-chunk string of args (tables via tell)

_, _, _ = unique_id()           -- 3 ints unique to this crow (use all 3)
time()                          -- MILLISECONDS since power-on
cputime()                       -- main loops per dsp block (last 8ms); higher = more idle

justvolts(fraction [, offset])  -- just ratio -> V/oct volts (1/1 -> 0V)
                                --   accepts a table -> table; offset is a just ratio
just12(fraction [, offset])     -- just ratio -> 12TET semitones
hztovolts(freq [, reference])   -- Hz -> volts; DEFAULT reference 261.63 Hz (middle C) = 0V;
                                --   2-arg form: log2(freq/ref)
```

Math: `hztovolts(f)` = `log2(f / 261.63)`; `justvolts(r)` = `log2(r)` (+ offset);
`just12(r)` = `log2(r) * 12`.

Undocumented but real (firmware source): `get_out(chan)` / `get_cv(chan)` (instant
`^^output`/`^^stream` tells), `send_usb(str)`, `sys_bootloader()`, `debug_usart(str)`, `nop_fn`.

## 8. norns interaction

- norns auto-detects crow over USB; **only one crow** is supported by the norns API (multiple
  crows work fine from druid/ii).
- On norns, **prefix every crow command with `crow.`**: `crow.output[1].volts = 3.33`,
  `crow.ii.jf.mode(1)`, `crow.input[1].mode('scale', dorian)`.
- Host-side public calls use `norns.crow.`: `norns.crow.public.discover()`.
- Reset/clear from norns: `crow.reset()` / `crow.clear()` (sends `^^r` / `^^c`).
- Send raw Lua to crow: `crow.send("output[1].volts = 3")` — the string must be a single Lua
  chunk. (Max/M4L equivalent: `tell_crow "output[1].volts = 3"`.)
- Firmware updates from norns: `;install https://github.com/monome/fledge`.
- norns study scripts: https://github.com/monome/crow-studies.
- From norns, this whole skill's Lua works unchanged behind the `crow.` prefix — but scripts
  loaded *onto crow* never use the prefix (it's a host-side namespace).

## 9. Crow as an ii follower (Teletype calls THIS crow)

See `ii-overview.md` for the full slot table (`ii.self.call`/`ii.self.query`, `call1..4`,
`query0..3`). Quick shape:

```lua
ii.self.call2 = function(x, y)
  output[1].volts = x
  output[2](true)
end

ii.self.query1 = function(x)
  return input[x].volts
end
-- caller side: ii.crow.call2(x, y) / ii.crow.get('query1', x)
```
