# File tree sidebar

The left panel: a `CollapsibleSidebar` (`src/ui/collapsiblesidebar.cpp`) holding the search row,
a thin search progress line, a hidden error banner, and the tree (`SmoothTreeWidget`, a
`QTreeWidget`). Built in `MainApplication::setupTreeView()` (`src/ui/mainapplication.cpp:2673-2959`).

## 1. Where the tree list comes from

The app runs in "AWS-only" mode: the tree is one JSON catalogue delivered with sign-in, never a
folder on disk (`src/ui/mainapplication.cpp:247-248`).

1. **At login** the server reply carries the catalogue (`syncFiles.treeData`, base64, maybe
   gzip). `AuthService` stores it in RAM as `treeview.json` and writes an encrypted copy to
   `%LOCALAPPDATA%\…\cache\treeview.enc` in the background (`src/network/authservice.cpp:364-407`).
2. The login request sends the last ETag (`treeDataEtag`); if the server says the tree is
   unchanged, the cached `treeview.enc` is decrypted instead (`src/network/authservice.cpp:238-243`,
   `:388-397`, `:475-489`). Older server replies (presigned URLs) are also handled (`:412-473`).
3. After login, MainWindow calls `configureAwsFromAuth` then `switchToAwsTreeview()`
   (`src/ui/mainwindow.cpp:2537-2555`) → `setTreeSource(AWS, true)` → `loadTreeFromMemory()` +
   `fetchDbSolutionKeys()` (`src/ui/mainapplication.cpp:4065-4125`).
4. `loadTreeFromMemory()` parses `treeview.json` (object, or a bare array wrapped as
   `{"items": [...]}`) into `m_inMemoryTreeCache` (`src/ui/mainapplication.cpp:9463-9553`).
5. `loadTreeFromCache()` builds **only the root level** (`src/ui/mainapplication.cpp:9258-9431`).

Each catalogue item is flat: `i`/`Id` (key), `n`/`Name`, `t`/`Type` (`d`/`folder` = folder),
`p`/`ParentId` (`ROOT` or `NONE` = top level).

### Load errors (all shown as a toast via `showTreeIssue`, `src/ui/mainapplication.cpp:398-404`)
| Situation | Message |
|---|---|
| Download client not ready | Yellow banner in the panel: "File tree not loaded. Required Visual C++ components are missing. Click 'Install VC++ Runtime' to fix this issue." + button **Install VC++ Runtime** (`:9465-9477`, `:2869-2926`) |
| No tree in memory or cache, server gave a reason | "File tree not loaded: <reason>" (`:9494-9498`) |
| No tree, no reason | "File tree not loaded: No data received from server. Please check your internet connection and login again." (`:9500`) |
| JSON broken | Cache files deleted; "File tree not loaded: Cached data was corrupted and has been cleared. Please login again to reload." (`:9508-9520`) |
| Not object/array | Cache deleted; "...Received invalid data format. Cache has been cleared — please login again to reload." (`:9523-9530`) |
| Could not build items | "...Data could not be displayed. Please login again or contact support." (`:9551`) |

**Install VC++ Runtime** button: runs `Additional/Vc_Redist_Aio.exe` next to the exe. Missing →
card *Error / "Installer Not Found"*. Launch fails → *"Failed to Launch Installer"*. Success →
info cards "Installing Visual C++ Runtime" then "Application Will Close", and the app quits after
0.5 s (`src/ui/mainapplication.cpp:3065-3097`).

After the first successful tree load, if `update log.txt` exists next to the exe it is shown once
in the "What's New" modal and deleted (`src/ui/mainapplication.cpp:9433-9461`).

## 2. Lazy loading and the "Loading..." child

1. Root folders that have children get one dummy child with text **"Loading..."**, marked
   `UserRole+2 = true`, so the expand arrow shows (`src/ui/mainapplication.cpp:9377-9385`).
2. User expands a folder → `onTreeItemExpanded` (`src/ui/mainapplication.cpp:7952-8165`):
   1. **Accordion:** every other expanded sibling folder (and everything inside it) is collapsed
      and its children freed (`:7964-8026`). Only one branch per level is open at a time.
   2. If the only child is the dummy, it is deleted and the real children are built from the
      cached catalogue items whose parent = this folder's key (`:8032-8120`).
   3. `.txt` files are skipped — they never appear in the tree (`:8077-8079`).
   4. Child folders that have children get their own "Loading..." dummy (`:8103-8109`).
   5. Children are sorted (section 3) and added (`:8122-8126`).
   6. Smart scroll: only if the last child is below the visible area, the folder is scrolled to
      the top (`:8137-8159`).
