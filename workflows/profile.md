# Profile / account workflow

What the main window knows about the signed-in account, how it shows it, and every way the
account state changes while the app is open.

## 1. Sign-in state handed to the main window

1. The login window signs in (`AuthService`), then builds a `UserSession`: username, fullName,
   email, **plan**, **planExpiry**, loginTime = now (`src/ui/mainwindow.cpp:2476-2483`,
   `include/ui/mainapplication.h:107-114`).
2. It creates `MainApplication`, passes the password (kept **encrypted in memory only**, used to
   sign the Home tab into the website) (`src/ui/mainwindow.cpp:2486-2489`,
   `src/ui/mainapplication.cpp:691-703`).
3. `configureAwsFromAuth(creds, token)` stores the desktop token; downloads go through the
   server proxy with it (`src/ui/mainapplication.cpp:3860-3921`).
4. The reply's `trialAccess` block is applied with `setTrialAccess` (`:7434-7448`):
   `present`, `trialType` (`signup` / `key` / `free`), `fullLibrary`, `freeFolders`,
   `dailyLimit`, `totalLimit`, `usedToday`, `usedTotal` (`include/network/authservice.h:36-48`).
5. The window title becomes "Schemax Schematics - <full name>" (`src/ui/mainapplication.cpp:8994-8998`).

## 2. Plan shown

| Where | Text | Code |
|---|---|---|
| Title-bar chip caption | plan upper-cased; empty → "STANDARD" | `src/ui/identitybar.cpp:621` |
| Profile card badge + Plan row | plan upper-cased; empty session plan → "Standard" | `src/ui/mainapplication.cpp:1834`, `src/ui/profilecard.cpp:269-273` |

`displayPlan()`: if the server plan is "Trial" and the trial kind is `key`, the app shows
**"Trial key"** so it is not confused with the sign-up trial (`src/ui/mainapplication.cpp:7452-7458`).

## 3. Days left

