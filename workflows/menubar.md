# Menu bar (MenuNavBar)

The 34 px row under the title bar. Class `MenuNavBar` (`src/ui/menunavbar.cpp`). The menu
content is plain data in `MenuModel::makeMenuDefs()` (`include/ui/menumodel.h:149-308`).
The bar performs nothing itself: every click emits `itemTriggered(id)` and
`MainApplication::setupMenuBar()` routes the id (`src/ui/mainapplication.cpp:1421-1745`).

Left to right: **File ▾, Settings ▾, Help ▾, Tools ▾** (Tools sweeps a strip open inline), a
stretch, then the **PDF | PCB / PDF | PDF** pair switch and a hidden **Swap** pill
(`src/ui/menunavbar.cpp:189-237`, `:263-295`).

## How the menus behave (all menus)

| Interaction | Result | Code |
|---|---|---|
| Click a title | Opens its dropdown panel; click again closes it | `src/ui/menunavbar.cpp:347-362` |
| Hover another title while a panel is open | Switches to that menu (but hovering **Tools** does nothing) | `:467-478` |
| Click outside | Closes the panel | `:480-500` |
| Alt held | Underlines the first letter of each English title | `:118-125`, `:590-597` |
| Alt+F / Alt+S / Alt+H / Alt+T | Opens File / Settings / Help, or toggles the Tools strip, with the first row selected | `:607-645` |
| Tooltip on a title | e.g. "File  (Alt+F)" | `:198-202` |
| ↑ / ↓ / Home / End | Move the highlighted row | `src/ui/menupanel.cpp:525-536` |
| → | Open the highlighted submenu, else go to the next menu | `src/ui/menupanel.cpp:537-545` |
| ← | Close one submenu level, else go to the previous menu | `:546-553` |
| Esc | Close one submenu level, else close the panel | `:554-563` |
| Enter / Space | Activate the highlighted row | `:564-569` |

- Shortcut texts shown in rows (e.g. "Ctrl +") are **captions only**; the menu registers no
  accelerators (`include/ui/menumodel.h:18-20`).
- Clicking a tick/radio row flips it **before** the app reacts, and the panel stays open for
  tick rows (`src/ui/menunavbar.cpp:506-524`, `src/ui/mainapplication.cpp:1510-1513`).
- When a translated UI language is active, titles lose the underline but keep the English Alt
  letter (`src/ui/menunavbar.cpp:190-200`).

---

## File

| Row | Icon | Shortcut | What happens | Enabled when |
|---|---|---|---|---|
| Logout | Logout | – | Logout flow below | Always |
| ─ separator ─ | | | | |
| Exit | Power | – | `animateClose()` — same as the title-bar X (`src/ui/mainapplication.cpp:1555-1556`) | Always |

Defined at `include/ui/menumodel.h:155-159`.

### File > Logout
1. `onLogoutClicked()` (`src/ui/mainapplication.cpp:8559-8598`).
2. If a phone is being flashed → card *Notice / "A phone is being flashed"* and stop (`:8561`).
3. Confirm card: heading **"Logout Confirmation"**, title **"Log out?"**, body "Are you sure you
   want to logout?", buttons **Logout** / **Cancel** (`:8562-8566`).
4. On **Logout**:
   1. Remove the website sign-in from the Home tab and reset Home sign-in state (`:8569-8570`).
   2. Close every PDF tab and every PCB tab (`:8573-8580`).
   3. Delete the saved auth token from Windows Credential Manager for this username (`:8584`).
   4. Clear the in-memory login password (`:8589`).
   5. Emit `logoutRequested()`; another handler also clears the Home page's localStorage /
      sessionStorage and the Panic Logger result cache (`:656-686`).
   6. Close the main window (`:8596`).
5. MainWindow (`endSession`) then: waits if a phone is flashing or a download must unwind,
   frees all in-memory files, and shows the login window again (`src/ui/mainwindow.cpp:2625-2694`).

### File > Exit
- Fades and closes the window; refused while a phone is being flashed (`src/ui/mainapplication.cpp:1332-1366`).
  No confirmation.

---

## Settings

Shown as "Settings"; internal id is still `preferences` (`include/ui/menumodel.h:167-169`).
Full description of every option and storage key: **[settings.md](settings.md)**.

| Row | Type | Shortcut | Enabled when |
|---|---|---|---|
| **PCB Theme ▸** | submenu | | |
| · Apply Theme ▸ | radio list of the active PCB tab's themes, each with a colour swatch | | Rebuilt on open; shows disabled "No PCB file open" if no PCB tab (`src/ui/mainapplication.cpp:1447-1450`) |
| · Create New Theme… | command | | Always clickable; without a PCB tab shows card *Notice / "PCB Theme"*: "Please open a PCB file first." (`:1685-1691`) |
| · Edit Theme ▸ | list of every theme | | Needs a PCB tab |
| · Delete Theme ▸ | list of user themes only | | Needs a PCB tab; "No custom themes" when none |
| **File Tree Text Size ▸** | submenu | | |
| · "Current: N px" | heading | | – |
| · Increase | command | "Ctrl +" | Disabled at 24 px (`:1491`) |
| · Decrease | command | "Ctrl -" | Disabled at 10 px (`:1495`) |
| · Reset to Default (13 px) | command | "Ctrl 0" | Disabled when already 13 px (`:1501`) |
| **PCB Viewer ▸** | submenu of ticks / radios | | Sub-animation rows greyed while "Animations" is off (`:1516-1522`) |
| **PDF Viewer ▸** | submenu of radios / tick | | Always |
| **Language ▸** | radio list + "Suggest a wording fix" | | Always |

