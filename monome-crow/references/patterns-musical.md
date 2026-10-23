# Musical intent & the Maps curriculum

Technique in service of music. This file (a) walks the official **Maps** learning series — the
best compact curriculum for thinking musically on crow — and (b) distills design principles for
building patches with intent: voice, gesture, dynamics, phrasing, tuning, and performance surface.

---

## 1. Maps — "musical frameworks" (official, with video)

Three livestream episodes by Trent (Whimsical Raps), each with a text companion. Play the video,
then read the page and paste the code:

| Episode | Title | Teaches | Links |
|---|---|---|---|
| S2E1 | **Found Shapes** | sequins + clock fundamentals: `clock.tempo` get/set, `clock.run/sync/cleanup`, sequins `step`/`settable`/`count`/`every`, sequenced clock divisions | [page](https://monome.org/docs/crow/maps/s2e1) · [video](https://youtu.be/WQyoDRRQ5Qg) |
| S2E2 | **Zippers** | output `.scale` quantization, melodies as ASL contours, `to` shapes (`'exp'`), **dyn with `:step`/`:mul`/`:wrap`** — live "zipping" of running actions | [page](https://monome.org/docs/crow/maps/s2e2) · [video](https://youtu.be/8O34RzutxRI) · [ASL oscillator gist](https://gist.github.com/trentgill/23d6c8376d371997d7933d959ddba258) |
| S2E3 | **Long Distance** | clock review + **`fnl()` ("funnel")** — a slope language *for functions*: slew any Lua function (including ii calls) over time | [page](https://monome.org/docs/crow/maps/s2e3) · [video](https://youtu.be/gMeFH4AvSBw) · [gist](https://gist.github.com/trentgill/ec9fcf2d2583bb75934b7130baf231bb) |

Full playlist: ["Maps w/Trent"](https://www.youtube.com/playlist?list=PLTboTBC9twfLmlUqGBUwOvG_EQhc4I5Bx).

### `fnl()` — interpolating ii calls without saturating the bus

```lua
fnl( fn, start, { to(dest, dur), ... } [, fps = 15] )
```

Slides `fn` (any function — `ii.wdel.feedback`, `output[n].volts`, a dyn setter …) through a
journey of destinations at ~15 fps. Why it matters: **raw per-frame ii writes saturate the i2c
bus** (glissandi, feedback sweeps); `fnl` throttles to a frame rate the bus can carry. Community
scripts use it (or equivalent throttling) for every continuous ii gesture — see
`troubleshooting.md` §5.6 and `community-techniques.md` §5.

Other supplemental resources the official docs point at: [bowery](https://github.com/monome/bowery)
script collection, [asllib.lua](https://github.com/monome/crow/blob/main/lua/asllib.lua),
[norns crow-studies](https://github.com/monome/crow-studies),
[Max/M4L toolkit](https://github.com/monome/crow-max-and-m4l),
[Just-Type](https://github.com/whimsicalraps/Just-Friends/blob/main/Just-Type.md),
[Lua 5.2+ manual](https://www.lua.org/manual/5.3/) (crow embeds Lua 5.2+).

---

## 2. Designing with intent — the questions to ask before writing code

### Voice: what is sounding, and how many of them?

- **4 outputs are 4 voices** (or 4 gates, or 2 voices + 2 modulators). Assign roles explicitly:
  pitch / gate / timbre / space is a classic split.
- **Just Friends = 6 more voices** with two personalities (synth / rhythm) — decide *which*
  personality the piece wants (`ii-just-friends.md`).
- **W/Syn** = 4 FM voices with vactrol articulation — velocity *is* timbre there.
- **W/Tape & W/Del** = one "meta-voice" that transforms everything else — treat them as
  arrangement, not notes.

### Gesture: what does a player/patch actually *do*?

Crow's panel-less nature means your **inputs and publics are the instrument**. Decide the gesture
map first:

- 1V/oct input → pitch (through `mode('scale', …)` for quantized play)
- `window` mode → one CV as a **scene selector** (tempos, scale sets, patterns)
- `change` mode → gates; `peak`/`volume`/`freq` → audio-responsive control
- `public` vars → druid/norns knobs; `public.view.*` → visual feedback
- ii panel getters (JF knobs, TXi faders) → macro control (`community-techniques.md` §5/§6)

### Dynamics & articulation: velocity is not optional

- `ii.jf.vtrigger(ch, level)` and `ii.wsyn.play_note(p, vel)` take **volts as velocity** —
  sequence velocity, don't leave it at 5. A `sequins{3, 5, 2.5, 4}` velocity lane makes the same
  melody breathe.
- JF remembers the last velocity per channel — set-and-forget dynamics, then trigger.
- Crow-side dynamics: `ar`/`adsr` levels, `pulse` polarity/width, `output[n].scale` for pitch
  safety.

### Phrasing: structure time, not just triggers

- `sequins` flow-mods are phrasing: `:every(3)` = a bar-long accent cycle, `:count()` =
  sub-phrases, nested sequins = sections. Alternation via two `:count()` branches
  (`api-sequins.md` gotchas).
- `timeline.score{0, intro, 32, verse, 64, 'reset'}` = a literal song form; return `'reset'`
  probabilistically for variation-on-repeat.
- Swing and humanization: `clock.sleep(clock.get_beat_sec()/4 + swing)` on odd 16ths; velocity
  and strum-sparsity lanes (`patterns-generative.md` §11).

### Tuning: choose deliberately

- **12TET** (default) — `output[n].scale({0,2,4,7,9})`, JF `transpose`, semitone math `/12`.
- **Just intonation** — `input.mode('scale', ratios, 'ji')`, `output.scale(ratios, 'ji')`,
  `justvolts(3/2)` into `ii.jf.play_note`. JF's `retune(ch, n, d)` gives per-channel ratio banks
  (savable with `retune(-1,0,0)`); W/Tape `speed(num, den)` ratios harmonize two tapes.
- **god_mode(1)** (A=432) or `hztovolts(f, ref)` for custom references.
- Community reference scripts: La Monte Young Well-Tuned Piano JI patch and foldedworlds' retune
  interval-set selector (`community-techniques.md` §2/§5).

### Performance & failure: plan the exits

- Every patch gets a **stop**: idle-timeout (`community-techniques.md` §1), `:unless()` end
  conditions, `clock.cleanup()` (sledgehammer!), `ii.jf.mode(0)` unlatch in `cleanup()`.
- Every live-coding script gets **hotswap**-friendly data (`hotswap.seq = …`) so the piece can be
  rewritten mid-flight.
- Every generative system gets **one macro input** that can pull it back to sanity (tempo,
  density, filter).

---

## 3. Patch archetypes — recommended stacks

| Archetype | Stack | Key references |
|---|---|---|
| Generative sequencer | `sequins` + `clock.sync` + `output.scale` / `ii.jf.play_note` | Maps S2E1; `api-sequins.md`; `patterns-generative.md` §9 |
| Rhythmic engine | `output[n]:clock(div)` ×4 + Euclidean + window-mode preset swap | `patterns-generative.md` §1/§8 |
| Polyrhythm rig | one fast metro + counters (Dial), or `timeline` with `sequins` times | `community-techniques.md` §1 |
| Drone/chaos system | Lorenz/gingerbreadman + ASL `dyn` zippers + W/Syn drones | `patterns-generative.md` §2/§3; Maps S2E2 |
| Tape/echo performance | W/Tape loop cues (`timestamp`+table) + W/Del `fnl` sweeps + freeze gestures | `ii-wslash.md` |
| JF synth/Geode duo | `mode(1)` + velocity lanes + retune banks + `tick`/`quantize` | `ii-just-friends.md`; `community-techniques.md` §5 |
| Audio-reactive | `volume`/`peak`/`freq` input modes → `public` → ii gestures | `api-input-output.md` |
| Live-coding showpiece | `hotswap` + `public` command functions + `scrolls`-style queues | `community-techniques.md` §3/§6 |

---

## 4. Where to look next

- Concrete algorithms: `patterns-generative.md`
- Forum-tested techniques with attribution: `community-techniques.md`
- When things misbehave: `troubleshooting.md`
- API details: `api-input-output.md`, `api-asl.md`, `api-sequins.md`, `api-time.md`,
  `api-environment.md`
- Devices: `ii-overview.md`, `ii-just-friends.md`, `ii-wslash.md`, `ii-other-devices.md`
