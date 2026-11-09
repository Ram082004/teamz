# Title bar (IdentityBar)

The top 48 px row. Class `IdentityBar` (`src/ui/identitybar.cpp`), created by `AppShellHeader`
and wired by `MainApplication::setupTitleBar()` (`src/ui/mainapplication.cpp:1103-1118`).

> The old `TitleBar` / `TopMenu` / `TitleBarWidget` classes (`src/ui/titlebar.cpp`,
> `src/ui/topmenu.cpp`, `src/ui/titlebarwidget.cpp`) are retired and not built
> (`CMakeLists.txt:252`, `:336`). Ignore them.

## Elements, left to right

| # | Element | What it is | Code |
|---|---|---|---|
| 1 | Logo mark | `:/assets/icons/w2r-logo.svg`, rendered once at 30 px high, aspect kept | `src/ui/identitybar.cpp:571-583`, painted `:725-726` |
| 2 | Wordmark | "Schemax" (bold 19 px) + "Schematics" (regular, accent colour) | `src/ui/identitybar.cpp:745-754` |
| 3 | Divider + tagline | thin vertical line, then "WORLD'S FASTEST SCHEMATICS" (mono 10 px, faint) | `src/ui/identitybar.cpp:756-769` |
| 3b | DEV banner | In a DEV test build the tagline is replaced by red `BuildEnv::devBannerText()` | `src/ui/identitybar.cpp:761-767` |
| 4 | Trial allowance pill | e.g. "8/10 files left today · 48/50 files in trial" — hidden unless the server sends trial data | `src/ui/identitybar.cpp:586-587`, `:636-645` |
| 5 | Days-left pill | "N days left" with a green dot | `src/ui/identitybar.cpp:185-243`, `:620-634` |
| 6 | Theme button | sun (light) / moon (dark) in a rounded square, tooltip "Toggle theme" | `src/ui/identitybar.cpp:248-290` |
| 7 | Profile chip | avatar letter + username + plan caption, tooltip "Account" | `src/ui/identitybar.cpp:297-509` |
| 8 | Minimize / Maximize-Restore / Close | 46 px wide flush buttons, tooltips "Minimize", "Maximize / Restore", "Close" | `src/ui/identitybar.cpp:515-551`, `:590-605` |

Note: the wordmark reads **"Schemax Schematics"** in code. "W2R Schematics" is not painted
anywhere in `identitybar.cpp` **(screenshot text not confirmed in code)**.

The right-hand cluster is placed by hand from right to left: Close, Max, Min, 10 px gap,
chip, theme button, days-left pill (skipped when hidden), allowance pill (skipped when hidden)
(`src/ui/identitybar.cpp:649-674`). It re-runs on every resize and whenever identity changes
(`:681-685`, `:772-786`).

---

## Window dragging, snapping, maximize

### Drag the window
1. User presses the left mouse button on an empty part of the title bar.
2. Windows hit-test: `nativeEvent` answers `HTCAPTION` for any point inside the IdentityBar that
   is not a button (`src/ui/mainapplication.cpp:880-895`). Buttons return `HTCLIENT` so clicks
   reach them (`:885-892`).
3. Windows moves the window natively (Aero Snap works because it is a real caption area).
   IdentityBar also has its own Qt drag fallback (`src/ui/identitybar.cpp:687-708`).

### Double-click the title bar
1. User double-clicks an empty part of the bar.
2. Because the area is `HTCAPTION`, Windows sends `WM_SYSCOMMAND / SC_MAXIMIZE`.
3. The app intercepts `SC_MAXIMIZE` and runs its own `doMaximize()` (`src/ui/mainapplication.cpp:797-805`).
4. `doMaximize()` saves the normal geometry and sets the window to the monitor's available area
   (taskbar excluded) (`src/ui/mainapplication.cpp:211-236`).

Edge case: once custom-maximized, `doMaximize()` returns immediately (`:213`). Whether a second
double-click restores the window depends on what Windows sends (SC_RESTORE is not intercepted)
**(restore-on-double-click not confirmed in code)**. Use the Maximize/Restore button to restore.

