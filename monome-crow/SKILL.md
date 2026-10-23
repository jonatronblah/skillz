---
name: monome-crow
description: Program the monome crow eurorack module using Lua. Covers the full crow Lua API (input, output, ASL, sequins, metro, clock, timeline, hotswap, ii, public, calibration), i2c control of companion modules (Just Friends, W/Tape, W/Del, W/Syn), a cheat sheet of ii commands for every supported device (ansible, Disting EX, ER-301, 16n Faderbank, TXi, TXO, and more), and algorithmic/generative patterns (Euclidean rhythms, boids, chaos maps, shift registers).
version: 1.1.0
license: MIT
author: "@monome-community"
tags:
  - monome
  - crow
  - eurorack
  - modular-synth
  - lua
  - i2c
  - just-friends
  - wslash
---

## Instructions

Use this skill when writing, debugging, or explaining Lua scripts for the **monome crow** eurorack module, or when controlling i2c-connected companion modules (Just Friends, W/Tape, W/Del, W/Syn) from crow.

### When to Use

- Writing or editing crow Lua scripts
- Explaining crow API functions (input, output, ASL, sequins, metro, clock, timeline, ii, public)
- Controlling Just Friends via i2c from crow
- Controlling W/Tape, W/Del, or W/Syn via i2c from crow
- Controlling other ii devices: ansible, Disting EX, ER-301, 16n Faderbank, TXi, TXO
- Building algorithmic/generative patches: Euclidean rhythms, boids, chaos maps, shift registers
- Debugging crow scripts or i2c communication
- Designing sequencers, arpeggiators, or generative systems on crow
- Questions about crow's Lua scripting environment

### Key Principles

- crow scripts are written in **Lua** and run on the crow hardware
- The `init()` function is called when the script starts
- Voltages are in **volts-per-octave** (V/8) — `0.0` = C3
- Times are in **seconds**
- i2c commands use the `ii.<device>.<command>()` pattern
- Use `sequins` for step-sequencer patterns, `timeline` for rhythmic loops/scores
- Use `ASL` for output actions (LFOs, envelopes, oscillators)
- Use `clock` for tempo-synced coroutines, `metro` for simple repeating timers

---

## crow Lua API Reference

### input

`input` is a table representing the 2 CV inputs (1 and 2).

#### Queries

```lua
_ = input[n].volts  -- returns the current value on input n
input[n].query      -- send input n's value to the host -> ^^stream(channel, volts)
```

#### Modes

```lua
input[n].mode = 'stream'  -- set input n to stream with default time (0.1s)
input[n].mode('none')     -- disable events
input[n].mode('stream', time)  -- stream every time seconds
input[n].mode('change', threshold, hysteresis, direction)  -- event on threshold crossing
  -- direction: 'rising', 'falling', or 'both'
input[n].mode('window', windows, hysteresis)  -- event when entering a new voltage window
  -- windows: table of boundary voltages, eg: {-3,-1,1,3}
input[n].mode('scale', notes, temperament, scaling)  -- quantize input, event on new note
  -- notes: table of note values, eg: {0,2,4,5,7,9,11}
  -- temperament: notes per octave (default 12), or 'ji' for just intonation
  -- scaling: volts-per-octave (default 1.0, use 1.2 for Buchla)
input[n].mode('volume', time)  -- track RMS amplitude
input[n].mode('peak', threshold, hysteresis)  -- detect audio transients
input[1].mode('freq', time)  -- frequency detection (input 1 only!)
input[n].mode('clock', division)  -- drive internal clock from input
  -- division: rate at which clock arrives, eg 1/4 for 4 ticks per beat
```

#### Table-call syntax

```lua
input[n]{ mode   = 'stream'
        , time   = 0.2
        , stream = function(v) print(v) end
        }
```

#### Default Values

```
time       = 0.1
threshold  = 1.0
hysteresis = 0.1
direction  = 'both'
temp       = 12
scaling    = 1.0
div        = 1/4
```

#### Event Handlers

Default modes call specific events sent to the host:
- `'stream'` → `^^stream(channel, volts)`
- `'change'` → `^^change(channel, state)`

Custom handlers:

```lua
input[1].stream = function(volts) <your_function> end
input[1].change = change_handler_function
input[1].scale  = function(note) <use note.volts> end  -- 'scale' mode: fires on each newly detected note
input[1].window = function(win, dir) end                -- 'window' mode: fires on window change (win = window index)
```

---

### output

`output` is a table representing the 4 CV outputs (1–4).

#### Setting CV

```lua
output[n].volts = 3.0  -- set output n to 3 volts instantly
```

#### Slewing CV

```lua
output[n].slew  = 0.1       -- set slew time to 0.1 seconds
output[n].volts = 2.0       -- move toward 2.0V over the slew time
_ = output[n].volts         -- inspect instantaneous voltage
```

#### Shaping CV

```lua
output[n].shape = 'expo'  -- non-linear path to destination
output[n].slew  = 0.1
output[n].volts = 2.0
```

Available shapes: `'linear'`, `'sine'`, `'logarithmic'`, `'exponential'`, `'now'`, `'wait'`, `'over'`, `'under'`, `'rebound'`

#### Quantize to Scales

```lua
output[n].scale({0,2,3,5,7,9,10})       -- quantize to scale
output[n].scale({1/1, 9/8, 5/4, 4/3, 3/2, 11/8}, 'ji')  -- just intonation
output[n].scale('none')                   -- deactivate scaling
output[n].scale = {0,7,2,9}              -- modify scale table (can be out of order for arpeggios)
```

#### Clock Mode

```lua
output[n]:clock(division)   -- set output to clock mode with division
output[n]:clock('none')     -- disable output clock
output[n].clock_div = 1/4   -- update division without re-syncing
```

#### Actions (ASL-based)

```lua
output[n].action = lfo()     -- set action to default LFO
output[n]()                  -- start the action
output[n](lfo())             -- shortcut: set action and start immediately
```

Built-in actions (from asllib.lua):

```lua
lfo(time, level, shape)                      -- low frequency oscillator
pulse(time, level, polarity)                  -- trigger / gate generator
ramp(time, skew, level)                      -- triangle LFO with skew
ar(attack, release, level, shape)            -- attack-release envelope
adsr(attack, decay, sustain, release, shape) -- ADSR envelope
oscillate(freq, level, shape)                -- audio rate oscillator
```