3. User collapses a folder → all its children are deleted to save memory and a fresh
   "Loading..." dummy is put back (`src/ui/mainapplication.cpp:8167-8209`). Re-expanding rebuilds
   them from the cache.

## 3. Sorting

`naturalSortLessThan` (`src/ui/mainapplication.cpp:126-176`), applied to the root level
(`:9407`) and to each folder's children (`:8123`):

1. **Priority first** (`UserRole+12`): `-1` folders whose name contains "[Beginner]" → `0`
   folders → `1` non-PDF files → `2` PDF files. So PDFs are always listed last.
2. **Then natural order:** digit runs compare as numbers ("2. Samsung" before "10. Honor"),
   other characters compare case-insensitively; shorter name first on a tie.
3. **Signup-trial / Free plan only:** after sorting, root folders that are in the account's free
   folder list (e.g. "Free - Trail Users") are moved to the top, keeping their order
   (`pinFreeFoldersToTop`, `src/ui/mainapplication.cpp:7504-7534`). A Premium account
   (no restriction) keeps them in natural order — which is why "Free - Trail Users" can appear last.

## 4. Names, icons and the green dot

- **Display name**: folders show the full name; files show the name **without extension**
  (`completeBaseName`, so "a.b.pdf" → "a.b") (`src/ui/mainapplication.cpp:8082`, `:9388-9389`).
  Tooltip = the same display name.
- **Folder icon**: flat amber folder, closed / open, painted in code; the same in both themes
  (`src/ui/mainapplication.cpp:8344-8414`). Swapped open/closed on expand/collapse (`:8253-8273`).
- **"[Beginner]" folders**: green beginner folder SVG (`:8448-8497`).
- **File icons** (`getFileIcon`, `:8275-8342`):
  | File | Icon |
  |---|---|
  | `.w2re` whose name contains "HS" (hardware solution) | `tree/hardware.svg` |
  | `.pdf` | `tree/pdf.svg` |
  | `.pcb .xzz .xzzpcb .brd .brd2 .cad .bvr .w2r .w2s .w2re .pro` | `tree/pcb.svg` |
  | anything else | Qt standard icons by type |

### The green dot ("hardware solution available")
- A 5 px green dot drawn just after the row's name: `#16A34A` light / `#159A4E` dark
  (`HWBadgeDelegate::paint`, `src/ui/mainapplication.cpp:2594-2608`).
- Hover over the dot area → tooltip **"Hardware solution available"** (`:2621-2644`).
- A **folder** gets the dot (`UserRole+20`) when any of these is true
  (`src/ui/mainapplication.cpp:9289-9329`, `:9365-9368`, `:8085-8087`):
  1. it contains a subfolder whose name contains "hardware solution";
  2. it contains a `.txt` file (a solution overlay);
  3. it contains a `.w2re` file with "HS" in the name;
  4. the solutions database has a community solution for a board inside it
     (`fetchDbSolutionKeys`, `:9559-9596` — fetched after the tree loads, so the dot may appear a
     moment later; only root rows are updated live).
- A **file** gets the dot when it is an "HS" `.w2re` hardware-solution file.

## 5. Single click

`onTreeItemClicked` (`src/ui/mainapplication.cpp:6962-6992`):
- **Folder** → toggles expand/collapse (one click is enough).
- **File** → just selects it (status text "double-click to open", not visible).
- **Folder inside search results** → selects only.

## 6. Search

Box: placeholder **"Search files..."**, magnifier icon inside on the left, built-in clear (×)
button, 28 px tall (`src/ui/mainapplication.cpp:2720-2736`). It does not take focus at startup;
**Ctrl+F** focuses it (`:9210-9216`).

### When it fires
- On **Enter** in the box, or a click on the magnifier icon (`src/ui/mainapplication.cpp:2735`, `:2867`).
- **Not** while typing — there is no search-as-you-type.

### Flow (`onTreeSearchTriggered`, `src/ui/mainapplication.cpp:3110-3208`)
1. Empty box → nothing (status message only).
2. A thin indeterminate progress line appears under the box; the box is disabled (`:3118-3123`).
3. If the term differs (case-insensitive) from the last one, the search runs 10 ms later:
   `findMatchingFiles` scans the whole cached catalogue (`:3345-3401`):
   - match = case-insensitive **substring** of the folder name, or of the file name without extension;
   - **top-level folders are excluded** from results;
   - `.txt` files are excluded;
   - stops at 10 000 hits (marks the search "trimmed").
4. **No results** → the box is cleared and refocused, the normal tree is restored, and card
   *Notice / "Search"*: "No files found for "<term>". Try a different name." (`:3141-3161`).
