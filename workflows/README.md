# Schemax main window — workflow documentation

Source of truth: `E:\schemax\tool-script` (Qt 6 / C++). Every citation below is `path:line`
relative to that folder. Only behaviour visible in the code is documented; anything that could
not be confirmed is marked **(not confirmed in code)**.

## Documents

| File | Covers |
|---|---|
| [titlebar.md](titlebar.md) | Title bar (IdentityBar): logo/wordmark, days-left pill, trial allowance pill, theme button, profile chip, window controls, drag / snap / maximize |
| [menubar.md](menubar.md) | Menu bar (MenuNavBar): File, Settings, Help, Tools strip, PDF \| PCB / PDF \| PDF pair switch, Swap, Alt shortcuts |
| [treeview.md](treeview.md) | File tree sidebar: data source, lazy loading, sorting, icons, green dot, search, opening files, download queue, limits, resize/collapse |
| [settings.md](settings.md) | Everything under Settings + every `settings.ini` key the main window reads/writes |
| [profile.md](profile.md) | Account: sign-in state, plan, days left, trial allowance, Buy Premium (Razorpay vs WhatsApp), activation key, plan re-check, logout, HWID |
| [home-notice-plans.md](home-notice-plans.md) | The Home tab (embedded website dashboard): notice ticker, plans carousel, offer banner, WhatsApp button — and what the app does around them |

## Important facts that apply everywhere

1. **The code paints "Schemax Schematics", not "W2R Schematics".** The title bar wordmark is
   hard-coded to `Schemax` + `Schematics` (`src/ui/identitybar.cpp:747`, `:753`) and the window
   title is `Schemax Schematics - <full name>` (`src/ui/mainapplication.cpp:8997`). If a
   screenshot shows "W2R Schematics", it came from a different build or a different branch
   **(not confirmed in code)**.
2. **The status bar is hidden.** `setupStatusBar()` hides it and sizes it to 0×0
   (`src/ui/mainapplication.cpp:2445-2455`). Every `statusBar()->showMessage(...)` call in the
   main window is therefore **never seen by the user**. These messages are listed in the docs
   only as internal notes.
3. **All pop-up cards are the "glass" Notify card.** `Notify::show` routes every notice/error/
   confirm to `GlassNotice::show` (`src/ui/notify.cpp:33-38`). A card has a small **heading**
   (eyebrow) and a bigger **title**. Docs quote both as *heading / title*.
   `showNoticeDialog(message, title)` uses the heading **"Notice"** (`src/ui/notify.cpp:60-63`).
4. **The right-hand panel (notice ticker, plans, offer banner, WhatsApp button) is a web page.**
   It is the website page `/user/app_dashboard.php` loaded into a WebView2 "Home" tab
   (`src/ui/mainapplication.cpp:294-297`, `:6738-6858`). Its HTML is not in this repository,
   so its inner behaviour cannot be confirmed from the app code.
5. **Settings file.** Unless stated otherwise every preference lives in
   `%APPDATA%\<org>\<app>\settings.ini` = `QStandardPaths::AppDataLocation + "/settings.ini"`
   (INI format), e.g. `src/ui/apptheme.cpp:23`. Full key list in [settings.md](settings.md).

## Window layout map

```
+----------------------------------------------------------------------------------------------+
| IdentityBar (48px)                                                         titlebar.md       |
| [W2R logo] Schemax Schematics | WORLD'S FASTEST SCHEMATICS                                   |
|        [8/10 files left today*] [* 72 days left] [sun/moon] [(B) buddies400 / PREMIUM] _ [] X |
+----------------------------------------------------------------------------------------------+
| MenuNavBar (34px)                                                          menubar.md        |
| File v  Settings v  Help v  Tools v [Chip Finder|RFFE|UART Terminal|Apple]  [PDF|PCB|PDF|PDF] [Swap*] |
+------------------------------+---------------------------------------------------------------+
| CollapsibleSidebar (350px)  >| Tab area (DualTabWidget)                                      |
| treeview.md                 <| [Home(house icon)] [PDF tabs ...]   /  [PCB tabs ...]          |
| +--------------------------+ |                                                               |
| | (search) Search files... | |  Home tab = WebView2 -> website /user/app_dashboard.php       |
| +--------------------------+ |  home-notice-plans.md                                         |
| | [VC++ error banner*]     | |   +-----------------------------------------------------+    |
| | > 1. Mobile Phones       | |   | NOTICE  >> INFO ... INFO ... (ticker)               |    |
| | > 2. ...                 | |   | Premium plans / Extend your access (carousel)       |    |
| | > 11. Testing  (green .) | |   |  [Buy Rs600] [WhatsApp]  INR|USD  3 MONTHS..3 USERS |    |
| | > Free - Trail Users     | |   |  secure checkout / Razorpay   My Purchases          |    |
| |                          | |   | Diwali offer banner                                 |    |
| |                          | |   |                     [Need a key? WhatsApp] (float)  |    |
| +--------------------------+ |   +-----------------------------------------------------+    |
|  notch blade (collapse) / right-edge resize grip                     [RFFE dock, hidden*]    |
+------------------------------+---------------------------------------------------------------+
  * = only shown in some states.  Status bar exists but is hidden (0x0).
```

Code anchors for the map:

- Header = `AppShellHeader` wrapping `IdentityBar` + `MenuNavBar` (`src/ui/mainapplication.cpp:928-935`).
- Body = `[CollapsibleSidebar | ViewerBackdrop(tab widget) | RffeDock]` in a plain `QHBoxLayout`
  (`src/ui/mainapplication.cpp:940-977`).
- When no tab at all is open, the tab widget is hidden and a branded backdrop
  ("SCHEMAX / SCHEMATICS" watermark) shows instead (`src/ui/mainapplication.cpp:1795-1805`).
  Because the Home tab is always present, this is normally not visible.

## Global keyboard shortcuts (main window)

Defined in `setupKeyboardShortcuts()` (`src/ui/mainapplication.cpp:9156-9237`):

| Keys | Action |
|---|---|
| Ctrl+T | Show/hide the file-tree sidebar |
| Ctrl+Shift+T | Show the file-tree sidebar |
| F11 | Immersive full screen on/off (only when a PDF, PCB or hardware-solution viewer is active) |
| Esc | Leave immersive full screen (does nothing otherwise) |
| Ctrl+Tab / Ctrl+Shift+Tab | Next / previous tab in the current lane (PDF or PCB) |
| Ctrl+F | Focus the tree's "Search files..." box and select its text |
| Ctrl+= or Ctrl++ / Ctrl+- / Ctrl+0 | Tree text bigger / smaller / reset — **only while the tree has focus** |
| Alt+F / Alt+S / Alt+H / Alt+T | Open File / Settings / Help, or sweep open Tools (`src/ui/menunavbar.cpp:607-645`) |
