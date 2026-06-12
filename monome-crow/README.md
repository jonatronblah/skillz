# monome-crow skill

A coding agent skill for programming the [monome crow](https://monome.org/docs/crow/) eurorack module using Lua, including i2c control of companion modules.

## What it covers

- **crow Lua API**: input, output, ASL, sequins, metro, clock, timeline, hotswap, ii, public, calibration
- **Just Friends** (`ii.jf`): synthesis mode, geode mode, triggers, tuning, velocity
- **W/Tape** (`ii.wtape`): recording, playback, looping, vari-speed
- **W/Del** (`ii.wdel`): delay, feedback, freeze, modulation, clock sync
- **W/Syn** (`ii.wsyn`): FM synthesis, polyphony, vactrol LPG, patching

## Usage

This skill is automatically invoked when writing or debugging crow Lua scripts, or when controlling i2c-connected modules from crow.

## References

- [crow reference](https://monome.org/docs/crow/reference/)
- [crow source](https://github.com/monome/crow)
- [Just Friends Just-Type](https://github.com/whimsicalraps/Just-Friends/blob/main/Just-Type.md)
- [W/Tape ii](https://github.com/whimsicalraps/wslash/wiki/Tape#ii)
- [W/Del ii](https://github.com/whimsicalraps/wslash/wiki/Del#ii)
- [W/Syn ii](https://github.com/whimsicalraps/wslash/wiki/Syn#ii)