5. **Results** → `renderSearchResultsFlat` replaces the tree with a **flat list** (no folders
   above them) (`:3277-3342`):
   - each row shows only the item's own name (files without extension), with the folder/file
     icon and the green dot rule;
   - the matched part of each name is drawn **bold blue `#4D74F8`** (not on the selected row)
     (`:2526-2592`);
   - at most 5 000 rows are shown; beyond that a toast "Search trimmed for performance:
     N total results; showing first 5000" (`:3332-3336`).
6. Pressing Enter again with the **same** term does nothing new (`:3197-3207`).

### Leaving search
- Clearing the box (× button or deleting the text) restores the full tree from the cache and
  resets the highlight (`src/ui/mainapplication.cpp:2848-2866`).
- **Double-click a folder in the results** → the box is cleared, the tree is restored, and after
  50 ms the app expands down to that folder and selects it (`revealS3FolderInTree`)
  (`src/ui/mainapplication.cpp:7001-7014`, `:3530-3593`).
- **Double-click a file in the results** → opens it (same as section 7).

## 7. Open a file (double-click)

1. User double-clicks a file row → `onTreeItemDoubleClicked` (`src/ui/mainapplication.cpp:6994-7048`).
   - Double-clicking a folder just toggles it.
   - The tree uses Qt's default **single selection** (no `setSelectionMode` call), so in practice
     one file is opened at a time even though the code collects "all selected" keys.
2. `startAwsDownloadQueue(keys)` (`src/ui/mainapplication.cpp:7609-7663`):
   1. **Plan re-check for locked files**: if the file is outside the account's free folders and
      the plan was last checked over 60 s ago, the app asks MainWindow for a token refresh and
      waits up to 8 s before deciding (`:7614-7631`, `:7667-7676`).
   2. **Locked file** (signup trial / Free plan, outside the free folder) → never downloaded; the
      **Buy Premium** card is shown instead (see [profile.md](profile.md)) (`:7633-7649`).
   3. If a download is already running, the new keys are appended to the queue (`:7651-7655`).
   4. Otherwise the loader card appears ("Loading files...") over a dimmed scrim (`:7656-7662`).
3. For each queued key, `processNextAwsDownload` (`src/ui/mainapplication.cpp:7678-7950`):
   1. **Tab limits**: max **10 PDF tabs** (Home and tool tabs not counted) and **10 PCB tabs**.
      At the limit → queue stopped, card *Notice / "PDF Tabs"* or *"PCB Tabs"*: "Tab limit (10)
      reached. Close a tab to open more." (`:7680-7708`).
   2. Loader card: eyebrow "Schematic · Fetching" (PDF, red accent), "Boardview · Fetching"
      (board, copper) or "Hardware Solution · Fetching" (green); title = file name without
      extension; tag "2 of 7" for multi-file queues; status "Connecting…" then "x MB of y MB"
      with a progress bar (`:7095-7152`, `:7192-7211`, `:7730-7747`).
   3. The file is downloaded **into memory** through the app's server proxy (`m_aws.downloadToMemory`)
      — nothing is saved to disk (`:7750`, `:7902-7904`).
   4. Opened with `openFileFromMemory` (`:7928`, `:4659-4927`):
      - `.pdf` → new tab in the **PDF lane** (`PDFViewerWidget`), user's name as watermark.
      - `.xzz .pcb .xzzpcb .brd .brd2 .cad .bvr .bdv .w2r .w2s .w2re .pro` → new tab in the
        **PCB lane** (`PCBNewViewerWidget`); then a sibling `Solution.txt` overlay and community
        solutions are loaded (`:4908-4912`).
      - `.w2rs`, or a `.w2re` whose decrypted content is a solution → read-only solution viewer
        (`:4691-4698`).
      - Anything else → nothing opens.
      - **Already open** (same key) → that tab is just selected (`:4705-4721`, `:4823-4839`).
      - Tab name = file name without extension.
   5. Trial usage alert from the server: level 1 → small toast; level 2 → card *Download /
      "Download Limit Warning"* (`:7917-7926`).
4. Loader safety: a **Cancel** on the loader stops the queue ("Loading cancelled by user")
   (`src/ui/mainapplication.cpp:1082-1100`); a 45 s watchdog hides a stuck loader
   (`:1072-1080`).

