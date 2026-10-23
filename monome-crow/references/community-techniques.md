# Community techniques — recipes from the monome lines forum

This file collects **community-tested techniques** for crow, harvested from two long-running lines
threads: **"^^ crow: druid scripts"** ([25974](https://llllllll.co/t/crow-druid-scripts/25974), ~284
posts) and **"^^ crow strategies: Just Friends"**
([25937](https://llllllll.co/t/crow-strategies-just-friends/25937), ~166 posts). Everything here is
attributed to its author with a direct post URL of the form
`https://llllllll.co/t/crow-druid-scripts/25974/N` (thread A) or
`https://llllllll.co/t/crow-strategies-just-friends/25937/N` (thread B). Code is copied verbatim from
the threads, forum attachments, or the linked repos/gists; `…` marks elided lines.

These are *folk recipes*, not firmware documentation — where the community disagreed or a behaviour
looks firmware-dependent, the disagreement is preserved rather than resolved. Reference facts live in
the other files: timing/clocks in `api-time.md`, ASL in `api-asl.md`, sequins in `api-sequins.md`,
inputs/outputs in `api-input-output.md`, ii/JF command semantics in `ii-just-friends.md` and
`ii-overview.md`, and known pitfalls in `troubleshooting.md` (which you should always read before
copying a pattern from this file).

Fixed reference points used throughout: pitch `0V` = middle C = 261.63 Hz (monome docs call it
"C3"); crow handles **-5V to +10V**; `time()` returns **milliseconds**; the built-in is
**`delay(action, time [, repeats])`** (action FIRST); `metro.init` is **lowercase**;
`input[n].mode('clock', division)` (division like `1/4`) exists and raises **no** Lua event.

---

## 1. Clock & tempo techniques

### 1.1 External-clock BPM detection + metro slaving (forrest)

Detect the interval between incoming pulses with `time()` (milliseconds!), derive BPM, and re-time a
metro so everything downstream rides the external clock. ([forrest, 25974/48](https://llllllll.co/t/crow-druid-scripts/25974/48))

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

"Then you could just take that metro and fire on every X events to divide or whatever, have another
metronome set to be a multiple of it to multiply." Forrest flags one uncertainty himself: "I don't
know if updating the metro speed that often has any drawbacks but it seems to work ok for me."

Forrest points to voidstar's `modes.lua` for the smoother variant: a **running average** of recent
timestamps rather than the last interval alone
([voidstar attachment, 25974/1](https://llllllll.co/t/crow-druid-scripts/25974/1); see §7):

```lua
input[1].change = function(state) 
	table.remove(intervalRegister) -- Remove last timestamp
	table.insert(intervalRegister,1,time())	-- Place the current count into the first spot in the register

	-- Calculate the average difference of timestamps 
	average = 0
	for i=1,#intervalRegister-1 do 
		average = average+intervalRegister[i]-intervalRegister[i+1]
	end
	average = average/(#intervalRegister-1)
	…
```

Note `elapsed * 0.001` above: `time()` is in **milliseconds**, `metro[].time` and `lfo()` want
**seconds** (see `api-time.md`).

### 1.2 4-division clock rack, with CV tempo select (Justmat)

Four clock divisions straight out of the box; the second version selects tempo from a table via
voltage windows on input 1. ([Justmat, 25974/223](https://llllllll.co/t/crow-druid-scripts/25974/223),
reposted for a beginner in [25974/275](https://llllllll.co/t/crow-druid-scripts/25974/275))

```lua
-- clock source thing
--

clock.tempo = 120

function init()
  for i = 1, 4 do
    output[i]:clock(i/4)
  end
end
```

"you can set the tempo from druid using beats per minute/bpm. just issue `clock.tempo = your_bpm`"
— and Justmat's clarification of the divisions
([25974/277](https://llllllll.co/t/crow-druid-scripts/25974/277)): "output 1 will be the straight
clock, output 2 is divided by 2, out 3 is divided by 3 and out 4 by 4."

Voltage-controlled variant ("if you want something you can control with voltage, instead of a
computer") — note this one writes `output[i]:clock(1/i)`: periods of 1, 1/2, 1/3, 1/4 **beats**
(1–4 pulses per beat — a multiplier bank). The first snippet's `clock(i/4)` gives periods of
1/4, 2/4, 3/4, 4/4 beats — every 1st/2nd/3rd/4th **16th-note**, which is what Justmat's
"divided by 2/3/4" means. (`:clock(div)` takes a **period in beats**, same units as
`clock.sync`.)

```lua
-- clock source thing
--

clock.tempo = 120

bpms = {76, 86, 100, 120, 140, 160}

function init()

  input[1].mode( 'window', {0, 1, 2, 3, 4, 5})
  input[1].window = function(v) clock.tempo = bpms[v] end


  for i = 1, 4 do
    output[i]:clock(1/i)
  end
end
```

"the above uses voltage present at input 1 to select a bpm from the bpm table. 0 volts is 76bpm, and
5 volts is 160bpm."

### 1.3 Input `'clock'` mode: slave `clock` to an external pulse (Galapagoose)

`input[n].mode('clock', …)` feeds the `clock` module directly from a gate/clock source — "this is
actually built into the input lib since v3 (though is undocumented)":
([Galapagoose, 25974/155](https://llllllll.co/t/crow-druid-scripts/25974/155))

```lua
-- uses 'change' mode detection
input[1].mode('clock')

-- or you can customize
input[1].mode('clock', 1.0, 0.2) -- 1V threshold with 0.2V hysteresis
```

Two hard facts from the debugging that followed
([Galapagoose, 25974/146](https://llllllll.co/t/crow-druid-scripts/25974/146) and
[25974/148](https://llllllll.co/t/crow-druid-scripts/25974/148)):

- **`'clock'` mode raises no `.change` event** ("`input[2].change` will never be called because
  you're in `clock` mode (which doesn't have an event)"), and **you cannot combine `'clock'` and
  `'change'` on the same input** ("you only have a access to the raw voltage on the jack (you can't
  use `clock` and `change` mode at the same time)").
- If you need both external sync *and* edge-triggered behaviour, **poll the raw voltage from a
  `clock.run` loop** (this was Galapagoose's answer to Oootini's "stop the clocks on idle" problem —
  full context in §1.6):

```lua
function init()
    input[2].mode('clock', 1, 'rising')

    -- monitor input for timeout
    clock.run(function()
        local timeout = clock.run(function() end)
        local in_state = false
        while true do
          local in_bool = (input[2].volts > 2) -- 2 is voltage threshold for clock
          if in_state ~= in_bool then -- if the state changed
            set_timeout(timeout, clock.cleanup)
            in_state = in_bool -- save new state
          end
          clock.sleep(0.01) -- poll the input. might need to be faster if clock is very narrow
        end
      end)

    -- save your note-updater in it's own clock for clarity
    note_clock = clock.run(function()
      while true do
        clock.sync(1)
        get_notes()
      end
    end)
    print('loaded seq m8')
end
```

(`set_timeout` is §1.6.) Galapagoose summarises the design gap honestly: "it's really not pretty, and
i think that's because this is a use-case i didn't even think about when designing". The alternative
is `input[n].mode('change')` — "the main thing you lose doing that is the ability to create clock
routines that run faster than the external clock, or at non-integer divisions of the clock."

The docs' form of the call takes the division as the first argument — `input[n].mode('clock',
division)` with a division like `1/4` — while Galapagoose's example above shows the
threshold/hysteresis form; whichever your firmware exposes, `'clock'` mode never fires a Lua event.
See `troubleshooting.md` and `api-input-output.md`.

### 1.4 Correct swing: `clock.sync` on-beat, `clock.sleep` off-beat (Galapagoose)

`clock.sync` cannot express swung beats directly, because it anchors to the origin:
([Galapagoose, 25974/187](https://llllllll.co/t/crow-druid-scripts/25974/187), answering
[ktnsft, 25974/186](https://llllllll.co/t/crow-druid-scripts/25974/186))

> Unfortunately `clock.sync` can't do this directly as you've written. The issue is that
> `clock.sync` looks for multiples of the sync value relative to the origin beat. So simple ratios
> work fine (1/4 will recur precisely inside every beat). Meanwhile `1/4 + 0.00001` (for example)
> will shift further and further forward in time every beat.
>
> What you want to do is `clock.sync(1/4)` every even beat (as you are already), then calculate a
> `clock.sleep(...)` every odd beat. The duration can be calculated with the `clock.get_beat_sec()`
> helper function. This will return the number of seconds for 1 beat. So something like:

```lua
clock.sleep(clock.get_beat_sec()/4 + swing_in_seconds)
```

The drifted attempt to compare against — additive offset on `clock.sync`, which accumulates error
([ktnsft, 25974/186](https://llllllll.co/t/crow-druid-scripts/25974/186)):

```lua
function clockit()
    while true do
        output[1]()
        if (oddbeat)
        then
            clock.sync(1/4)
        else
            -- thanks zebra on llllllll.co
            clock.sync(0.5 + swingfactor/16)
        end
        oddbeat = not oddbeat
        …
```

Related: jmsole's `swingclock` takes a different route entirely — "I first multiplied the clock by 2
to get a slower clock and then used normal outputs together with delayed outputs … Controlling how
much the even beats are delayed as well as adding repeats based on the division factor did the
trick. I also used nested delays" ([jmsole, 25974/87](https://llllllll.co/t/crow-druid-scripts/25974/87);
gist linked in §7). Note his caveat that dynamic `metro` re-timing needs crow firmware at commit
`f491781` or later.

### 1.5 One metro to rule them all: phase reconciliation (license / Dial)

Multiple loose metros jitter and go out of phase when tempos change
([license, 25974/112](https://llllllll.co/t/crow-druid-scripts/25974/112)). Both
[synthetivv, 25974/113](https://llllllll.co/t/crow-druid-scripts/25974/113) and
[Galapagoose, 25974/114](https://llllllll.co/t/crow-druid-scripts/25974/114) converge on the same
fix: **run a single high-resolution metro** (at the greatest common factor of the desired rates, or
higher) and derive everything else from a phase/threshold comparison inside it. License's motto for
the resulting design: *"Always off-course, always correcting"* — the polyrhythmic outputs continually
reconcile to the master phase instead of free-running. That architecture became **Dial**, "a
polyrhythmic, polyphasic clock source/metronome and ramp/phasor generator"
([license, 25974/116](https://llllllll.co/t/crow-druid-scripts/25974/116), library thread
[here](https://llllllll.co/t/dial-0-2-0/37732)). Helpers used along the way
([license, 25974/115](https://llllllll.co/t/crow-druid-scripts/25974/115)):

```lua
function v_to_bpm(v)
    v = math.min(math.max(v, v_min), v_max)
    local ratio = (v - v_min) / (v_max - v_min)
    local new_bpm = bpm_min + (ratio * (bpm_max - bpm_min))
    local frac = new_bpm % 1
    new_bpm = new_bpm - frac
    return new_bpm + (frac < 0.5 and 0 or 1)
end
```

```lua
function div_to_seconds(divisor)
    return 1 / (bpm / 60 * divisor)
end
```

Timing reality-check from the same exchange: metros and input streams run on hardware timers with
sub-microsecond accuracy; jitter is scheduling, not hardware
([Galapagoose, 25974/111](https://llllllll.co/t/crow-druid-scripts/25974/111)). Coroutines are
cooperative multitasking, not extra threads
([Galapagoose, same post](https://llllllll.co/t/crow-druid-scripts/25974/111)). See `api-time.md`.

### 1.6 Idle-timeout / cleanup pattern (Galapagoose)

"Stop my clocks when the input goes quiet": a **timeout clock** that is cancelled and respawned on
every input event, calling `clock.cleanup()` when it expires
([Galapagoose, 25974/144](https://llllllll.co/t/crow-druid-scripts/25974/144)):

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

Reusable version ("this seems kinda generally useful"):

```lua
function set_timeout(coro, fn)
  clock.cancel(coro) -- stop the current timeout
  coro = clock.run(function() -- start the new timeout
      clock.sleep(1) -- set timeout in seconds
      fn() -- call the on-timeout function
    end)
end
```

(The original post has a stray `)` after `clock.cancel(coro)`. Note `set_timeout` cannot rebind the
caller's `coro` variable — pass and store the returned id yourself if you need repeated resets.) If
the input is in `'clock'` mode, combine this with the polling loop from §1.3.

---

## 2. Quantization & tuning

### 2.1 Hand-rolled scalar quantizer (voidstar)

"Expects a voltage and a table of semitone notes" — nearest-degree snap with octave folding:
([voidstar, 25974/5](https://llllllll.co/t/crow-druid-scripts/25974/5))

```lua
local scaleExample = {0,2,4,5,7,9,11,12} -- Major Scale

function quantize(volts,scale) 
	local oct = math.floor(volts)
	local interval = volts - oct
	local semitones = interval / (n2v(1))
	local degree = 1
	while degree < #scale and semitones > scale[degree+1]  do
		degree = degree + 1
	end
	local above = scale[degree+1] - semitones
	local below = semitones - scale[degree]
	if below > above then 
		degree = degree +1
	end
	note = scale[degree]
	note = note + 12*oct
	return note
end
```

One-line warning: the `interval / (n2v(1))` line predates the current `n2v` behaviour — see §2.6 and
substitute `/12`; joshhh reports exactly this broke the quantizers in voidstar's Ornithopters
([25974/231](https://llllllll.co/t/crow-druid-scripts/25974/231)).

### 2.2 Built-in scale quantization: `output[n].scale` / `input[n].mode('scale', …)` (crow 2.0+)

Since crow 2.0 the firmware ships a quantizer. csboling first pointed at it
([25974/91](https://llllllll.co/t/crow-druid-scripts/25974/91)); Galapagoose clarified that
`output[n].scale` is **setup only** — "it is a setup function, not a function that changes the value
at the output jack. … you first need to setup the scale quantizer (best in the `init` script), then
simply set `output.volts` in the input event"
([25974/135](https://llllllll.co/t/crow-druid-scripts/25974/135)):

```lua
function init()
  output[4].scale( {}, 12, 1.0 )
end

input[1].change = function()
  output[4].volts = r() -- call the random function again
end
```

(`{}` = chromatic; `12` = notes per octave, so `1.2` volts/oct for Buchla; deactivate with
`output[n].scale('none')`.)

The input-side version — quantize incoming CV and get an event on every new note, with the whole
thing live-swappable from the druid prompt
([dani_derks, 25974/175](https://llllllll.co/t/crow-druid-scripts/25974/175), building on the bowery
quantize snippet):

```lua
--- quantize
-- in1: input signal to quantize
-- out1: in1 quantized to provided scale
-- out2: generates a trigger pulse every time a new note occurs

maj = {0,2,4,5,7,9,11}
min = {0,2,3,5,7,8,10}
dorian = {0,2,3,5,7,9,10}
majPent = {0,2,4,7,9}

function init()
  input[1].mode('scale', majPent)

  input[1].scale = function(s)
    output[1].volts = s.volts
    output[2]()
  end

  output[2].action = pulse()
end
```

```
> input[1].mode('scale', maj)
> input[1].mode('scale', min)
> input[1].mode('scale', dorian)
> input[1].mode('scale', {0,4,19})
```

The full `input[n].mode('scale', {scale}, temperament, scaling)` signature also appears in the wild
with explicit temperament/scaling, e.g. `input[1].mode('scale', {0,3,5,7,10}, 12, 1.0)`
([vicimity, 25974/117](https://llllllll.co/t/crow-druid-scripts/25974/117)).

### 2.3 Scale handler fields: `s = {index, octave, note, volts}`

The `input[n].scale` handler receives a table with (at least) `index`, `octave`, `note`, `volts`.
From the JI debugging thread ([ObscBySds, 25974/181](https://llllllll.co/t/crow-druid-scripts/25974/181),
[25974/183](https://llllllll.co/t/crow-druid-scripts/25974/183) and
[Galapagoose, 25974/184](https://llllllll.co/t/crow-druid-scripts/25974/184)):

- **`s.volts`** = quantized volts, ready for `output[n].volts` — with temperament `'ji'` this already
  has the just-intonation conversion applied.
- **`s.index`** = index into your notes/ratio table — *raw*, not converted. "The `s.index` call just
  returns the index into your scale that is currently active. This is provided so you can
  decorrelate input windows from output scale selection (or other weird edge cases)."
- **`s.octave`** = octave offset of the detected note. ObscBySds found "`s.index` is one upper octave
  apart from `s.volts` for the same ratio and input voltage", needing `- 1` compensation when
  rebuilding volts by hand:

```lua
function vox2(s)
  local note = myscale[s.index] + s.octave - 1
  output[1].volts = note
  output[2]( pulse() )
end
```

Galapagoose's resolution: if you index the table yourself, convert explicitly —
`just12(myscale[s.index])` for a 12TET number, `justvolts(myscale[s.index])` for volts.

### 2.4 Just intonation: `'ji'` temperament + `justvolts()` / `just12()`

With `input[n].mode('scale', myscale, 'ji')` the scale table holds **ratios**. "It's important to
realize that there is nothing special about this line. It's just plain Lua. That means it's a table
of floating point values which are calculated via division when this line is executed. That's why
`9/8 == 1.125`. Lua doesn't have type-level support for ratios"
([Galapagoose, 25974/184](https://llllllll.co/t/crow-druid-scripts/25974/184)). Convert with
`justvolts(ratio)` (→ volts for an output) or `just12(ratio)` (→ 12TET number). juhbuba traced the
implementation to `just12()` in `lua/crowlib.lua`, applied in `lua/input.lua` when the temperament
argument is a string like `'ji'` ([25974/182](https://llllllll.co/t/crow-druid-scripts/25974/182);
PR [#380](https://github.com/monome/crow/pull/380)).

### 2.5 La Monte Young "Well-Tuned Piano" JI script (ObscBySds)

Full script: one JI scale feeding two JF voices (offset by table indices for harmony) plus a CV
output, with a running transpose. Posted by
[ObscBySds, 25974/185](https://llllllll.co/t/crow-druid-scripts/25974/185) (built on Galapagoose's
`s.index`/`justvolts` guidance; `vox2` gained its missing `note` line in the author's edit):

```lua
lamonte = {1/1,567/512,9/8,147/128,21/16,1323/1024,189/128,3/2,49/32,7/4,441/256,63/32}
myscale = lamonte

function init()
    ii.jf.mode(1)
    ii.jf.run_mode(1)
    ii.jf.run(5)
    input[1].mode( 'scale', myscale, 'ji')
end

function vox1(s)
  local indx = (s.index + 5) % #myscale + 1
  local note = justvolts(myscale[indx]) + s.octave
  ii.jf.play_note(note, (math.random() * 5.0 + 0.5))
end

function vox2(s)
  local indx = (s.index + 2) % #myscale + 1
  local note = justvolts(myscale[indx]) + s.octave 
  ii.jf.play_note(note, (math.random() * 5.0 + 0.5))
end

function vox3(s)
  local note = justvolts(myscale[s.index]) + s.octave
  output[1].volts = note + 3/12
  output[2]( pulse() )
end

input[1].scale = function(s)
  vox1(s)
  vox2(s)
  vox3(s)
  ii.jf.transpose(3/12 - 1)
end
```

The `indx = (s.index + k) % #myscale + 1` trick is how you decorrelate voices from the input window
while staying in one scale — harmony from a single quantized source.

### 2.6 `n2v()` returns a FUNCTION, not a number (csboling)

A classic trap, debugged live in thread A
([Wisdom.Water, 25974/94](https://llllllll.co/t/crow-druid-scripts/25974/94) reports *'attempt to
perform arithmetic on a function value (global offset)'*; diagnosis by
[csboling, 25974/95](https://llllllll.co/t/crow-druid-scripts/25974/95) and
[25974/97](https://llllllll.co/t/crow-druid-scripts/25974/97)):

> I think this is because `n2v` is a helper for building ASLs and returns a function. For just
> converting a note value to a V/oct voltage it's overkill, just divide by 12 instead of calling n2v.

> This is because it's intended for helping write ASL expressions, where you want it to calculate a
> voltage at runtime rather than a fixed value determined at ASL definition time. … If you just want
> to convert a semitone to a volt-per-octave voltage value, you just need to divide by 12, since 12
> semitones = 1 octave maps to 1 volt.

So: inside ASL, `n2v(semitones)` is fine; in plain script code use `semitones/12`. (Octave maths is
direct: `-1` volt = one octave down, not `-12` — see
[Galapagoose, 25974/267](https://llllllll.co/t/crow-druid-scripts/25974/267): "the .volts signals are
all direct voltage levels where 1V = 1 octave.")

---

## 3. Sequins & pattern design

### 3.1 One-liner REPL sequencer (dani_derks)

Paste into druid and you have a melody + per-note pulse stream
([dani_derks, 25974/179](https://llllllll.co/t/crow-druid-scripts/25974/179)):

```lua
mys = sequins{0,2,4,7,9}; clock.run(function() while true do clock.sync(1); output[1].volts = mys()/12; output[2](pulse()) end end)
```

### 3.2 The nested-`:every()` trap — use `:count()` and fractions instead (Galapagoose)

Eastwood wanted "a triplet pattern every four bars"
([25974/211](https://llllllll.co/t/crow-druid-scripts/25974/211)):

```lua
kick_pattern = sequins{ 1, 1, 1, sequins{ sequins{0.33, 0.33, 0.33}:all():every(4), 1 } }
```

```lua
kick_timeline = timeline.loop{ kick_pattern, kick_function }
```

This does **not** work — "`:all():every(4)` is captured inside the doubly-nested sequins. perhaps it
doesn't bubble out to the next layer … basically i'm not sure sequins is designed to express this
kind of dual-branching condition"
([Galapagoose, 25974/212](https://llllllll.co/t/crow-druid-scripts/25974/212)). The working shape
counts the plain beats and substitutes the triplet for the last slot:

```lua
sequins{ sequins{1}:count(15), sequins{1/3}:count(3) }
```

Also from that answer: **use fractions, not decimals** — "because the timing in `timeline` is not
synchronized after it launches, it's best to use a fraction for triplets to avoid drift across many
repetitions" (`1/3`, never `0.33`). See `api-sequins.md` and `api-time.md`.

### 3.3 Sequins-of-functions as a phrase selector + step size at call time (onoma)

In the jazz-ensemble script (§3.4) the phrase generator itself is a sequins of *functions*:

```lua
sw = s{fstr,slwr} -- jazzing sequins
```

then each phrase slot runs one: `clock.run(sw())`. Likewise the main melody takes its step size from
another sequins **at call time**: `mld:step(stp())()` — the parentheses order matters: `stp()`
produces the step, `:step(...)` re-targets `mld`, the final `()` reads the note. See
[onoma, 25937/164](https://llllllll.co/t/crow-strategies-just-friends/25937/164).

### 3.4 Sequins arithmetic `s{…} + s{…}` for layered parts (onoma's jazz ensemble)

`+` on two sequins adds their values element-wise, so a melody line and its harmony/transpose layer
stay locked while both wander independently. Full script as posted
([onoma, 25937/164](https://llllllll.co/t/crow-strategies-just-friends/25937/164) — "JF jazz ensemble
based on jfstrumm", with audio on the post):

```lua
--- jazzt piano and double bass

--- reqs
output[1]:clock(4)
ii.jf.mode(1)
clock.tempo = 100 -- base tempo

s = sequins


--- musical playground here
mld = s{-2,0,3,8,5,8,14,s{15,18,12}} + s{0,0,s{-12,0}} -- main jazz improv, rn dom7
stp = s{1,2,3,-1} -- 2nd line direction 
bss = s{0,0,7,s{-2,5,-2,-4}} + s{-12} -- reference bassline


t = 1 -- strumming start (lower = faster)
hm = s{7.5,15,30,60} -- strumming sparcity sequence 

vl1 = s{1,2,2.5,1} -- melody velocity
vl2 = s{2,1.5,1.5,1.5} -- bass velocity


slp = 480 / clock.tempo -- lower faster (ratios of 60 to make it tempo related) 


---  voicing
fstm = function()
    ii.jf.play_note(mld()/12, vl1())
end

tndm = function()
    ii.jf.play_note(mld:step(stp())()/12, vl2())
end

cnst = function()
    ii.jf.play_voice(1,bss()/12, v2())
end


--- strumming
divi = function()
    while true do
        clock.sync(16) -- strumming pattern sequis quantizaion
        t = hm()/clock.tempo
        print(t)
    end
end

fstr = function()
    local tm = 0.75
    for i=1,7 do
        fstm()
        clock.sleep(t)
        t = t * tm
        
    end
end

slwr = function()
    local tm = 1.25
    for i=1,7 do
        tndm()
        clock.sleep(t)
        t = t * tm
    end
end

sw = s{fstr,slwr} -- jazzing sequins


--- clocked output
jazz = function()
    while true do
        clock.run(sw())     
        clock.sleep(sleep)
    end
end


bass = function()
    while true do
        clock.sync(1)
        cnst()
    end
end



function init()
    playbass = clock.run(bass) --- druid ==> clock.cleanup(play*) to make voice stop
    playjazz = clock.run(jazz) --""--
    lessjazz = clock.run(divi) --- druid ==> clock.cleanup(lessjazz) to stop strumming changes
end
```

Two names in this script (`v2()` in `cnst`, `sleep` in `jazz`) are unresolved as posted — treat them
as authoring stubs (`vl2` / `slp` are the likely intents) and fix locally. The strumming idea: each
`fstr`/`slwr` phrase fires 7 notes whose spacing multiplies by `tm` (`0.75` = accelerating strum,
`1.25` = decelerating), re-entrained by `divi` every 16 beats.

### 3.5 Pattern-string drum machine with bar-quantized swap queue (nattog, drummer.lua)

Four tracks of `'x---x---x---x---'` notation, live-editable from druid; new patterns are **queued**
and land on the next bar boundary. (Posted as a truncated embed by
[nattog, 25974/131](https://llllllll.co/t/crow-druid-scripts/25974/131) — "Heyyyyyyy! This is so dope"
— [Galapagoose, 25974/132](https://llllllll.co/t/crow-druid-scripts/25974/132); full source from the
linked [gist](https://gist.github.com/nattog/841bcfb813aefdc751cacf0d4b79f4ab)):

```lua
-- drummer
-- 4 track pattern sequencer
-- pattern notation looks like this 'x---x---x---x---'
-- update sequence track with set(track, pattern)
-- 
-- queue a new sequence
-- progress(seq table, period - by default 16)
--
-- control transport
-- start() stop() reset()

counter = 0 -- used to get index by modulo of seq items
seq = {"", "", "", ""}
pause = false
preset = {'x---', '----x---', '--x-', 'xx--x--'}
queued = {}
wait = 0

function init()
    -- set up input modes
    input[1].mode("change", 1, 0.1, "rising")
    input[2].mode("change", 1, 0.1, "rising")
    input[1].change = function() if pause == false then step() end end
    input[2].change = reset

    -- set up output modes
    for i = 1, 4 do 
        output[i].action = pulse(0.02,8)
    end
end

-- resets counter variable to initial value
function reset()
    counter = 0
end 

-- update counter and trigger for each 'x' found at index of each seq
function step()
    local to_print = ""
    for chan = 1, #seq do
        if chan <= 4 then
            if #seq[chan] > 0 then
                local step_idx = math.floor(counter % #seq[chan]) + 1
                local step_value = seq[chan]:sub(step_idx, step_idx)
                if step_value == 'x' then
                    output[chan]()
                end
                to_print = to_print .. (step_value == 'x' and step_value or "_") .. " "
            else 
                to_print = to_print .. "_ "
            end
        end
    end

    -- increments counter
    counter = counter + 1

    if #queued > 0 then
        -- countdown wait if queue has elements
        if wait > 0 then
            wait = wait - 1
        else 
            -- can set the queue
            seq = queued
            wait = 0
            queued = {}
        end
    end
    print(to_print)
end

function set(idx, val)
    -- sets all tracks to same val
    local to_print = ''
    if idx == "*" then
        for i = 1, #seq do
            seq[i] = #val > 0 and val or ''
            to_print = 'SET ALL TO ' .. string.upper(seq[1])
        end
    -- only set sequences for valid output indexes
    elseif idx < 5 and idx > 0 then
        seq[idx] = #val > 0 and val or ''
        to_print = 'SET ' .. idx .. ' TO ' .. string.upper(seq[idx]);
    end

    print('\n' .. to_print .. '\n')
end

function stop()
    pause = true
end

function start()
    pause = false
end

function progress(tab, period)
    local progress_on = period or 16
    if tab == nil then
        print('no input')
        return
    end
    -- if input is empty table, set all channels to empty strings
    local new_seq = #tab < 1 and {"", "", "", ""} or table.shallow_copy(tab)

    -- if running then a wait period is necessary
    if pause == false then
        wait_period = progress_on % counter 
        -- add seq to queue
        if wait_period ~= 0 then
            queued = new_seq
            wait = wait_period
        else
            seq = new_seq
        end
    else 
        -- update seq
        seq = new_seq
        reset()
    end
end

function table.shallow_copy(t)
    local t2 = {}
    for k,v in pairs(t) do
      t2[k] = v
    end
    return t2
end
```

The bar-quantized swap is the `progress()` tail: if the transport is running, the new table goes into
`queued` with `wait = progress_on % counter`; `step()` counts `wait` down and only then swaps `seq`
in. Different pattern lengths per track (`#seq[chan]` modulo) give polymeters for free.

### 3.6 Integers as rhythm: `bit32` shifts and binary decomposition

**`bit32` walk (gigasturtz's jfInt)** — "it plays notes from an array of arrays using integers for
rhythmic information"; each beat is an integer whose bits are clocked out one per step
([gigasturtz, 25974/46](https://llllllll.co/t/crow-druid-scripts/25974/46), attachment verbatim):

```lua
--generate rhythms from integers


function init()
  ii.pullup(true)
  ii.jf.mode(1)
  ii.jf.transpose(-1)
  beat = { 255, 255, 255, 255 }
  chords = {
    { 15, 15, 19, 15, 15, 19, 15, 15 },
    { 14, 15, 17, 14, 15, 17, 14, 15 },
    …
    { 8, 15, 15, 8, 15, 15, 8, 15 }
  }
  chordDegree = {1}
  currentChord = {1}
  pattern = {1}
  beatReset = {1}
  input[1].mode('change', 1.0,0.1,'rising')
end

input[1].change = function(state)
  rhythm_loop(8)
end

function rhythm_loop(beatLength)
    if math.fmod(bit32.rshift(beat[pattern[1]], beatReset[1]), 2) == 1 then
      if chordDegree[1] == 1 then
        ii.jf.play_note(.0833  * (chords[currentChord[1]][chordDegree[1]] - 12),5)
      end
      ii.jf.play_note(.0833 * (chords[currentChord[1]][chordDegree[1]]),5)
      incrementValue(chordDegree, (#chords[currentChord[1]] + 1))
    end
    if incrementValue(beatReset, beatLength) then
      incrementValue(pattern, #beat)
      incrementValue(currentChord, #chords)
    end
end

function incrementValue(value, max)
  value[1] = value[1] + 1
  if value[1] == max then
    value[1] = 1
    return true
  end
  return false
end
```

`bit32.rshift(beat[i], step)` + `% 2 == 1` reads one bit per step — a whole 8-step gate pattern is a
single number like `255` (or `144`, `18`, `43691` in tejomay's `seq.lua`, §7). Also note the
single-entry-array trick for mutable state: "All the variables used for incrementing live in
single-entry arrays because Lua passes arrays by reference (and passes numbers/strings by value),
which is useful for maintaining global state while also allowing me to cut down the amount of
repeated code" ([25974/46](https://llllllll.co/t/crow-druid-scripts/25974/46)).

**Binary decomposition via lookup table (ringsandrocks' binarray / binarrayJF)** — "inspired by
Xaoc's Drezno. It converts an input into a four digit binary number whenever it receives a clock"
([ringsandrocks, 25974/219](https://llllllll.co/t/crow-druid-scripts/25974/219),
[demo video](https://www.youtube.com/watch?v=NeAauwc11ME)). The JF variant gates the bits straight
to Just Friends — see §5.10. Core of the attachment (`binArray` is a 32-row `{0,1}` table, rows
elided):

```lua
--binArray
--binary output of input inspired by xoac devices drezno
--in1 value to be sampled (0-5V, negative voltages are converted to absolute)
--in2 is clock
--out1-4 outputs value at in1 in binary with 0 being 0000 and 5 being 1111
inputVolts = 5 --set maximum volts
scaling = 32/inputVolts
clockDivision1 = 8
clockDivision2 = 32

inVal = 0
cDivCount1 = 0
cDivCount2 = 0
binArray = {{0,0,0,0,0},
             {0,0,0,0,1},
             …
             {1,1,1,1,0},
             {1,1,1,1,1}}
…
function vToB(inVal)
  vToI = clamp((math.ceil(math.abs(inVal * scaling))))
  for i=1,5 do
    ii.jf.trigger(i+1,binArray[vToI][i])
  end
  if inVal > 0 then ii.jf.trigger(1,1)
  else ii.jf.trigger(1,0)
  end

  output[1].volts = adder()
  output[2].volts = wiggle()
  output[3].volts = clockDiv1()
  output[4].volts = clockDiv2()

end
```

The plain `binarray.lua` variant sends the four bits to crow outputs 1–4 *and* the first four JF
channels; `binarrayJF.lua` repurposes the crow outputs as (1) sum of set bits, (2) random voltage
scaled to the input, (3–4) clock divisions `8` and `32` "for repeating patterns by patching it into
the sync or trigger of the voltage source that Input 1 is sampling".

### 3.7 Meta-sequencing: queued edit functions (crim's scrolls)

scrolls is "a little sequencer that's intended to be live-coded via druid"
([crim, 25974/21](https://llllllll.co/t/crow-druid-scripts/25974/21), grown-up version with docs in
[25974/25](https://llllllll.co/t/crow-druid-scripts/25974/25), repo
[crimclark/scrolls](https://github.com/crimclark/scrolls)). The architecture worth stealing has three
parts (source quoted from the repo):

1. **Edits are queued as functions, one per loop start.** Each druid command (`oct`, `mod`, `rv`,
   `mv`, `rm`, `rndm`, …) returns an *action* pushed through `updateQueue`:

```lua
function updateQueue(fn, count)
  table.insert(queues[currentSeq], {fn = fn, count = count or 1, id = time()})
  return fn
end
```

   and at every sequence wrap only one queued action is consumed
   (`handleSeqStart`/`doQueuedAction`):

```lua
function handleSeqStart(i)
  if #prevSeq[i] > 0 and queueLocations[i] > #queues[i] then
    sequences[i] = prevSeq[i]
    queueLocations[i] = 1
  end
  local queueLoc = queueLocations[i]
  if #queues[i] > 0 then
    if #prevSeq[i] == 0 then prevSeq[i] = sequences[i] end
    if currentLoop[i] <= queues[i][queueLoc].count then doQueuedAction(i, queueLoc) end
    if queues[i][queueLoc].count == -1 then loopAction(i, queueLoc) end
  end
end
```

2. **Edits are relative to a snapshot, never cumulative.** `s()` overwrites the snapshot with what is
   currently playing; until then every transform runs against the stored original, so calling
   `oct()` in a loop pitches up an octave *relative to the original* every time instead of running
   away ([README](https://github.com/crimclark/scrolls) — "Script will always store a reference to
   the original sequence, functions will be relative to that sequence"):

```lua
function s(seq)
  seq = seq or currentSeq
  prevSeq[seq] = sequences[seq]
  ptrn()
end

function oct(...)
  local args = parseArgs({...})
  return updateQueue(function(seq)
    for i=args.first,args.last do seq[i].note = seq[i].note + args.value*12 end
    return seq
  end, -1)
end
```

3. **`ptrn` composes the edits themselves into a sequence** — a meta-sequence of function calls with
   loop counts ("Pattern will loop infinitely until `ptrn()` is called with no args"):

```lua
function ptrn(...)
  local args = {...}
  local newQueue = {}
  local i = 1
  while i <= #args do
    if type(args[i]) == 'function' then
      local fn = args[i]
      local count = 1
      if type(args[i+1]) == 'number' then
        count = args[i+1]
        i = i+2
      else i = i+1 end
      table.insert(newQueue, {fn = fn, count = count, id = time()+i})
    else i = i+1
    end
  end
  queues[currentSeq] = newQueue
  queueLocations[currentSeq] = 1
end
```

   usage: `ptrn(mod(3), 2, oct(-1), 3, rndm(), rv(), 2)` — "modulate up a 3rd for 2 loops, transpose
   down an octave for 3 loops, randomize notes for 1 loop, reverse sequence for 2 loops."

Command surface (README): `ed(n)` picks sequence 1/2, `rndm()`, `sync(n)`, `oct(first,last,value)`
with flexible arity, `mod(...)` (transpose by *scale degree*), `rv`, `rm`, `mv(step,position)`,
`slw`, `eg`, `div[1] = 2` for clock division. "Updates take effect at the start of the sequence loop,
and you can currently only invoke 1 update per cycle." The earlier prototype in-thread
([crim attachment, 25974/21](https://llllllll.co/t/crow-druid-scripts/25974/21)) shows the same
druid-callable style in miniature: `rndm()`, `oct()`, `tp(7, 5, 6)`, `rev(9, 16)`, `cp()`,
`slew(4, 1)`, `eg(6, adsr())` — "It's also fun without the live coding if you send a trigger or lfo
into input 2 to change up the random melody."

### 3.8 Turing-machine shift registers (Ornithopters, voidstar)

Five independent feedback shift registers with per-register storage semantics (digital rhythm,
weighted-bit-sum "Turing Volts expander" analog, quantized variants, velocity envelopes), each
output reading a chosen `locations[i]` tap of a chosen register
([voidstar, 25974/14](https://llllllll.co/t/crow-druid-scripts/25974/14)). Live-druid configurable:

```lua
rhythm.length = 8
velocityEnvelope.length = 56
```

```lua
analog.min = -5
analog.max = 3.4
velocityEnvelope.max = 7.5
```

```lua
modes[3] = 'quantizedAnalog'
modes[4]='quantizedAnalog'
locations[3] = 1
locations[4] = 2
```

The output generation is reprogrammable per mode ("Advanced" section of the post):

```lua
actions.analog = function(outputIndex) 
    return lfo(output[outputIndex].rate,analog.register[locations[outputIndex]])
end
```

and new registers can be created at runtime — `myRegister = newRegister('voidstarReg',16,'digital',0,1)`,
fed via `processShiftRegisters(register,cv)`, read via `<varName>.register[location]`.

---

## 4. ASL, dyn & output techniques

### 4.1 LFO bank with re-bang phase sync (Galapagoose)

Matched-frequency LFOs whose phases can be re-synced by simply re-banging the outputs
([Galapagoose, 25974/71](https://llllllll.co/t/crow-druid-scripts/25974/71)):

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

Two forms of the LFO itself ([Galapagoose, 25974/65](https://llllllll.co/t/crow-druid-scripts/25974/65)):

```lua
-- assign an LFO to out1 with duration of 0.1s, 5V scaling & square shape
output[1].action = lfo( 0.1, 5, 'now' ) -- assign it
output[1]() -- start the LFO

-- or execute it immediately
output[1]( lfo( 0.1, 5, 'now' ) )
```

or hand-rolled in ASL ("Or in pre 1.0.3 ASL"):

```lua
output[1].action =
  loop{ to(5     , 0)
      , to('here', 0.05)
      , to(-5    , 0)
      , to('here', 0.05)
      }
```

### 4.2 Freeze an output mid-ASL (Galapagoose)

To hold the current voltage you must unset any active ASL — the idiom is one line
([Galapagoose, 25974/74](https://llllllll.co/t/crow-druid-scripts/25974/74)):

```lua
output[n].volts = output[n].volts -- set destination to current voltage
```

"This works because `output[n].volts` returns the current voltage at the output jack (not the
destination)."

### 4.3 `delay()` one-shots — the built-in takes ACTION FIRST

The built-in is **`delay(action, time [, repeats])`** — a function (not a call result!) first,
seconds second, optional repeat count third; the action receives the iteration index
`1 … 1+repeats`, and `delay` returns a coro id you can `clock.cancel`. The historical thread code
uses a custom wrapper with the **opposite** order — do not copy that order.

Working gate-delay, the outcome of a live debugging session
([Neither-cloud, 25974/227](https://llllllll.co/t/crow-druid-scripts/25974/227)):

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

The bug that led there: `delay(phase1, 0.5)` where `phase1 = output[1]()` — "the first argument to
`delay` needs to itself be a function (currently it's the result of a function call)"
([Galapagoose, 25974/225](https://llllllll.co/t/crow-druid-scripts/25974/225)). "You can probably just
pass `output[1]` directly to the delay function."

The historical wrapper to *avoid copying verbatim* — argument order `delay(time, action)`, from
before `delay` was built in ([Galapagoose, 25937/48](https://llllllll.co/t/crow-strategies-just-friends/25937/48)):

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
```

If you meet this wrapper in an old script, translate it to `delay(action, time)`. Two warnings from
the same exchange: `metro.init` is **lowercase** ("that needs to be lowercase `metro`" —
[Galapagoose, 25937/51](https://llllllll.co/t/crow-strategies-just-friends/25937/51)), and crow has
only 8 metros (`Metro.init: nothing available` means you exhausted them or used the wrong case).

### 4.4 `'window'` input mode as a voltage-controlled behavior switch (yaw / echophon)

"takes a clock signal & selects a mode based off the voltage window. If you attenuate the clock
signal, you'll still get tempo information but also be able to select different behaviors"
([yaw, 25974/129](https://llllllll.co/t/crow-druid-scripts/25974/129)):

```lua
-- switch between behaviors based on high voltage of clock
-- in1: attenuated clock
-- out1: voltages from mode a
-- out2: voltages from mode b
-- out3: voltages from mode c
-- out4: voltages from mode d
-- TODO implement clocking 


function init()
    input[1].mode( 'window', {0,1,2,3,4,5}, 0.1 )
end

input[1].window = function(state)
    -- skip the low window (clock is down)
    if state ~= 1 then  
        -- switch behavior based off state
        if state == 2 then
            mode_a(state)
        end
        if state == 3 then
            mode_b(state)
        end
        if state == 4 then
            mode_c(state)
        end
        if state == 5 then
            mode_d(state)
       end 
    end
end

function mode_a(state)
    output[1].volts = state
    output[2].volts = 0
    output[3].volts = 0
    output[4].volts = 0
    
end
…
```

The polished utility form — a voltage-controlled switch routing input 2 to one of four outputs
([yaw/echophon, 25974/136](https://llllllll.co/t/crow-druid-scripts/25974/136);
[echophon/crowscripts vcswitch.lua](https://github.com/echophon/crowscripts/blob/main/vcswitch.lua),
full source):

```lua
-- voltage controlled switch 
-- in1: select output based on 0-5v value
-- in2: cv input
-- out1-4: voltage from in2 (when selected)

function init()
    -- set window ranges of 1.25v to choose outputs
    input[1].mode( 'window', {0,1.25,2.5,3.75,5}, 0.1 )
    input[2].mode( 'stream', 0.01)
    for n=1,4 do
        output[n].slew=0
        output[n].volts=0
    end
end

dest = 1

input[1].window = function(state)
    dest = state

    for n=1,4 do
        if n == state then
        -- do nothing, maintain the current cv
        else
        -- reset to zero
            output[n].volts=0
        end
    end
end

input[2].stream = function(v)
    output[(dest%4)+1].volts = v
end
```

The same repo's `xfade.lua` softens the switch into a scan/width crossfade ("Inspired by the scan
section of the Verbos Scan & Pan. You might try it paired to a Quad VCA") — targets
`{0,1.666,3.333,5}`, a `width` input, `output[n].slew=0.01`. Conceptual framing from the thread:
"Single signal can encode up to four values — offset, frequency, amplitude and phase"
([poed, 25974/130](https://llllllll.co/t/crow-druid-scripts/25974/130)); Galapagoose recommends window
mode precisely for table-driven mode selection ([25974/160](https://llllllll.co/t/crow-druid-scripts/25974/160)).

### 4.5 dyn-mutated slew (Galapagoose) — two hard rules

Full pattern: a `to()` ASL whose slew time is a `dyn`, mutated live from an input stream
([Galapagoose, 25974/252](https://llllllll.co/t/crow-druid-scripts/25974/252), following the request
from [brick, 25974/246](https://llllllll.co/t/crow-druid-scripts/25974/246)):

```lua
-- this value is a "time" not a "rate", hence the name
slew_time = 1 -- start with an initial value

function rand_volts()
  for i=1,4 do
    --define an ASL and execute it immediately
    output[i](to(math.random(-5,5), dyn{slew=slew_time}))
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

(An earlier helper for the same idea, from [25974/250](https://llllllll.co/t/crow-druid-scripts/25974/250):
"a function that works like 'output.volts'" — `output[channel]( to(volts, dyn{slew=1} ))`, then
`output[1].dyn.slew = 0.1` to retune in motion. Also note there: `output[n].slew` only takes effect
when `output[n].volts` is next assigned.)

**Rule 1 — run the action once in `init()` *before* touching `output[i].dyn.x`,** or you get
`bad argument #N to 'casl_setdynamic' (number expected, got nil)`. "the error arises when trying to
set an `output[n].dyn._` element when it hasn't been initialized by an ASL constructor (like the one
inside `rand_volts`)" ([Galapagoose, 25974/256](https://llllllll.co/t/crow-druid-scripts/25974/256)).
The same error surfaces if the *value* you assign is itself `nil` — dani_derks' diagnosis of koerby's
`output[1].dyn.time = voltage` before `voltage` had ever been set: "when crow looks for the value of
`voltage` when your script is loaded, it doesn't find any `voltage` history and it presents that
error" ([dani_derks, 25974/238](https://llllllll.co/t/crow-druid-scripts/25974/238)).

**Rule 2 — `dyn` values are snapshotted when an ASL stage starts; mid-stage changes do not retarget
the running stage.** EvanEverfield's minimal repro (`to(5, dyn{time=10})` updated after 5s keeps its
original rate) was confirmed by the crowd: "The call to execute the action snapshots the dyn values
unless I'm mistaken. (EDIT: It may actually be that the start of each ASL segment snapshots the
value)" ([dessertplanet, 25974/281](https://llllllll.co/t/crow-druid-scripts/25974/281)); "I've run
into this before as well, I think this is exactly it. It's possible to get around it by breaking the
motion down into much smaller chunks by looping short `to()` calls"
([driedstr, 25974/282](https://llllllll.co/t/crow-druid-scripts/25974/282)); "the asl to() stage has
to complete first then it'll update I think … it'll take like 5 seconds for the oscillator to respond"
([postsolarpunk, 25974/283](https://llllllll.co/t/crow-druid-scripts/25974/283)). §4.6 is the
chunked-steps solution.

### 4.6 Mid-cycle-modulatable stepped LFO: `dyn{...}:step():wrap()` + `asl._if`/`asl._while` (EvanEverfield)

"an LFO that I want to respond to frequency changes in the middle of the cycle instead of having to
wait for it to end. I figured how to get this working by dividing the motion into a bunch of 'steps'
and using a `dyn` with `step` and `wrap` mutations"
([EvanEverfield, 25974/284](https://llllllll.co/t/crow-druid-scripts/25974/284), crediting
postsolarpunk's drumcrow research doc linked in
[25974/282](https://llllllll.co/t/crow-druid-scripts/25974/282)). Ramp oscillator with configurable
time and direction:

```lua
-- -5v to 5v
lo = -5.0
hi = 5.0
t=0.5
step_size = 0.01
steps = math.floor((hi - lo) / step_size) - 1

output[1](loop{
    to(lo + (dyn{step_counter = steps}:step(dyn{dir = -1}):wrap(0, steps) * step_size), (dyn{t = t} / steps))
})

-- t is the time in seconds of one cycle
output[1].dyn.t = 1

-- dir is -1 for falling, 1 for rising
output[1].dyn.dir = 1
```

Floating-point-precise variant of the destination ("stepping the destination value directly … does
seem to result in accumulating floating point error, which means that the start and end of a cycle
aren't fully precise and stable"):

```lua
output[1](loop{
    to(dyn{pos=hi}:step(dyn{dir = -1} * step_size):wrap(lo, hi), (dyn{t = t} / steps))
})
```

`times` does **not** retrigger inside a `loop` ("maybe because the underlying mutable counter isn't
being reset?"); rebuild it with `asl._while`:

```lua
output[1](loop{
    asl._while( dyn{counter = steps+1}:step(-1):wrap(0, steps+1), {
        to(dyn{pos=hi}:step(-1 * step_size):wrap(lo, hi-step_size), dyn{t=(t/steps)})
    }),
    -- do stuff after each cycle
})
```

And a pause gate via `asl._if(pred, t)` ("`pred` is a 'predicate' that is true if it evaluates to
>0, false otherwise (I think)") — note the **second `_if` is required**:

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

"The second `_if` seems necessary to prevent lockup from an infinitely `loop`ing sequence with zero
time passing each loop. It still might be burning a bunch of CPU time."

### 4.7 ASL as strings: a table of envelope shapes (xmacex, Eckelopes)

Store ASL **as text** and assign it to `output[n].action` when selected
([xmacex, 25974/232](https://llllllll.co/t/crow-druid-scripts/25974/232)):

```lua
-- Eckelopes
-- Inspired by René Kersting's
-- exhibition 162 Eckes at
-- the pool in Düsseldorf
--
-- input 1:  trigger
-- input 2:  select Ecke
-- output 1: eckelope
-- output 2: gate
--
-- by xmacex

voices = {
   {o=1,g=2},
   {o=3,g=4}
}

eckelopes = {
   "{ to(0, 0),   to(0, 0.2),   to(1 ,0.1),   to(0.7, 0.6), to(0, 0) }",
   "{ to(1.8, 0), to(3, 0.2),   to(3, 0.3),   to(0, 0) }",
   "{ to(2.3, 0), to(2.5, 0.1), to(2, 0.2),   to(0.4, 0),   to(0.4, 0.2), to(0, 0) }",
   "{ to(2, 0.2), to(3.1, 0.3), to(0, 0)}",
   "{ to(0, 0),   to(2, 0.3),   to(3.5, 0),   to(0, 0.2) }"
}

eckelection = 1

function init()
   input[1].mode('change', 1.0, 0.1, 'rising')
   input[1].change = trig

   input[2].mode('none')
   input[2].stream = selecke
end

function trig(volts)
   input[2].query()
   caw(voices[1], eckelopes[eckelection])
end

function selecke(volts)
   index = math.floor(volts/2)+1
   -- TODO: allow non-5 number of Eckelopes
   if index ~= eckelection and index >= 1 and index <= #eckelopes then
      eckelection = index
      print("changed to ecke "..eckelection)
      return math.floor(volts)
   end
end

function caw(voice, eckelope)
   output[voice.o].action = eckelope
   output[voice.g].volts = 2
   output[voice.o].done = function() output[voice.g].volts = 0 end
   output[voice.o]()
end
```

Two idioms hiding in there: `input[2].query()` — force a manual re-read of an input whose mode is
`'none'` — and **`output[n].done`** to end-of-ASL gating (gate up on trigger, gate down when the
shape finishes).

### 4.8 `output[n].done` gates (xmacex; EvanEverfield)

Besides Eckelopes above, the dual-AR envelope script uses `done` to fire end-of-fall trig outputs and
free the voice ([EvanEverfield, 25974/278](https://llllllll.co/t/crow-druid-scripts/25974/278)):

```lua
    output[1].done = function()
        parameters[1].active = false
        output[3]()
    end
    output[2].done = function()
        parameters[2].active = false
        output[4]()
    end
```

The same script shows the dyn-parameterized envelope shape — one ASL with `dyn { attack = 1 }` /
`dyn { release = 1 }` stages, then `output[num].dyn.attack = len * ratio` from the knob handler:

```lua
        output[1].action = {
            to(0, 0),
            to(8, dyn { attack = 1 }, 'linear'),
            to(0, dyn { release = 1 }, 'linear')
        }
```

(Full ~400-line script with txi polling, mode switching and response curves is in the post; it also
demonstrates `input[2]` held at steady 1V as a secondary-mode switch.)

### 4.9 Getter functions as ASL arguments: `ar(get_a, get_d)` (bloc)

ASL stage args accept **functions evaluated at runtime** in these positions (built-in action args;
likewise bare stage elements, §5.7) — note function support is position- and version-dependent:
the compiler's `Do you have a function in your ASL?` error fires where numeric expressions are
expected (`api-asl.md`). Pass `get_a`/`get_d` (not calls) to keep
attack/release live while the envelope runs
([bloc, 25937/89](https://llllllll.co/t/crow-strategies-just-friends/25937/89)):

```lua
function set_d(t) decay = (t-1)/8 + 0.05 end
function set_a(t) attack = (t-1)/64 + 0.003 end
…
function get_d() return decay end
function get_a() return attack end
…
  output[2].action = ar(get_a,get_d)
  …
  output[4].action = ar(get_a,get_d)
```

Galapagoose states the boundary: "The functions as arguments feature is specific to the `ASL`
scripts" — outside ASL, just write a generator function and call it
([25974/135](https://llllllll.co/t/crow-druid-scripts/25974/135)):

```lua
function r()
  return math.random() * 10.0 - 5.0
end
```

```lua
input[1].change = function()
  output[2].volts = r() -- not the parenthesis to call the function
end
```

---

## 5. ii / Just Friends techniques

Command semantics for `ii.jf.*` live in `ii-just-friends.md`; this section is community practice on
top of them. (0V = middle C = 261.63 Hz, "C3"; level in volts.)

### 5.1 Simultaneous notes drop: the 10 ms spacing workaround (dani_derks; Rodrigo)

"Just Friends is a 6 voice polyphonic synth, but in `transient/sound` it sometimes loses track of
pitch data when it receives simultaneous messages over i2c … I found that if I spaced out the message
delivery between crow and Just Friends by ~10 milliseconds, I got perceivable polyphony with no
dropped notes or weird pitches" ([dani_derks, 25937/1](https://llllllll.co/t/crow-strategies-just-friends/25937/1)).
The norns version (from `less_concepts`) — a one-shot 10 ms metro that delivers the second voice:

```lua
jf_note_spacer = metro.init()
jf_note_spacer.time = 0.01
jf_note_spacer.count = 1
```

```lua
if i == 1 then
     crow.ii.jf.play_note((BLAH-BLAH-NOTE-MATHS)/12,5)
elseif i == 2 then
     jf_note_spacer:start()
end
```

"if the note is for the first voice in the pair, that can be played straightaway; if the note is for
the second voice in the pair, start that 10ms timer; at the end of the 10ms timer, crow will send JF
the data for that second voice. but it all sounds like simultaneous events!!"

Rodrigo's smarter scheduler (Max-side, generalizable):
([Rodrigo, 25937/15](https://llllllll.co/t/crow-strategies-just-friends/25937/15)) — "The patch above
will send any message received, instantly. IF it receives a message within 10ms of the last message,
it will then add it to `zl queue` and send it after 10ms. This gives you instant response time and
note staggering when you're dealing with polyphony." (He also warns the naive fixed-`metro 10` queue
can "jam up the scheduler" and adds up to 10 ms latency to every message.)

Galapagoose reports this class of bug **fixed in later crow firmware** ("This will be fixed in the
next Crow firmware release. I've got proper polyphony working already"
[25937/39](https://llllllll.co/t/crow-strategies-just-friends/25937/39)) — treat 10 ms spacing as the
legacy workaround, and verify on your firmware. See `troubleshooting.md`.

### 5.2 Synthesis vs Geode argument semantics — and the ÷1638.3 wrinkle

**Status: resolved against crow firmware source** (see `ii-just-friends.md` §5 for the full
treatment). Two layers of confusion exist in the community record:

1. **Wiki semantics.** The same ii command changes meaning with the SPEED switch (see
   `ii-just-friends.md`): Synthesis `play_note(pitch, level)` with pitch in volts (fifth = `7/12`),
   Geode `play_note(divs, repeats)` with `divs=4` = quarter notes. voidstar's summary in-thread writes
   the Geode order as `play_note(repeats,division)`
   ([25937/20](https://llllllll.co/t/crow-strategies-just-friends/25937/20)) — the wiki/crow docs say
   `(divs, repeats)`. The two accounts disagree on argument order; check `ii.jf.help()` output (§5.8)
   on your unit.

2. **The scaling wrinkle.** The Max Geode example divides its numbers by 1638 before sending; nattog
   asked why ([25937/29](https://llllllll.co/t/crow-strategies-just-friends/25937/29)), and voidstar
   explained the Just Type ii protocol was designed for Teletype's integer-only world
   ([25937/32](https://llllllll.co/t/crow-strategies-just-friends/25937/32)):

   > Make an extremely large number equal to 10vpp. Let's call this our "super precise large number
   > scaling system." … The velocity argument of `JF.NOTE` expects to be given an integer from our
   > special scale …

   > Crucially, the same command is used to talk to both modes; the range switch on JF is what
   > determines the difference in meaning, not the sending device.

   > Fast forward to crow, which can handle floating point numbers after all. The `play_note`
   > functions was written with the pitch priority in mind however JF is still expecting to receive an
   > integer in our "super large integer scaling system" …

   > So on crow, `play_note` expects a floating point v/oct voltage; it then converts this to our
   > "super large integer scale" and sends it to jf. This poses an issue when we try to use geode - if
   > you type in "1" to play_note, it converts that into a super large integer before sending it to
   > JF. As such, when we actually want to send a 1 to JF, eg in geode mode, we can't give
   > `play_note` a 1 as an argument - we need to give it a much smaller decimal number which will
   > then be immediately made bigger again by play_note, landing on the integer 1.

   **Resolution (firmware source).** crow's `lua/ii/jf.lua` types `play_note`/`play_voice` args as
   `s16V` — always multiplied by **1638.3** (`II_TT_VOLT`, `lib/ii.c`) before sending — and JF's
   Geode engine reads the raw wire integers, so **divide Geode `divs`/`repeats` by 1638.3** exactly
   as voidstar describes: `ii.jf.play_note(4/1638.3, -1/1638.3)`. The firmware table also confirms
   the **argument order is `(divs, repeats)`** (`pitch_or_divs` first) — the `(repeats, division)`
   label in [25937/20](https://llllllll.co/t/crow-strategies-just-friends/25937/20) is swapped. One
   caveat: `ii.jf.retune` args are **`s8` raw integers (0–127), not scaled** —
   `ii.jf.retune(2, 2/1638, 1/1638)` in [25937/43](https://llllllll.co/t/crow-strategies-just-friends/25937/43)
   truncates to `retune(2, 0, 0)` and does nothing ("no response", [25937/44](...)). Never divide
   `s8` args. A worked Geode session (Max patch `geode_ex.maxpat`):
   "set channel [1], division [4], repeats [3] … you should see 4 pulses from channel 1 (repeats
   assigns how many pulses after initial pulse)"
   ([dani_derks, 25937/27](https://llllllll.co/t/crow-strategies-just-friends/25937/27)).

Synthesis pitch/level are plain volts though — no lookup tables like Teletype's `N`/`V`
([voidstar, 25937/37](https://llllllll.co/t/crow-strategies-just-friends/25937/37)):

```lua
ii.jf.play_note(1,5)          -- one octave above C3 at 5Vpp
ii.jf.play_note(2,3.4)        -- two octaves above at 3.4Vpp
ii.jf.play_note(7/12,5)       -- perfect fifth above C3
ii.jf.play_note(4/12 - 37/1200,5)  -- major third minus 37 cents (1 cent = 1/1200 V)
```

### 5.3 Just intonation on JF: `play_note(justvolts(ratio), level)` — not `retune` (Galapagoose)

Asked whether `retune` could give ratio tuning per voice in Synthesis
([ObscBySds, 25937/116](https://llllllll.co/t/crow-strategies-just-friends/25937/116)):

> `retune` won't get you what you want, but you can use the regular `ii.jf.play_note` (or
> `play_voice`), and generate the pitch argument with the `justvolts` function. It'd look something
> like: ([Galapagoose, 25937/117](https://llllllll.co/t/crow-strategies-just-friends/25937/117))

```lua
ii.jf.play_note( justvolts(3/2), 3)
```

```lua
jf_play_ratio( 3, 2 )
```

("you could wrap this in a helper function if you want it feel different".) The La Monte Young script
in §2.5 is the full-scale example of this approach.

### 5.4 Interval-set selector: `retune` all six voices from voltage windows (foldedworlds)

Seven hand-tuned ratio sets, selected by input voltage via `'window'` mode, applied to all six
channels ([foldedworlds, 25937/154](https://llllllll.co/t/crow-strategies-just-friends/25937/154),
posted complete):

```lua
--- jf retune
-- in1: selector
-- in2: none
-- out1-4: none

-- choose interval numerators and denominators 
num = { {1,1,3,9,9,16} -- -5V
       , {1,1,9,4,3,9}
       , {1,2,18,8,6,18}
       , {1,2,18,12,8,6} -- 0V
       , {1,2,18,8,6,54}
       , {1,18,4,36,24,48}
       , {1,1,6,18,18,32} -- +5V
       }

den = { {1,2,4,8,5,9} -- -5V
       , {1,1,8,3,2,5}
       , {1,1,8,3,2,5}
       , {1,1,8,5,3,2} -- 0V
       , {1,1,8,3,2,16}
       , {1,5,1,8,5,9}
       , {1,1,4,8,5,9} -- +5V
       }


function init()
input[1].mode('window',{-5,-4,-3,-2,-1,1,2,3,4,5},0.1)
input[2].mode('none')
end

input[1].window=function(s)
    set = math.floor(input[1].volts/2 + 4.5)
    if set > 7 then set = 7 elseif set < 1 then set = 1 end
    if s then
        for n=1,6 do
             ii.jf.retune (n, num[set][n], den[set][n]) 
        end
    end
end
```

Galapagoose's recipe this implements: "using the window to choose a list of ratios from a table, then
iterating over each ratio set and calling `.retune` for each one. The bulk of the work will be the
fun musical part – choosing which ratios to use!"
([25937/148](https://llllllll.co/t/crow-strategies-just-friends/25937/148)). Field note from
foldedworlds: boundary fuzz near +5V ("Sounds like the voices are unsure whether it should be at set
6 or set 7") solved by feeding a wider-range attenuator so any voltage >5V forces the top set; use
case was PLUME mode with IDENTITY's LPG forced open into a tuner. Numerator/denominator max is
**127** (see §5.5).

### 5.5 Live `retune` from a JF output (xmacex)

Patch any JF output back into crow and retune a channel from its own voltage
([xmacex, 25937/163](https://llllllll.co/t/crow-strategies-just-friends/25937/163)):

```lua
function retune(v)
  crow.ii.jf.retune(6, math.floor(v), 1)
end
crow.input[1].stream = retune
crow.input[1].mode = 'stream'
```

Limits and history: "A numerator or denominator of 127 seems to be the limit!"
([foldedworlds, 25937/167](https://llllllll.co/t/crow-strategies-just-friends/25937/167)). And note
that `ii.jf.retune` was **silently broken on crow for a while** — `retune(1, x, 1)` shut the channel
off instead of tuning ([dani_derks, 25937/56](https://llllllll.co/t/crow-strategies-just-friends/25937/56))
— "the i2c descriptor is wrong for Just Friends on crow. I corrected it on my end now, so this
command will be fixed in the next crow update"
([Galapagoose, 25937/57](https://llllllll.co/t/crow-strategies-just-friends/25937/57)). Working
escapes at the time: `ii.jf.retune(0,0,0)` restores defaults. Check firmware before trusting
`retune` (see `troubleshooting.md`).

### 5.6 `mode(1)` lifecycle gotcha

JF stays in Synthesis/Geode until told otherwise, and a stale `mode(1)` makes the module look
"stuck" even with no crow attached. Recovery and cleanup recipes are catalogued in
**`troubleshooting.md`** — in short: `ii.jf.mode(0)` restores normal operation
([alanza, 25937/63](https://llllllll.co/t/crow-strategies-just-friends/25937/63);
[hghrlearning, 25937/108](https://llllllll.co/t/crow-strategies-just-friends/25937/108)); see that
file before shipping any script that calls `ii.jf.mode(1)`.

### 5.7 ASL-driven ii polling: `bonk()` (desolationjones)

"The getter function is happening as part of an ASL loop with a triangle LFO" — poll a faderbank and
play its value on JF
([desolationjones, 25937/88](https://llllllll.co/t/crow-strategies-just-friends/25937/88)):

```lua
function init()
	output[2](bonk())
	ii.jf.mode(1)
end

ii.faders.event = function (e, data, id)
	ii.jf.play_note(data, 10)
	--crow.tell('fb', e, data, id)
end

function bonk()
	return loop{ function() ii.faders[1].get(1) end
				, to(5, 1)
				, to(0, 1)
				}
end
```

Note the bare `function() ii.faders[1].get(1) end` as an ASL stage — a zero-time side-effect step.

### 5.8 JF knob polling via getters (ObscBySds; maaark; csboling)

Getters never return values; the reply arrives later in `ii.jf.event`, it **lags**, and polling too
fast kills JF. ObscBySds's TIME/INTONE experiment
([25937/119](https://llllllll.co/t/crow-strategies-just-friends/25937/119)):

```lua
function init()
    getters()
    ii.jf.mode(1)
    input[1].mode('change',1.0,0.1,'rising')
end

input[1].change = function(state)
  ii.jf.get('time')
  ii.jf.play_voice(1, 0, 5)
  print(time)
end

function getters()
    ii.jf.event = function(e, value)
        if e.name == 'time' then
            time = value
        end
    end
end
```

Observed behaviour: "the value (time) is not updated right away when I change the knob position. It
needs two triggers to do so" (reads lag **two** triggers), and crashes appear "related to the speed
at which I trig crow … around 5Hz max I think" — **don't poll faster than ~5 Hz**. maaark's voice-free
inspection attempt runs into the companion rule — `crow.ii.jf.get(...)` **returns nothing**; "It
triggers an asynchronous event that is received in the `crow.ii.jf.event()` handler"
([Galapagoose, 25937/103](https://llllllll.co/t/crow-strategies-just-friends/25937/103)); and
`get('trigger')` "might mean it's always 1 when in synthesis mode … when the oscillator goes silent it
is still oscillating (just being muted)"
([25937/105](https://llllllll.co/t/crow-strategies-just-friends/25937/105)).

The command list you get from `ii.jf.help()` in druid (`crow.ii.jf.help()` on norns)
([csboling, 25937/86](https://llllllll.co/t/crow-strategies-just-friends/25937/86)):

```lua
ii.jf.trigger( channel, state )
ii.jf.run_mode( mode )
ii.jf.run( volts )
ii.jf.transpose( pitch )
ii.jf.vtrigger( channel, level )
ii.jf.mode( mode )
ii.jf.tick( clock-or-bpm )
ii.jf.play_voice( channel, pitch/divs, level/repeats )
ii.jf.play_note( pitch/divs, level/repeats )
ii.jf.god_mode( state )
ii.jf.retune( channel, numerator, denominator )
ii.jf.quantize( divisions )
```

### 5.9 Custom round-robin allocation with `play_voice` (Galapagoose)

If you want control over which voice plays (e.g. "notes never change pitch while they are playing"),
bypass the built-in allocator and roll your own: "i'd use `jf.play_voice` and build a simple
round-robin allocator – it's just a wrapping counter from 1 to 6"
([Galapagoose, 25937/105](https://llllllll.co/t/crow-strategies-just-friends/25937/105)). Real-world
explicit-voice calls look like
([vagueintl, 25937/96](https://llllllll.co/t/crow-strategies-just-friends/25937/96)):

```lua
for n=1,2 do
  scale_ix = (scale_ix + 3) % #penta +1
  local note = penta[scale_ix]/12
  ii.jf.play_voice( n, note, level )
end
```

(the wrapping counter `n` here is the round-robin seed; a dedicated `voice = voice % 6 + 1` is the
general form). Note `play_voice(…, 0)` targets **all** channels (see `ii-just-friends.md`).

### 5.10 Rhythm gates to `ii.jf.trigger` from binary decomposition (ringsandrocks, binarrayJF)

Turn a sampled voltage into a 5-bit rhythm played directly on JF's trigger inputs — JF becomes a
Drezno-style rhythm decoder ([ringsandrocks, 25974/219](https://llllllll.co/t/crow-druid-scripts/25974/219);
full code in §3.6):

```lua
  for i=1,5 do
    ii.jf.trigger(i+1,binArray[vToI][i])
  end
  if inVal > 0 then ii.jf.trigger(1,1)
  else ii.jf.trigger(1,0)
  end
```

"JF Channel 1 shows 0 for negative and 1 for positive. JF Channels 2-5 show the value in 5 digits of
binary rather than 4." The demo video
([youtube](https://www.youtube.com/watch?v=NeAauwc11ME)) patches "the 4 channels of Just Friends
… triggering 4 channels of Landscape's Noon. I am only adjusting knobs and switches on Just Friends."

### 5.11 Run-mode gestures: `run_mode(1); run(5); transpose(...)` (ObscBySds)

Performance gestures over ii, straight from the Well-Tuned script's `init` (§2.5):

```lua
    ii.jf.mode(1)
    ii.jf.run_mode(1)
    ii.jf.run(5)
```

and per-event whole-module transposition from the scale handler:

```lua
  ii.jf.transpose(3/12 - 1)
```

`run()` is a virtual voltage added to the physical RUN state and needs `run_mode(1)` first
(`ii-just-friends.md`); `transpose` takes volts where `1.0` = octave, so `3/12 - 1` = "a third up
from one octave down".

### 5.12 Random velocity for expression (ObscBySds)

Humanize every note by randomizing level in the voltage domain
([ObscBySds, 25974/185](https://llllllll.co/t/crow-druid-scripts/25974/185)):

```lua
  ii.jf.play_note(note, (math.random() * 5.0 + 0.5))
```

(`math.random()*5 + 0.5` spans roughly 0.5–5.5V. `vtrigger(ch, level)` remembers the last velocity
per channel — useful for "accents on every Nth note"; see `ii-just-friends.md`.)

### 5.13 Binary-mask rhythm gates + JF/First hybrid (buma)

One clocked `play(out,ix)` drives four rhythm streams whose steps are integers tested with bit masks:
`rhythm[ix][step] & 8 == 8`
([bloc, 25937/89](https://llllllll.co/t/crow-strategies-just-friends/25937/89) — "JF with transpose
and first combined"; note the post also defines its own legacy `delay(time, action)` wrapper, see
§4.3):

```lua
function play(out,ix)
  if rhythm[ix][ step[ix] ] & 8 == 8 then
    if ix == 1 then set_d(t[ix]) else set_a(t[ix]) end
    t[ix] = 0
    output[out+1]()
  end
  if rhythm[ix+2][ step[ix+2] ] & 8 == 8 then
    n1 = notes[ix][ step[ix+2] ]
    n2 = notes[ix][ step[ix] ]
    abs = math.abs(input[2].volts)/5
    note = n1 + abs*(n2-n1)
    note = math.floor(note * (abs*3 + 0.1))
    s = scale[input[2].volts > -0.04 and 1 or 2]
    nn = s[ note%(#s) + 1 ]
    oct = math.floor(note/12)
    output[out].volts = nn/12 + oct
  end
  step[ix] = (step[ix] % length[ix]) + 1
  step[ix+2] = (step[ix+2] % length[ix+2]) + 1
end
```

Supporting cast worth reading in full on the post: an `lcg(seed)` PRNG seeded from `unique_id()`
for the rhythm/note tables, `get_a`/`get_d` feeding `ar(get_a,get_d)` (§4.9), and a `tmetro` that
re-transposes JF from input 2 as it moves (`trans(count)` calling `ii.jf.transpose(transpose - 2)`),
with `getters()` polling `speed`/`tsc` to decide between `ii.jf.mode(1)` synth play and `mode(0)`
First-style behaviour. "Note the outputs acts as the First script so when you aren’t playing Just
friends synth mode you can use the First script."

### 5.14 Faderbank → JF (koerby; vinc; dani_derks)

The recurring `nil` trap: `Fader1 = ii.faders.get(1)` returns **nothing** — "the get function does
not return a value" ([vinc, 25974/240](https://llllllll.co/t/crow-druid-scripts/25974/240)). Route
the reply through the event handler instead:

```lua
ii.faders.event = function(e, value)
    ii.jf.vtrigger(e, value)
  end
end
```

(verbatim as posted — the stray `end` is the author's; combine with a `metro` or clock polling
`ii.faders.get(n)` as in §5.7. `ii.jf.vtrigger` *triggers* the channel; use `ii.jf.pitch`/level
writes if you only want to set levels — see `ii-just-friends.md`.) dani_derks' general nil lesson
from the same debugging: everything outside functions runs **before** `init()`, so "if along your
scripting journey you continue to run into errors due to `nil` values, i'd recommend checking to make
sure that you aren't executing functions which require the code inside of your `init` to have already
executed" ([25974/241](https://llllllll.co/t/crow-druid-scripts/25974/241)) — the full parse-order
print test is in §6.6.

### 5.15 Teletype `CROW.Q4` as a dispatch table via `ii.self.call4` (dani_derks)

Only four crow callbacks exist for Teletype, but the 4-argument one can carry a **function selector**
([dani_derks, 25974/238](https://llllllll.co/t/crow-druid-scripts/25974/238)):

```lua
crow_functions = {
  function(x,y,z)
    print("function 1!", x, y, z)
  end,
  function(x,y,z)
	  print("function 2!", x, y, z)
  end,
  function(x,y,z)
	  print("function 3!", x, y, z)
  end,
  function(x,y,z)
	  print("function 4!", x, y, z)
  end,
  function(x,y,z)
	  print("function 5!", x, y, z)
  end
  -- etc etc
}

function ii.self.call4(thing_1,thing_2,thing_3,func_id)
  crow_functions[func_id](thing_1,thing_2,thing_3)
end
```

Then on Teletype: `> CROW.Q4 7 4 9 1` — "calls function 1 with 7, 4 and 9 as arguments and we get
this printed to druid: `function 1!  7.0  4.0  9.0`". "you can use whichever argument you want to
index the functions, just make sure you're using the same spot for both TT and crow."

### 5.16 Miscellaneous JF field notes

- **Microtonality needs no MIDI tricks on the ii path** — pitch is a plain volts argument
  ([voidstar, 25937/37](https://llllllll.co/t/crow-strategies-just-friends/25937/37), examples in
  §5.2).
- **Knobs are read-only over ii**: "You can get the values in code, but can't set those ones" — TIME,
  INTONE, CURVE etc. can be polled but not written; patch crow outputs to JF CV inputs to modulate
  them ([PiecesOfQuiet, 25937/132](https://llllllll.co/t/crow-strategies-just-friends/25937/132);
  [wminor, 25937/131](https://llllllll.co/t/crow-strategies-just-friends/25937/131)).
- **Panels as script source material**: bloc reads `get('speed')`/`get('tsc')` to detect whether the
  user has JF in sound or shape mode and switches script behaviour accordingly (§5.13).
- **i2c leaders**: two crows + JF + Teletype on one bus works "as long as you aren't sending i2c stuff
  at the same time" ([Justmat, 25974/261](https://llllllll.co/t/crow-druid-scripts/25974/261));
  second JF is `ii.jf[2]` after `ii.jf.address(2)`
  ([Justmat, 25974/263](https://llllllll.co/t/crow-druid-scripts/25974/263)) — never cache the handle
  (`ii-overview.md`).

---

## 6. Host interaction & live-coding workflow

### 6.1 Public variables from the druid prompt (dani_derks)

Publics were built so "norns function as a remote user interface", but they also sync to any USB host
and are settable live ([dani_derks, 25974/175](https://llllllll.co/t/crow-druid-scripts/25974/175);
pointer from [mouseflies, 25974/173](https://llllllll.co/t/crow-druid-scripts/25974/173)):

```
> public.scale1 = "dorian"
-- which will confirm the change by returning:
^^pupdate("scale1", "dorian")
```

For druid-only workflows, plain globals changed at the prompt are often simpler ("for druid, you can
simply use variables to control aspects of your script via the command line").

### 6.2 Typed declarations and host visualization (xmacex)

`public{...}:range(...)` / `:type(...)` / `:options{...}` declare typed, ranged parameters the host
renders as sliders/selects
([xmacex, 25974/279](https://llllllll.co/t/crow-druid-scripts/25974/279) — window-crowparator):

```lua
public{truev  = 5}:range(MINV, MAXV)
public{falsev = 0}:range(MINV, MAXV)

public{window_center = 0}:range(MINV, MAXV):type('slider')
public{window_width  = 1}:range(0.1, MAXV):type('slider')
public{input_voltage = 0}:range(MINV, MAXV):type('slider')
public{comp = 'inside'}:options{'above', 'inside', 'below'}
```

and population-crowth pushes values to the norns screen with `public.view.output[i]()`:

```lua
    public{r      = 3.2}:range(MINR, MAXR):type('@')
    public{values = {0.1, 0.1, 0.1, 0.1}}:type('@')
    public{case   = 'osc 2'}:options{'osc 2', 'osc 4', 'osc 8', 'osc n', 'kaos'}:type('@')

    input[1]{mode = 'change', direction='rising'}
    input[1].change = function()
        local t = l(public.values[1])

        table.insert(public.values, 1, t)
        if #public.values > 4 then
            table.remove(public.values, 5)
        end

        for i=1,4 do
            output[i].volts = public.values[i] * SCALE
            public.view.output[i]() -- visualize
        end
    end
```

(If you expose params without publics, the `public.add` idiom is the alternative — e.g.
`public.add('trans 1', 0, {-2, 2}, function(v) t1 = v end)` in Oootini's seq-m8
[25974/143](https://llllllll.co/t/crow-druid-scripts/25974/143).)

### 6.3 Pre-planned command functions for live druid control (dani_derks)

"it gets a little more complicated to execute a bunch of stuff on a single line" — so write
argument-taking functions into the script and drive *those* from the prompt. Clock
cancel/respawn pattern ([dani_derks, 25974/179](https://llllllll.co/t/crow-druid-scripts/25974/179)):

```lua
function init()
  out_clock = {} -- we want to store clocks to cancel + respawn them
end

function stream_pulses(out,beat_div)
   -- if there's a clock at this output already, cancel it:
  if out_clock[out] ~= nil then
    clock.cancel(out_clock[out])
  end

  -- spawn a new clock:
  out_clock[out] = clock.run(
    function()
      while true do
        clock.sync(beat_div)
        output[out](pulse())
      end
    end
  )

end
```

"with that scaffolding in a script, the command line execution to get a pulse for every beat on out3
is simply:" `stream_pulses(3,1)`. Same post: one-line static voltages ("if you want to just send a
static voltage equal to 3 semitones to out3"):

```lua
output[3].volts = 3/12
```

### 6.4 Max and norns as remotes

- **Max**: send Lua as a quoted symbol to `tell_crow` — "The quotes are crucial here because they make
  it so that the Lua code is treated a single chunk. Typically, the I use the `sprintf` object with
  the `symout` argument to format my Lua chunks and turn them into a single symbol, followed by a
  `prepend` object to tack on the `tell_crow`" ([voidstar, 25937/20](https://llllllll.co/t/crow-strategies-just-friends/25937/20)):

  ```
  tell_crow "output[1].volts = 3"
  ```

- **norns REPL (maiden)**: `crow.send("your_crow_command_in_quotes")` — "If you type something that
  causes crow to spit out standard out, it will show up in the repl output too"
  ([jlmitch5, 25937/71](https://llllllll.co/t/crow-strategies-just-friends/25937/71)); or skip the
  wrapper entirely for ii commands — "you can just use `crow.ii.jf.mode(0)`, no need for the
  `crow.send` workaround" ([Galapagoose, 25937/75](https://llllllll.co/t/crow-strategies-just-friends/25937/75)).

- The `^^jf_synth` / crow Max-for-Live toolkit lives at
  [monome/crow-max-and-m4l](https://github.com/monome/crow-max-and-m4l)
  ([dani_derks, 25937/1](https://llllllll.co/t/crow-strategies-just-friends/25937/1)).

### 6.5 Druid workflow tips (Galapagoose; csboling; damon; Justmat)

- **`r your_script.lua` while developing, `u` once at the end**: "we suggest using
  `r your_script_name.lua` as the 'run' command is faster than 'upload', and doesn't write to the
  flash (hence reduces wear on the memory). then just run `u` (no name needed) at the end of a
  session to freeze it to the module"
  ([Galapagoose, 25937/97](https://llllllll.co/t/crow-strategies-just-friends/25937/97)). Scripts do
  **not** survive power-cycling until `u`'d.
- **Launch druid from the scripts folder**: "Be sure to launch `druid` from the same folder where the
  script files are" ([csboling, 25974/124](https://llllllll.co/t/crow-druid-scripts/25974/124)); the
  working directory is wherever you ran the command
  ([csboling, 25974/126](https://llllllll.co/t/crow-druid-scripts/25974/126)):

  ```
  $ cd /path/to/my/bowery
  $ druid
  r boids.lua
  ```

- **Recovery**: `^^c` clear and `^^r` reset — "sometimes a reset of Crow (`^^r`) before uploading the
  script can help" ([damon, 25974/32](https://llllllll.co/t/crow-druid-scripts/25974/32)); a
  successful load after trouble may require `^^c` first
  ([Oootini, 25974/36](https://llllllll.co/t/crow-druid-scripts/25974/36)). `^^v` prints firmware
  version, `druid --version` the host tool
  ([Galapagoose, 25974/35](https://llllllll.co/t/crow-druid-scripts/25974/35)).
- **Built-in First**: `^^F` / `^^First` reloads the firmware's `First.lua`
  ([csboling, 25974/60](https://llllllll.co/t/crow-druid-scripts/25974/60)); it lives in the crow
  source, not the Bowery
  ([csboling, 25974/58](https://llllllll.co/t/crow-druid-scripts/25974/58)).
- **`p` prints the currently uploaded script** into the druid session
  ([Justmat, 25937/128](https://llllllll.co/t/crow-strategies-just-friends/25937/128)).
- **Script loading quirks** (line endings / trailing newline) are discussed in thread A
  ([damon, 25974/32](https://llllllll.co/t/crow-druid-scripts/25974/32);
  [csboling, 25974/34](https://llllllll.co/t/crow-druid-scripts/25974/34)) and settled by firmware
  version — see `troubleshooting.md` before blaming your editor. Upload size limit is ~16 kB of
  script ([juhbuba, 25974/182](https://llllllll.co/t/crow-druid-scripts/25974/182)).
- **Script selector CLI**: driedstr's `crow-script-nest.sh` shows the running script (`druid
  download`) and picks a new one via `fzf` + `druid upload`
  ([driedstr, 25974/285](https://llllllll.co/t/crow-druid-scripts/25974/285)).

### 6.6 Know the parse order: top → bottom, then `init()` (dani_derks; alanza)

The canonical demonstration to keep in your head
([dani_derks, 25974/241](https://llllllll.co/t/crow-druid-scripts/25974/241)):

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

which prints:

```
hello from the top of the script
hello from below the init
Running: _test flow_
hello from init
hello from another function!
```

"all parts of a script are parsed upon load, which means standalone variables and actions … are
executed as well, and only after everything's loaded in does the `init` run." alanza's lower-level
explanation: `dofile` evaluates the whole file top-to-bottom (Lua can't look ahead), then the C layer
calls `init()`, then the event loop runs handlers
([alanza, 25974/244](https://llllllll.co/t/crow-druid-scripts/25974/244)) — including the gotcha `a =
b; b = 5; print(a) -- nil`.

---

## 7. Script & author index

Named scripts from the threads and their repos — all worth reading or cloning.

| Script | Author | Source | What it is |
|---|---|---|---|
| modes.lua | voidstar (sam wolk) | [25974/1](https://llllllll.co/t/crow-druid-scripts/25974/1) | Multi-mode quad utility "Pam's New Workout": S&H, slewed/quantized/random S&H, clock-synced LFOs, sequencer, pulses; per-output clock division & level; tempo detection via averaged `time()` deltas |
| Ornithopters | voidstar | [25974/14](https://llllllll.co/t/crow-druid-scripts/25974/14) | Five Turing-Machine-style feedback shift registers (digital, weighted-bit-sum, quantized, analog, velocity-envelope) with reprogrammable `actions.*` |
| livecodesequencer / **scrolls** | crim (crimclark) | [25974/21](https://llllllll.co/t/crow-druid-scripts/25974/21), [25974/25](https://llllllll.co/t/crow-druid-scripts/25974/25), [repo](https://github.com/crimclark/scrolls) | Live-coded dual-melody sequencer; druid commands (`oct`, `mod`, `rv`, `mv`, `ptrn`, …) queued as functions applied at loop starts, snapshot-relative edits |
| histo-qt | csboling | [25974/26](https://llllllll.co/t/crow-druid-scripts/25974/26), [source](https://github.com/csboling/crow-scripts/blob/aca2c850bc23fdc5b7810987811c290098405420/sequencing/histo-qt.lua) | "Leaky histogram quantizer": recent-input voltage histogram with decaying bins, sampled on trigger; `set_scale`, `set_halflife`, `monitor` |
| dual_clock_quad_lfo.lua | damon | [25974/24](https://llllllll.co/t/crow-druid-scripts/25974/24) | Dual-clock quad LFO: 2 clocks or 1 clock + depth CV; wave/skew/unipolar/rate-ratio per output |
| curvedrandom.lua | damon | [25974/41](https://llllllll.co/t/crow-druid-scripts/25974/41) | Weighted random voltage with curveLevel / bellFactor / direction shaping |
| krowll.lua | damon | [25974/45](https://llllllll.co/t/crow-druid-scripts/25974/45) | Krell patch: envelope + pitch + two CVs, self-clocked |
| vults.lua | damon | [25974/42](https://llllllll.co/t/crow-druid-scripts/25974/42) | CV range scaler / buffered mult utility |
| jfInt.lua | gigasturtz | [25974/46](https://llllllll.co/t/crow-druid-scripts/25974/46) | Integer-as-rhythm (`bit32` bit-walking) note player for JF |
| swingclock.lua | jmsole (capleniz) | [25974/85](https://llllllll.co/t/crow-druid-scripts/25974/85), [25974/87](https://llllllll.co/t/crow-druid-scripts/25974/87), [gist](https://gist.github.com/jmsole/6572916b58f79bb75392cd589b5e25da) | CV-controlled clock with swing + clock division (nested-delay approach) |
| seq.lua (OOP) | tejomay | [25974/88](https://llllllll.co/t/crow-druid-scripts/25974/88) | Sequencer class + `bseq` binary sequences (`bseq:new(144)`), JF or CV triggers, `mlb` mode |
| Probabilities.lua | ringsandrocks | [25974/141](https://llllllll.co/t/crow-druid-scripts/25974/141) | Per-output trigger probabilities with CV scaling of odds |
| binarray.lua / binarrayJF.lua | ringsandrocks | [25974/219](https://llllllll.co/t/crow-druid-scripts/25974/219), [demo](https://www.youtube.com/watch?v=NeAauwc11ME) | Drezno-style voltage→binary decomposition to outputs/JF triggers, plus clock divisions |
| drummer.lua | nattog | [25974/131](https://llllllll.co/t/crow-druid-scripts/25974/131), [gist](https://gist.github.com/nattog/841bcfb813aefdc751cacf0d4b79f4ab) | 4-track `'x---x---'` pattern sequencer with bar-quantized pattern-swap queue |
| vcswitch.lua / xfade.lua | yaw (echophon) | [25974/136](https://llllllll.co/t/crow-druid-scripts/25974/136), [repo](https://github.com/echophon/crowscripts) | Voltage-controlled switch / scan-and-width crossfade using `'window'` mode |
| Eckelopes | xmacex | [25974/232](https://llllllll.co/t/crow-druid-scripts/25974/232) | ASL-as-string shape selector with `output.done` gating |
| leibniz.lua | xmacex | [25974/279](https://llllllll.co/t/crow-druid-scripts/25974/279) | Xaoc Leibniz-style bitwise AND/OR on digitized inputs |
| window-crowparator | xmacex | [25974/279](https://llllllll.co/t/crow-druid-scripts/25974/279), [repo](https://github.com/xmacex/window-crowparator) | Window comparator (above/inside/below) with typed `public` UI, runs standalone or with norns |
| population-crowth | xmacex | [25974/279](https://llllllll.co/t/crow-druid-scripts/25974/279), [repo](https://github.com/xmacex/population-crowth) | Logistic map chaos source, `public.view.output[i]()` visualization |
| dual-ar-envelopes.lua | EvanEverfield | [25974/278](https://llllllll.co/t/crow-druid-scripts/25974/278) | txi-controlled dual AR envelopes with `done` end-of-fall trigs, response curves, secondary mode |
| stepped LFO / pause gate | EvanEverfield | [25974/284](https://llllllll.co/t/crow-druid-scripts/25974/284) | Mid-cycle-modulatable stepped oscillator via `dyn:step():wrap()`, `asl._if`/`asl._while` |
| wdelrandjump.lua | coreyr | [25974/292](https://llllllll.co/t/crow-druid-scripts/25974/292) | W/ delay buffer position jumper: random jumps via the script's own `positionslices` helper (slices of `ii.wdel.position(n,d)`) + random CV outs + scale-quantized out |
| WMD rand levels V1–V3 | Sanjuro_77 | [25974/193](https://llllllll.co/t/crow-druid-scripts/25974/193) | 4-channel VCA "auto-mix": random levels summing to a max voltage, random/selected channel order |
| feedback ASR (mono + dual) | driedstr | [25974/228](https://llllllll.co/t/crow-druid-scripts/25974/228) | Analog shift register with feedback table (linear/tanh variants), dual version cross-clocked |
| coordinates.lua | vinc | [25974/203](https://llllllll.co/t/crow-druid-scripts/25974/203), [gist](https://gist.github.com/vinc6/d2b55f3e64f1b1c2bcab9b87e051e72a) | Quadrant magnitude CV processor (polar decomposition of two CVs) |
| synmatrixseq.lua | haze | [25974/214](https://llllllll.co/t/crow-druid-scripts/25974/214) | 4×4 matrix sequencer for w/syn; two clock divisions walk x/y |
| seq m8 | Oootini | [25974/143](https://llllllll.co/t/crow-druid-scripts/25974/143) | External-sequence variation engine: 16-stage shift register + `public.add` params + sequins steps |
| sample & hold harmony | nolliegray | [25974/286](https://llllllll.co/t/crow-druid-scripts/25974/286) | S&H with +3rd / +5th / octave-down harmony outs |
| **Dial** | license (with Galapagoose / synthetivv consulting) | [25974/116](https://llllllll.co/t/crow-druid-scripts/25974/116), [library](https://llllllll.co/t/dial-0-2-0/37732) | Polyrhythmic polyphasic metronome / ramp generator with phase reconciliation |
| jazz ensemble (jfstrumm) | onoma | [25937/164](https://llllllll.co/t/crow-strategies-just-friends/25937/164) | Sequins-arithmetic strumming piano + bass over JF (§3.4) |
| JF + First hybrid | bloc | [25937/89](https://llllllll.co/t/crow-strategies-just-friends/25937/89) | Binary-mask rhythm gates, LCG patterns, JF transpose tracking, First-compatible outputs |
| jf retune interval selector | foldedworlds | [25937/154](https://llllllll.co/t/crow-strategies-just-friends/25937/154) | Seven retune ratio sets selected by voltage window (§5.4) |
| less_concepts | dndrks (dani_derks) | [25937/1](https://llllllll.co/t/crow-strategies-just-friends/25937/1), [repo](https://github.com/dndrks/less_concepts) | Norns script (2-voice JF via crow) — origin of the 10 ms `jf_note_spacer` pattern |
| crow-max-and-m4l | monome / voidstar / dani_derks | [repo](https://github.com/monome/crow-max-and-m4l) | Max/M4L toolkit incl. `^^jf_synth` polyphonic JF device |
| First.lua | monome (built-in) | [25974/58](https://llllllll.co/t/crow-druid-scripts/25974/58), [source](https://github.com/monome/crow/blob/master/lua/First.lua) | Firmware-shipped default script; reload with `^^F` |
| Boids | (druid examples) | [25974/12](https://llllllll.co/t/crow-druid-scripts/25974/12) | Flocking simulator from the (now-moved) druid examples collection |
| crow-script-nest.sh | driedstr | [25974/285](https://llllllll.co/t/crow-druid-scripts/25974/285) | Bash+fzf druid script picker/uploader |

---

## 8. Sources

Primary threads (both complete rendered dumps; some posts are deleted and unavailable — thread A
posts 13, 66, 78, 89, 123, 156, 161, 165, 168, 170, 197, 210, 273; thread B posts 18, 21, 24, 47,
53, 54, 66, 130, 160, 162):

- **"^^ crow: druid scripts"** — https://llllllll.co/t/crow-druid-scripts/25974
- **"^^ crow strategies: Just Friends"** — https://llllllll.co/t/crow-strategies-just-friends/25937

External code cited (thread embeds truncate; these are the full sources):

- scrolls — https://github.com/crimclark/scrolls
- histo-qt — https://github.com/csboling/crow-scripts/blob/aca2c850bc23fdc5b7810987811c290098405420/sequencing/histo-qt.lua
- drummer.lua — https://gist.github.com/nattog/841bcfb813aefdc751cacf0d4b79f4ab
- maps_s2e3 (contains `fnl()` — continuous i2c sweep at a given framerate, for glissandi; cited by
  Galapagoose in [25937/105](https://llllllll.co/t/crow-strategies-just-friends/25937/105)) —
  https://gist.github.com/trentgill/ec9fcf2d2583bb75934b7130baf231bb
- swingclock — https://gist.github.com/jmsole/6572916b58f79bb75392cd589b5e25da
- vcswitch / xfade — https://github.com/echophon/crowscripts
- window-crowparator — https://github.com/xmacex/window-crowparator
- population-crowth — https://github.com/xmacex/population-crowth
- crow-max-and-m4l — https://github.com/monome/crow-max-and-m4l
- less_concepts — https://github.com/dndrks/less_concepts

Historical note: the original druid examples collection linked in thread A
(`github.com/monome/druid/tree/master/examples`) is **dead** — the successor is the Bowery,
**https://github.com/monome/bowery**. Companion reference files: `api-time.md`, `api-sequins.md`,
`api-asl.md`, `api-input-output.md`, `ii-just-friends.md`, `ii-overview.md`, `druid-and-protocol.md`,
`troubleshooting.md`.