Action default values:

```
lfo:       time=1, level=5, shape='sine'
pulse:     time=0.01, level=5, polarity=1
ramp:      time=1, skew=0.25, level=5
ar:        attack=0.05, release=0.5, level=7, shape='log'
adsr:      attack=0.05, decay=0.3, sustain=2, release=2, shape='linear'
oscillate: freq=1, level=5, shape='sine'
```

Directives for `adsr`:

```lua
output[1].action = adsr()
output[1](true)   -- start attack phase, pause at sustain
output[1](true)   -- re-start attack from current location
output[1](false)  -- enter release phase
```

#### done Event

```lua
output[1].done = function() print('done!') end
```

---

### ASL (Action Sequence Language)

Build custom actions from primitives.

#### Primitive: `to(destination, time, shape)`

```lua
myjourney = { to(1,1)
            , to(2,2)
            , to(3,3,'log')
            }
output[1](myjourney)
```

#### Constructs

```lua
loop{ <asl> }                    -- repeat forever
held{ <asl> }                    -- freeze at end until false/'release' directive
lock{ <asl> }                    -- ignore directives until sequence completes
times(count, { <asl> })          -- repeat count times
asl._if(pred, { <asl> })        -- conditional execution
asl._while(pred, { <asl> })      -- loop while predicate is true
```

#### Dynamic Variables

```lua
-- dynamic variables can be updated while the ASL is running
output[1].action = loop{ to(dyn{height=1}, 1)
                       , to(-1, 1)
                       }
output[1].dyn.height = 2  -- update the rising destination

-- arithmetic on dynamics
output[1].action = loop{ to(1, dyn{time=1}/2)
                       , to(-1, dyn{time=1}/2)
                       }
output[1].dyn.time = 2  -- set overall LFO time

-- mutations: applied every time the dynamic is used
dyn{k=v}
  :step(inc)           -- add inc on each access
  :mul(factor)         -- multiply by factor on each access
  :wrap(min, max)      -- wrap value into range

-- example: ramp LFO decelerating from 0.1s to infinity
output[1].action = loop{ to(5, dyn{time=0.1}:step(0.1))
                       , to(0, 0)
                       }

-- example: wrap to create looping patterns
output[1].action = loop{ to(5, dyn{time=0.1}:step(0.1):wrap(0.1,5))
                       , to(0, 0)
                       }
```

---

### sequins

`sequins` are Lua tables for building sequencers and arpeggiators.

```lua
s = sequins  -- idiomatic alias

seq = s{1,2,3}       -- create a sequins
_ = seq()             -- returns next value: 1, 2, 3, 1, ...

-- any datatype allowed
fnseq  = s{lfo, ar, pulse}
strseq = s{'+', '-', '*', '/'}
tabseq = s{{1,2,3}, {4,5,6}}

-- step control
seq = s{1,2,3,4,5}:step(2)  -- step by 2: 1,3,5,2,4,1,...

-- index control
seq:select(n)  -- set next index
_ = seq.ix     -- query current index

-- modify elements
seq[n] = _     -- update value at index n
_ = seq[n]     -- access nth element
seq:settable(new_table)  -- replace whole table, preserving index

-- nested sequins
seq = s{1, 2, s{3, 4}}  -- seq() --> 1, 2, 3, 1, 2, 4 ...
```

#### Flow Modifiers

```lua
seq:every(n)   -- produce a value every nth call
seq:times(n)   -- only produce a value the first n times
seq:count(n)   -- produce n values in a row (greedy)
seq:all()      -- return all values before yielding (greedy)
seq:reset()    -- reset all modifiers and indices
```

#### String Shortcut

```lua
seq = sequins"abcd"  -- same as sequins{'a','b','c','d'}
```

#### Transformers

```lua
seq = sequins{0,4,7,10}:map(function(n) return n/12 end)  -- volts instead of notes
seq = sequins{0,4,7,10}/12                                -- operator shortcut
seq = sequins{0,4,7,10} + sequins{0,12,24}                -- chain sequences
seq:map()  -- cancel transformer
```

#### Copy & Bake

```lua
copy = seq:copy()          -- complete copy with modifiers & transformers
cookie = seq:bake(16)      -- sample next 16 values into a new sequins
```

#### Helpers

```lua
print(seq)    -- pretty-print: s[1]{1,2,3}
#seq          -- length (nested sequins count as 1)
seq:peek()    -- current value without advancing
```

---

### metro

8 independent timers, each with its own timebase and event.

```lua
-- indexed access
metro[1].event = function(c) print(c) end
metro[1].time  = 1.0
metro[1]:start()

-- named initialization
mycounter = metro.init{ event = count_event
                      , time  = 2.0
                      , count = -1  -- -1 = forever
                      }
mycounter:start()
mycounter:stop()

-- update while running
metro[1].event = a_different_function
mycounter.time = 0.1
```

---

### clock

The clock system for tempo-synced coroutines.

```lua
-- run a function as a clock coroutine
coro_id = clock.run(func [, args])
clock.cancel(coro_id)
clock.sleep(seconds)
clock.sync(beats)    -- sleep until next sync at interval 'beats' (eg 1/4)
clock.cleanup()      -- kill all running clocks
```

#### Tempo & Timing

```lua
clock.tempo = t               -- set BPM
_ = clock.tempo               -- get BPM
_ = clock.get_beats()         -- beats since clock started
_ = clock.get_beat_sec()      -- length of a beat in seconds
```

**NB**: `clock.get_beats` and `clock.get_beat_sec` are **functions** — call them with `()`. Using the bare symbol in arithmetic raises `attempt to perform arithmetic on a function value` (verified on hardware).

#### Start/Stop

```lua
clock.start([beat])       -- start clock (optionally from 'beat')
clock.stop()              -- stop clock
clock.transport.start = start_handler  -- function called on start
clock.transport.stop = stop_handler    -- function called on stop
```

#### Looping Example

```lua
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

#### One-shot Example

```lua
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

---

### delay

Simple delayed execution.

```lua
delay(action, time [, repeats])  -- delay action (function) by time (seconds)
                                  -- (optional) repeat repeats times
```

---

### timeline

Create rhythmic loops, scores, or timed-event sequences.

