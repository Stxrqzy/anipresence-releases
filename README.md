# AniPresence

Detect the anime you are watching in Google Chrome and mirror it to Discord
Rich Presence automatically — cover art from AniList, episode number, playback
progress, watching duration and quick links back to the site and AniList page.

Functionally similar to PreMiD / Kaguya, but specialized for anime: it reads
the **actual HTML5 video element** (via Chrome DevTools Protocol), so it knows
the real `currentTime`, `duration` and play/pause state rather than guessing
from the page title.

---

## Features

- **Real player-state detection** — reads the live `<video>` element with CDP.
  Works even for cross-origin iframe players (GoGoAnime, AnimePahe, AniWave…)
  by attaching to each iframe's own DevTools target.
- **Supported sites** — Crunchyroll, Netflix, Hulu, HIDIVE, AniWave/9anime,
  GoGoAnime mirrors, AnimePahe, HiAnime/AniWatch/Zoro, Miruro, ani.pm, Bilibili,
  plus a generic extractor for Prime Video, Disney+, WcoFun, KissAnime and
  friends.
- **ani.pm episodes** — reads the active episode from the player heading or
  selected playlist entry, including opaque `?ep=` links and episode switches
  without a page reload. Named episodes show their title beside the episode number
  in Discord, including while paused. After upgrading, reload AniPresence Bridge in your
  browser's extensions page and refresh the ani.pm tab to load the updated reader.
- **AniList identification** — fuzzy matching with multi-term search fallback
  and a disk cache (default 7-day TTL) to avoid hammering the API.
- **Two-way AniList sync** — pulls your AniList list into the Library and Stats
  (the token makes this work for private profiles too) and pushes episode
  progress after an episode reaches 90% playback; progress is never lowered.
- **Account indicator** — the active AniList account shows its profile picture
  next to the account selector, so you always know whose watch history/stats
  you're looking at.
