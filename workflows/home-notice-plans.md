# Home tab: notice ticker, plans carousel, offer banner, WhatsApp button

## Key fact: this whole area is a web page

The right-hand area in the screenshot (NOTICE ticker, "Premium plans / Extend your access"
carousel, Buy ₹600 / WhatsApp buttons, INR | USD toggle, duration chips, secure checkout /
Razorpay row, My Purchases link, Diwali offer banner, floating "Need a key? Contact on WhatsApp"
button) is **not built in Qt**. It is the website page **`/user/app_dashboard.php`** shown in the
permanent **Home** tab, a `WebView2Widget` (embedded Edge) (`src/ui/mainapplication.cpp:290-297`,
`:6738-6858`).

The HTML/JS of that page is not in `E:\schemax\tool-script` (and `app_dashboard.php` was not
found in the local `E:\WRSOLUTION` checkout either). So the inner behaviour of the ticker,
carousel, currency toggle, chips, banner and floating button is **(not confirmed in code)**.
This document describes exactly what the **app** does around that page.

Searches for "ticker", "carousel", "Diwali", "Need a key", "Extend your access", "My Purchases"
in the app's `src/` and `include/` return nothing.

---

## 1. The Home tab itself

### Creation at startup (`addPermanentHomeTab`, `src/ui/mainapplication.cpp:6738-6858`)
1. Browser profile folder: `%LOCALAPPDATA%\…\WebView2Data` (DEV builds use their own) (`:6749-6752`).
2. First URL: `<site>/user/signin.php?client=app` (`appSignInUrl`, `:290-293`). `<site>` is
   `https://w2rsolutions.in` in a release build (`:288-289`, `:307-309`).
3. Marked `isHomeTab` **before** it is added, so its tab pill is icon-only (house), neutral and
   **cannot be closed** (`:6760-6764`, `:4602-4605`).
4. Theme is handed to the page before it loads (`wv->setTheme(dark)`, `:6769`).
5. Start-up scripts injected into every page:
   - `kHomeBootJs` — once per run, removes any website sign-in left by an earlier run (shared PCs) (`:336-342`, `:6772`).
   - `kHomeCapsJs` — sets `window.W2R_APP_PAY = 1` so the website knows this app can pay in-app (`:387-391`, `:6780`).
6. Messages from the website are accepted only from the site root (`setWebMessageOrigin`, `:6781`).
7. In-app payment popups enabled (`setInAppPaymentPopups(true)`, `:6782`).
8. The Home tab is made the current tab and an auto sign-in starts 50 ms later (`:6846-6853`).

The Home tab is in the **PDF lane** but is not counted toward the 10-PDF limit
(`countExcludingHome`). It is not available for immersive full screen
("Full screen is available while viewing a board, PDF or hardware solution", `:9054-9061`).

### Automatic website sign-in (`src/ui/mainapplication.cpp:1955-2262`)
1. Get a website token:
   - password logins → `POST <accountApi>/api/auth/login` with username + in-memory password;
   - Google / auto-login (no password) → `POST <desktopApi>/auth/web-token` with the desktop token (`:2037-2106`).
2. Write `session` (+ `w2r_auth` when there is a token) into the page's localStorage, and a
   "Welcome, <user>!" flash message (`kInjectHomeSessionJs`, `:357-372`, `:2166-2207`).
3. Navigate to `/user/app_dashboard.php` — **this is the page with the notice and plans**
   (`:2195-2197`).
4. Token renewal 30 minutes before expiry (12 h tokens), retry every 5 min on failure (`:2109-2141`).
5. If the page bounces back to the sign-in page while the app is signed in, it re-signs in at
   most twice per 5 minutes, then stops ("Home: please sign in on the page") (`:6826-6843`).
6. Sign-in refused (HTTP ≥ 400 with a password) → no retry until the user clicks Home again (`:2245-2251`).

### Theme follow
Light/dark toggles call `setTheme(dark)` on every WebView2 (`:8959-8970`); the page reads it from
a localStorage value seeded before each document (`src/webview2/WebView2Widget.cpp:147-172`).
Popup windows (Razorpay/bank) are never themed (`:149-153`).

### Logout
File > Logout removes the website sign-in from the page and clears its localStorage /
sessionStorage and caches (`src/ui/mainapplication.cpp:8569`, `:656-686`).

---

## 2. Notice ticker ("NOTICE" + INFO items)

- Rendered by the website dashboard page **(content, scrolling speed and item source not confirmed in code)**.
- App-side: nothing reads or controls it. It is visible whenever the Home tab is current and
  the dashboard page has loaded.
- Not related to the Qt `GlassNotice` class, which is the app's pop-up card, not a ticker
  (`include/ui/glassnotice.h`).

## 3. Plans carousel ("Premium plans / Extend your access")

Website content **(card layout, INR | USD toggle, duration chips such as 3 MONTHS … 3 USERS,
prices such as ₹600 — not confirmed in code)**. What the app does that affects it:

| App behaviour | Effect on the page | Code |
|---|---|---|
| `window.W2R_APP_PAY = 1` injected | Website may show its in-app Pay button; older apps without the flag are sent to the browser to pay | `src/ui/mainapplication.cpp:387-391` |
| Buy Premium card → `openPremiumPlans()` | Opens `/user/app_dashboard.php?upgrade=1&n=<n>#plans` in the Home tab so the page highlights the plans; nothing preselected | `:2288-2316`, `src/core/buildenv.cpp:20-28` |
| No website token | Same list opened in the external browser at `/user/dashboard.php?upgrade=1&for=<user>#plans` | `src/ui/mainapplication.cpp:2321-2324` |

### Buy (Razorpay) inside the Home tab
1. User clicks a Pay/Buy button on the page (website).
2. The page opens Razorpay checkout. New-window requests from the Home tab are routed by
   `W2RPopup::decide` (`include/webview2/PopupPolicy.h:63-93`):
   - Razorpay hosts (`razorpay.com` and subdomains), blank windows, and windows opened by a
     Razorpay frame → **in-app payment window** (max 4 at once, `src/webview2/WebView2Widget.cpp:480`).
   - Bank / 3-D Secure pages opened from a payment window → also in-app.
3. When the **last** payment window closes, the app starts a **plan watch**
   (`src/ui/mainapplication.cpp:6788-6793`).
4. When the website posts `w2r:plan-changed`, or `/user/payment-status.php` loads, the app also
   starts a plan watch (`:6784-6787`, `:6808-6811`).
5. Plan watch refreshes the token at 0/5/15/30/60 s until the plan changes; then the title-bar chip
   and days-left pill update (details in [profile.md](profile.md#8-plan-re-check-after-paying-plan-watch)).

### WhatsApp buttons on the cards / "My Purchases" link
- Website content **(not confirmed in code)**.
- App-side routing of such links from the Home tab: any popup that is **not** a payment window
  keeps WebView2's own default popup window, because `kHomeLinksFollowPolicy = false`
  (`src/webview2/WebView2Widget.cpp:488`, `:886-892`). So a `target=_blank` WhatsApp link opens in a
  separate WebView2 popup window, not in the system browser **(exact page behaviour not confirmed in code)**.
- Same-tab links (no new window) simply navigate the Home tab.

## 4. Diwali offer banner

Website content **(not confirmed in code)**. The only app-side references are design docs
(e.g. `docs/razorpay-phase3/design/api-contract.md:240`, `:279` — an `offer` object with
`label`, `bonusMonths`, `minMonths`, dates), not running app code.

## 5. Floating "Need a key? Contact on WhatsApp" button

Website content **(not confirmed in code)**. Link handling is the same as section 3 (non-payment
popup → WebView2 default popup window).

The **app's own** WhatsApp entry points, for comparison (all open the system browser /
WhatsApp via `QDesktopServices::openUrl`):

| Entry point | Target | Code |
|---|---|---|
| Buy Premium card → "Buy on WhatsApp" (or "Buy Premium" when online pay is off) | `api.whatsapp.com/send/?phone=917708684274`, "Hi, I want to buy Schemax Premium. Username: …" | `src/ui/mainapplication.cpp:2328-2334` |
| Download failure card → "Get support" | `wa.me/917708684274` with a pre-filled error report | `:7308-7313` |
| Settings > Language > Suggest a wording fix / Help > Language help | `api.whatsapp.com/send/?phone=917708684274` with a wording template | `:7366-7399` |
| Help > About → WhatsApp tile | support chat | `src/ui/aboutdialog.cpp:238` |

---

## 6. Clicking the Home tab / Home action

Clicking the Home tab pill just makes it the current tab (normal tab switch).

`onHomeClicked()` (`src/ui/mainapplication.cpp:1887-1953`) would reset the tree, clear page
caches and re-sign the dashboard in with a fresh token, but **nothing calls it** in the current
code (a search of `src/ui/*.cpp` finds only its definition). It is dead code.

## 7. Edge cases

| Case | Behaviour | Code |
|---|---|---|
| WebView2 not compiled in (`HAVE_WEBVIEW2` off) | No Home tab; branded idle backdrop shows when no file is open | `src/ui/mainapplication.cpp:6740`, `:6855-6857`, `:1795-1805` |
| No desktop token and no password yet | Sign-in waits until `configureAwsFromAuth` provides the token | `:2231-2233`, `:3868-3872` |
| Website has tokens turned off / older server | Only the `session` key is written (no `w2r_auth`) | `:2253-2256`, `:2014-2032` |
| Logged out mid-request | Late replies are ignored (generation counter) | `:1982-1998`, `:2070-2071` |
| Too many in-app windows (>4) | Extra payment popup dropped | `src/webview2/WebView2Widget.cpp:893-896` |