#### :loop — Rhythmic Loop

```lua
t1 = timeline.loop{duration, event, duration, event, ...}

-- conditional loop
t2 = timeline.loop{1, kick, 2, snare}:unless(function() return input[1].volts > 2 end)

-- fixed repetitions
t3 = timeline.loop{0.55, hihat, 0.45, hihat}:times(16)
```

#### :score — Beat-based Score (runs once)

```lua
t4 = timeline.score{0, intro, 32, verse}

-- loop with 'reset'
t5 = timeline.score{0, intro, 32, verse, 64, 'reset'}

-- conditional reset
t6 = timeline.score{ 0, intro
                    , 32, verse
                    , 64, function() if math.random() > 0.5 then return 'reset' end
                    }
```

#### :real — Realtime (seconds) Sequence

```lua
t7 = timeline.real{0, note_1, 0.33, note_2, 0.5, note_3}

-- with reset
t8 = timeline.real{0, note_1, 0.33, note_2, 0.5, note_3, 1.2, 'reset'}
```

#### Control

```lua
-- queue for later playback
tt = timeline.queue:loop{2, kick, 2, snare}
tt:play()   -- begin
tt:stop()   -- halt
tt:play()   -- restart

-- launch quantization
tt = timeline.launch(8):loop{2, kick, 2, snare}  -- quantize to 8 beats
timeline.launch_default = 0  -- disable quantization globally

-- stop all
timeline.cleanup()
```

#### Function Tables & sequins

```lua
-- function table: first element is function, rest are args
tt = timeline.loop{1, {print, "boop!"}}

-- with sequins for evolving patterns
tt = timeline.loop{sequins{3,3,2}, {ii.jf.play_note, sequins{0,4,7,11}/12, 2}}
```

---

### hotswap

Live-coding support — preserves playback position when re-assigning sequins/timelines.

```lua
hotswap.seq = sequins{1,2,3}
hotswap.seq()       --> 1
hotswap.seq()       --> 2
hotswap.seq = sequins{4,5,6,7}  -- playhead preserved
hotswap.seq()       --> 6

hotswap.tt = timeline.loop{sequins{3,3,2}, {ii.jf.play_note, sequins{0,4,7,11}/12, 2}}
-- update while running:
hotswap.tt = timeline.loop{sequins{3,2,2,1}, {ii.jf.play_note, sequins{0,4,7,10}/12, 2}}
```

---

### ii (i2c)

crow can lead i2c devices on the ii bus.

```lua
ii.help()              -- list supported devices
ii.<device>.help()     -- list functions for a device
```

#### General Pattern

```lua
-- commands (setters)
ii.mydevice.command([args])

-- queries (getters)
ii.mydevice.get('param' [, args])

-- event handler for query responses
ii.mydevice.event = function(e, value)
  -- e is a table: { name='param', arg=first_arg, device=device_number }
  if e.name == 'param' then
    -- handle value
  end
end
```

#### Duplicate Devices

```lua
ii.txi[1].get('param',1)  -- first device
ii.txi[2].get('param',1)  -- second device

ii.txi.event = function(e, value)
  if e.name == 'param' then
    print('txi[' .. e.device .. '][' .. e.arg .. ']=' .. value)
  end
end
```

**NB — there are no per-device handles** (verified against crow source: `lua/ii.lua`, `lib/l_ii_mod.c`): `ii.mod[n]` switches the module's *shared* i2c address and returns the **same table**. Always index at call time (`ii.wtape[2].record(1)`). Caching the result (`local t = ii.wtape[2]`) makes later calls silently send to whichever address was set *last*. Commands to an address with no device attached are silently ignored — no ACK checking, no error. Devices ship at a factory address (W/Tape: `0x71`); a second unit must be configured at the next address (W/Tape: `0x72`, per crow's `lua/ii/wtape.lua`) before `ii.mod[2]` can reach it.

#### Raw ii Access

```lua
ii.raw(addr, bytes [, rx_len])
  -- addr: i2c address (hex, eg 0x70)
  -- bytes: bytestring message
  -- rx_len: optional response bytes to read (default 0)

ii.event_raw = function(addr, cmd, data)
  if addr == 0x70 then
    -- handle custom event
    return true  -- prevent normal ii system from processing
  end
end
```

#### Addressing Multiple crows

```lua
print(ii.address)  --> 1 by default (1-4)
ii.address = 2     -- set this crow to address 2

ii.crow[1].volts(1,2.9)  -- set output[1] on crow[1]
ii.crow[2].volts(1,4.2)  -- set output[1] on crow[2]
```

#### Advanced Settings

```lua
ii.fastmode(state)  -- true = max speed (~4x), false = default
ii.pullup(state)     -- true = on (default), false = off
```

#### Crow as an ii Follower (`ii.self`)

Other devices (e.g. Teletype) can call *this* crow over ii. Bind handlers to the fixed slots:

```lua
ii.self.call2 = function(x, y)   -- "call" slots fire side effects (call1-4 = 1-4 args)
  output[1].volts = x
  output[2](true)
end

ii.self.query1 = function(x)     -- "query" slots must return a value (query0-3 = 0-3 args)
  return input[x].volts
end
```

On the caller's side these appear as `ii.crow.call2(x, y)` / `ii.crow.get('query1', x)` (see the ii cheat sheet below).

---

### public

Expose variables to a connected USB host (e.g. norns) with automatic synchronization.

```lua
public{name = init_value}  -- create public variable
public.name = 'jin'        -- set value
_ = public.name            -- get value

-- supported types: numbers, strings, flat tables, sequins (first layer only)
-- NOT supported: functions, coroutines, nested tables

-- metadata
public{volts = 0}:range(-5, 10)                    -- clamp range
public{seconds = 1}:range{0.001, 10}:type('exp')   -- exponential scaling
public{count = 0}:type('int')                       -- integer steps
public{readonly = 'hello'}:type('@')                -- read-only
public{myslide = 0}:range{-5,10}:type('slider')     -- slider UI
public{myopt = '+'}:options{'+', '-', '*', '/'}     -- enumerated options

-- action on remote update
public{speed = 1.0}:action(update_speed)
function update_speed(value)
  metro[1].time = value
end
```

#### .view — I/O State Sharing

```lua
public.view.all([state])        -- all input & output views
public.view.input[n]([state])   -- nth input view
public.view.output[n]([state])  -- nth output view
```

