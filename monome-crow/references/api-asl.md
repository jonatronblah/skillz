# ASL — A Slope Language

ASL is crow's declarative language for **output actions**: journeys through voltage over time.
Every output runs one ASL action at a time; the built-in actions (`lfo`, `adsr`, … — see
`api-input-output.md`) are just pre-built ASL. This file is the full language so you can build
your own.

## Assigning an action

```lua
-- as a table of stages:
output[1].action = { to(0, 1), to(5, 1), to(0, 0.5) }
output[1]({ to(1, 0.1), to(-1, 0.1) })     -- set + start in one call

-- as a string of ASL Lua (norns-compatible):
output[1].action = "{ to(0,0), to(5,0.1) }"

-- then:
output[1]()            -- (re)start
output[1](true)        -- directive: gate/sustain
output[1](false)       -- directive: release
output[1]()            -- nil: restart
output[1]('unlock')    -- release a lock{} construct
output[1].done = function() end   -- fires when the journey completes
```

## Primitive: `to(destination, time, shape)`

```lua
to( 1, 1 )            -- go to 1 V over 1 s (defaults: volts 0, time 1, shape 'linear')
to( 2, 2 )
to( 3, 3, 'log' )     -- any shape string from api-input-output.md ('expo', 'sine', ...)
```

An ASL is a table of stages, left to right, top to bottom (a "journey"). `output[n].volts = x`
mid-action restarts the action as `to(x, slew, shape)` — see the gotchas.

## Constructs

```lua
loop{ <asl> }                     -- restart the sequence when complete (forever)
held{ <asl> }                     -- freeze at the end until the false/'release' directive
lock{ <asl> }                     -- ignore directives until the sequence completes
times( count, { <asl> } )         -- repeat `count` times
asl._if( pred, { <asl> } )        -- execute only if pred is truthy
asl._while( pred, { <asl> } )     -- loop while pred is truthy
```

Predicates for `_if`/`_while` are values or dynamics (see below) — e.g.
`asl._if(dyn{run=1}, { to(5, 0.1) })`.

### Directives (exact semantics, `lua/asl.lua` + `lib/casl.c`)

| call | effect |
|---|---|
| `output[n](true)` or `(1)` | set `_held=1` + restart — run the `held{}`/attack body, pause at sustain |
| `output[n](false)` or `(0)` | set `_held=0` + release — jump to the release phase |
| `output[n]()` (nil) | restart from the top (program counters only) |
| `output[n]('unlock')` | release a `lock{}` early |

**Critical**: a freshly described `held{}`/`adsr` action has `_held = 0`, so a **bare
`output[1]()` skips the held body entirely** — drive gates/envelopes with
`output[1](true)` / `output[1](false)`, not bare restarts. Also: restart does **not** clear
dynamics — `times()` counters and `mutable` state persist across `output[n]()` calls; only
**re-describing** (re-assigning `.action` or calling `output[n](asl_table)`) clears them.
The `asl._if` predicate is tested as `<= 0` → skip body.

## Dynamic variables (`dyn`)

Dynamics are **live-updateable variables** read when each stage runs — CV-modulate your actions
from Lua without rebuilding them:

```lua
output[1].action = loop{ to( dyn{height = 1}, 1)
                       , to(-1, 1)
                       }
output[1]()                  -- start
output[1].dyn.height = 2     -- update the rising destination while it runs
```

Arithmetic works on dynamics (returns a dyn expression):

```lua
output[1].action = loop{ to( 1, dyn{time = 1}/2 )
                       , to(-1, dyn{time = 1}/2 )
                       }
output[1].dyn.time = 2       -- retimes the whole LFO
```

Same-named dynamics **share state** across expressions (`dyn{time=1}/2` twice uses one `time`).

### Mutations & mutables

Chained, applied **every time the dynamic is accessed** (there are exactly three):

```lua
dyn{ k = v }:step(inc)        -- add `inc` on each access
            :mul(factor)      -- multiply by `factor` on each access
            :wrap(min, max)   -- wrap into [min, max] (min==0 → t % max)
```

