# ii (i2c) — how the bus works from crow

crow can **lead** (control) up to 14 families of i2c ("ii") devices: Just Friends, W/Tape, W/Del,
W/Syn, Ansible (kria/levels/meadowphysics), Disting EX, ER-301, 16n faders, TXi, TXO, and other
crows. This file covers the bus mechanics shared by all of them. Per-device command tables:
`ii-just-friends.md`, `ii-wslash.md`, `ii-other-devices.md`.

## 1. Physical layer

- Rear ii connector; **white stripe = GND**. 3.3 V open-drain bus with pullups.
- **`ii.pullup(state)`** — crow supplies the bus pullups (default **on**; the docs say it "should
  probably stay on"). Required when crow is the only/first device on the bus.
- **`ii.fastmode(state)`** — true = max speed (~4×, default **false** = "modest speed for
  stability").
- **One leader transmits at a time**: multiple *listeners* are fine; two leaders (two crows, or
  crow + Teletype) transmitting simultaneously collide. Stagger traffic or split buses.
  Leader/follower roles switch per transaction automatically.
- Cable length matters (long runs have caused failures for users).

## 2. The Lua API

```lua
ii.help()               -- print supported devices
ii.<device>.help()      -- print that device's commands (from firmware tables)
ii.m_help(address)      -- list commands by raw i2c address (undocumented alias)

ii.<device>.command([args])          -- setter
ii.<device>.get('param' [, args])    -- query — does NOT return a value! see §3
ii.<device>.event = function(e, value) ... end   -- replies arrive here
```

Every device table is generated from crow's `lua/ii/*.lua` descriptors. **The on-device
`ii.<device>.help()` output is the authoritative, firmware-matched reference** — if this skill and
the device disagree, trust the device.

### Duplicate devices — the shared-table caveat

```lua
ii.txi[1].get('param', 1)   -- first TXi
ii.txi[2].get('param', 1)   -- second TXi
```

**There are no per-device handles.** `ii.<dev>[n]` switches the module's shared i2c address and
returns **the same table** (out-of-range indices clamp; address stays unchanged). Always index at
call time — `ii.wtape[2].record(1)`, never `local t = ii.wtape[2]` (cached handles silently send
to whichever address was set last). Replies are disambiguated by `e.device`.

Devices ship at a factory address; a second unit must be set to the alternate address (each device
family has its own procedure — e.g. W/ boot gestures, `ii.jf.address(2)`). Commands to an address
with no device are **silently ignored** — no ACK check, no error.

### This crow's own address

```lua
print(ii.address)   --> 1 by default; valid 1-4 (maps to i2c 0x01-0x04)
ii.address = 2      -- when several crows share a bus
```

Other crows are reachable as devices: `ii.crow[1].volts(1, 2.9)`.

### Raw access

```lua
ii.raw(addr, bytes [, rx_len])   -- e.g. ii.raw(0x70, '\x09\x00\x00\x20', 2)

ii.event_raw = function(addr, cmd, data, arg)
  -- ALL ii traffic passes through this FIRST. Match the address and
  -- `return true` to consume the event (else it also goes to the normal
  -- ii dispatcher — which errors on unknown commands).
end
```

Raw is how you talk to devices crow doesn't have tables for. `ii.event_raw` actually receives four
args (`addr, cmd, data, arg`) — many docs show three.

### Crow as an ii **follower** (Teletype → this crow)

Another leader (Teletype, another crow) can call *this* crow. Firmware routes incoming calls to
`ii.self` slots:

| wire cmd | args | user slot | default behavior |
|---|---|---|---|
| `volts` | (chan, level) | `ii.self.volts` | `output[chan].volts = level` |
| `slew` | (chan, time) | `ii.self.slew` | `output[chan].slew = time` |
| `call1` | (a) | `ii.self.call1` → funnels to `ii.self.call{a}` | prints |
| `call2` | (a,b) | `ii.self.call2` → `ii.self.call{a,b}` | prints |
| `call3` | (a,b,c) | `ii.self.call3` → `ii.self.call{a,b,c}` | prints |
| `call4` | (a,b,c,d) | `ii.self.call4` → `ii.self.call{a,b,c,d}` | prints |
| `reset` | — | `ii.self.reset` | `crow.reset()` |
| `pulse` | (chan, time, level, pol) | `ii.self.pulse` | `output[chan](pulse(...))` |
| `ar` | (chan, atk, rel, level) | `ii.self.ar` | `output[chan](ar(...))` |
| `lfo` | (chan, freq, level, skew) | `ii.self.lfo` | `output[chan](ramp(2^-freq, skew, level))` — freq 0 = 1 Hz |
| `input`/`output` queries | (chan) | C only | return the jack value |
| `query0` | — | `ii.self.query0` → `ii.self.query{}` | returns a number |
| `query1..3` | (1..3 args) | `ii.self.query1..3` → `ii.self.query{args}` | returns a number |

Two assignment styles both work (the per-arity slots funnel into the unified ones when you don't
override them):

```lua
-- per-arity (matches Teletype CROW.C1..C4 / CROW.Q1..Q4):
ii.self.call2 = function(x, y) output[1].volts = x end
ii.self.query1 = function(x) return input[x].volts end

-- unified (receives an args table):
ii.self.call  = function(args) output[1].volts = args[1] end
ii.self.query = function(args) return input[args[1]].volts end
```

Caller side: `ii.crow.call2(x, y)`, `ii.crow.get('query1', x)`. Dispatch-table idiom (route one
Teletype op to many functions): `community-techniques.md` §5. Note the older `ii._c.call*` naming
seen in some dev docs is outdated — use `ii.self.*`.

## 3. Queries are asynchronous

**`get()` never returns the value.** The reply arrives later in the device's `.event` handler:

```lua
ii.jf.event = function(e, value)
  -- e.name = 'ramp' etc., e.arg = first get-arg, e.device = unit index
  print(e.name .. ' = ' .. value)
end

ii.jf.get('ramp')       -- fire the query
```

- With **no handler installed, replies vanish silently** on current firmware (no automatic
  host forwarding).
- An unknown getter **name** raises a local Lua error `getter not found for 'x'` even with no
  device attached (typo guard — the name check is local).
- Silence after a valid query = device absent, busy, or (W/Del) never answering at all.
- **Presence probe**: install a printing `.event` handler per family, `get()` one cheap getter,
  wait ~10 ms — reply = present.

### Polling & bus hygiene

- Pace requests: ~1.5 ms between `get()` calls, full sweeps every few hundred ms (bowery's
  `w_slash_stats.lua` pattern — see `patterns-generative.md` §10).
- Continuous parameter sweeps saturate the bus easily — throttle continuous gestures to ~15 fps
  (`fnl()` from Maps S2E3, or a metro). See `troubleshooting.md` §5.6.
- Don't poll fragile devices fast (JF panel getters crashed at ~5 Hz in one report;
  `ii.wtape.get('play')` has hung crow in another).

## 4. Value wire types

Command args are encoded per the device's table type (`lib/ii.c`):

| type | encoding | meaning |
|---|---|---|
| `s8` / `u8` / `s16` / `u16` | raw integers | modes, channels, small counts — **pass plain integers** |
| `s16V` | volts × **1638.3** (`II_TT_VOLT`) | pitch/level/run values — **pass natural volts** |
| `s16ms` | seconds × 1000 | millisecond times |
| `s32T` | int16 seconds + fractional × 1638.3 | timestamps |
| `float` | raw 4 bytes | (rare) |

Practical rule: volts arguments ("V" types) take **volts** and are scaled for you. Integer
arguments are raw. The one trap: when the *far device* reads a V-scaled value as a raw integer
(JF's Geode `divs`/`repeats`), pre-divide by 1638.3 — see `ii-just-friends.md` §5.

## 5. Quick device index

| namespace | device | addresses | details |
|---|---|---|---|
| `ii.jf` | Just Friends | 0x70, 0x75 | `ii-just-friends.md` |
| `ii.wtape` / `ii.wdel` / `ii.wsyn` | W/ Tape, Del, Syn | 0x71/0x72, 0x78/0x79, 0x76/0x77 | `ii-wslash.md` |
| `ii.ansible` | Ansible | 0x20–0x23 | `ii-other-devices.md` |
| `ii.kria` / `ii.levels` / `ii.meadowphysics` | Ansible apps | 0x28, 0x2c, 0x2A | `ii-other-devices.md` |
| `ii.disting` | Disting EX | 0x41–0x44 | `ii-other-devices.md` |
| `ii.er301` | ER-301 | 0x31–0x33 | `ii-other-devices.md` |
| `ii.faders` | 16n faderbank | 0x34–0x37 | `ii-other-devices.md` |
| `ii.txi` | TXi | 0x68–0x6F | `ii-other-devices.md` |
| `ii.txo` | TXO | 0x60–0x67 | `ii-other-devices.md` |
| `ii.crow` | another crow | 0x01–0x04 | `ii-other-devices.md` (+ §2 above) |

## 6. Sources

- crow reference (ii section): https://monome.org/docs/crow/reference/
- ii overview (monome): https://monome.org/docs/modular/ii
- firmware: `lua/ii.lua`, `lua/ii/*.lua`, `lib/ii.c`, `lib/l_ii_mod.c`
  (https://github.com/monome/crow)
- live `ii.help()` capture: https://github.com/awesomebrick/bowery/blob/main/twigs/ii_help.txt
- i2c background: https://llllllll.co/t/a-users-guide-to-i2c/19219
