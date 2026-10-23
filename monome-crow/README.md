# monome-crow skill

A coding-agent skill for programming the [monome crow](https://monome.org/docs/crow/) eurorack
module in Lua — full API, druid/serial workflow, i2c control of companion modules, generative
patterns, and community-tested techniques. Everything is cross-checked against the official docs,
the crow firmware source, and hardware-tested protocol notes.

## Layout

- `SKILL.md` — entry point: principles, workflow, routing table, top gotchas
- `references/api-input-output.md` — input modes/handlers, output volts/slew/shape/scale/clock, built-in actions
- `references/api-asl.md` — ASL journeys, `dyn`, mutations, limits
- `references/api-sequins.md` — complete sequins reference
- `references/api-time.md` — metro, clock, delay, timeline
- `references/api-environment.md` — lifecycle, tell/print, public, hotswap, cal, randomness, globals, norns
- `references/druid-and-protocol.md` — druid, the `^^` wire protocol, crow-mcp server
- `references/technical.md` — hardware, limits, firmware, recovery
- `references/ii-overview.md` — ii bus mechanics, `ii.self`, wire types
- `references/ii-just-friends.md` — Just Friends / Just Type / Geode
- `references/ii-wslash.md` — W/Tape, W/Del, W/Syn
- `references/ii-other-devices.md` — ansible, crow, disting, er301, faders, kria, levels, meadowphysics, txi, txo
- `references/patterns-generative.md` — Euclidean, chaos, boids, ASR, CV delay, strum, data banks
- `references/patterns-musical.md` — musical intent, Maps curriculum, patch archetypes
- `references/community-techniques.md` — attributed techniques from the lines forum
- `references/troubleshooting.md` — consolidated pitfalls + symptom index

## References

- [crow reference](https://monome.org/docs/crow/reference/) ·
  [timeline](https://monome.org/docs/crow/timeline/) ·
  [technical](https://monome.org/docs/crow/technical/) ·
  [FAQ](https://monome.org/docs/crow/faq/) ·
  [maps](https://monome.org/docs/crow/maps/) ·
  [druid](https://monome.org/docs/crow/scripting-druid/)
- [crow source](https://github.com/monome/crow) · [bowery scripts](https://github.com/monome/bowery)
- [Just Friends Just-Type](https://github.com/whimsicalraps/Just-Friends/blob/main/Just-Type.md)
- [W/ wiki](https://github.com/whimsicalraps/wslash/wiki/)
- [lines forum](https://llllllll.co/)
