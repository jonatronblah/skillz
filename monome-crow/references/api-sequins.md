# `sequins` — step-sequencer tables

`sequins` are Lua tables that advance a playhead each call — the core pattern language for crow.
Any Lua value can live in one: numbers, strings, functions, tables, nested sequins.

```lua
s = sequins            -- idiomatic alias
seq = s{1, 2, 3}
_ = seq()              -- next value: 1, 2, 3, 1, 2, 3, ...
seq = sequins"abcd"    -- string form == sequins{'a','b','c','d'}
```

## Elements & nesting

```lua
fnseq  = s{lfo, ar, pulse}        -- functions are fine
strseq = s{'+', '-', '*', '/'}
tabseq = s{{1,2,3}, {4,5,6}}      -- tables are fine

nested = s{1, 2, s{3, 4}}         -- 1, 2, 3, 1, 2, 4, ... (inner advances per outer step)
```

Nested sequins count as **one** element for `#seq`. Type predicate: `sequins.is_sequins(x)`.

## Index & element control

```lua
seq:step(2)          -- advance by 2 per call (default 1); chainable; n may itself be a sequins
seq:select(n)        -- next call jumps to index n
_ = seq.ix           -- current index
seq[n] = value       -- write element n (table syntax)
_ = seq[n]           -- read element n
seq:settable(t)      -- replace the whole table, PRESERVING the current index
                     --   (recursively merges nested sequins/flow state)
seq:reset()          -- reset flow-mod counters, indices, nested sequins
```

## Flow modifiers

Control *when* values are produced:

```lua
seq:every(n)   -- produce a value only every nth call (skips otherwise)
seq:times(n)   -- produce values only for the first n calls
seq:count(n)   -- GREEDY: produce n values in a row (captures focus, then yields)
seq:all()      -- = seq:count(#seq): all values before returning control
seq:flow(k, n) -- underlying setter used by the above (rarely called directly)
```

Greedy modes (`count`/`all`) are how you make phrase-level groupings inside nested sequins.
`print(seq)` shows flow state (`:e[n](3)` = every).

## Transformers

Map values at output time:

```lua
seq = sequins{0, 4, 7, 10}:map(function(n) return n/12 end)   -- semitones -> volts
seq = sequins{0, 4, 7, 10}/12                -- operator shortcut for a single op
seq = sequins{0, 4, 7, 10} + sequins{0,12,24} -- RHS may be a sequins (advanced each call)
seq:map()                                    -- cancel the transformer
```

Operators `+ - * / %` are sugar for single-op `:map`. Extra args pass through:
`seq:map(fn, another_sequins)`.

## Copy & bake

```lua
copy   = seq:copy()       -- deep independent copy (flow-mods + transformers included)
cookie = seq:bake(16)     -- resample the next 16 values into a NEW plain sequins
                         --   (flow/transformers NOT copied; bake() with no arg = bake(#seq))
```

## Helpers

```lua
print(seq)    -- pretty-print: s[1]{1,2,3} (+ flow/transformer state)
#seq          -- length (nested sequins count as 1)
seq:peek()    -- current value WITHOUT advancing
```

## Full method index (verified against `lua/sequins.lua`)

| method | semantics |
|---|---|
| `step(n)` | step increment (chainable) |
| `select(n)` | set next index |
| `ix` | current index (read) |
| `[n]` get/set | element access |
| `settable(t)` | replace data, keep index |
| `every(n)` / `times(n)` / `count(n)` / `all()` / `flow(k,n)` | flow modifiers |
| `reset()` | reset state |
| `peek()` | look without advancing |
| `copy()` | deep copy |
| `bake(n)` | resample n values into a new sequins |
| `map(fn[, args])` | transformer; `map()` cancels |
| `is_sequins(x)` | predicate (static) |
| `+ - * / %` | operator transformers |
| `#` | length |
| `print()` | pretty-print |

## Musical idioms

- **One table, many lanes**: read the same data with different transforms —
  `notes()`, `notes():step(3)()`, `s{}:map(...)()` — melody, harmony, and rhythm from one source
  (the bowery "krahenlied" trick, see `patterns-generative.md`).
- **Sequins of functions as phrase selectors**: `sw = s{strum, arpeggio}` then `clock.run(sw())`
  (community, `community-techniques.md`).
- **Live editing from the REPL**: `seq[3] = 7` or `seq:settable(newdata)` mid-performance.
- **In timeline tables**: times *and* function-table args may be sequins —
  `timeline.loop{sequins{3,3,2}, {ii.jf.play_note, sequins{0,4,7,11}/12, 2}}` (`api-time.md`).
- **Arithmetic layers**: `melody = s{0,4,7,11} + s{0,0,s{-12,0}}` — base line plus octave drift.
- **Fractions for triplets**: `s{1/3}:count(3)` not decimals, especially with timeline.

## Gotchas

- **Methods that do NOT exist** (call them and you get nil/`attempt to call`): `:once`, `:func`
  (the transformer setter is `:map` only), instance `:is_sequins()` (it's the library function
  `sequins.is_sequins(t)`), `:next()` (just call `seq()`), unary minus `__unm`.
- **Assigning unknown keys is silently ignored** (`__newindex` drops everything but numeric
  indices and `'n'`) — `seq.step = 3` does nothing; use `seq:step(3)`.
- **Operators need the sequins on the LEFT**: `s{...} + s{...}` and `seq/12` work;
  `2 + s{...}` breaks (it tries `S.func(2, ...)`).
- `:reset()` resets counters/indices but does **not** remove a `:map` transformer.
- First call forces the first value (`qix`) — `s{1,2,3}()` returns 1 even after `:step(2)`.
- `:all():every(n)` inside a **doubly-nested** sequins doesn't bubble out — express alternation
  as two `:count()` branches (community lesson).
- `settable` preserves the index — if the new table is shorter, you may land past the end briefly.
- Greedy flow-mods (`count`/`all`) capture focus: an outer sequins will pull all n values before
  advancing.
- `bake()` snapshots *values* — later edits to the source table don't propagate.