#### Discovery & Synchronization

```lua
public.discover()                    -- list all public vars
public.update(name, value [, subkey]) -- update from remote host
```

---

### cal (Calibration)

```lua
cal.save()              -- save calibration to flash
cal.source(chan)        -- configure output->input multiplexer

_ = cal.input[n].offset   -- get/set input offset
cal.input[n].scale = _    -- get/set input scale
_ = cal.output[n].offset  -- get/set output offset
cal.output[n].scale = _   -- get/set output scale
```

---

### Random Values

```lua
math.random()           -- float 0.0 to 1.0 (true hardware random)
math.random(max)        -- int 1 to max
math.random(min, max)   -- int min to max

math.srandomseed(math.random() * 2^31)  -- seed pseudo-random
math.srandom()                           -- pseudo-random 0.0 to 1.0
```

---

### Other Globals

```lua
crow.reset()           -- deactivate inputs, zero outputs, free metros

tell(event, <args>)    -- send ^^event(arg1, ...) to host
quote(...)             -- string-representation for sending tables to host

_, _, _ = unique_id()  -- 3 numbers unique to each crow
time()                 -- milliseconds since power-on
cputime()              -- main loops per dsp block (higher = lower CPU)

justvolts(fraction [, offset])  -- just ratio to V/8
just12(fraction [, offset])     -- just ratio to 12TET semitones
hztovolts(freq [, reference])   -- frequency to voltage (default: C3 = 0V)
```

---

## i2c Device References

### Just Friends (`ii.jf`)

Just Friends is a 6-channel function generator / synthesizer / rhythm machine.

#### Basic Remote Control

```lua
ii.jf.trigger(channel, state)    -- trigger channel (1-6, or 0 for all). state: 1=high, 0=low
ii.jf.run_mode(mode)            -- set RUN state. 1=active, 0=inactive
ii.jf.run(volts)                 -- virtual RUN voltage (-5 to +5). Requires run_mode(1)
```

#### Extended Behaviour

```lua
ii.jf.transpose(pitch)          -- transpose by pitch (1.0 = 1 octave, 1/12 = 1 semitone)
ii.jf.vtrigger(channel, level)  -- trigger with velocity (level in volts, 0 = low state)
ii.jf.retune(channel, n, d)     -- retune INTONE ratio. retune(0,0,0) resets. retune(-1,0,0) saves to flash
ii.jf.address(index)            -- set ii address for dual-JF setups (1 or 2)
```

#### Synthesis Mode (sound + mode(1))

```lua
ii.jf.mode(1)                              -- enter Synthesis/Geode mode
ii.jf.play_note(pitch, level)              -- polyphonic note (dynamic voice allocation)
ii.jf.play_voice(channel, pitch, level)     -- note on specific voice (1-6, or 0 for all)
ii.jf.pitch(channel, pitch)                -- set pitch without triggering envelope
ii.jf.god_mode(state)                       -- 1 = A=432Hz, 0 = A=440Hz
```

Pitch: `0.0` = C3, in volts-per-octave. Level: in volts (5.0 = 5V peak-to-peak).

#### Geode Mode (shape + mode(1))

```lua
ii.jf.play_note(divs, repeats)             -- rhythmic sequence, dynamically allocated
ii.jf.play_voice(channel, divs, repeats)   -- rhythmic sequence on specific channel
ii.jf.tick(divs)                            -- clock with ticks per measure (1-48, 0=reset)
ii.jf.tick(bpm)                             -- set timebase with BPM (49-255, 0=reset)
ii.jf.quantize(divisions)                  -- quantize events to divisions (0=off, 1-32)
```

- `divs`: divides the measure into this many segments (4 = quarter notes)
- `repeats`: number of retriggerings (-1 = forever, 0 = initial only, 1 = 2 events total)

#### Getters

```lua
ii.jf.get('trigger', channel)  -- true if channel is moving
ii.jf.get('run_mode')          -- RUN mode state
ii.jf.get('run')               -- current RUN voltage
ii.jf.get('transpose')         -- current transpose
ii.jf.get('mode')              -- 1 if Synthesis/Geode active
ii.jf.get('tick')              -- current Geode BPM
ii.jf.get('god_mode')          -- 1 if god mode active
ii.jf.get('quantize')          -- quantize divisions
ii.jf.get('speed')             -- shape(0) or sound(1) switch
ii.jf.get('tsc')               -- MODE switch: 1=transient, 2=sustain, 3=cycle
ii.jf.get('ramp')              -- RAMP knob (-5,5)
ii.jf.get('curve')             -- CURVE knob (-5,5)
ii.jf.get('fm')                -- FM knob (-5,5)
ii.jf.get('time')              -- TIME knob + CV (-5,5)
ii.jf.get('intone')            -- INTONE knob + CV (0=C3, V/8 scaled)
```

#### Event Handler

```lua
ii.jf.event = function(e, value)
  if e.name == 'trigger' then
    -- e.arg: channel, e.device: device index
  elseif e.name == 'ramp' then
    -- handle ramp value
  end
end
```

---

### W/Tape (`ii.wtape`)

W/Tape is a virtual tape deck with ~3 hours of recording time, vari-speed, and live-looping.

#### Setters

```lua
ii.wtape.record(is_recording)       -- 1=on, 0=off
ii.wtape.play(is_playing)          -- 1=forward 1x, 0=stop, -1=reverse 1x
ii.wtape.reverse()                  -- reverse direction (no args)
ii.wtape.speed(rate [, denominator]) -- set speed as rate/ratio. Negative = reverse
ii.wtape.freq(v/8)                  -- set speed as V/8 frequency. 0 = speed(1). Maintains direction.
ii.wtape.erase_strength(level)      -- 0=overdub, 1=overwrite
ii.wtape.monitor_level(gain)        -- dry path gain from IN to OUT
ii.wtape.rec_level(gain)            -- input gain before recording
ii.wtape.echo_mode(active)          -- 1=playback before erasing, 0=erase first (default)
ii.wtape.loop_start()               -- set current position as loop start
ii.wtape.loop_end()                 -- set current position as loop end, jump to start
ii.wtape.loop_active(is_active)     -- 1=activate, 0=deactivate (preserves start/end)
ii.wtape.loop_scale(scale)          -- positive=multiply, negative=divide, 0=reset
ii.wtape.loop_next(direction)       -- positive=forward by loop length, negative=backward, 0=retrigger
ii.wtape.timestamp(seconds)         -- move to absolute position (seconds)
ii.wtape.seek(seconds)              -- move relative to current position (seconds)
```

