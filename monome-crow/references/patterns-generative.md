# Generative & algorithmic patterns

Reusable algorithms distilled from the community script collections — mostly
[bowery](https://github.com/monome/bowery) (the official crow script collection) and the fork's
[ii_help.txt](https://github.com/awesomebrick/bowery/blob/main/twigs/ii_help.txt). Each pattern is
self-contained: paste, patch, mutate. Cross-references: `api-sequins.md`, `api-time.md`,
`api-asl.md`, `community-techniques.md` (forum-tested variants), `troubleshooting.md` (pitfalls).

Musical intent is marked per pattern — say what a patch *is* before choosing an algorithm.

---

## 1. Euclidean rhythms (Bjorklund interleave)

*Intent: evenly-distributed pulse patterns — the backbone of W/African/ techno rhythm; rotate for
variation.*

Distribute `fill` pulses as evenly as possible across `length` steps; rotate by an offset for
classic Euclidean rhythms (bowery/euclidean.lua):

```lua
-- returns true if step ix of (fill, length) is a pulse
function er(fill, length, ix)
  local r = {}
  for i = 1, length do r[i] = { i <= fill } end
  local function cat(t, dst, src)  -- append table src into table dst
    for _, v in ipairs(t[src]) do table.insert(t[dst], v) end
    t[src] = nil
  end
  while #r > fill do  -- interleave empties into pulses
    for i = 1, math.min(fill, #r - fill) do cat(r, i, #r) end
  end
  while #r > 1 do cat(r, #r - 1, #r) end  -- flatten
  return r[1][ix]
end

-- per clock trigger, per channel n:
--   locations[n] = (locations[n] + 1) % length
--   if er(fill, length, (locations[n] + offset) % length + 1) then output[n]() end
```

Typical use: 4 outputs as kick/snare/hat/rim with `(3,8)`, `(5,8)`, `(7,16)`, `(2,5)` — prime
lengths against each other generate long phrasing.

## 2. Chaos source: Lorenz attractor

*Intent: slow, organic, never-repeating modulation — "weather" for CV; great for drone systems.*

Euler-integrate the Lorenz system and map state to CV; run at ~1 kHz (bowery/lorenz.lua):

```lua
lorenz = { sigma = 10, rho = 28, beta = 8/3, state = {0.01, 0, 0}, dt = 0.001 }

function lorenz:process()
  local s = self.state
  local dx = self.sigma * (s[2] - s[1])
  local dy = s[1] * (self.rho - s[3]) - s[2]
  local dz = s[1] * s[2] - self.beta * s[3]
  s[1], s[2], s[3] = s[1] + dx*self.dt, s[2] + dy*self.dt, s[3] + dz*self.dt
end

function lorenz_loop()
  while true do
    lorenz:process()
    -- normalize x (-25..35) to -5..5 V
    output[1].volts = 10 * (lorenz.state[1] + 25) / 60 - 5
    -- exponential speed mapping from a CV input
    lorenz.dt = math.exp((input[2].volts - 1) / 3) / 1000 - 0.00005
    clock.sleep(1/1000)
  end
end
```

Map other state axes to other outputs for correlated-yet-independent modulation.

## 3. Chaos source: gingerbreadman map

*Intent: chaotic but discrete — stepped random-walk melodies that feel "composed", not white noise.*

Tiny chaotic map stepped per clock trigger — great for melodies (bowery/gingerbread.lua):

```lua
bread = { x = 0.5, y = 0.5 }

function bread:step(xoff, yoff)  -- offsets from input[n].volts
  local x1 = self.x
  self.x = 0.5 - self.y + math.abs(x1) + xoff
  self.y = x1 + yoff
  if self.x > 5 then self.x = 5.0 end
  if self.y > 5 then self.y = 5.0 end
end
```

Quantize the result through `output[n].scale` for pitch material.

## 4. Boids flocking (1D)

*Intent: emergent ensemble motion — four related-but-autonomous voices drifting together; poetic
for quadraphonic / four-oscillator patches.*

Flocking across the 4 outputs: each boid has position (volts) and velocity; steer toward a centre
(blend with a CV input), avoid neighbours, match their velocity. Update one boid per tick
(bowery/boids.lua):

```lua
boids = { {p=0,v=0}, {p=1,v=0}, {p=2,v=0}, {p=3,v=0} }

function move_boid(n, c, accel)
  local b = boids[n]
  local avoid = 0
  for m = 1, #boids do
    if m ~= n and math.abs(boids[m].p - b.p) < public.avoid then
      avoid = avoid - (boids[m].p - b.p) / 2
    end
  end
  local v_avg = 0
  for m = 1, #boids do if m ~= n then v_avg = v_avg + boids[m].v end end
  v_avg = v_avg / (#boids - 1)
  b.v = b.v + (c - b.p) * public.pull + avoid + (v_avg - b.v) * public.sync
  b.v = math.max(-public.limit, math.min(public.limit, b.v))
  b.p = b.p + b.v * accel
  output[n].volts = b.p
end

function flock()
  local n = 1
  while true do
    local c = (input[1].volts + 5) / 2     -- centre follows input 1
    local accel = (input[2].volts + 5) / 5 -- acceleration follows input 2
    move_boid(n, c, accel)
    n = (n % 4) + 1                        -- round-robin
    clock.sleep(0.01 / 4)
  end
end
```

Expose `avoid`/`pull`/`sync`/`limit` as `public` vars for live shape control.

## 5. Analog shift register (ASR)

*Intent: the classic Serge/Buchla gesture — the same melodic cell echoed across voices, each a
beat older; self-harmonizing.*

On each trigger, outputs present the previous stages, then shift a new sample in at the front
(bowery/shiftregister.lua):

```lua
reg = {0, 0, 0, 0}

input[1].mode('change', 1, 0.1, 'rising')
input[1].change = function()
  for n = 1, 4 do
    output[n].volts = reg[n]
  end
  table.remove(reg)                    -- drop the oldest
  table.insert(reg, 1, input[2].volts) -- push the new sample to the front
end
```

Community extension: feedback-weighted ASR ("entangled S&H", `fdbk` amounts per stage, chaotic
drift) in `community-techniques.md`.

## 6. Boolean logic

*Intent: gate processing — combine clock streams rhythmically (XOR = fills, AND = drops).*

Two gate inputs through a selectable logic function per output (bowery/booleans.lua):

```lua
fns = { ['&']  = function(a,b) return a and b end
      , ['|']  = function(a,b) return a or b end
      , ['^']  = function(a,b) return a ~= b end    -- XOR
      , ['~^'] = function(a,b) return a == b end    -- XNOR
      }  -- also buildable: ~|, ~&, >, <, <=, >=
fnames = {'~|','>','<','&','~^','^','~&','<=','>=','|'}
public{f1 = '&'}:options(fnames):action(apply)

function gate(n) return input[n].volts > 1.0 end

function apply()
  output[1].volts = fns[public.f1](gate(1), gate(2)) and 5 or 0
end

function init()
  for n = 1, 2 do
    input[n].mode('change', 1, 0.1, 'both')
    input[n].change = apply
  end
end
```

## 7. CV delay (bucket brigade)

*Intent: voltage memory — canon at any delay time; smears gestures into chords.*

Stream an input at 1 kHz into a circular buffer; each output reads an independent tap; a CV
crossfades between endlessly looping and overwriting (bowery/cvdelay.lua):

```lua
LENGTH = 1000
bucket = {}
write_ix = 1

function peek(tap)
  local ix = (math.floor(write_ix - tap - 1) % LENGTH) + 1
  return bucket[ix] or 0
end

function poke(v, ix)
  -- 0 = keep looping the old value, 1 = overwrite with the new
  local c = math.min(1, math.max(0, input[2].volts / 4.5))
  bucket[ix] = v + c * ((bucket[ix] or v) - v)
end

function init()
  input[1].mode('stream', 0.001)
end

input[1].stream = function(v)
  output[1].volts = peek(100)  -- tap 1: 100 samples behind
  output[2].volts = peek(250)  -- tap 2
  poke(v, write_ix)
  write_ix = (write_ix % LENGTH) + 1
end
```

## 8. Runtime clock-division presets

*Intent: arrangement-level rhythm changes from one CV gesture — the "Pamela's preset" move.*

Swap all four output divisions live via a window-quantized CV input, without re-syncing the clocks
(bowery/clockdiv.lua):

```lua
presets = { {5,7,11,13}, {3,5,7,11}, {1,2,3,4}, {8,16,32,64} }
preset_ix = 1

function init()
  for n = 1, 4 do output[n]:clock(4) end
  input[2].mode('window', {-3,-1,1,3})
end

input[2].window = function(win, dir)
  preset_ix = win
  for n = 1, 4 do
    output[n].clock_div = presets[preset_ix][n]  -- live re-divide, no re-sync
  end
end
```

## 9. One sequins as a data bank

*Intent: text-as-score — a whole piece driven from one string; literary/aleatoric.*

krahenlied.lua drives a whole piece from a single sequins built from a text string; `:settable()`
hot-swaps the data from the REPL:

```lua
-- turn a string into a sequins of numbers (ASCII % 32 + 1)
function process_string(s)
  local t = {}
  for i = 1, #s do table.insert(t, s:byte(i) % 32 + 1) end
  return t
end

melody = sequins(process_string('crow poetry here'))

function text(str)  -- call from the REPL to hot-swap the data
  melody:settable(process_string(str))
end
```

krahenlied.lua also reads the same sequins with different `:step(n)()` values to derive note,
duration, and timbre lanes from one table (see the idiom in `api-sequins.md`).

## 10. Polling ii getters

*Intent: bring external panel state into the composition — knobs as macros.*

Request params one at a time and handle replies in the event handler; pace requests to avoid i2c
congestion (bowery/twigs/w_slash_stats.lua):

```lua
ii.wdel.event = function(e, value)
  print(e.name .. ' = ' .. value)
end

function poll()
  while true do
    for _, name in ipairs{'feedback', 'mix', 'time', 'rate'} do
      ii.wdel.get(name)
      clock.sleep(0.0015)  -- pace requests to avoid i2c congestion
    end
    clock.sleep(0.3)       -- full sweep every 300ms
  end
end
```

Two-arg getters work the same way (e.g. `ii.wsyn.get('patch', 1)`). NB: W/Del hardware never
answers gets (`ii-wslash.md`); for W/Tape/W/Syn/JF this works.

## 11. Strum (spreading a chord)

*Intent: guitar-like gesture — humanizes chords; lovely on Just Friends' LPGs.*

```lua
function strum(notes)
  local t, mult = 0.02, 1.3
  for i = 1, #notes do
    ii.jf.play_note(notes[i] / 12, 3)
    clock.sleep(t)
    t = t * mult  -- each gap longer than the last (accelerando)
  end
end
```

Richer variants (velocity lanes, sequins-driven strum sparsity, two-hand jazz ensemble) in
`community-techniques.md`.

## 12. Idioms (one-liners)

```lua
dest = (dest % 4) + 1                       -- rotate a destination: sequential switch / round-robin
ii.jf.tick(clock.tempo)                     -- JF Geode timebase lock (tempos 49..255)
output[n](lfo(dyn{freq=1}, 5))              -- CV-controlled LFO bank: update output[n].dyn.freq
                                            --   from an input stream handler
output[1].volts = hztovolts(f)              -- Hz -> V/oct inside an input[1].freq handler
clock.sleep(clock.get_beat_sec()/4 + swing) -- swung 16ths (see community-techniques.md §1)
output[n].volts = output[n].volts           -- freeze an output mid-ASL
```
