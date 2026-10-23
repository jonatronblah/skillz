---
name: monome-crow
description: "Program the monome crow eurorack module in Lua: the complete crow API (input modes, output/ASL, sequins, metro, clock, delay, timeline, hotswap, public, cal), the druid/USB serial workflow (incl. raw wire protocol and the crow-mcp server), and i2c (ii) control of companion modules — Just Friends (Just Type/Geode), W/Tape, W/Del, W/Syn, Ansible, Disting EX, ER-301, TXi, TXO, 16n faderbank — plus generative algorithms and community-tested techniques. WHEN: writing/debugging crow Lua, monome crow, druid upload, crow sequins/ASL/timeline, ii or i2c from crow, Just Friends or W/ control, crow + norns or Teletype, generative eurorack scripting."
version: 2.0.0
---

# monome crow

crow is a 2hp eurorack module — 2 CV inputs, 4 CV outputs (16-bit, −5 V…+10 V), an ii/i2c bus —
scripted in **Lua**. It stores **one script** which auto-runs at power-on without a computer.
This skill is a complete, source-verified reference for programming it at expert level.

## When to use

- Writing, editing, or debugging crow Lua scripts (`.lua` for druid)
- Explaining crow API: `input`, `output`, ASL, `sequins`, `metro`, `clock`, `delay`, `timeline`,
  `hotswap`, `public`, `cal`
- Driving ii devices from crow: Just Friends, W/Tape, W/Del, W/Syn, Ansible, Disting EX, ER-301,
  TXi/TXO, 16n, other crows — or Teletype calling *this* crow (`ii.self`)
- Automating crow over USB: druid, the `^^` wire protocol, or the crow-mcp server
- Generative/algorithmic patch design: Euclidean rhythms, chaos, boids, shift registers, JI
- crow ↔ norns / Max interaction (`public`, `tell`, `crow.send`)

## Key principles

- **Units**: plain volts; pitch is **1V/oct** with **0 V = middle C = 261.63 Hz** (monome/Whimsical
  docs call this "C3"; scientific pitch notation calls it C4). One semitone = `1/12`. Times are
  **seconds**. `time()` returns **milliseconds**. I/O clips outside **−5…+10 V**.
- **Script lifecycle**: top-level code runs once at load, *before* `init()`; `init()` does setup;
  `cleanup()` runs before the next script replaces this one; `crow.reset()` clears everything.
  `init()` must be spelled exactly.