#### Getters

```lua
ii.wtape.get('record')          -- is_recording
ii.wtape.get('play')            -- is_playing
ii.wtape.get('speed')           -- rate
ii.wtape.get('freq')            -- v/8
ii.wtape.get('erase_strength')  -- level
ii.wtape.get('monitor_level')   -- gain
ii.wtape.get('rec_level')       -- gain
ii.wtape.get('echo_mode')       -- is_active
ii.wtape.get('loop_start')      -- timestamp in seconds
ii.wtape.get('loop_end')        -- timestamp in seconds
ii.wtape.get('loop_active')     -- is_active
ii.wtape.get('loop_scale')      -- scale
ii.wtape.get('timestamp')       -- timestamp in seconds
```

---

### W/Del (`ii.wdel`)

W/Del is a tape echo / BBD delay with 4 seconds down to 1ms buffer, vari-speed, and modulation.

#### Setters

```lua
ii.wdel.feedback(level)           -- feedback amount (-5 to +5)
ii.wdel.mix(fade)                 -- 0=dry, 1=wet
ii.wdel.filter(cutoff)            -- filter center frequency in feedback loop
ii.wdel.freeze(is_active)         -- 0=off, 1=on (freeze buffer)
ii.wdel.time(seconds)             -- delay buffer length in seconds (at 1x rate)
ii.wdel.length(num, denominator)  -- buffer loop size as fraction of buffer time
ii.wdel.position(num, denominator)-- loop location as fraction of buffer time
ii.wdel.cut(num, denominator)     -- jump to loop location as fraction
ii.wdel.rate(multiplier)          -- tape speed multiplier
ii.wdel.freq(volts)               -- tape speed as V/8
ii.wdel.clock()                   -- send clock pulse for sync (no args)
ii.wdel.clock_ratio(mul, div)     -- clock pulses per buffer time
ii.wdel.pluck(volume)             -- pluck delay line with noise at volume
ii.wdel.mod_rate(rate)            -- modulation rate multiplier
ii.wdel.mod_amount(amount)        -- modulation depth
```

#### Getters

```lua
ii.wdel.get('feedback')    -- feedback amount
ii.wdel.get('mix')         -- wet/dry mix
ii.wdel.get('filter')      -- filter cutoff
ii.wdel.get('freeze')      -- freeze state
ii.wdel.get('time')        -- delay time
ii.wdel.get('rate')        -- tape speed multiplier
ii.wdel.get('freq')        -- tape speed as V/8
ii.wdel.get('mod_rate')    -- modulation rate
ii.wdel.get('mod_amount')  -- modulation depth
```

---

### W/Syn (`ii.wsyn`)

W/Syn is a 2-op FM polyphonic synthesizer with vactrol LPG.

#### Setters

```lua
ii.wsyn.velocity(voice, velocity)       -- strike vactrol at velocity (volts)
ii.wsyn.pitch(voice, pitch)            -- set pitch (V/8)
ii.wsyn.play_voice(voice, pitch, velocity) -- set pitch (V/8) and strike at velocity (V)
ii.wsyn.play_note(pitch, velocity)     -- dynamically assign voice, set pitch (V/8), strike (V)
ii.wsyn.ar_mode(is_ar)                  -- 1=attack-release mode (plucked), 0=off
ii.wsyn.curve(curve)                    -- waveform: -5=square, 0=triangle, 5=sine
ii.wsyn.ramp(ramp)                      -- symmetry: -5=ramp, 0=triangle, 5=sawtooth
ii.wsyn.fm_index(index)                -- FM amount: -5=negative, 0=min, 5=max
ii.wsyn.fm_env(amount)                 -- envelope to FM: -5=min, 5=max
ii.wsyn.fm_ratio(num, denominator)      -- FM modulator:carrier ratio (float up to 20.0)
ii.wsyn.lpg_time(time)                 -- vactrol time: -5=drone, 0=vtl5c3, 5=blits
ii.wsyn.lpg_symmetry(symmetry)         -- vactrol AR: -5=fastest attack, 5=long swells
ii.wsyn.voices(count)                   -- polyphonic voices: 0=unison
ii.wsyn.patch(jack, param)             -- assign jack to param:
  -- jack: 1=THIS, 2=THAT
  -- param: 1=ramp, 2=curve, 3=fm_env, 4=fm_index, 5=lpg_time,
  --        6=lpg_symmetry, 7=gate, 8=pitch, 9=fm_ratio(num), 10=fm_ratio(denom)
```

#### Getters

```lua
ii.wsyn.get('ar_mode')       -- 1 if attack-release mode active
ii.wsyn.get('curve')         -- waveform position (-5..5)
ii.wsyn.get('ramp')          -- symmetry (-5..5)
ii.wsyn.get('fm_index')      -- FM amount (-5..5)
ii.wsyn.get('fm_env')        -- FM envelope amount (-5..5)
ii.wsyn.get('fm_ratio')      -- current FM ratio
ii.wsyn.get('lpg_time')      -- vactrol time (-5..5)
ii.wsyn.get('lpg_symmetry')  -- vactrol AR character (-5..5)
ii.wsyn.get('patch', jack)   -- param assigned to jack (1=THIS, 2=THAT)
ii.wsyn.get('voices')        -- current voice count
```

---

## ii Cheat Sheet — All Supported Devices