Undocumented siblings (`lua/asl.lua`): **`mutable(n)`** / **`mutable{named = 4}`** — destructive
counters that decrement/increment on each access (what `times()` is built from), and the **`#`
freeze operator** for mutables (`#dyn_expr` freezes its current value). `times(n, {asl})` is
literally `asl._while(mutable(n+1)-1, {asl})`; `asl._while(pred, t)` is `loop(asl._if(pred, t))`.

Worked examples:

```lua
-- ramp that decelerates from 0.1s toward infinity:
output[1].action = loop{ to(5, dyn{time = 0.1}:step(0.1))
                       , to(0, 0)
                       }

-- same but wrapping: endlessly looping 0.1..5 s cycle:
output[1].action = loop{ to(5, dyn{time = 0.1}:step(0.1):wrap(0.1, 5))
                       , to(0, 0)
                       }

-- CV-controlled LFO rate (see community-techniques.md for the full bank):
output[1](lfo(dyn{freq = 1}, 5))
input[1].stream = function(v) output[1].dyn.freq = 1 + v end
```

### dyn hard rules (learned the hard way)

1. **The action must have run at least once before you assign `output[n].dyn.X`** — otherwise:
   `casl_setdynamic (number expected, got nil)`. Standard fix: bang once in `init()`.
2. **Values snapshot at stage start** — changing `output[1].dyn.time` mid-stage does *not*
   retarget the motion in flight; it applies at the next stage/next run. For instant response use
   many short stages (the "stepped LFO" technique in `community-techniques.md`).
3. **Functions in ASL are limited and position-dependent.** The compiler errors *"Do you have a
   function in your ASL? Replace it with dyn."* when a function turns up where it expects a
   numeric expression (e.g. inside `to()` destination slots or dyn math). Two function positions
   *have* been observed working on real firmware — bare stage elements
   (`loop{ function() ii.faders[1].get(1) end, to(5,1), to(0,1) }`) and built-in action arguments
   (`ar(get_a, get_d)`) — see `community-techniques.md` §4.9/§5.7. Treat function support as
   position- and version-dependent; prefer `dyn{}` for anything that must be reliable.

## Resource limits (per output, `lib/casl.h`)

| resource | limit | overrun message |
|---|---|---|
| `to` stages | 16 | `ERROR: no stages left!` |
| sequences | 8 | `ERROR: no sequences left!` |
| stages per sequence | 8 | (with the above) |
| dynamics | 40 | `ERROR: no dynamic slots remain` |

Practical consequence: big repeating journeys should **compose** (`times`, `loop`, nesting)
rather than enumerate every stage; split huge gestures across outputs or drive finer material
from Lua (`clock` + `.volts`).

## ASL gotchas

- `times()` **doesn't retrigger inside `loop{}`** (its counter never resets) — use
  `asl._while(dyn{counter=...}:step(-1):wrap(0, n), {...})` for counted cycles inside loops.
- **Zero-time stages can lock up the ASL** — an `_if` with a 0-second stage and no else-branch
  can spin; add a tiny-time second `_if` for the idle case (community pattern, see
  `community-techniques.md`).
- `output[n].volts = x` mid-action **restarts** the action as `to(x, slew, shape)` — deliberate
  for envelopes (gate via volts), surprising mid-LFO. Freeze instead with
  `output[n].volts = output[n].volts`.
- `.done` fires on completion of non-looping journeys only (a `loop{}` never completes).
- Shapes are matched on first character (see `api-input-output.md`) — `'log'` needs the `lo`;
  anything unrecognized is silently linear.

## More

- Built-in actions and their defaults: `api-input-output.md`.
- Timed/beat-locked triggering of actions: `api-time.md`.
- dyn-mutated LFOs, stepped micro-stage LFOs, ASL-as-strings: `community-techniques.md`.
- ASL sources: [`lua/asl.lua`](https://github.com/monome/crow/blob/main/lua/asl.lua),
  [`lua/asllib.lua`](https://github.com/monome/crow/blob/main/lua/asllib.lua).