### Resize by the window edge
- An 8 px border on every side and corner returns the matching `HT*` resize code, **only when
  not maximized** (`src/ui/mainapplication.cpp:863-878`).
- Minimum size = smaller of 1200×800 and 80 % of the screen (`src/ui/mainapplication.cpp:564-567`).
- On first show after login the window is maximized (`src/ui/mainwindow.cpp:2524-2528`).

---

## Window controls

### Minimize
1. Click `_` → `IdentityBar::minimizeRequested` → `showMinimized()` (`src/ui/mainapplication.cpp:1110`).
   (An animated `animateMinimize()` exists, `:1280-1330`, but the button is wired to plain
   `showMinimized()`.)

### Maximize / Restore
1. Click the square → `toggleMaximizeRestore()` (`src/ui/mainapplication.cpp:1111`, `:198-209`).
2. If currently custom-maximized → `doRestore()` puts back the saved normal geometry (`:238-251`).
   Otherwise → `doMaximize()` (above).
3. The glyph shows two overlapping squares ("restore") while `window()->isMaximized()`
   (`src/ui/identitybar.cpp:541-546`); `updateMaximizeState()` repaints it (`:788-792`).

### Close
1. Click `X` (hover turns it red `#d01c29`, `src/ui/identitybar.cpp:45`, `:533-537`) →
   `animateClose()` (`src/ui/mainapplication.cpp:1112`).
2. If a phone is being flashed (Panic Logger) → card *Notice / "A phone is being flashed"*:
   "Wait until the flash has finished — closing now could leave the phone unusable." and the
   window stays open (`src/ui/mainapplication.cpp:1165-1171`, `:1335`).
3. Otherwise the window fades out and shrinks to 98 % over 160 ms, then really closes
   (`src/ui/mainapplication.cpp:1340-1365`).
4. `closeEvent`: if a download is in progress (`m_aws.isBusy()`), it cancels the operation and
   retries close after 100 ms (`src/ui/mainapplication.cpp:1129-1143`).
5. When the main window is destroyed and the login window is hidden, the whole app quits
   (`src/ui/mainwindow.cpp:2516-2522`).

There is **no** "are you sure?" prompt on Close.

---

## Days-left pill

### What it counts
1. `applyIdentity()` reads `m_userSession.planExpiry` (ISO-8601, with or without milliseconds)
   (`src/ui/mainapplication.cpp:7462-7473`).
2. `daysLeft = max(0, today.daysTo(expiry in local time))` — whole days from now to the expiry
   date/time, never negative.
3. If the expiry is empty or cannot be parsed, `daysLeft = -1`.
4. `IdentityBar::setIdentity(username, displayPlan(), daysLeft)` (`src/ui/identitybar.cpp:772-778`).

### What it shows
| daysLeft | Pill |
|---|---|
| -1 (no expiry, e.g. Free plan) | **hidden**, slot removed from the layout (`src/ui/identitybar.cpp:625-627`, `:660-664`) |
| 0 | "0 days left" (plural form via `%n`) |
| 1 | "1 day left" (`src/ui/identitybar.cpp:631`) |
| N | "N days left" (`:632`) |

### Colours / thresholds
- **The title-bar pill has no thresholds.** It is always the same green pill: background
  `#d5f5da` light / `#0f3118` dark, border `#a9ddb2` / `#225a31`, text `#0b5d2a` / `#b4efbf`
  (`src/ui/identitybar.cpp:43`, `:52`), with a green dot `#4BCB71` (`kPlanDot`, `:34`, `:242`).
- Urgency colours exist only in the **profile card** expiry line: ≤ 7 days = `@ERROR` (bold),
  ≤ 30 days = `@COPPER`, otherwise `@TEXT_SECONDARY` (`src/ui/profilecard.cpp:43-44`, `:239-258`).
  See [profile.md](profile.md).
- The pill is not clickable and has no tooltip **(none set in code)**.

### When it updates
- At startup (`setupTitleBar` → `applyIdentity`).
- When a token refresh returns a different plan or expiry (`updatePlan`, `src/ui/mainapplication.cpp:7423-7432`).
- When trial info changes (`setTrialAccess`, `:7434-7448`).
- It is **not** re-evaluated on a timer, so the count only changes on one of the events above.