- **Rich Presence** — AniList cover art (highest-resolution variant) as the large
  image, `Watching: <Title>`, `Episode N of M · Next: Ep N+1 · Fri` (episode
  totals + the next airing date, shown when you're caught up), a native playback
  progress bar (hidden while paused, restored at the playhead on resume), and two buttons (*Watch Anime*,
  *Open AniList*).
- **Vencord bridge** — optional broadcast through a Vencord userplugin
  (`vencord-plugin\anipresence\`) so friends see your presence even with an
  unverified Application ID; pick `Vencord bridge` or `Both` under Broadcast.
- **Multiple browsers** — Chrome, Brave, Edge and Opera are supported for
  detection and as PWA launchers (any Chromium that exposes the debug port).
- **Modern chrome** — borderless window with macOS-style traffic-light controls
  (close-to-tray, minimize, maximize), native edge-resize and a themed title
  bar; the app ships with its own logo as the window and tray icon.
- **Auto behaviour** — hides the presence when nothing is playing, shows a
  paused state when paused, auto-launches Chrome with the debugging flag.
- **Desktop UI** — dark PySide6 window with live status, cover art, progress
  bar, controls, recently-watched history and a log panel.
- **Watch anything** — force a presence for anything AniPresence can't detect
  (non-anime sites, movies, games) with a title/episode/website dialog.
- **Continue watching** — a *Next episode* button opens the next episode of the
  current show, and *Copy URL* grabs the page link.
- **Watch stats** — playback time is tracked per title and shown in a stats
  dialog; covers are cached on disk so they load instantly on repeat visits, and
  recently-watched history shows each show's poster thumbnail.
- **Idle presence** — optionally show a "Browsing anime" presence instead of
  clearing it when nothing is playing.
- **Windows 11 Mica** — frosted-glass backdrop (Mica on Windows 11, acrylic on
  Windows 10) with translucent panels and a theme-matched title bar; presence is
  cleared on quit so nothing lingers in Discord.
- **Cinematic backdrop** — dims the current cover art behind the whole window
  for a movie-theatre look (toggle under Controls). Rendered from the largest
  cover variant at display resolution, so it stays crisp on HiDPI screens.
  Pairs with Mica for a frosted-glass key-art glow.
- **Auto accent** — samples a vibrant accent color from the current cover art and
  re-tints the buttons, spinner and tray icon per show. Turn it off under
  Controls (or pick a manual accent) to pin a fixed color.
- **Accent presets** — a curated palette (Spring Bloom, Summer Sunset, Autumn
  Ember, Winter Frost, Cherry Blossom, Midnight Violet, Ocean Depths, Forest
  Dew) sits in the header; pick a preset or "Custom color…" for any color.
- **Continue watching** — the dashboard shows a strip of titles you've watched
  before with a one-click *Resume Ep N* jump straight back to the last spot
  (covers and tooltips included; the current show is hidden from the strip).
- **Rich Presence preview** — a live mockup of your Discord activity card
  (details, state, elapsed clock, buttons) updates under the dashboard as you
  watch.
- **Adaptive tray icon** — the tray logo grows a progress ring around the cover
  while something plays, and the tooltip shows title · episode · elapsed.
- **Global hotkeys** — `Ctrl+Alt+A` toggles the window, `Ctrl+Alt+P` pauses/
  resumes detection, `Ctrl+Alt+X` clears the presence, `Ctrl+Alt+N` opens the
  next episode. All four are **reassignable** from the Settings tab (click a
  shortcut, press the new combination; Esc cancels; Reset restores the defaults).
- **Settings tab** — a dedicated page for global-hotkey assignment, the Discord
  application ID, broadcast method, browser selection and poll interval, plus
  the less-common knobs: minimum match score, AniList cache lifetime, the Chrome
  debugging port, and the paused / no-video presence behaviours.
- **Fresh look** — a remodeled dark/light interface: rounded card system,
  accent-pill tabs, gradient progress, refined chips and buttons.

---

## Architecture

```
main.py                          entry point (arg parsing, Qt bootstrap)
src/
  config.py                      typed JSON config (app data folder)
  logger.py                      console + rotating file logging
  browser/
    cdp.py                       Chrome DevTools Protocol client (tabs + iframe
                                 targets, Runtime.evaluate over WebSocket)
    detectors.py                 per-site extractors + DetectorRegistry
                                 (add a site by subclassing BaseDetector)
  anime/
    titles.py                    slug/title/episode parsing utilities
    anilist.py                   AniList GraphQL client
    matcher.py                   rapidfuzz title matching
    cache.py                     disk cache of AniList results (TTL)
  discord/
    rpc.py                       pypresence wrapper (connect/update/clear)
  core/
    controller.py                orchestrator (poll loop, session tracking,
                                 presence throttling)
    worker.py                    runs the controller on a QThread
    recent.py                    recently-watched history
    watchlog.py                  per-day/per-title watch-time analytics
    hotkeys.py                   global-hotkey spec parsing/formatting
  ui/
    bridge.py                    Qt signals worker -> UI + Qt log handler
    app.py                       PySide6 main window
```

### How detection works

1. Every `poll_interval_seconds` (default 5 s) the controller asks Chrome's
   debugging endpoint for all page + iframe targets.
2. It evaluates one JavaScript snippet per page (and its child iframes) that
   returns `{title, visibility, focused, video: {currentTime, duration, paused,
   ended, readyState, src}}`.
3. The tab with the best score (visible, focused, playing, loaded) wins.
4. The matching site detector parses the URL / page title / `__NEXT_DATA__`
   into `{title, episode}`.
5. The title is matched against AniList (cache first, then search). A
   multi-term fallback (full title → first 3 → first 2 → first significant
   token) handles cases where AniList's strict search rejects the extracted
   phrasing.
6. The result is pushed to Discord, throttled so the RPC pipe isn't spammed.

---

## Limitations

- **Chrome must run with `--remote-debugging-port=9222`.** A normal Chrome
  session is not inspectable. AniPresence can launch Chrome for you (with a
  dedicated profile if Chrome is already running); see below.
- **Discord activity images** must be asset keys or publicly reachable image
  URLs with a file extension. AniList covers qualify; if Discord ever rejects an
  image the app retries without it so text/timestamps/buttons still show.
- **Netflix / Hulu episode numbers** are hidden inside DRM-protected streams
  and are not exposed to the page. The title is extracted from the page title;
  the episode number is shown only when the page title contains something like
  `S1:E7`. Progress still works (the `<video>` element exposes `currentTime`).
- **Sites change their DOM/URLs.** Detectors are intentionally isolated so a
  broken site can be fixed or replaced without touching the rest of the app.
- **AniList coverage.** Obscure or mistyped titles may not resolve; the app
  falls back to showing the raw detected title.
- **Discord must be running** and logged in for Rich Presence to work.

---

## Why can't my friends see my Rich Presence?

Discord only broadcasts Rich Presence to other users for applications it has
**verified** (games, bots, etc.). A regular, self-created application ID is
"unverified", so the presence it sets is visible **only to you** — you'll see
it in your own profile, but friends/other users won't.

To make it visible to others you'd have to get your application verified via
the Developer Portal (Settings → Verification) and meet Discord's requirements,
which is intended for real games/apps and usually won't pass for a personal
anime-tracking tool. Practical workarounds people use:

- **Accept it as a personal-only presence** (works great for your own profile).
- **Use a shared, already-verified app** that exposes RPC and paste its
  application ID into the *Discord Application ID* field instead of your own —
  see "1b. Broadcast to friends".
- **Verify the app anyway** via
  <https://discord.com/developers/applications> → your app → **Verification**
  → *Submit for verification* and provide the requested info. Approval is
  discretionary and typically requires a real public product.

Before assuming it's a verification issue, double-check the other person's
**User Settings → Activity Privacy** (activity sharing must be enabled) and
that they're viewing you in a server/friend list where activity is shown.

---

## Installation

Requires **Python 3.10+** and **Chrome/Chromium**.

```powershell
git clone <this repo> AniPresence
cd AniPresence
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

### 1. Create a Discord Application

1. Open <https://discord.com/developers/applications> and click
   **New Application** (name it e.g. `AniPresence`).
2. Copy the **Application ID** (top of the page, under *General Information*).
3. Open the app's **Rich Presence → Art Assets** tab (optional but nice):
   - You can upload any assets, but **you don't have to** — AniPresence uses
     public URLs for cover art, so no assets are required.
4. Paste the Application ID into the app's config (below) or into the config
   file in `%APPDATA%\AniPresence\config.json`.

> Note: your Discord account does **not** need to be in the developer team of
> the application for it to use its own Rich Presence. Any application ID
> works with the local Discord client.

### 1b. (Optional) Broadcast to friends via a verified app

A self-made App ID is unverified, so its Rich Presence is visible **only to
you**. To make it visible to friends/other users, paste a **verified**
application's ID into the *Discord Application ID* field instead. Caveats:

- The status line will show that app's name (e.g. `Playing PreMiD`).
- Your Discord account must have authorized that application.
- Using another app's identity is not what Discord intends; it may be
  suppressed, and doing it against an app's wishes could risk your account.
  Prefer getting your own app verified via the Developer Portal if you can.

### 1c. (Optional) Broadcast to friends via the Vencord plugin

Vencord (a client mod) accepts local activity updates that skip Discord's
server-side app verification, so friends can see your presence without
verification. AniPresence ships a small userplugin that bridges its activity
into that channel:

1. Copy the `vencord-plugin\anipresence\` folder into Vencord's
   `src\userplugins\` and rebuild Vencord (`pnpm build`).
2. In the app (Settings → Discord & browser), set **Broadcast** to `Vencord
   bridge` or `Both`.
3. The plugin opens a tiny local server on `127.0.0.1:6339` and dispatches a
   `LOCAL_ACTIVITY_UPDATE` whenever AniPresence posts a new activity, which
   Vencord turns into a presence visible to other users.

The bridge binds to `127.0.0.1` only. If the plugin isn't running, AniPresence
falls back to the Discord pipe automatically (`is_connected` status stays
grayed).

### 2. Prepare the browser

AniPresence needs a Chromium-based browser (Chrome or Brave) exposed on its
debugging port. Pick which one in the UI (`Settings → Discord & browser`) or via
the `browser` config key. Two options:

- **Let the app do it.** Keep `chrome_auto_launch = true` in the config. When
  no debug port is found the app launches Chrome itself:
  - if Chrome is closed → your normal profile is used;
  - if Chrome is already open → a **separate AniPresence window** with a
    dedicated profile is opened (the debug flag is ignored on a running
    default-profile Chrome). Use that window for anime.
- **Manual (recommended for your real profile):** fully quit Chrome, then start
  it once with the flag, e.g.:
  ```powershell
  & "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222
  ```
  Subsequent launches from the taskbar/desktop keep the flag for that session.

### 3. Run

```powershell
python main.py
```

The window shows connection status for Chrome and Discord, the currently
detected anime, and a log. Toggles for RPC / auto-detection and the poll
interval are persisted in the config.

**Background mode**: closing the window minimizes AniPresence to the system
tray instead of quitting (toggle in Controls). The tray menu shows the current
anime, can open it as a site or PWA, clear the presence, or quit for real.
Desktop notifications appear when the browser or Discord connects.

**Correcting a wrong match**: hit "Wrong match?" on the detected-anime card to
set the correct title/episode. The fix is remembered per page, so the same URL
detects correctly next time.

**Recently watched** stays across restarts (`%APPDATA%\AniPresence\recent.json`).
Double-click an entry to reopen it, or right-click for "Open as PWA". Playback
time is accumulated per title — open **Watch stats** in the controls to see your
totals. Covers are cached in `%APPDATA%\AniPresence\covers\`.

**Privacy mode** strips URLs out of the on-screen log (and recent tooltips).

**Accent color & theme**: pick a preset from the dropdown next to the theme
selector, or choose **Custom color…** to open the picker (either pins the color
and disables auto accent). With **Auto accent from cover art** enabled, the
accent follows the current show's key art instead. The theme can be `Dark`,
`Light` or `System` — `System` follows Windows' light/dark app mode and even
updates live while the app runs. "Compact layout" tightens spacing for smaller
windows.

**Start with Windows** registers the packaged exe in the Windows startup (it
then starts minimized to the tray).

**Backup config** copies `config.json` to your Desktop.

**Launching a site as a PWA**: use the "Launch as PWA" button on the detected
anime card, or the "OPEN A SITE AS PWA" URL field in the controls. Either way
the URL is opened in a standalone, tab-less app window of your configured
browser (Chrome, Brave, Edge or Opera) and is detected by AniPresence like any
other tab. If the browser is already running on the debug port the window is
opened directly via CDP; otherwise it is launched with `--app=<url>`.

### Working with a normal (non-debug) browser

By default AniPresence connects over Chrome's DevTools Protocol, which only
works if the browser was started with `--remote-debugging-port`. To track your
everyday browser tabs instead (any window, any profile, no relaunching), use
the bundled extension:

1. In AniPresence, note the two values logged at startup:
   `Browser-extension bridge listening on 127.0.0.1:<port> (token: <token>)`.
2. Open your browser's extension page (e.g. `chrome://extensions` for
   Chrome/Edge), enable **Developer mode**, then **Load unpacked** and select
   the `browser-extension\` folder (or the `extension` folder, when installed
   from the setup program).
3. Open the extension's **Options** page. The port and token are usually
   filled in automatically (AniPresence writes `settings.json` into the folder
   on startup); hit **Test connection** to confirm. Only if you copied the
   folder elsewhere do you need to type them manually.

The extension reports the active player back to AniPresence, so tabs you open
normally start showing in your Discord presence. It works alongside the CDP
path: whichever reports first each poll wins, so PWA windows and regular tabs
both work.

The bridge only listens on `127.0.0.1` and requires the token (stored in
`config.json` as `extension_token`) in every POST, so another site running in
your browser can't spoof your presence.

**Troubleshooting**: open AniPresence's **Log** tab — it prints
`browser extension connected (token accepted)` on the first good report, or
`extension bridge rejected a request ... (wrong token)` if the token doesn't
match. If you see the "wrong token" line: press **Reload** on the extension in
`chrome://extensions` (its files change every app launch) so it re-reads
`settings.json`, and confirm the folder you loaded is the one sitting next to
the app you're running (repo `browser-extension\` for a source run,
`extension` in the install folder for a setup-program install). The extension's
Options **Test connection** distinguishes "can't reach the app" (app not
running / different port) from "wrong token".

