# UART Terminal — workflow

Source of truth: `E:\schemax\tool-script` (Qt 6 / C++). Citations are `path:line` relative to that folder.
Visual version of this workflow + 10 design concepts: [schemax-uart-terminal-designs.html](../schemax-uart-terminal-designs.html).

Files:
- `include/ui/uartterminalpanel.h`, `src/ui/uartterminalpanel.cpp` — the UI (tool tab).
- `include/ui/uartsession.h`, `src/ui/uartsession.cpp` — the serial side (`QSerialPort` wrapper). The UI never touches `QSerialPort` directly.

---

## 1. Opening the tool

| Entry point | Code |
|---|---|
| Menu bar → **Tools** strip → **UART Terminal** | `src/ui/mainapplication.cpp:1771-1772` (`tools.uartTerminal`) |
| Activity rail tool `"uart"` | `src/ui/mainapplication.cpp:1003` |

`MainApplication::onUartTerminalClicked()` (`src/ui/mainapplication.cpp:8526`):

1. Scans the **PDF lane** for a tab with property `isUartTerminalTab`. If found → just focus it (**single instance**).
2. Otherwise creates `UartTerminalPanel`, tags it `isUartTerminalTab = true`, adds it as tab **"UART Terminal"** in the PDF lane and focuses it.
3. The panel's header **X** emits `closeRequested` → the host closes the tab (`onTabCloseRequestedByType`).
4. Closing the tab destroys the panel → destroys `UartSession` → **port is closed**.

## 2. Panel layout (top → bottom)

```
┌ Top bar ─ [W2R mark] UART Terminal · Serial monitor ........................... [X] ┐
├ Conn bar ─ Port [COM5 · USB-SERIAL CH340 ▾][⟳]  Baud [115200▾]  Data [8] Parity [None]│
│            Stop [1]  Flow [None] ................................... [ Connect ]       │
├ View bar ─ [Text|Hex|Mixed]  Timestamps  Auto-scroll  Wrap  Echo   Find Clear Save… Log│
├ Search bar (Ctrl+F, hidden by default) ─ [Find in output…] Prev Next  "n matches"  ✕  │
├ Terminal output (QPlainTextEdit, read-only)                                            │
├ Send row ─ [Type a command and press Enter · ↑/↓ for history] [Text▾] [LF \n▾] Send   │
│            [Repeat] [1000 ms]                                                          │
└ Footer ─ ● status text ...................... DTR  RTS   ●RX 12.4 KB  ●TX 210 B  Reset┘
```
Build functions: top bar `:257`, conn bar `:290`, body `:384`, view bar `:409`, search `:490`, send row `:532`, footer `:603` (all `src/ui/uartterminalpanel.cpp`).

## 3. Port discovery

- `UartSession::availablePorts()` (`src/ui/uartsession.cpp:87`) — sorted by number (COM2 before COM10). Label: `COM5 · USB-SERIAL CH340`; tooltip shows manufacturer / serial / VID:PID.
- `refreshPorts()` (`src/ui/uartterminalpanel.cpp:665`):
  - Auto-polls every **1 s** (`m_portPoll`), plus the **⟳** button forces a rescan.
  - **Skipped while connected** (the combo is locked).
  - Rebuilds only if the list actually changed.
  - Keeps the current selection, else re-selects the **last used port** (`m_preferredPort`) when it reappears.
  - No ports → combo shows **"No serial ports found"**, disabled; **Connect disabled**.

## 4. Configure line settings

| Field | Options | Default |
|---|---|---|
| Baud | 300 … 3 000 000 (19 presets incl. **74880**) — editable, any value 50–20 000 000 | 115200 |
| Data bits | 5 / 6 / 7 / 8 | 8 |
| Parity | None / Even / Odd / Mark / Space | None |
| Stop bits | 1 / 1.5 / 2 | 1 |
| Flow | None / RTS/CTS / XON/XOFF | None |

Status/log format: `115200 8N1` (`Settings::frameSummary`, `src/ui/uartsession.cpp:31`).

**Live change:** editing any line setting while connected calls `applyLineSettingsLive()` (`:736`) → `applyLineSettings()` on the open port → system line *"Line settings changed to 115200 8N1"*. The **port name cannot change while open** (disconnect first).

## 5. Connect

`toggleConnection()` (`src/ui/uartterminalpanel.cpp:713`):

```
Connect pressed
  ├─ no port selected ──────────────► status (error): "Pick a serial port first."
  └─ UartSession::open(settings)      (src/ui/uartsession.cpp:135)
       ├─ fail ─► status + system line, by error:
       │     PermissionError  → "COM5 is in use by another program (or the RFFE tool). Close it there and try again."
       │     DeviceNotFound   → "COM5 was not found. Is the adapter still plugged in?"
       │     OpenError        → "COM5 is already open."
       │     Unsupported      → "COM5 does not support these line settings."
       │     other            → "Could not open COM5: <reason>"
       └─ ok ─► DTR/RTS applied (both OFF by default — ESP32 auto-reset safety)
                read buffer unbounded (bytes never dropped)
                emit opened() → onOpened() (:757)
                   • decoder / ANSI state reset (fresh line)
                   • system line "Connected to COM5 · 115200 8N1"
                   • button → "Disconnect" (danger style); port combo + ⟳ locked
                   • Send / Repeat enabled; status "Connected · COM5 · 115200 8N1"
                   • port remembered + settings saved
```