## Trial allowance pill (next to days-left)

Shown only when the server sends `trialAccess` with `dailyLimit > 0` (`src/ui/mainapplication.cpp:7478-7500`).

| Case | Text |
|---|---|
| Daily limit only | "X/Y files left today" |
| With lifetime cap | "X/Y files left today · A/B files in trial" |

- Tooltip: "Your trial can download Y files a day (the count restarts every day at 5:30 AM IST)."
  or, for the Free plan, "Your Free plan can download Y files a day from the free folder (...)",
  plus "Your free trial includes B downloads in total; A are left." when capped.
- Dot turns **amber `#F59E0B`** when ≤ 2 left today or ≤ 5 left in the trial; otherwise green
  (`src/ui/mainapplication.cpp:7498`, `src/ui/identitybar.cpp:35`, `:643`).
- Updated after each download from the server's usage numbers (`src/ui/mainapplication.cpp:7911-7916`)
  and forced to 0 when the server says the limit is reached (`:7810-7817`).

---

## Theme toggle (sun / moon)

Icon = the **current** theme: sun while light, moon while dark (`src/ui/identitybar.cpp:283-288`).

### Flow
1. User clicks the button → `themeToggleRequested` → `MainApplication::onThemeToggled()`
   (`src/ui/mainapplication.cpp:1113`, `:8707-8783`).
2. If a switch is already running, the click is ignored (`:8710-8711`).
3. A `ThemeSwitchOverlay` appears: "Switching to dark theme…" / "Switching to light theme…" (`:8775-8779`).
4. Work runs as small queued steps, one per event-loop turn (`:8725-8769`, `:8787-8807`):
   1. `AppTheme::setDarkMode(goingDark)` — flips ThemeManager **and saves** `theme/darkMode`.
   2. Global stylesheet, tree, shell chrome.
   3. One step per open PDF/PCB tab.
   4. Solution editor.
   5. The Home web page: `WebView2Widget::setTheme(dark)` (`:8959-8970`).
   6. Final repaint of every widget.
5. Overlay finishes (`:8792-8793`).

### Persistence
- Key `theme/darkMode` (bool) in `settings.ini`, written by `AppTheme::setDarkMode`
  (`src/ui/apptheme.cpp:19-25`).
- Read at app start (default **false = light**) (`src/ui/mainapplication.cpp:533-538`) and by the
  login window (`src/ui/mainwindow.cpp:274`).
- An open profile card re-skins live (`src/ui/mainapplication.cpp:1881-1882`).

---

## Profile chip

### Look
- Avatar circle with the first letter of the username, upper-case ("?" if empty).
- Line 1: username. Line 2: plan in upper case ("STANDARD" if the plan is empty)
  (`src/ui/identitybar.cpp:620-624`).
- Plan text comes from `displayPlan()`: the server plan, except a key-based trial shows
  **"Trial key"** (`src/ui/mainapplication.cpp:7452-7458`).
- Hover: border and fill cross-fade over 160 ms, avatar grows 1 px (`src/ui/identitybar.cpp:348-355`, `:452-456`).
- A light shimmer band crosses the chip about every 4.5 s while visible (`:328-346`, `:370-380`).

### Click → there is **no dropdown menu**
1. User clicks the chip → `profileRequested` → `onProfileRequested()` (`src/ui/mainapplication.cpp:1114`).
2. A frameless, always-on-top **modal** dialog opens containing a `ProfileCard`
   (`src/ui/mainapplication.cpp:1809-1885`).
3. Card contents and actions are described in [profile.md](profile.md). In short:
   - **Subscription details**: Username (click to copy), Plan, Expiry date + "· N days left",
     Last login, **Auto login** switch.
   - **System diagnostics**: Hardware "Active", OS version, Architecture, Hostname (click to copy).
   - Close button (X).
4. The card has **no** Logout, My Purchases, Buy/Renew or account-details actions. Logout is under
   **File > Logout** ([menubar.md](menubar.md)); purchases live on the website
   ([home-notice-plans.md](home-notice-plans.md)).