CLI options:

```powershell
python main.py --config .\config.example.json   # use a specific config file
python main.py --port 9223                      # override the debug port
python main.py --debug                          # verbose logging
python main.py --autostart                      # start minimized (used by Windows startup)
```

---

## Configuration

The app reads `%APPDATA%\AniPresence\config.json` on first run (created with
defaults). All keys:

| Key | Default | Meaning |
| --- | --- | --- |
| `discord_application_id` | `""` | Your Discord application ID (required for RPC) |
| `chrome_debugging_port` | `9222` | Chrome remote-debugging port |
| `poll_interval_seconds` | `5` | Detection poll rate |
| `chrome_auto_launch` | `true` | Launch the browser with the debug flag if needed |
| `browser` | `"chrome"` | Browser to detect/launch: `chrome`, `brave`, `edge`, or `opera` |
| `chrome_path` | `""` | Explicit path to the browser exe if auto-detect fails |
| `anilist_cache_ttl_days` | `7` | How long AniList results are cached |
| `anilist_username` | `""` | Public AniList username used for account lists and stats imports |
| `anilist_sync_enabled` | `false` | Two-way AniList sync: pull your list and push progress after an episode is substantially watched |
| `anilist_access_token` | `""` | OAuth bearer token for the sync account (enables private-list reads and progress updates) |
| `anilist_client_id` | `""` | Public OAuth client ID for browser-based AniList login |
| `show_paused_presence` | `true` | Show a "Paused" presence instead of hiding it |
| `presence_episode_names` | `true` | Include detected episode names in Discord |
| `presence_airing_dates` | `true` | Include the next airing date when available |
| `presence_cover_art` | `true` | Include anime cover art in Discord |
| `presence_paused_label` | `true` | Include the Paused text label (timers still stop when disabled) |
| `auto_check_updates` | `true` | Check at startup and every six hours |
| `update_feed_url` | GitHub latest-release API for `Stxrqzy/anipresence-releases` | HTTPS GitHub release API or custom JSON feed |
| `rpc_enabled` | `true` | Master switch for Discord Rich Presence |
| `rpc_broadcast` | `"discord"` | Where to announce activity: `"discord"` (RPC pipe), `"vencord"` (Vencord plugin bridge), or `"both"` |
| `detection_enabled` | `true` | Master switch for Chrome detection |
| `hide_when_no_video` | `true` | Clear the presence when nothing is playing |
| `min_match_score` | `60` | Fuzzy-match confidence floor (logging only) |
| `theme` | `"dark"` | Interface theme: `dark`, `light`, or `system` (follows Windows light/dark) |
| `accent` | `"#7c5cff"` | Accent color as a hex string |
| `auto_accent` | `true` | Sample the accent color from the current cover art |
| `compact` | `false` | Compact layout (tighter spacing/padding) |
| `minimize_to_tray` | `true` | Close hides to the tray instead of quitting |
| `start_windows` | `false` | Register in Windows startup (packaged exe only) |
| `privacy_mode` | `false` | Strip URLs from the on-screen log and recent tooltips |
| `overrides` | `{}` | Per-page "wrong match" corrections (`url -> {title, episode}`) |
| `idle_presence` | `false` | Show a "Browsing anime" presence when nothing is playing |
| `mica_backdrop` | `true` | Frosted-glass backdrop (Mica on Win 11, acrylic on Win 10) |
| `cinematic_backdrop` | `true` | Dim the cover art behind the whole window (cinematic look) |
| `extension_enabled` | `true` | Enable the localhost browser-extension bridge |
| `extension_port` | `8793` | Port the browser extension reports to |
| `extension_token` | `""` | Bridge token (auto-generated on first run; set it in the extension options) |