Every time the Settings panel opens, the app refreshes the dynamic parts first via the
`aboutToOpen("preferences")` signal (`src/ui/mainapplication.cpp:1442-1550`).

---

## Help

| Row | What happens | Code |
|---|---|---|
| About | Opens `AboutDialog` (window title "About Schemax Schematics"), version pill "Version x.y.z" with the full version; Quick Support tiles Website, WhatsApp, Instagram, YouTube, Report Bug | `src/ui/mainapplication.cpp:8546-8557`, `src/ui/aboutdialog.cpp:72`, `:236-246` |
| Schemax Schematics Website | Opens `https://schemaxsolutions.com` in the default browser | `src/ui/mainapplication.cpp:1562-1564` |
| Check my connection | Opens the modal `NetworkCheckDialog` (window title "Connection check") | `src/ui/mainapplication.cpp:1565-1567`, `src/ui/networkcheckdialog.cpp:427` |
| Language help & wording fixes | Opens a pre-filled WhatsApp chat (see Language in settings.md) | `src/ui/mainapplication.cpp:1559-1561`, `:7366-7399` |

All Help rows are always enabled. Defined at `include/ui/menumodel.h:278-290`.

---

## Tools (inline strip, no dropdown)

Clicking **Tools** (or Alt+T) sweeps a `ToolsStrip` open to the right of the title; clicking
again collapses it. Opening any dropdown collapses it (`src/ui/menunavbar.cpp:364-376`, `:388-391`).
The chevron flips ▴ while open (`:599-605`).

| Chip | Tooltip | What happens | Code |
|---|---|---|---|
| Chip Finder | "Browse CPU, storage and board-part databases" | Opens (or focuses) a "Chip Finder" tab in the PDF lane; only one instance | `src/ui/toolsstrip.cpp:33-36`, `src/ui/mainapplication.cpp:8601-8640` |
| RFFE | "Scan a phone's RFFE bus over a COM-port probe" | Toggles the RFFE dock on the right edge of the window | `src/ui/toolsstrip.cpp:38-41`, `src/ui/mainapplication.cpp:1680-1684` |
| UART Terminal | "Serial monitor for any COM port" | Opens (or focuses) a "UART Terminal" tab | `src/ui/toolsstrip.cpp:43-46`, `src/ui/mainapplication.cpp:8644-8673` |
| Apple | "Panic Logger — read iPhone panic / crash logs" | Opens (or focuses) a "Panic Logger" tab | `src/ui/toolsstrip.cpp:51-54`, `src/ui/mainapplication.cpp:8676-8705` |

Each chip collapses the strip after it fires (`src/ui/menunavbar.cpp:216-232`). Tool tabs do not
count toward the 10-PDF-tab limit (`src/ui/mainapplication.cpp:8618-8621`).

---

## PDF | PCB and PDF | PDF (pair switch) + Swap

One two-segment track (`PairSwitch`) at the right end of the bar (`src/ui/menunavbar.cpp:263-295`).

| Segment | Label | Tooltip | Needs |
|---|---|---|---|
| 0 | PDF \| PCB | "Split view: show PDF and PCB side by side" | ≥ 1 real PDF tab **and** ≥ 1 PCB tab |
| 1 | PDF \| PDF | "Compare PDFs: show two PDF documents side by side" | ≥ 2 real PDF tabs |

"Real" PDF tabs exclude the Home tab (`countExcludingHome`, `src/ui/mainapplication.cpp:1761`).

### Availability (`updatePairSwitchAvailability`, `src/ui/mainapplication.cpp:1756-1789`)
Runs at start, whenever tab count changes and whenever the current tab changes (`:1735-1739`).

| State | Disabled-reason text (shown as a hint bubble when the greyed segment is clicked) |
|---|---|
| No PDF and no PCB | "Open a PDF and a PCB to use split view" |
| No PDF | "Open a PDF to use split view" |
| No PCB | "Open a PCB to use split view" |
| No PDF (compare) | "Open two PDFs to compare" |
| One PDF (compare) | "Open one more PDF to compare" |

A greyed segment gets no hover effect and clicking it only pops the reason
(`src/ui/pairswitch.cpp:110-139`). If a mode is on and its second document is closed, the mode
is switched off automatically (`src/ui/mainapplication.cpp:1784-1788`).

### Flow: turn split view on
1. User clicks **PDF | PCB** (enabled).
2. The segment becomes checked (clicking the checked segment again unchecks it)
   (`src/ui/pairswitch.cpp:137`).
3. **Swap** pill appears whenever a segment is checked (`src/ui/menunavbar.cpp:300-305`).
4. App clicks the tab widget's own split button (`view.splitPdfPcb`, `src/ui/mainapplication.cpp:1709-1710`).
5. The tab widget's `splitModeChanged` is reflected back onto the switch (`:1726-1728`).

Same for **PDF | PDF** → `getCompareButton()->click()` (`:1711-1712`, `:1729-1731`).

### Swap
- Tooltip "Swap the two panes". Hidden until a pair is engaged (`src/ui/menunavbar.cpp:277-284`).
- Click → swaps the panes of whichever mode is active (`src/ui/mainapplication.cpp:1713-1720`).
