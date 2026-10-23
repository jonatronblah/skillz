# ii Devices (`ii.*`) — complete reference (everything except Just Friends & W/)

The ii (i2c) bus lets crow lead a whole ecosystem of modules. This file is the complete
command/query index for **every** supported device other than the two with their own curated
references: **Just Friends** (`ii.jf` → `ii-just-friends.md`) and the W/ family
(**W/Del, W/Syn, W/Tape** (`ii.wdel`, `ii.wsyn`, `ii.wtape`) → `ii-wslash.md`). Covered here:
ansible, crow (as an ii *target*), disting (Disting EX), er301, faders (16n), kria, levels,
meadowphysics, txi, txo.

**Sources:** compiled from `ii.<module>.help()` outputs on crow (early 2025), via
[bowery ii_help.txt](https://github.com/awesomebrick/bowery/blob/main/twigs/ii_help.txt); crow
firmware device definitions [`lua/ii/*.lua`](https://github.com/monome/crow/tree/main/lua/ii).
Provenance notes in §16.

---

## 1. The shared command / query / event pattern

Every device namespace works the same way — only the device name and its argument lists change:

```lua
ii.<device>.command(args)            -- fire-and-forget write to the module
ii.<device>.get('param'[, args])     -- request a value back from the module
ii.<device>.event = function(e, value) end   -- where the reply lands
```

Responses to every `.get()` arrive through the shared event pattern (identical for all devices):

```lua
ii.<device>.event = function(e, value)
  if e.name == '<param>' then
    -- e.arg: first argument of the get (eg channel); e.device: index of device
  end
end
```

- **Getter replies need the `.event` handler.** Without a handler installed, replies are dropped
  silently — a query just produces nothing (this is silent; there is no error).
- **An unknown getter name raises a LOCAL Lua error `getter not found for 'x'`** — even with no
  device attached. That makes it a free typo guard. An *unanswered* query (device absent) simply
  produces nothing; crow does not check for ACKs.
- **`ii.<device>.help()` prints the live command/query list on the hardware** — that is the
  authoritative source. The tables below are a faithful snapshot of those help outputs (see §16);
  if your firmware is newer, trust `help()`.

---

## 2. ansible (`ii.ansible`)

monome ansible — the grid-controlled trigger/CV hub that hosts the Kria, Meadowphysics and Levels
apps; these base commands drive its 4 trigger outs and 4 CV outs directly, plus input reads.

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

---

## 3. crow (`ii.crow`) — crow as an ii target

crow can also be a **follower** on the bus: another ii leader (Teletype, a second crow, a
norns-based leader) can set this crow's outputs, trigger arbitrary Lua on it via `call`, query it
via `query`, and reset it. This crow's own ii identity is `ii.address` (1–4 → i2c
`{0x01,0x02,0x03,0x04}`).

Caller side — what the *other* device sends (a leader would use these as `ii.crow.*`):

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

**NB — query arities look shifted**: the help capture prints `query2` with 1 arg and `query3` with
2, while the follower-side naming is `query0..query3` = 0–3 args. When in doubt prefer the
**unified** follower form `ii.self.query = function(args)` (below) and/or send the full arg list;
trust `ii.crow.help()` on your own firmware.

### Handling incoming traffic on the target crow (`ii.self`)

Bind handlers to the fixed per-arity slots — "call" slots fire side effects (call1–4 = 1–4 args),
"query" slots must return a value (query0–3 = 0–3 args):

```lua
ii.self.call2 = function(x, y)   -- "call" slots fire side effects (call1-4 = 1-4 args)
  output[1].volts = x
  output[2](true)
end

ii.self.query1 = function(x)     -- "query" slots must return a value (query0-3 = 0-3 args)
  return input[x].volts
end
```

Alternatively assign the **unified dispatch** forms: the firmware routes *all* `call1`–`call4`
into `ii.self.call` and *all* `query0`–`query3` into `ii.self.query` if you assign those. Both
take the incoming arguments as a table. Unassigned slots (the defaults) just print.

```lua
-- unified form: one handler for every call1..call4
ii.self.call = function(args)
  -- args = table of the call's arguments, eg {x, y} for call2(x, y)
  output[1].volts = args[1]
end

-- unified form: one handler for every query0..query3 (must return a value)
ii.self.query = function(args)
  return input[args[1]].volts
end
```

---

## 4. disting — Disting EX (`ii.disting`)

Expert Sleepers Disting EX — the multi-algorithm swiss-army module (poly player, sampler, looper,
MIDI utilities, …); ii selects algorithms, sets parameters, plays polyphonic notes, and drives the
looper/recorder.

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

---

## 5. er301 (`ii.er301`)

Orthogonal Devices ER-301 sound computer — ii drives its trigger/CV inputs (units/slices/etc. via
the ER-301's own ii assignments).

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

---

## 6. faders — 16n Faderbank (`ii.faders`)

16n faderbank — 16 sliders readable over ii.

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

---

## 7. kria — ansible kria (`ii.kria`)

ansible's Kria — grid sequencer with per-track parameters, patterns, presets, scales and clock;
ii exposes preset/pattern recall, playback position, looping, mutes and CV reads.

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

---

## 8. levels — ansible levels (`ii.levels`)

ansible's Levels — preset-based CV level sequencer; ii sets preset, playback position, loop
points/direction and reads track CV.

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

---

## 9. meadowphysics (`ii.meadowphysics`)

monome Meadowphysics (ansible app) — cascading polyrhythmic trigger/CV generator; ii sets preset,
scale, period and per-track stop/reset, and reads track CV.

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

---

## 10. txi (`ii.txi`)

monome txi — 4-knob / 4-input CV source expander; over ii it is read-only.

```lua
-- queries only (no commands)
ii.txi.get( 'param', channel )
ii.txi.get( 'param_quant', channel )
ii.txi.get( 'param_N', channel )
ii.txi.get( 'in', channel )
ii.txi.get( 'in_quant', channel )
ii.txi.get( 'in_N', channel )
```

---

## 11. txo (`ii.txo`)

monome txo — 4-port trigger/CV/oscillator/envelope output expander; ii drives everything.

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

---

## 12. Devices covered in sibling files

- **jf — Just Friends (`ii.jf`)**: full curated reference in `ii-just-friends.md` (trigger modes,
  synthesis/Geode personalities, getters, tuning, the Geode wire-scaling story).
- **wdel / wsyn / wtape — W/Del, W/Syn, W/Tape (`ii.wdel`, `ii.wsyn`, `ii.wtape`)**: full curated
  references in `ii-wslash.md` (their command/query indexes and getters live there).

---

## 13. Device index — namespaces and i2c addresses

| Device | crow namespace | i2c address(es) | Reference |
|---|---|---|---|
| ansible | `ii.ansible` | {0x20,0x21,0x22,0x23} | §2 |
| crow | `ii.crow` | {0x01,0x02,0x03,0x04} (crow's own `ii.address` 1-4) | §3 |
| disting | `ii.disting` | {0x41,0x42,0x43,0x44} | §4 |
| er301 | `ii.er301` | {0x31,0x32,0x33} | §5 |
| faders | `ii.faders` | {0x34,0x35,0x36,0x37} | §6 |
| jf | `ii.jf` | {0x70,0x75} | `ii-just-friends.md` |
| kria | `ii.kria` | 0x28 | §7 |
| levels | `ii.levels` | 0x2c | §8 |
| meadowphysics | `ii.meadowphysics` | 0x2A | §9 |
| txi | `ii.txi` | {0x68..0x6F} | §10 |
| txo | `ii.txo` | {0x60..0x67} | §11 |
| wdel | `ii.wdel` | {0x78,0x79} | `ii-wslash.md` |
| wsyn | `ii.wsyn` | {0x76,0x77} | `ii-wslash.md` |
| wtape | `ii.wtape` | {0x71,0x72} | `ii-wslash.md` |

---

## 14. ii value wire types

Command args are encoded per the device's type table (crow firmware `lib/ii.c`, device tables in
`lua/ii/*.lua`):

| Wire type | Encoding | Typical use |
|---|---|---|
| `s8` / `u8` / `s16` | raw integers, sent as-is | states, channel/port indices, modes, counts, ratios |
| `s16V` | volts × **1638.3** (the Teletype `V` unit) before sending | pitch / level values ("V" args) |
| `s16ms` | seconds × 1000 | millisecond time values |
| `s32T` | int16 seconds + fractional part × 1638.3 | longer time values |

**Practical rule:** pass **natural volts** for V args (crow does the ×1638.3 encoding for you), and
**plain integers** for `s8`/`u8`/`s16` args — never scale `s8` args by 1638. Pitch follows the crow
convention throughout: **0 V = middle C = 261.63 Hz** (the docs call this "C3"; in scientific pitch
notation it is C4), 1 V/octave; levels are plain volts (5.0 ≈ standard modular level).

The one place the V-scaling bites is JF Geode: `ii.jf.play_note`/`play_voice` args are typed
`s16V`, but the Geode engine on the far side reads the *raw wire integers* as `divs`/`repeats` —
so from crow you divide those args by 1638.3. That story (and only that story) is documented in
`ii-just-friends.md` §5.

---

## 15. Duplicate devices

`ii.<dev>[n]` does **not** give you a per-device handle: it switches the module's **shared** i2c
address and returns the **same table**. Always index at call time, and rely on `e.device` in the
event handler to tell the replies apart:

```lua
ii.txi[1].get('param',1)  -- first device
ii.txi[2].get('param',1)  -- second device

ii.txi.event = function(e, value)
  if e.name == 'param' then
    print('txi[' .. e.device .. '][' .. e.arg .. ']=' .. value)
  end
end
```

Caching the handle (`local t = ii.txi[2]`) makes later calls silently send to whichever address
was set **last**. A second unit must first be configured at its next address (see §13); commands
to an address with no device attached are silently ignored — no ACK checking, no error.

---

## 16. Sources

- Old skill provenance (the origin of the command/query tables above): compiled from
  `ii.<module>.help()` outputs on crow (early 2025), via
  [bowery ii_help.txt](https://github.com/awesomebrick/bowery/blob/main/twigs/ii_help.txt).
  **Note:** that capture predates 2025-01-22, so it lacks `ii.wtape.get('loop_active')` (covered in
  `ii-wslash.md`).
- crow firmware ii device definitions (addresses, command arities, wire types):
  [`lua/ii/*.lua`](https://github.com/monome/crow/tree/main/lua/ii) and `lib/ii.c`.
- On hardware, `ii.<device>.help()` is always the authoritative list.