Logs: `%APPDATA%\AniPresence\logs\anipresence.log`.

### Two-way AniList sync

In Settings → Other Settings, use **Connect AniList** to approve access in your
browser. AniPresence verifies the token's owner, selects that account, and enables
sync. You can also paste an existing token into **AniList access token** and enable
**Keep AniList in sync (two-way)**. Sync runs in both directions:

- **Pull (AniList → app).** Your list is read into the Library and Stats tabs on
  every launch and whenever you press **Sync now**. With a token the read uses
  the authenticated viewer's own list, so it works for private profiles and
  reflects changes you make on AniList; without a token it falls back to the
  active account's public username list.
- **Push (app → AniList).** After an episode reaches 90% playback, AniPresence
  advances the token owner's progress (and marks the entry *Completed* on the
  final episode). Progress is never lowered, and failures retry automatically.

The token belongs to the account that receives updates; the username selector
only controls the public fallback. The token is stored locally in `config.json`;
do not share it.

#### Browser login setup (once)

Create an application at <https://anilist.co/settings/developer> named AniPresence.
Set its redirect URL to **`http://127.0.0.1:8794/callback`** and paste its public
client ID into the Settings field. No client secret is used or bundled. Then
**Connect AniList** opens your default browser, and the returned token is saved
automatically. Login expires after three minutes and can be cancelled. The
loopback listener only runs during login. The app opens a local launch URL which
redirects to AniList with the complete authorization parameters. Disconnect clears the local token;
access may also be revoked from AniList's settings.

