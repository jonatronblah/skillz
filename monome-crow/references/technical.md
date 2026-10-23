# Technical — hardware, limits, firmware, recovery

Hard constraints and hardware facts for the monome crow. Script design decisions (memory budgets,
watchdog behavior, voltage ranges) come from here.

**Sources:** [overview](https://monome.org/docs/crow/), [technical](https://monome.org/docs/crow/technical/),
[FAQ](https://monome.org/docs/crow/faq/), [update](https://monome.org/docs/crow/update/) /
[manual_update](https://monome.org/docs/crow/manual_update/),
[readme-development.md](https://github.com/monome/crow/blob/main/readme-development.md), firmware
source.

---

## 1. Hardware

- **Eurorack module, 2hp**, 41 mm depth.
- **Power**: 60 mA @ +12 V, −15 mA @ −12 V, **no 5 V rail used**.
- **2 CV inputs, 4 CV outputs**, all **16-bit**, range **[−5 V, +10 V]**. Voltages outside the
  range are **clipped** (inputs clipped on read, outputs at the DAC).
- Rear **ii / i2c** bus connector (white stripe = GND).
- **CPU**: STM32F7-series (512 kB flash part; repo uses `stm32f7xx` HAL). **DAC: DAC8565** (16-bit).
  **Input ADC: ADS131**. Input impedance is not documented officially.
- **USB**: CDC virtual serial port (enumerates as `0483:5740`; product string
  "crow: telephone line"). crow is **not** a USB host — a computer or norns must supply USB power
  and initiate communication. Docs say "mini USB" in the FAQ table; current hardware is USB-C.
- Debug UART (solder-on header) runs 115200 baud — the USB CDC link has no meaningful baud rate.

## 2. Signal timing

- Outputs are synthesized at **48 kHz** (block size 32 samples) — you never touch samples directly;
  `output.volts`/ASL shape smooth curves at that rate.
- Inputs are sampled at **1.5 kHz** — fine for CV and envelopes, *not* audio-rate. Audio analysis
  modes (`volume`, `peak`, `freq`) do their own detection on that stream.

## 3. CPU & the watchdog

- A user Lua loop that **doesn't yield for ~2 s** trips the watchdog: crow prints `CPU timed out.`
  and errors the script. `while true do end` without `clock.sleep`/`clock.sync` is a guaranteed
  crash — always yield.
- `cputime()` reports main loops per DSP block (last 8 ms) — **higher = more idle CPU**. Use it to
  profile: print it before/after a heavy section.
- Heavy ASL/clock activity, lots of i2c traffic, and 1 kHz `input.stream` handlers all cost Lua
  time; budget accordingly.

## 4. Memory & script size

Flash layout (512 kB total):

| Region | Size | Contents |
|---|---|---|
| `0x08000000` | 48 kB | DFU bootloader |
| `0x0800C000` | 16 kB | calibration data |
| `0x08010000` | 64 kB | **user script** |
| `0x08020000` | 384 kB | application (firmware) |

- **User script limit: ~16 kB (16380 bytes practical)** — the FAQ says "up to 16kb, which is
  ~700 lines". (The FAQ's Teletype comparison table says "8kb" and the dev readme mentions an
  8 kB upload block — treat 16380 bytes as the working limit; if an upload is rejected, trim.)
- **REPL chunks must stay under 2048 bytes** (crow replies `!chunk too long!` past that); keep
  individual lines/chunks under ~2000 bytes.
- ASL resource limits per output (`lib/casl.h`): **16 `to` stages, 8 sequences, 8 stages per
  sequence, 40 dynamics**. Overruns print `ERROR: no stages left!` / `ERROR: no sequences left!` /
  `ERROR: no dynamic slots remain`. An ASL containing a *function* errors with
  "Do you have a function in your ASL? Replace it with dyn."
- Lua runtime: Lua **5.2+** semantics (embedded). `math.random` is a **true hardware random**
  (on-chip TRNG); `math.srandom*` is the seeded pseudo-RNG (see `api-environment.md`).

## 5. Persistence & power-on behavior

- crow stores **exactly one** user script in flash. It **auto-runs at power-on** — no USB needed
  (USB *power* is still needed if nothing else powers the case... crow runs from eurorack power;
  USB is only for host comms).
- Factory default script is **First** (a generative sequencer).
- druid `r file.lua` = run now (RAM, lost on power cycle); `u file.lua` = upload to flash
  (persists). See `druid-and-protocol.md`.

## 6. Firmware

- Current source: **crow firmware 4.0.5** (`version.txt`, repo last pushed 2026-04-18; recent Lua
  additions: `clock.sync(beats, offset)` Dec 2025, `tempo_change` handler, wtape `loop_active`
  getter Jan 2025). Older docs/community posts predate these.
- Version: query over the wire (`^^version`, or in druid).
- **Update**: `druid firmware` (checks GitHub releases, enters DFU, flashes) — see
  `druid-and-protocol.md` for the manual DFU path.
- From norns: `;install https://github.com/monome/fledge` then its update flow.

## 7. Recovery procedures

| Problem | Fix |
|---|---|
| Script crashes at boot / inputs unresponsive | `crow.reset()` from the REPL; norns: `crow.reset()` + `crow.clear()` |
| Crashing script blocks everything | `^^clearscript` (`^^c`) — erases the flash script without running it |
| druid can't find the device | Force the **bootloader**: jumper the centre pin to GND on the ii/i2c header, power-cycle; then run `osx_linux-erase_userscript.command` / `windows-erase_userscript.bat` from the firmware zip, or flash manually |
| Manual flash | `dfu-util -a 0 -s 0x08020000 -R -D crow.bin -d ,0483:df11` |
| Windows 7 driver issues | Zadig → "USB Serial (CDC)" driver; Windows 10/11, macOS, Linux need none |
| Linux permission denied on `/dev/ttyACM0` | add user to the `dialout` (or `uucp`) group |

## 8. Electrical / patching notes

- All jacks accept and output [−5, +10] V; **16-bit** resolution ≈ 0.3 mV/step.
- Inputs are high-impedance CV inputs (suitable for gates, envelopes, LFOs, and — with `freq`/
  `volume`/`peak` modes — audio analysis at reduced rate).
- 1V/oct pitch tracking: 0 V = middle C (261.63 Hz; monome docs call it "C3"), 1.0 V = one octave.
- The ii bus is 3.3 V logic, **open-drain with pullups** (crow can supply pullups —
  `ii.pullup(state)`, default on; see `ii-overview.md`).
