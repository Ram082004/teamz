# Settings

Everything reachable from the **Settings** menu (internal id `preferences`), plus every main-window
preference stored on disk. Menu shape: `include/ui/menumodel.h:169-276`. Handlers:
`src/ui/mainapplication.cpp:1442-1722`.

There is **no separate Settings dialog/window**: every option is a row inside the Settings
dropdown, applied the moment it is clicked. The only dialogs reached from it are the PCB theme
editor, the delete-theme confirmation and the language-restart confirmation.

## Where settings are saved

All keys below are in one INI file: `QStandardPaths::AppDataLocation + "/settings.ini"`
(e.g. `src/ui/apptheme.cpp:23`, `include/ui/pcbanimsettings.h:39-44`, `include/ui/pointerinput.h:242-247`).

| Key | Type / values | Default | Written by | Code |
|---|---|---|---|---|
| `theme/darkMode` | bool | false (light) | Title-bar theme button | `src/ui/apptheme.cpp:24`, read `src/ui/mainapplication.cpp:535` |
| `tree/fontPx` | int 10–24 | 13 | File Tree Text Size, Ctrl+wheel, Ctrl+=/-/0 | `src/ui/mainapplication.cpp:3022`, read `:2681` |
| `pcb/animations` | bool | true | PCB Viewer > Animations (master) | `include/ui/pcbanimsettings.h:49` |
| `pcb/animNetPulse` | bool | **false** | Net & component highlight blink | `:50`, default `:72-75` |
| `pcb/animTraceReveal` | bool | true | Solution trace reveal | `:51` |
| `pcb/animSolutionGlow` | bool | true | Solution glow & signal particles | `:52` |
| `pcb/animSpotlightDim` | bool | true | Dim other nets when one is focused | `:53` |
| `pcb/animRffeBlink` | bool | true | RFFE blink | `:54` |
| `pcb/animChrome` | bool | true | Panel & toolbar animations | `:55` |
| `pcb/animFlyTo` | bool | true | Smooth jump to search results & net steps | `:56` |
| `pcb/hoverInfo` | bool | true | Show info card on hover | `include/ui/pcbanimsettings.h:148`, `:170` |
| `pcb/panMode` | `floating` / `standard` | floating | PCB Viewer > Panning | `include/ui/pointerinput.h:240-282` |
| `pdf/panMode` | `floating` / `standard` | floating | PDF Viewer > Panning | same |
| `pdf/animFlyTo` | bool | true | Smooth jump to search results & pages | `include/ui/pcbanimsettings.h:191`, `:212` |
| `app/language` | `ru` `ar` `th` `ta` `fa` `id`; **absent = English** | absent | Language | `src/ui/mainapplication.cpp:1626-1638` |
| `app/languageNoticePending` | language code | absent | Language (marker for the welcome note) | `:1645-1649`, `:7324-7360` |
| `login/auto_login` | bool | true | Profile card "Auto login" switch (also the login window) | `src/ui/mainapplication.cpp:1863-1874` |

Read-only tuning keys also exist (`viewer/pinch_zoom_gain`, `viewer/touchpad_zoom_speed`,
`viewer/fling_friction`, `include/ui/pointerinput.h:112-173`); nothing in the menu writes them.

**PCB themes** are not in `settings.ini`. Built-ins load from `config/pcb_themes_builtin.json`,
user themes from `config/pcb_themes.json` next to the exe (`src/viewers/pcb_new/qt/PCBNewViewerWidget.cpp:219-234`).
Whether the *selected* theme is remembered between runs: **(not confirmed in code)**.

The sidebar width is **not** saved ([treeview.md](treeview.md) §10).

---

## Settings > PCB Theme ▸

All PCB-theme rows act on the **last active PCB tab** (`getActivePCB`,
`src/ui/mainapplication.cpp:1431-1436`). Lists are rebuilt each time Settings opens (`:1442-1482`).

### Apply Theme ▸
1. User opens Settings > PCB Theme > Apply Theme.
2. No PCB tab → one disabled row "No PCB file open". No themes → "No themes available".
3. Otherwise one radio row per theme with the theme's colour dot; the active theme is ticked
   (`:1452-1462`).
4. Click a theme → `pcbV->applyThemeByName(name)` re-colours that board (`:1692-1694`).

### Create New Theme…
1. Click → no PCB tab: card *Notice / "PCB Theme"*: "Please open a PCB file first." (`:1685-1690`).
2. Otherwise `pcbV->createNewTheme()` opens the PCB theme editor (`:1691`).