- Title-bar pill: computed from `planExpiry` → hidden when there is no expiry; otherwise
  "N days left", always green ([titlebar.md](titlebar.md#days-left-pill)).
- Profile card "Expiry date": `dd MMM yyyy` in local time, plus a suffix
  (`src/ui/mainapplication.cpp:1840-1855`, `src/ui/profilecard.cpp:275-293`):

| daysLeft | Suffix | Colour |
|---|---|---|
| no expiry (-1) | hidden; date shows "N/A" | – |
| unparseable expiry | raw text shown as the date, no suffix | – |
| 0 | "· Expired" | `@ERROR`, bold |
| 1–7 | "· 1 day left" / "· N days left" | `@ERROR`, bold |
| 8–30 | "· N days left" | `@COPPER` |
| > 30 | "· N days left" | `@TEXT_SECONDARY` |

Thresholds: `kExpiryUrgentDays = 7`, `kExpiryWarnDays = 30` (`src/ui/profilecard.cpp:43-44`, `:239-258`).

## 4. Profile card (click the title-bar chip)

`onProfileRequested()` (`src/ui/mainapplication.cpp:1809-1885`). Frameless, always-on-top,
**modal** dialog with a `ProfileCard` inside. Labels from `resources/ui/profilecard.ui`.

1. User clicks the chip (tooltip "Account").
2. Card fades in once (`src/ui/profilecard.cpp:367-374`).
3. Header: avatar letter, username, plan badge, close (X, tooltip "Close").
4. **SUBSCRIPTION DETAILS**
   - Username — click copies it; tooltip "Click to copy", then "Copied" (`src/ui/profilecard.cpp:422-440`).
   - Plan.
   - Expiry date + suffix (section 3).
   - Last login — the time this session started, `yyyy-MM-dd HH:mm` (`src/ui/mainapplication.cpp:1835`).
     It is the **current** login, not the previous one.
   - **Auto login** switch — reads/writes `login/auto_login` (default true) immediately on toggle
     (`src/ui/mainapplication.cpp:1863-1874`). The login window uses this key to sign in
     automatically next time (`src/ui/mainwindow.cpp:721-727`).
5. **SYSTEM DIAGNOSTICS**
   - Hardware: always "Active" — hard-coded `setHardwareActive(true)` (`src/ui/mainapplication.cpp:1858`).
   - OS version (`QSysInfo::prettyProductName`), Architecture (`currentCpuArchitecture`).
   - Hostname (`machineHostName`) — click to copy.
6. Close with X → dialog closes (`:1877`). If the theme changes while open, the card re-skins (`:1881-1882`).

The card has **no** Logout, Buy/Renew, My Purchases, account-details or HWID row.

## 5. HWID (hardware ID)

- Sent by the login request as `hardwareId` (`src/network/authservice.cpp:234-236`); login can be
  refused with `MISSING_HARDWARE_ID`, `INVALID_HARDWARE_ID` or `HARDWARE_ACCESS_DENIED`
  (`:296-301`).
- Also used to derive the key that encrypts the on-disk tree cache (`src/ui/mainapplication.cpp:3926-3927`).
- **Not displayed anywhere in the main window.** The profile card shows the hostname instead.

## 6. Trial / Free-plan limits

- Title-bar allowance pill ([titlebar.md](titlebar.md#trial-allowance-pill-next-to-days-left)).
- Free folders pinned to the top of the tree ([treeview.md](treeview.md) §3).
- A file outside the free folders is **locked** for `restrictsDownloads()` accounts (trial present
  and not full library) (`include/network/authservice.h:46`, `src/ui/mainapplication.cpp:7539-7548`).

## 7. Buy Premium card (Razorpay vs WhatsApp)

Shown when a locked file is opened, or when the server refuses a download for plan reasons
(`showPremiumFileDialog`, `src/ui/mainapplication.cpp:7550-7600`).

Card: heading **"Premium"**; title **"This file is included with Premium"** (or "Your trial
downloads are used up" when the lifetime cap is hit); body = the server's message, else e.g.
"Your free trial can open the files in the "Free - Trail Users" folder. Buy Premium to access
every file." (Free-plan wording: "Your Free plan can open ...").

### Is online payment on?
- 3 s after start the app asks the public `POST /api/pay/plans`; payment is "on" only if the reply
  has `ok: true` **and** `enabled: true`. Fails closed (off) until a server says yes
  (`src/ui/mainapplication.cpp:624-627`, `src/network/authapi.cpp:89-105`).
- Each card refreshes the answer in the background if it is older than 10 minutes
  (`include/network/authapi.h:102`, `src/ui/mainapplication.cpp:7573`).

### Buttons
| | Online payment ON | Online payment OFF |
|---|---|---|
| Primary | **Buy Premium** → `openPremiumPlans()` | **Buy Premium** → `openWhatsAppBuy()` |
| Secondary | **Buy on WhatsApp** → `openWhatsAppBuy()` | **Enter Activation Key** |
| Tertiary | **Enter Activation Key** | **Close** |

### openPremiumPlans() (`src/ui/mainapplication.cpp:2288-2325`)
1. If there is a Home tab **and** a website token:
   1. Switch to the Home tab (no tree reset).
   2. URL = `<site>/user/app_dashboard.php?upgrade=1&n=<counter>#plans` (`src/core/buildenv.cpp:20-28`, `:179-182`).
   3. If the page is already signed in → navigate now; otherwise remember the URL, sign in first,
      and open the plans instead of the dashboard (`:2305-2314`).
   4. The user picks a plan and pays inside the Home tab (Razorpay). Nothing is preselected.
2. Otherwise opens `<site>/user/dashboard.php?upgrade=1&for=<username>#plans` in the user's
   browser (username omitted if it contains "@") (`:2321-2324`, `src/core/buildenv.cpp:184-187`).

### openWhatsAppBuy() (`src/ui/mainapplication.cpp:2328-2334`)
Opens `api.whatsapp.com/send/?phone=919999999999` with "Hi, I want to buy Schemax Premium.\nUsername: <username>".

### Enter Activation Key
1. Emits `activationKeyRequested` → `MainWindow::showActivationKeyDialog()` (`src/ui/mainwindow.cpp:2495`, `:1965-…`).
2. From inside the app (login window hidden) it is the classic dialog: title "Enter Activation Key",
   "Enter the activation key you received:", placeholder "e.g. XXXX-XXXX-XXXX-XXXX", **Cancel / Activate**
   (`src/ui/mainwindow.cpp:2043-2190`, title at `:2077`). Empty → "Please enter your activation key."; while waiting →
   "Activating...". Spaces and dashes are removed and the key upper-cased before sending.
3. On success the trial lock is cleared and a token refresh pulls the new plan (`:2005-2008`).

## 8. Plan re-check after paying (plan watch)

Triggered by (`src/ui/mainapplication.cpp:6784-6811`):
- the website posting `w2r:plan-changed` to the app;
- the last in-app payment window closing;
- the Home tab loading `/user/payment-status.php`.

Flow (`startPlanWatch` / `planWatchTick`, `:2361-2421`):
1. Remember a fingerprint of plan + expiry + trial kind + lock.
2. Ask MainWindow for a token refresh at **0, 5, 15, 30, 60 s** (then wait 35 s for the last answer).
3. Every refresh result goes through `updatePlan` / `setTrialAccess`; as soon as the fingerprint
   changes the watch stops and the title-bar chip, days-left pill and allowance update.
4. Budget: at most 12 watch refreshes in any 5 minutes. Triggers within 3 s merge into one watch.

## 9. Plan changes from the server at other times

- Every token refresh (background, or `sessionRefreshRequested`) calls `setTrialAccess` and
  `updatePlan` (`src/ui/mainwindow.cpp:2497-2510`, `src/ui/mainapplication.cpp:3894-3903`).
- A locked click with a plan older than 60 s refreshes first (up to 8 s) ([treeview.md](treeview.md) §7).
- Server says the token is invalid → `logoutRequested("Your session is expired so please logout
  and relogin.")` (`src/ui/mainapplication.cpp:645-653`) → MainWindow signs out and shows
  **"Session Expired"**, or a specific card for blocked / plan expired / trial expired /
  "Your next key is ready" (`src/ui/mainwindow.cpp:2597-2623`, `:2673-2681`).

## 10. Logout

Only from **File > Logout** (or a forced sign-out). Full steps in [menubar.md](menubar.md#file--logout).
Summary: confirm "Log out?" → clear Home website sign-in → close all tabs → delete saved token from
Windows Credential Manager → clear in-memory password → back to the login window
(`src/ui/mainapplication.cpp:8559-8598`, `src/ui/mainwindow.cpp:2625-2694`).
If the Solution Editor has unsaved work during a forced sign-out: card heading "Signing out",
title "Unsaved solution", buttons **Save / Discard** (`src/ui/mainwindow.cpp:2643-2654`).

## 11. Purchases / "My Purchases"

There is no purchases screen in the desktop app code. "My Purchases" is a website feature shown
inside the Home tab (per `docs/Razorpay-Payment-Gateway-Plan.md:463`, a design doc, not app code)
**(website behaviour not confirmed in code)**.