Compiled from `ii.<module>.help()` outputs on crow (early 2025). Source: [bowery ii_help.txt](https://github.com/awesomebrick/bowery/blob/main/twigs/ii_help.txt). The curated "i2c Device References" above go deep on four modules; this section is the complete command/query index for **every** device crow can lead on the ii bus: ansible, crow, disting, er301, faders (16n), jf, kria, levels, meadowphysics, txi, txo, wdel, wsyn, wtape.

Responses to every `.get()` arrive through the shared event pattern (identical for all devices):

```lua
ii.<device>.event = function(e, value)
  if e.name == '<param>' then
    -- e.arg: first argument of the get (eg channel); e.device: index of device
  end
end
```

### ansible (`ii.ansible`)

```lua
-- commands
ii.ansible.trigger( channel, state )
ii.ansible.trigger_toggle( channel )
ii.ansible.trigger_pulse( channel )
ii.ansible.trigger_time( channel, time )
ii.ansible.trigger_polarity( channel, polarity )
ii.ansible.cv( channel, volts )
ii.ansible.cv_slew( channel, time )
ii.ansible.cv_offset( channel, volts )
ii.ansible.cv_set( channel, volts )
-- queries
ii.ansible.get( 'trigger', channel )
ii.ansible.get( 'trigger_time', channel )
ii.ansible.get( 'trigger_polarity', channel )
ii.ansible.get( 'cv', channel )
ii.ansible.get( 'cv_slew', channel )
ii.ansible.get( 'cv_offset', channel )
ii.ansible.get( 'input', channel )
```

### crow (`ii.crow`) — crow as an ii target

```lua
-- commands
ii.crow.volts( channel, level )
ii.crow.slew( channel, time )
ii.crow.call1( arg )
ii.crow.call2( arg1, arg2 )
ii.crow.call3( arg1, arg2, arg3 )
ii.crow.call4( arg1, arg2, arg3, arg4 )
ii.crow.reset( )
ii.crow.pulse( chan, time, level, polarity )
ii.crow.ar( chan, attack, release, level )
ii.crow.lfo( chan, freq, level, skew )
-- queries
ii.crow.get( 'input', channel )
ii.crow.get( 'output', channel )
ii.crow.get( 'query0' )
ii.crow.get( 'query1', arg1 )
ii.crow.get( 'query2', arg1 )
ii.crow.get( 'query3', arg1, arg2 )
```

Handle incoming `call`/`query` on the target crow with `ii.self` (see "Crow as an ii Follower" above).

### disting — Disting EX (`ii.disting`)

```lua
-- commands
ii.disting.load_preset( number )
ii.disting.save_preset( value )
ii.disting.reset_preset( )
ii.disting.algorithm( algorithm )
ii.disting.controller( controller, value )
ii.disting.parameter( parameter, value )
ii.disting.scale_parameter( parameter, value )
ii.disting.wav_record( action )
ii.disting.wav_playback( action )
ii.disting.al_pitch( pitch )
ii.disting.al_clock( )
ii.disting.lp_clear( )
ii.disting.midi( status, b0, b1 )
ii.disting.select_bus( status, b0, b1 )
ii.disting.voice_pitch( voice, pitch )
ii.disting.voice_on( voice, velocity )
ii.disting.voice_off( voice )
ii.disting.note_pitch( note_id, pitch )
ii.disting.note_velocity( note_id, velocity )
ii.disting.note_off( note_id )
ii.disting.all_notes_off( )
ii.disting.dual_parameter( side, param, value )
ii.disting.dual_scale_parameter( side, param, value )
ii.disting.dual_algorithm( side, algorithm )
ii.disting.dual_algorithms( algorithm_left, algorithm_right )
ii.disting.load_dual_preset( side, preset )
ii.disting.save_dual_preset( side, preset )
ii.disting.take_over_z( side, value )
-- queries
ii.disting.get( 'preset' )
ii.disting.get( 'algorithm' )
ii.disting.get( 'parameter', parameter )
ii.disting.get( 'parameter_min', parameter )
ii.disting.get( 'parameter_max', parameter )
ii.disting.get( 'lp_state', loop )
ii.disting.get( 'dual_parameter', side )
ii.disting.get( 'dual_parameter_min', side )
ii.disting.get( 'dual_parameter_max', side )
```

### er301 (`ii.er301`)

```lua
-- commands (no queries)
ii.er301.tr( port, state )
ii.er301.tr_tog( port )
ii.er301.tr_pulse( port )
ii.er301.tr_time( port, ms )
ii.er301.tr_pol( port, rising )
ii.er301.cv( port, volts )
ii.er301.cv_slew( port, ms )
ii.er301.cv_set( port, volts )
ii.er301.cv_off( port, volts )
```

### faders — 16n Faderbank (`ii.faders`)

```lua
-- queries only (read fader 1-16)
ii.faders.get( 1 )
-- ... through ...
ii.faders.get( 16 )

-- responses: e.name is the fader number itself
ii.faders.event = function( e, value )
  if e.name == 1 then  -- fader 1 value
  end
end
```

### jf — Just Friends (`ii.jf`)

Full curated reference above (trigger modes, synthesis/Geode, getters). Complete index:

```lua
-- commands
ii.jf.trigger( channel, state )
ii.jf.run_mode( mode )
ii.jf.run( volts )
ii.jf.transpose( pitch )
ii.jf.vtrigger( channel, level )
ii.jf.mode( mode )
ii.jf.tick( clock_or_bpm )
ii.jf.play_voice( channel, pitch_or_divs, level_or_repeats )
ii.jf.play_note( pitch_or_divs, level_or_repeats )
ii.jf.god_mode( state )
ii.jf.retune( channel, numerator, denominator )
ii.jf.quantize( divisions )
ii.jf.pitch( channel, pitch )
ii.jf.address( index )
-- queries
ii.jf.get( 'trigger', channel )
ii.jf.get( 'run_mode' )
ii.jf.get( 'run' )
ii.jf.get( 'transpose' )
ii.jf.get( 'mode' )
ii.jf.get( 'tick' )
ii.jf.get( 'god_mode' )
ii.jf.get( 'quantize' )
ii.jf.get( 'speed' )
ii.jf.get( 'tsc' )
ii.jf.get( 'ramp' )
ii.jf.get( 'curve' )
ii.jf.get( 'fm' )
ii.jf.get( 'time' )
ii.jf.get( 'intone' )
```

### kria — ansible kria (`ii.kria`)

```lua
-- commands
ii.kria.preset( number )
ii.kria.pattern( number )
ii.kria.scale( number )
ii.kria.period( time )
ii.kria.position( track, param, pos )
ii.kria.loop_start( track, param, pos )
ii.kria.loop_length( track, param, pos )
ii.kria.reset( track, param )
ii.kria.mute( track, state )
ii.kria.toggle_mute( track )
ii.kria.clock( track )
ii.kria.page( page )
ii.kria.cue( pattern )
ii.kria.direction( track, direction )
-- queries
ii.kria.get( 'preset' )
ii.kria.get( 'pattern' )
ii.kria.get( 'scale' )
ii.kria.get( 'period' )
ii.kria.get( 'position', track, param )
ii.kria.get( 'loop_start', track, param )
ii.kria.get( 'loop_length', track, param )
ii.kria.get( 'reset', track )
ii.kria.get( 'mute', track )
ii.kria.get( 'cv', track )
ii.kria.get( 'duration' )
```

### levels — ansible levels (`ii.levels`)

```lua
-- commands
ii.levels.preset( number )
ii.levels.reset( now )
ii.levels.position( pos )
ii.levels.loop_start( pos )
ii.levels.loop_length( pos )
ii.levels.loop_direction( direction )
-- queries
ii.levels.get( 'preset' )
ii.levels.get( 'position' )
ii.levels.get( 'loop_start' )
ii.levels.get( 'loop_length' )
ii.levels.get( 'loop_direction' )
ii.levels.get( 'cv', track )
```

### meadowphysics (`ii.meadowphysics`)

```lua
-- commands
ii.meadowphysics.preset( number )
ii.meadowphysics.reset( track )
ii.meadowphysics.stop( track )
ii.meadowphysics.scale( number )
ii.meadowphysics.period( time )
-- queries
ii.meadowphysics.get( 'preset' )
ii.meadowphysics.get( 'stop' )
ii.meadowphysics.get( 'scale' )
ii.meadowphysics.get( 'period' )
ii.meadowphysics.get( 'cv', track )
```

### txi (`ii.txi`)

```lua
-- queries only (no commands)
ii.txi.get( 'param', channel )
ii.txi.get( 'param_quant', channel )
ii.txi.get( 'param_N', channel )
ii.txi.get( 'in', channel )
ii.txi.get( 'in_quant', channel )
ii.txi.get( 'in_N', channel )
```

### txo (`ii.txo`)

```lua
-- commands only (no queries)
-- triggers
ii.txo.tr( port, state )
ii.txo.tr_tog( port )
ii.txo.tr_pulse( port )
ii.txo.tr_time( port, ms )
ii.txo.tr_pol( port, rising )
ii.txo.tr_time_s( port, s )
ii.txo.tr_time_m( port, m )
ii.txo.tr_pulse_div( port, pulses )
ii.txo.tr_m( port, ms )
ii.txo.tr_m_s( port, s )
ii.txo.tr_m_m( port, m )
ii.txo.tr_m_bpm( port, bpm )
ii.txo.tr_m_act( port, state )
ii.txo.tr_m_sync( port, state )
ii.txo.tr_m_width( port, width )
ii.txo.tr_m_count( port, count )
ii.txo.tr_m_mul( port, mult )
ii.txo.tr_pulse_mute( port, state )
-- CV
ii.txo.cv( port, volts )
ii.txo.cv_slew( port, ms )
ii.txo.cv_set( port, volts )
ii.txo.cv_off( port, volts )
ii.txo.cv_slew_s( port, s )
ii.txo.cv_slew_m( port, m )
ii.txo.cv_qt( port, qt )
ii.txo.cv_qt_set( port, qt )
ii.txo.cv_n( port, note )
ii.txo.cv_n_set( port, note )
ii.txo.cv_scale( port, scale )
ii.txo.cv_log( port, scale )
-- oscillators
ii.txo.osc( port, volts )
ii.txo.osc_set( port, volts )
ii.txo.osc_qt( port, volts )
ii.txo.osc_qt_set( port, volts )
ii.txo.osc_n( port, note )
ii.txo.osc_n_set( port, note )
ii.txo.osc_fq( port, fq )
ii.txo.osc_fq_set( port, fq )
ii.txo.osc_lfo( port, fq )
ii.txo.osc_lfo_set( port, fq )
ii.txo.osc_wave( port, wave )
ii.txo.osc_sync( port, sync )
ii.txo.osc_phase( port, phase )
ii.txo.osc_width( port, width )
ii.txo.osc_rect( port, pol )
ii.txo.osc_slew( port, ms )
ii.txo.osc_slew_s( port, s )
ii.txo.osc_slew_m( port, m )
ii.txo.osc_scale( port, scale )
ii.txo.osc_cyc( port, ms )
ii.txo.osc_cyc_s( port, s )
ii.txo.osc_cyc_m( port, m )
ii.txo.osc_cyc_set( port, ms )
ii.txo.osc_cyc_s_set( port, s )
ii.txo.osc_cyc_m_set( port, m )
ii.txo.osc_ctr( port, ctr )
-- envelopes
ii.txo.env_act( port, state )
ii.txo.env_att( port, ms )
ii.txo.env_att_s( port, s )
ii.txo.env_att_m( port, m )
ii.txo.env_dec( port, ms )
ii.txo.env_dec_s( port, s )
ii.txo.env_dec_m( port, m )
ii.txo.env_trig( port, state )
ii.txo.env_eoc( port, state )
ii.txo.env_eor( port, state )
ii.txo.env_loop( port, state )
ii.txo.env( port, state )
-- management
ii.txo.kill( port )
ii.txo.tr_init( port )
ii.txo.cv_init( port )
ii.txo.cv_calib( port )
ii.txo.cv_reset( port )
ii.txo.init( unit )
```

### wdel / wsyn / wtape — W/Del, W/Syn, W/Tape

See the curated "i2c Device References" above — the cheat sheet's command/query indexes for these three devices match those sections (their getters have been merged in there).

---

## Common Script Patterns

### Basic Script Structure

```lua
function init()
  -- initialize your script
  input[1].mode('stream', 0.1)
  output[1].slew = 0.01
  output[1].volts = 0
end

-- input event handler
input[1].stream = function(volts)
  output[1].volts = volts
end

-- cleanup on script change
function cleanup()
  -- optional: called before a new script is loaded
end
```

### Sequencer with sequins + clock

```lua
function init()
  ii.jf.mode(1)  -- enable Just Type
  clock.run(sequencer)
end

function sequencer()
  while true do
    clock.sync(1/4)  -- sync to quarter notes
    ii.jf.play_note(s{0,4,7,12}()/12, 5)
  end
end
```

### Polyphonic Arpeggiator with Just Friends

```lua
function init()
  ii.jf.mode(1)
  notes = sequins{0, 4, 7, 11, 14}/12  -- V/8 values
  clock.run(arpeggio)
end

function arpeggio()
  while true do
    clock.sync(1/4)
    ii.jf.play_note(notes(), 5)
  end
end
```

### W/Tape Live Looper

```lua
function init()
  ii.wtape.play(1)       -- start playback
  ii.wtape.record(1)     -- start recording
end

-- input 1: trigger record toggle
input[1].mode('change', 1, 0.1, 'rising')
input[1].change = function()
  recording = not recording
  ii.wtape.record(recording and 1 or 0)
end

-- input 2: set loop points
input[2].mode('change', 1, 0.1, 'rising')
input[2].change = function()
  if not looping then
    ii.wtape.loop_start()
    looping = true
  else
    ii.wtape.loop_end()
    ii.wtape.record(0)  -- disable recording when loop is set
  end
end
```

### W/Del Clock-Synced Delay

```lua
function init()
  ii.wdel.time(0.5)     -- 500ms delay
  ii.wdel.mix(0.7)      -- 70% wet
  ii.wdel.feedback(3.5) -- medium feedback
  clock.run(send_clock)
end

function send_clock()
  while true do
    clock.sync(1/4)
    ii.wdel.clock()
  end
end
```

### W/Syn FM Drone

```lua
function init()
  ii.wsyn.fm_ratio(3, 1)    -- 3:1 FM ratio
  ii.wsyn.fm_index(3)       -- moderate FM
  ii.wsyn.lpg_time(-2)      -- long vactrol
  ii.wsyn.voices(4)         -- 4 polyphonic voices
  ii.wsyn.play_note(0, 5)   -- play C3 at 5V
end
```

### Geode Rhythm Generator

```lua
function init()
  ii.jf.mode(1)           -- enter Just Type
  ii.jf.tick(120)         -- set BPM
  ii.jf.quantize(8)       -- quantize to 8ths
end

-- trigger different rhythmic patterns
input[1].mode('change', 1, 0.1, 'rising')
input[1].change = function()
  ii.jf.play_note(sequins{3,5,7,11}(), -1)  -- indefinite repeats at different divisions
end
```

### Hotswap Live-Coding

```lua
function init()
  hotswap.seq = sequins{0, 4, 7, 11}/12
  hotswap.tt = timeline.loop{1, {ii.jf.play_note, function() return hotswap.seq() end, 5}}
end

-- live-update the sequence without restarting:
-- hotswap.seq = sequins{0, 3, 7, 10}/12
-- hotswap.tt = timeline.loop{sequins{1,1,0.5}, {ii.jf.play_note, function() return hotswap.seq() end, 5}}
```

---

## Algorithmic & Generative Patterns

Distilled from the [bowery](https://github.com/awesomebrick/bowery) example scripts.

### Euclidean Rhythms (Bjorklund interleave)

Distribute `fill` pulses as evenly as possible across `length` steps; rotate by an offset for classic Euclidean rhythms (bowery/euclidean.lua):

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

### Chaos Source: Lorenz Attractor

Euler-integrate the Lorenz system and map state to CV; run at ~1kHz (bowery/lorenz.lua):

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

### Chaos Source: Gingerbreadman Map

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

### Boids Flocking (1D)

Flocking across the 4 outputs: each boid has position (volts) and velocity; steer toward a centre (blend with a CV input), avoid neighbours, match their velocity. Update one boid per tick (bowery/boids.lua):

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

### Analog Shift Register (ASR)

On each trigger, outputs present the previous stages, then shift a new sample in at the front (bowery/shiftregister.lua):

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

### Boolean Logic

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

### CV Delay (bucket brigade)

Stream an input at 1kHz into a circular buffer; each output reads an independent tap; a CV crossfades between endlessly looping and overwriting (bowery/cvdelay.lua):

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

### Runtime Clock-Division Presets

Swap all four output divisions live via a window-quantized CV input, without re-syncing the clocks (bowery/clockdiv.lua):

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

### One Sequins as a Data Bank

krahenlied.lua drives a whole piece from a single sequins built from a text string; `:settable()` hot-swaps the data from the REPL:

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

bowery/krahenlied.lua also reads the same sequins with different `:step(n)()` values to derive note, duration, and timbre lanes from one table.

### Polling ii Getters

Request params one at a time and handle replies in the event handler; pace requests to avoid i2c congestion (bowery/twigs/w_slash_stats.lua):

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

Two-arg getters work the same way, eg `ii.wsyn.get('patch', 1)`.

### Strum (spreading a chord on Just Friends)

```lua
function strum(notes)
  local t, mult = 0.02, 1.3
  for i = 1, #notes do
    ii.jf.play_note(notes[i] / 12, 3)
    clock.sleep(t)
    t = t * mult  -- each gap longer than the last
  end
end
```

### Idioms (one-liners)

```lua
dest = (dest % 4) + 1                       -- rotate a destination: sequential switch / round-robin fan-out
ii.jf.tick(clock.tempo)                     -- keep JF's Geode timebase locked to crow's tempo
output[n](lfo(dyn{freq=1}, 5))              -- CV-controlled LFO bank: update output[n].dyn.freq
                                            --   from an input stream handler (acquaintances.lua)
output[1].volts = hztovolts(f)              -- Hz -> V/8 inside an input[1].freq handler
                                            --   (input[1].mode('freq', 0.01) -- input 1 only)
```

---

## References

- **crow reference**: https://monome.org/docs/crow/reference/
- **crow source**: https://github.com/monome/crow
- **ASL library source**: https://github.com/monome/crow/blob/main/lua/asllib.lua
- **Just Friends (Just Type)**: https://github.com/whimsicalraps/Just-Friends/blob/main/Just-Type.md
- **W/Tape ii**: https://github.com/whimsicalraps/wslash/wiki/Tape#ii
- **W/Del ii**: https://github.com/whimsicalraps/wslash/wiki/Del#ii
- **W/Syn ii**: https://github.com/whimsicalraps/wslash/wiki/Syn#ii
- **crow community**: https://llllllll.co/
- **bowery example scripts**: https://github.com/awesomebrick/bowery
- **ii cheat sheet (all devices)**: https://github.com/awesomebrick/bowery/blob/main/twigs/ii_help.txt
