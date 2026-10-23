# Druid & the wire protocol — getting code onto crow

Three routes to drive crow, from highest to lowest level:

1. **druid** — monome's official Python REPL/upload tool (what humans use).
2. **crow-mcp** — an MCP server exposing crow as agent tools over the *same* serial protocol
   (experimental; see §4).
3. **Your own serial client** — the wire protocol is simple and documented here (§3).

All three are interchangeable clients of the same USB CDC serial port — but **only one process can
hold the port at a time**.

---

## 1. druid

### Install

Requires **Python 3.6+** (docs suggest 3.11).

```sh
# macOS / Linux
pip3 install monome-druid        # fallback: sudo pip3 install monome-druid
pip3 install --upgrade monome-druid   # update

# Windows (PowerShell)
python -m pip install pyserial asyncio prompt_toolkit
python -m pip install --upgrade setuptools
python -m pip install monome-druid
```

PATH fixups are described on the [druid page](https://monome.org/docs/crow/druid/). On Windows 7 a
Zadig "USB Serial (CDC)" driver may be needed; Windows 10/11, macOS, and Linux (user in `dialout`)
work out of the box.

### Run

```sh
druid        # shows <crow connected> / <crow disconnected>
```

The module must be powered and USB-connected. druid is a TUI (prompt-toolkit based, pins
`prompt-toolkit<3`) — **driving it from a subprocess is fragile**; for automation prefer crow-mcp
or a raw serial client.

### REPL commands (type `h` to list)

| command | meaning |
|---|---|
| `r filename.lua` | **run** the file now (RAM only — lost on power cycle) |
| `u filename.lua` | **upload** to crow's flash (persists; auto-runs at power-on) |
| `p` | print the currently stored script |
| `h` | help |
| `q` | quit |

- Any Lua expression can be typed live at the `>` prompt: `crow.reset()`, `output[1].volts = 1`,
  `print(input[2].volts)`, `ii.help()`. Up-arrow recalls history.
- After a failed `r`, wait and retry; if it sticks: type `^^kill` (restart the Lua env), then `r`.

### Firmware update & tools

- `druid firmware` — checks GitHub releases, enters DFU, flashes. Check the running version with
  `^^version`.
- **web-druid** (community browser REPL + editor, Chromium Web Serial; no firmware updates):
  clone `dessertplanet/web-druid`, `npm install`, `python3 -m http.server 8000`, open
  localhost:8000.
- **Editor integration**: druid listens on **websockets port 6666** — text lines sent there are
  forwarded to crow (e.g. vim + websocat). Handy for live-coding setups.

---

## 2. Persistence model

- crow stores **one** script in flash; it auto-runs at power-on.
- `r` / `^^e` = transient run (RAM); `u` / `^^w` = flash write (persist). `p` / `^^p` prints the
  **flash** copy — not whatever you last ran from RAM.
- Factory default script is **First** (generative sequencer).

---

## 3. The `^^` wire protocol

crow enumerates as a USB CDC serial port: **VID:PID = 0483:5740**, product string
"crow: telephone line" (macOS/Linux require both markers; Windows matches the VID:PID). Baud rate
is ignored (CDC); 115200 is the nominal value.

### Framing

- **host → crow**: lines terminated `\r\n` (CRLF).
- **crow → host**: messages terminated `\n\r` (split the stream on that sequence; buffer partials).
- **USB ZLP workaround**: any write whose byte length is an exact multiple of 64 must get a
  trailing `\n` appended or the link stalls/dies on macOS/Windows (same as druid's `writebin`).
- Decode incoming bytes with an **incremental UTF-8 decoder** (`errors='replace'`) — multibyte
  characters split across USB packets otherwise corrupt the stream (a known druid crash).
- crow messages are Lua chunks prefixed `^^`; parse by splitting on `^^`, then `name(args)` with
  args split on commas (druid does no smart quoting — `^^tell("a,b",3)` yields `['"a','b"','3']`).

### Commands (host → crow)

Parsed straight from USB even if the Lua environment is dead (except `^^s`/`^^e`/`^^w`, which need
the env — use `^^clearscript` first). **Only the first character after `^^` matters** (send 3
chars, e.g. `^^s`).

| full command | short | effect |
|---|---|---|
| `^^startscript` | `^^s` | enter reception mode; following lines buffered as the script |
| `^^executescript` | `^^e` | restart env; buffer error-checked + **run (not saved)** |
| `^^writescript` | `^^w` | restart env; error-checked, **written to flash**, run |
| `^^clearscript` | `^^c` | clear the saved user script (unstick a crashing script) |
| `^^First` | — | restart, set and run the factory `First` script |
| `^^printscript` | `^^p` | print the flash script over USB |
| `^^bootloader` | `^^b` | jump to the DFU bootloader |
| `^^reset` | `^^r` | reboot crow (**USB re-enumerates — reconnect after ~2 s**) |
| `^^kill` | `^^k` | restart the Lua env **without** running the user script |
| `^^identity` | — | → `^^identity('0x…')` serial number |
| `^^version` | — | → `^^version('x.y.z')` firmware version |

### Upload framing & pacing (druid-compatible)

```
^^s            (enter script reception)
<0.2 s pause>
<script line>  (one line per ~1 ms)
...
<0.1 s pause>
^^e            run from RAM        (or ^^w: flash + run)
```

- Whole-script limit: **16380 bytes** practical (FAQ: "16kb ≈ 700 lines"); REPL chunks must stay
  **< 2048 bytes** (crow replies `!chunk too long!`). Long lines should be split.
- Success is acked with **`^^ready()`** — crow sends it after a successful `init()`. If instead you
  see error text, the script did not run; fix and re-upload.
- A ``` ``` ```-style multiline helper exists for ≤ 2 kB blocks (dev readme).

### Messages (crow → host)

| message | meaning |
|---|---|
| `^^ready()` | script init completed |
| `^^stream(ch, volts)` / `^^change(ch, state)` / `^^window(ch, win, dir)` / `^^scale(ch, note)` / `^^volume(ch, val)` / `^^peak(ch, val)` / `^^freq(ch, hz)` | input-mode events (also `^^output(ch, volts)` from `output[n].query()`) |
| `^^ii.<name>(...)` | ii query replies **when a Lua `.event` handler forwards them** (current firmware does not auto-forward — install a handler, see `ii-overview.md`) |
| `^^pub …` / `^^pupdate …` | `public` discovery/sync traffic |
| `^^version('x.y.z')` / `^^identity('0x…')` | solicited responses |
| plain lines | `print()` output; also Lua errors: `syntax error`, `runtime error`, `evaluation failed`, `!chunk too long!`, `CPU timed out` (some paths arrive backslash-prefixed) |

### Error handling

- A Lua error in one REPL line does **not** stop later lines — check *all* returned output.
- Errors arrive as plain text lines matching the markers above (sometimes backslash-prefixed).
- `CPU timed out.` = the watchdog (see `technical.md`): your loop didn't yield for ~2 s.

### Writing your own client (essentials)

```
discover:   serial port whose hwid contains "0483:5740"
            (non-Windows: also require product "crow: telephone line")
open:       115200 8N1, timeout ~50 ms poll
read loop:  incremental UTF-8 decode; split on "\n\r"; classify lines:
            "^^" -> event(name, args); backslash-prefixed -> error; else print text
write:      line + "\r\n"; if len % 64 == 0 append one more "\n"
upload:     ^^s; sleep 0.2; lines at 1 ms; sleep 0.1; ^^e|^^w; wait for ^^ready()
```

A complete, tested implementation of exactly this is the `CrowDevice` class in the crow-mcp
project (`src/crow_mcp/crow.py`) — including the ring buffers, cursor polling, and upload ack
handling.

---

## 4. crow-mcp (agent-facing MCP server)

An experimental MCP server that exposes crow to coding agents as tools — live-coding via the REPL,
full-script upload (RAM/flash), event/output polling, and an embedded API cheat sheet. It speaks
the protocol above byte-for-byte like druid (same discovery, pacing, framing) so druid and crow-mcp
are interchangeable — but **the server owns the port** while running (close druid first).

Location (this user's machine): `C:\Users\jonathan\Desktop\other projects\mcp-dev\crow-mcp`.
Run with [uv](https://docs.astral.sh/uv/), Python ≥ 3.10:

```sh
cd crow-mcp
uv sync          # .venv + mcp + pyserial + pytest
uv run crow-mcp  # MCP server on stdio
```

Client config (opencode `opencode.json`, or any `mcpServers` JSON):

```json
{
  "mcp": {
    "crow": {
      "command": "uv",
      "args": ["--directory", "C:/Users/jonathan/Desktop/other projects/mcp-dev/crow-mcp", "run", "crow-mcp"]
    }
  }
}
```

### Tools

| Tool | What it does |
|---|---|
| `crow_list_ports` | List serial ports, tag crows (works without connecting) |
| `crow_connect` / `crow_disconnect` | Open/close the link (auto-discovery or explicit port) |
| `crow_status` | Connection, buffers, latest streamed CV, counters |
| `crow_execute` | **Live-coding workhorse**: run Lua on the REPL, collect output + events |
| `crow_upload` | Install a full script — `mode: "ram"` (run now) or `"flash"` (persist + run) |
| `crow_read_events` | Cursor-poll `^^stream`/`^^change`/custom `tell()` events (pass back `next_seq`) |
| `crow_read_output` | Cursor-poll `print()` output and errors |
| `crow_version` / `crow_identity` | `^^v` / `^^i` queries |
| `crow_print_script` | Print the FLASH copy (`^^p`) |
| `crow_clear_script` | Erase the flash script (`^^c`, destructive) |
| `crow_restart_lua` | Restart the Lua VM without the user script (`^^k`) |
| `crow_reboot` | Reboot (`^^r` — USB re-enumerates; reconnect after ~2 s) |
| `crow_api_help` | Embedded crow Lua API cheat sheet (12 topics, keyword search) |

Resource: `crow://status` — live status dict.

### Realtime workflow (as an agent)

1. `crow_connect` — auto-finds the module.
2. `crow_api_help` (`topic: "overview"`, then as needed) — or use this skill's references.
3. `crow_execute` to live-code. Iterate here; watch `errors` in the result.
4. Poll `crow_read_events` (CV telemetry, `tell()` customs) and `crow_read_output` (`print()`).
   **Values only reach you via `print()` / `tell()`** — nothing is reported automatically.
5. When it behaves: `crow_upload` with `mode: "flash"` so it survives power-cycling.

### Hardware-verified gotchas (from building/running crow-mcp)

- **Read-back race**: `output[n].volts = 4.0` then `print(output[n].volts)` on the next line prints
  the **old** value — reads lag one io cycle. Read again in a later call. (Not a slew artifact:
  default slew is 0.)
- **ii `get()` replies need a Lua `.event` handler** — with no handler, queries vanish silently
  (no `^^ii.*` forwarding on current firmware). Install a handler that `print()`s to see replies.
- **Unknown getter names raise a local Lua error** (`getter not found for 'x'`) even with no
  device attached — crow validates names against its ii tables.
- **W/Del never answers `get()`** at all — silence does not mean the device is absent (see
  `ii-wslash.md`).
- **Errors don't stop the REPL**: one bad line doesn't prevent later lines from running.
- **`!chunk too long!` / rejected uploads**: keep REPL chunks < 2000 bytes, scripts < 16380 bytes.

### Troubleshooting

- **No crow found** — check cable/port; `crow_list_ports` should show `USB VID:PID=0483:5740`.
  Win7 may need the Zadig driver.
- **Could not open port** — another app holds it (druid, maiden, a stale server). One process only.
- **Connection died after `crow_reboot`** — expected; reconnect.
- **Garbled characters** — should not happen (incremental UTF-8 decode); if it does, another
  client is mangling the stream.

---

## 5. Sources

- druid page: https://monome.org/docs/crow/druid/ · tutorial: https://monome.org/docs/crow/scripting-druid/
- technical (protocol table): https://monome.org/docs/crow/technical/
- dev readme (framing, limits): https://github.com/monome/crow/blob/main/readme-development.md
- manual update / recovery: https://monome.org/docs/crow/manual_update/
- web-druid: https://github.com/dessertplanet/web-druid
- crow-mcp (this user's project): `C:\Users\jonathan\Desktop\other projects\mcp-dev\crow-mcp`