- **Event-driven**: inputs/metros/clocks invoke handlers you assign (`input[1].stream = fn` —
  assign the function, don't call it). **Nothing reaches the host automatically** — use
  `print()` / `tell()` to see values.
- **Watchdog**: a Lua loop that doesn't yield (`clock.sleep`/`clock.sync`) for ~2 s dies with
  `CPU timed out.` Every `while true` needs a yield.
- **Musical time lives in `clock`** (`clock.sync(1/4)` follows BPM, external clock, tempo changes);
  `metro` is wall-clock timers; `timeline` is declarative beat/second scores; `delay()` is
  one-shot sugar over `clock`.
- **Gates/envelopes are ASL**: `output[1].action = adsr()` then `output[1](true)` / `(false)` —
  a bare `output[1]()` skips a held/adsr body.
- **ii is async and stateful**: `get()` replies arrive in a `.event` handler (silent without one);
  `ii.<dev>[n]` re-points a shared table (always index at call time); writes to absent devices are
  silently ignored. Volt args are auto-scaled (×1638.3); raw integer args are not.
- **Persistence**: one script in flash, auto-run at boot. `r file.lua` = run now (RAM),
  `u file.lua` = persist to flash. `^^p` prints the flash copy.

## Getting code onto crow

1. **druid** (official): `pip3 install monome-druid`, then `druid`; REPL: `r x.lua` (run),
   `u x.lua` (upload), `p` (print), `h`, `q`. Details + install troubleshooting:
   `references/druid-and-protocol.md`.
2. **crow-mcp** (agent tools over the same serial protocol; on this machine at
   `C:\Users\jonathan\Desktop\other projects\mcp-dev\crow-mcp`): `crow_connect`, `crow_execute`
   (live REPL), `crow_upload` (ram/flash), `crow_read_events`, `crow_read_output`, `crow_api_help`.
3. **Own client**: the `^^` wire protocol (`^^s`/`^^e`/`^^w`/`^^p`/`^^c`/`^^k`/`^^r`, upload
   pacing, `^^ready()` ack, 64-byte ZLP rule) is specified in `references/druid-and-protocol.md`.

Typical loop: live-code in the REPL (`crow_execute`) → iterate with `print()` feedback →
`crow_upload` with flash once it behaves.

## Reference map

| I need… | Read |
|---|---|
| Input modes/handlers, output volts/slew/shape/scale/clock, built-in actions (lfo/adsr/…) | `references/api-input-output.md` |
| Custom ASL journeys, `dyn`, mutations, ASL limits | `references/api-asl.md` |
| `sequins` — the full method set, flow modifiers, transformers | `references/api-sequins.md` |
| `metro`, `clock` (sync/tempo/transport/external clock), `delay`, `timeline` | `references/api-time.md` |
| Script lifecycle, `tell`/`print`, `public`, `hotswap`, `cal`, randomness, all globals, norns | `references/api-environment.md` |
| druid install & REPL, `^^` protocol, upload framing/limits, crow-mcp tools | `references/druid-and-protocol.md` |
| Hardware specs, voltage/sampling rates, CPU watchdog, memory limits, firmware, recovery | `references/technical.md` |
| ii bus mechanics: addressing, async queries, `ii.self` follower, wire types, polling | `references/ii-overview.md` |
| Just Friends / Just Type / Geode commands & techniques | `references/ii-just-friends.md` |
| W/Tape, W/Del, W/Syn commands & techniques | `references/ii-wslash.md` |
| All other ii devices (ansible, crow, disting, er301, faders, kria, levels, meadowphysics, txi, txo) | `references/ii-other-devices.md` |
| Generative algorithms (Euclidean, chaos, boids, ASR, CV delay, strum, data banks) | `references/patterns-generative.md` |
| Musical design: intent, dynamics, phrasing, tuning, Maps curriculum, patch archetypes | `references/patterns-musical.md` |
| Community-tested techniques with attribution (scripts, JF strategies, live-coding) | `references/community-techniques.md` |
| Something is broken / an error string / a gotcha | `references/troubleshooting.md` |

## Top gotchas (full list: `references/troubleshooting.md`)

1. `clock.get_beats()` / `clock.get_beat_sec()` are **functions** (`clock.tempo` is a property).
2. Handler assignment vs call: `input[1].change = fn` registers; `input[1].change = fn()` runs now
   and stores nil. Don't name a handler arg `volts` (it shadows the property).
3. `output[n].volts = x` interrupts any running ASL; read-backs lag one io cycle.
4. `held{}`/`adsr` need `output[n](true)` / `(false)` — bare `()` skips the held body.
5. `timeline.queue():loop{…}` needs the parens (the docs' paren-less form errors).
   `timeline.cleanup()` kills **all** clocks too.
6. ii `get()` returns nothing — replies come via `.event`; W/Del never answers gets at all.
7. `ii.jf.mode(1)` **latches** — send `mode(0)` in `cleanup()` or JF stops responding to pitch CV.
8. JF Geode args must be divided by 1638.3 from crow (`4/1638.3` = 4 segments); `retune` args are
   raw 0–127 and must **not** be scaled.
9. `output[n]:clock(...)` colon-only; `input[n].mode(...)` dot-only.
10. One process owns the USB port; scripts ≤ 16380 bytes; REPL chunks < 2048 bytes.

## References

- crow docs: [overview](https://monome.org/docs/crow/) ·
  [scripting with druid](https://monome.org/docs/crow/scripting-druid/) ·
  [reference](https://monome.org/docs/crow/reference/) ·
  [timeline](https://monome.org/docs/crow/timeline/) ·
  [technical](https://monome.org/docs/crow/technical/) ·
  [FAQ](https://monome.org/docs/crow/faq/) ·
  [maps](https://monome.org/docs/crow/maps/) ·
  [druid](https://monome.org/docs/crow/druid/)
- source: https://github.com/monome/crow (`lua/`, `lib/`) · scripts: https://github.com/monome/bowery
  (plus community fork with `twigs/ii_help.txt`: https://github.com/awesomebrick/bowery)
- Just Friends (Just Type): https://github.com/whimsicalraps/Just-Friends/blob/main/Just-Type.md
- W/ wiki: https://github.com/whimsicalraps/wslash/wiki/ (Tape, Del, Syn pages)
- community: [lines](https://llllllll.co/) ·
  [crow: druid scripts](https://llllllll.co/t/crow-druid-scripts/25974) ·
  [crow strategies: Just Friends](https://llllllll.co/t/crow-strategies-just-friends/25937)
- ii overview: https://monome.org/docs/modular/ii · Teletype↔crow: https://monome.org/docs/crow/teletype/