### Edit Theme ▸
- Lists **every** theme (built-ins too — edits to a built-in are saved to the user file as an
  override) (`:1469-1472`). Click → `openThemeEditor(name)` (`:1695-1697`).
- No PCB tab → "No PCB file open"; no themes → "No themes available".

### Delete Theme ▸
- Lists **user** themes only; built-ins cannot be deleted (`:1464-1474`). Empty → "No custom themes".
- Click → confirm card heading **"Delete Theme"**, title **"Delete theme?"**, body
  "Delete theme "<name>"? This cannot be undone.", buttons **Delete / Cancel** → on Delete,
  `deleteThemeByName(name)` (`:1698-1708`).

## Settings > File Tree Text Size ▸

Rebuilt on every open (`src/ui/mainapplication.cpp:1484-1507`):

| Row | Caption | Action | Disabled when |
|---|---|---|---|
| "Current: N px" | heading | – | – |
| Increase | Ctrl + | +1 px | at 24 px |
| Decrease | Ctrl - | −1 px | at 10 px |
| Reset to Default (13 px) | Ctrl 0 | back to 13 px | already 13 px |

Each change re-lays out the tree immediately and saves `tree/fontPx` (`:3008-3026`).
The Ctrl shortcuts themselves only work while the tree has focus; these menu rows work anywhere.

## Settings > PCB Viewer ▸

| Section | Row | Kind | Effect |
|---|---|---|---|
| DISPLAY | Show info card on hover | tick | Hover card on boards on/off; every open PCB tab repaints at once (`src/ui/mainapplication.cpp:1574-1582`) |
| PANNING | Floating — keeps gliding after you let go | radio | Drag ends with a glide |
| | Standard — stops when you let go | radio | Drag stops dead (`:1583-1593`) |
| ANIMATION & BLINKING | Animations | tick (master) | Off greys out the seven rows below but keeps their own ticks (`:1598-1611`) |
| | Net & component highlight blink | tick | default off |
| | Solution trace reveal | tick | |
| | Solution glow & signal particles | tick | |
| | Dim other nets when one is focused | tick | |
| | RFFE blink | tick | |
| | Panel & toolbar animations | tick | |
| | Smooth jump to search results & net steps | tick | |

Every animation change is pushed to **all** open PCB tabs (`applyAnimationSettings`, `:1613-1620`).
Ticks are re-seeded from stored values each time the menu opens (`:1509-1536`).

## Settings > PDF Viewer ▸

| Section | Row | Kind | Effect |
|---|---|---|---|
| PANNING | Floating — keeps gliding after you let go | radio | `pdf/panMode = floating` |
| | Standard — stops when you let go | radio | `pdf/panMode = standard` |
| ANIMATION | Smooth jump to search results & pages | tick | `pdf/animFlyTo` (`src/ui/mainapplication.cpp:1594-1597`) |

Takes effect on the next drag / jump in any PDF tab.

## Settings > Language ▸

Rows: English, Русский, العربية, ไทย, தமிழ், فارسی, Bahasa Indonesia (radio group; names always in
their own script), separator, **Suggest a wording fix** (`include/ui/menumodel.h:261-275`).
The current language is ticked from `app/language` on open (`src/ui/mainapplication.cpp:1540-1549`).

### Flow: change language (`src/ui/mainapplication.cpp:1621-1673`)
1. User clicks a language.
2. Same as current → nothing.
3. Saves `app/language` (English **removes** the key) and `app/languageNoticePending` (English removes it).
4. If a phone is being flashed → card *Notice / "Restart needed"*: "The new language applies the
   next time Schemax Schematics starts — a phone is being flashed now, so it cannot restart."
5. Otherwise confirm card heading **"Language changed"**, title **"Restart needed"**, body "The app
   needs to restart to switch languages. Restart now?", buttons **Restart Now / Later**.
6. **Restart Now** → starts a new copy with `--w2r-restart` and closes this one (`:1668-1670`).
   **Later** → applies on next start.
7. On the first start in the new language, a success card heading "Language changed", title
   "Dear customer", body "The app is now in <language>. If any word looks wrong ...", buttons
   OK / **Suggest a Correction** (`:7324-7360`).

### Suggest a wording fix (also Help > Language help & wording fixes)
Opens WhatsApp (`api.whatsapp.com/send/?phone=917708684274`) with an English template: app
language, Screen, Wording shown, Suggested wording, version and username (`:7366-7399`).

---

## Other preferences outside the Settings menu
- **Theme** (light/dark): title-bar button — see [titlebar.md](titlebar.md#theme-toggle-sun--moon).
- **Auto login**: profile card switch — see [profile.md](profile.md).