### Download refusals and errors (`src/ui/mainapplication.cpp:7761-7892`)
| Server reason / failure | What the user sees |
|---|---|
| `EMAIL_NOT_VERIFIED` | *Notice / "Verify your email"* — server text or "Please verify your email address on the website." |
| `QUEUED_KEY_PENDING` | Token refresh → sign-out with "Your next key is ready" (MainWindow); if refreshed < 60 s ago, *Notice / "Your next key is ready"* |
| `CHECK_FAILED` | *Notice / "Please try again"* — "The server is busy right now. Please try again in a moment." |
| `TRIAL_FOLDER_LOCKED`, or other "needs premium" reasons | Buy Premium card (title "Your trial downloads are used up" for `TRIAL_TOTAL_LIMIT_REACHED`) |
| `TRIAL_/PREMIUM_DAILY_LIMIT_REACHED` | *Download / "Daily Download Limit Reached"* — "You have reached your daily download limit of N. Your limit will reset tomorrow." |
| `*_COOLDOWN_ACTIVE` | *Download / "Download Cooldown"* — "Please wait N minute(s) before downloading again." + "You have N download(s) remaining today." |
| DNS failure | *Download / "Couldn't find the download server"*, code chip "DNS lookup failed", steps (restart router, DNS 1.1.1.1/8.8.8.8, VPN/hotspot), buttons **Try again / Get support / Close** |
| Unreachable / other network | Same card, title "Couldn't reach the download server", chip "Server unreachable" |
| TLS blocked | *Download / "Secure connection was blocked"*, chip "TLS blocked", steps (antivirus HTTPS scanning, hotspot, date/time) |
| Anything else | Toast "AWS download failed: <error>" and the queue moves on |

Guided-card buttons (`src/ui/mainapplication.cpp:7225-7319`): **Try again** retries the same file;
**Get support** copies an English report (version, user, file name, failure, time) to the
clipboard and opens WhatsApp `wa.me/919999999999` pre-filled; **Close** drops the rest of the queue.

## 8. Context menu

**None.** No `contextMenuPolicy` / `customContextMenuRequested` is set on the tree; right-click
does nothing beyond Qt's default (searched `src/ui/mainapplication.cpp`, no match).

## 9. Ctrl+wheel zoom (tree text size)

1. With Ctrl held, the mouse wheel over the tree changes the tree text size instead of
   scrolling: +1 px per notch up, −1 px per notch down; small trackpad deltas are accumulated
   (`SmoothTreeWidget::wheelEvent`, `src/ui/mainapplication.cpp:2655-2671`).
2. Also: **Ctrl+= / Ctrl++ / Ctrl+-** (±1 px) and **Ctrl+0** (reset) while the tree has focus
   (`:9218-9236`), and Settings > File Tree Text Size (any time).
3. Range **10–24 px**, default **13 px** (`include/ui/apptheme.h:36-38`).
4. Row height, icon size (16 px at 13 px) and indentation (20 px at 13 px) scale with it; the
   row that was at the top stays at the top (`src/ui/mainapplication.cpp:2973-3006`).
5. Saved as `tree/fontPx` in `settings.ini` and restored at startup (`:3008-3026`, `:2678-2682`).

## 10. Resize and collapse the sidebar

`CollapsibleSidebar` owns its own width (`src/ui/collapsiblesidebar.cpp`). Constants (`:27-43`):
default **350 px**, min **250**, max **800**, collapsed rail **13 px**, animation 240 ms.

| Action | Result |
|---|---|
| Hover near the right edge | A notch "blade" (10×36 px) fades in at mid-height; tooltip "Collapse panel" / "Expand panel" (`:51-188`, `:254-265`) |
| Click the blade | Toggle collapse/expand (`:219`) |
| Hover the 10 px right-edge grip | Resize cursor; hint "Drag to resize" (`:314-326`) |
| Drag the grip | Live resize between 13 and 800 px; content hides below 250 px (`:357-385`) |
| Release while narrower than 200 px (started open) | Collapses to the rail (`:404-411`) |
| Release between 200 and 250 px | Springs back to 250 px (`:412-423`) |
| Collapsed: drag the rail outward > 40 px | Opens at the dragged width (`:409`, `:426-427`) |
| Click the grip / rail without dragging | Toggles (`:394-403`) |
| Ctrl+T / Ctrl+Shift+T | Toggle / show (`src/ui/mainapplication.cpp:9159-9176`) |
| F11 immersive full screen | Hides the sidebar, restores its previous state on exit (`:9045-9097`) |

The expanded width is remembered while the app runs but **not saved** between runs (no
QSettings use in `collapsiblesidebar.cpp`); each start uses 350 px.

## 11. Not present in the current code
- Breadcrumb under the search box — removed (`src/ui/mainapplication.cpp:2928-2930`).
- A-/A+ buttons in the search row — removed (`:2738-2741`).
- Local / Server / AWS source toggle — unused (`:2754-2755`, `:3099-3104`).