## 6. Receive pipeline

1. `QSerialPort::readyRead` → `UartSession::onReadyRead` (`src/ui/uartsession.cpp:229`) → counts RX → `dataReceived(bytes)`.
2. `onDataReceived` (`:814`) **queues** bytes; RX LED pulses.
3. `flushRx` (`:822`) drains the queue every **30 ms** — one document insert per frame (handles 2 Mbaud).
4. Rendered by display mode (`setDisplayMode`, `:1018`):
   - **Text** — stateful UTF-8 decode (split multi-byte chars survive), **ANSI escapes stripped**, CR / LF / CRLF = one line break.
   - **Hex** — 16 bytes per row.
   - **Mixed** — hex bytes + printable characters alongside.
   - A short hex row is flushed once the line goes idle (`m_hexIdle`).
5. Options: **Timestamps** (prefix arrival time), **Auto-scroll** (scrolling up pauses it until you return to the bottom), **Wrap**.
6. Text colours by kind: RX / TX echo / system / timestamp.

## 7. Send pipeline

`sendCurrent()` (`:1097`) — Enter or **Send**:

```
not connected ─► "Not connected. Pick a port and press Connect."
buildPayload() (:1061)
  Text → input + line ending (None / LF / CR / CR+LF, default LF)
  Hex  → "48 65 6C" or "0x48,0x65"; line ending NOT added
       invalid ─► field turns red + "Not valid hex. Use pairs of digits, e.g. 48 65 6C or 0x48,0x65."
  empty ─► "Nothing to send."
sendPayload() (:1082) → write fails ─► "Write failed."
ok ─► TX LED pulse · Echo (if on) prints what was sent · counters update
     input pushed to history (↑/↓ browse, saved to settings) · field cleared · stale error cleared
```

**Repeat** (`setRepeat`, `:1121`): captures the payload **at the moment Repeat is pressed** (editing the input later doesn't change what repeats), sends immediately, then every **N ms** (default 1000). Stops automatically on disconnect.

## 8. Line controls & counters (footer)

- **DTR / RTS** toggles — both released on open. On ESP32-style auto-reset circuits RTS holds EN low (reset) and DTR holds IO0 low (download mode). With **RTS/CTS flow**, RTS is owned by flow control.
- **RX / TX** LEDs + byte counters (`formatBytes`, `:1318`); **Reset** zeros counters.

## 9. Find, Clear, Save, Log

| Action | Shortcut | Behaviour |
|---|---|---|
| Find | Ctrl+F | Shows search bar; Enter = next, Shift+Enter = previous, Esc closes; "No matches" / "N+ matches" |
| Clear | Ctrl+L | Clears output |
| Save… | — | Save dialog (`*.txt *.log`) → "Saved to <path>" / "Could not write <path>: <err>" |
| Log to file | — | Appends everything shown as it arrives → "Logging to <path> — click to stop"; stopping → "Logging stopped · <path>" |

## 10. Disconnect / errors during a session

```
User presses Disconnect ─► close() ─► closed("") ─► "Disconnected"
Cable unplugged / I/O error on a LIVE port (src/ui/uartsession.cpp:238)
  ResourceError / DeviceNotFound ─► "COM5 was disconnected."
  Permission / Read / Write error ─► "COM5: <errorString>"
  → port closed, closed(reason) emitted
  parity / framing glitches ─► cleared, port stays open
onClosed (:775): flush pending RX · stop Repeat · system line · status error (if reason) · force port rescan
```

## 11. Persistence

`settings.ini` (AppData), group **`[UartTerminal]`** — `loadSettings` `:1427`, `saveSettings` `:1464`:

`port, baud, dataBits, parity, stopBits, flow, displayMode, timestamps, autoscroll, wrap, echo, sendFormat, lineEnding, repeatMs, history`

Defaults: 115200 8N1, no flow, Text mode, timestamps off, auto-scroll on, wrap on, echo on, Text send, LF, 1000 ms.

## 12. State machine

```
            ports appear                    Connect ok
 [No ports] ───────────► [Ready] ───────────────────────► [Connected]
     ▲  ports vanish        │  ▲  Connect fails                 │   │
     └──────────────────────┘  └──── [Error status] ◄───────────┘   │ unplug / I/O error
                               ▲                                     │
                               └──────────── Disconnect ─────────────┘
 Close tab (any state) → panel destroyed → port closed
```