The sync panel shows the verified account, last successful sync, pending episode
updates, and errors. **Retry failed syncs** retries completed episodes even after
you leave their tabs, and refreshes the list. Failed episode updates retry after a
60-second cooldown while the app runs. The pending queue lasts for the current app
session. Progress belonging to another token is never retried under a new token.

### Presence controls and extension diagnostics

Settings → Other Settings includes toggles for episode names, next-airing dates,
cover art, and the Paused label. **Keep presence when video is paused** controls
whether the whole activity stays visible. Paused playback never displays a moving
timer, regardless of the label toggle.

Extension diagnostics show the bridge port, last report, background-worker
version, and the reader versions in open tabs. A mismatched worker version calls
for reloading/updating the extension; a stale tab reader calls for refreshing the
anime page. **Open installed extension folder** locates the bundle to load in the
browser. The panel also reports token mismatches and disconnected extensions.

### Automatic update checks and downloads

Settings → Other Settings → App updates checks on startup and every six hours,
notifies you when a newer stable version exists, and offers **Download installer**
with progress followed by **Install update**. Install launches setup and quits the
running app. Downloads go to `%APPDATA%\AniPresence\updates` and are checked against
the feed's SHA-256 digest when one is supplied.

The default feed is the GitHub latest-release API for `Stxrqzy/anipresence-releases`.
**The repository must be public and have a published release with an installer
asset**; pushing a commit/tag alone does not publish the installer. For private
development repositories, host a public HTTPS JSON feed instead:

```json
{
  "version": "2.4.0",
  "url": "https://your-host.example/AniPresence-Setup-2.4.0.exe",
  "sha256": "<64-character SHA-256 hash>"
}
```

`scripts/build.ps1` creates `installer/update.json` with the installer hash and
the expected GitHub release-asset URL. Publish the installer on release `v2.4.0`
(or the version being built), or upload both files to your own host and adjust
the feed URL. This app does not embed GitHub credentials for private releases.

---

## Building an .exe

Use PyInstaller with the checked-in spec:

```powershell
pip install -r requirements.txt pyinstaller
python -m PyInstaller --noconfirm AniPresence.spec
```

The app will be at `dist\AniPresence\AniPresence.exe`. It reads and writes its
config/logs/cache in `%APPDATA%\AniPresence\`, so the folder is portable.

### Making an installer

`scripts\build.ps1` runs both steps and keeps the version in sync:
the version is read from `src\__init__.py` and injected into Inno Setup via
`/DMyAppVersion`. Run it from PowerShell:

```powershell
powershell -ExecutionPolicy Bypass -File scripts\build.ps1
```

This produces `installer\AniPresence-Setup-<version>.exe` — a per-user
installer (no admin needed) with Start-menu/desktop shortcuts and uninstall.
(It also deletes the runtime `settings.json` artifacts from `dist` so the
bridge token is never packaged.)

To step through manually, first build the exe:

```powershell
pip install -r requirements.txt pyinstaller
python -m PyInstaller --noconfirm AniPresence.spec
```

Then compile the Inno Setup script (requires Inno Setup 6,
[jrsoftware.org](https://jrsoftware.org/isinfo.php)) — without the script the
`.iss` uses its hardcoded `MyAppVersion` fallback, so pass one explicitly to
keep it in sync:

```powershell
# e.g. ISCC.exe with Inno Setup installed elsewhere
& "C:\Users\tnahe\AppData\Local\Programs\Inno Setup 6\ISCC.exe" /DMyAppVersion=1.9.5 AniPresence.iss
```

---

## Adding a new website

Two ways, in `src/browser/detectors.py`:

1. **Bespoke parser** (best): subclass `BaseDetector`, set `name` and `hosts`
   (a list of compiled hostname regexes), implement `detect(ctx)` returning a
   `Detection`, and add the instance to `DetectorRegistry.__init__`.
2. **Quick fallback**: add an entry to `KNOWN_SITES` as `(host_regex,
   "Site Name")`. The generic extractor then uses page-title + URL-slug
   heuristics for that host.

`PageContext` gives you `url`, `host`, `path`, `query`, `document_title`, and
`next_data` (Crunchyroll's embedded JSON blob) to build the detection from.

---

## Project status

The project is production-ready in structure; site parsers are the part most
likely to need maintenance as websites change. Contributions and additional
site parsers are welcome.
"# anipresence" 
