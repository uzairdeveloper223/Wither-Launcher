# Wither — Game Launcher
## Complete Project Documentation

> **Purpose of this document:** This file is the single source of truth for the Wither project. It is written to be fully parseable by an LLM used as a coding assistant. Every module is self-contained and references other modules explicitly. No implicit knowledge is assumed.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Module 01 — General Architecture](#2-module-01--general-architecture)
3. [Module 02 — Rust Backend](#3-module-02--rust-backend)
4. [Module 03 — Local Database (SQLite)](#4-module-03--local-database-sqlite)
5. [Module 04 — Design System](#5-module-04--design-system)
6. [Module 05 — Hybrid Launch System](#6-module-05--hybrid-launch-system)
7. [Module 06 — Steam Integration](#7-module-06--steam-integration)
8. [Module 07 — Build and Distribution](#8-module-07--build-and-distribution)
9. [Module 08 — Adding Games](#9-module-08--adding-games)
10. [Module 09 — Steam Integration: Implementation Guide](#10-module-09--steam-integration-implementation-guide)
11. [Module 10 — Silent Steam Launch](#11-module-10--silent-steam-launch)
12. [Module 11 — Steam Store (Custom UI)](#12-module-11--steam-store-custom-ui)
13. [Module 12 — Frontend Stores & App Bootstrap](#13-module-12--frontend-stores--app-bootstrap)

---

## 1. Project Overview

**Name:** Wither  
**Type:** Desktop game launcher (cross-platform)  
**Goal:** A single unified interface to launch any executable and sync with external game libraries (starting with Steam), with minimal RAM and CPU footprint.  
**License:** Open Source (MIT or Apache 2.0 — TBD)  
**Reference product:** GOG Galaxy 2.0 — unified multi-platform library management  
**Key differentiators vs GOG Galaxy:**
- Open source
- Minimal resource usage (~10–30 MB RAM idle)
- Swiss editorial / Apple-inspired UI aesthetic
- No account required for core functionality
- Local-first: all data stored on device in SQLite

### 1.1 Core Principles

- **Performance first.** The app must never be the reason a game loads slowly. Background footprint must be minimal at all times.
- **Local first.** All game metadata, playtime, and settings are stored locally in SQLite. No cloud dependency for core features.
- **Hybrid launching.** Wither detects whether a game requires a platform client (e.g. Steam DRM) and routes accordingly — direct executable launch or protocol URL.
- **Extensible.** The platform integration system is designed to support additional launchers (Epic, GOG, EA App) in future modules.

### 1.2 Platforms Supported

| Platform | Status |
|---|---|
| Windows 10/11 | Primary target |
| macOS 12+ | Supported |
| Linux (Ubuntu 22+, Arch) | Supported |

---

## 2. Module 01 — General Architecture

### 2.1 Technology Stack

| Layer | Technology | Reason |
|---|---|---|
| App shell | Tauri 2.x | Rust backend + system WebView, ~10–30 MB RAM, no bundled Chromium |
| Backend | Rust (stable) | Process management, filesystem access, IPC, system APIs |
| Frontend | Svelte 5 + TypeScript | No virtual DOM, minimal bundle, fast rendering |
| Styling | Plain CSS (no framework) | Full control, no Tailwind overhead |
| Database | SQLite via `rusqlite` | Local, embedded, zero-config |
| IPC | Tauri `invoke` commands | Type-safe Rust↔Svelte communication |

### 2.2 High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│  FRONTEND (Svelte + TypeScript)                         │
│  Runs inside system WebView (WebView2 / WKWebView)      │
│  Responsible for: UI rendering, routing, state          │
├─────────────────────────────────────────────────────────┤
│  IPC BRIDGE (Tauri invoke / events)                     │
│  Type-safe commands and event emitters                  │
├─────────────────────────────────────────────────────────┤
│  RUST BACKEND (Tauri core)                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Process      │  │ Steam Sync   │  │ File Watcher │  │
│  │ Manager      │  │ Module       │  │ Module       │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────────┐                     │
│  │ SQLite DB    │  │ System Tray  │                     │
│  │ Layer        │  │ Manager      │                     │
│  └──────────────┘  └──────────────┘                     │
├─────────────────────────────────────────────────────────┤
│  OPERATING SYSTEM                                       │
│  Native APIs: process spawn, registry (Win), power mgmt │
└─────────────────────────────────────────────────────────┘
```

### 2.3 System Tray Behavior

Wither runs as a **system tray application**. This is the optimal balance between zero-footprint and background functionality.

**States:**

| State | RAM usage | CPU usage | Description |
|---|---|---|---|
| Tray idle | ~12–20 MB | ~0% | UI unloaded, Rust threads running |
| UI open | ~25–40 MB | ~1–3% | WebView loaded, Svelte rendered |
| Game running | ~12–20 MB | ~0.1% | UI auto-hides, process watcher active |

**Behavior rules:**
- On app launch: start Rust backend, initialize DB, register tray icon, begin Steam sync. Do NOT load the WebView yet.
- On tray icon click: load WebView, render UI, restore last view.
- On window close (X button): unload WebView (free RAM), keep Rust backend alive in tray.
- On tray icon right-click menu: show "Open Wither", "Last played: [game]", "Quit".
- On system shutdown: flush any open session to DB, then exit cleanly.

### 2.4 Folder Structure

```
wither/
├── src-tauri/                  # Rust backend
│   ├── src/
│   │   ├── main.rs             # Tauri app entry point
│   │   ├── commands/           # Tauri invoke handlers
│   │   │   ├── library.rs      # get_games, add_game, remove_game
│   │   │   ├── launcher.rs     # launch_game, kill_game
│   │   │   ├── steam.rs        # sync_steam_library, get_steam_games
│   │   │   └── stats.rs        # get_playtime, get_sessions
│   │   ├── db/
│   │   │   ├── mod.rs          # DB init, migrations
│   │   │   └── schema.sql      # Full schema (see Module 03)
│   │   ├── steam/
│   │   │   ├── acf_parser.rs   # Parse .acf manifest files
│   │   │   └── web_api.rs      # Steam Web API client
│   │   ├── process/
│   │   │   ├── launcher.rs     # Spawn executables, get PID
│   │   │   └── watcher.rs      # Monitor PID, track playtime
│   │   ├── tray.rs             # System tray setup and events
│   │   └── watcher/
│   │       └── file_watcher.rs # Watch Steam library dirs for changes
│   ├── Cargo.toml
│   └── tauri.conf.json
├── src/                        # Svelte frontend
│   ├── lib/
│   │   ├── components/         # UI components (see Module 04)
│   │   ├── stores/             # Svelte stores (games, sessions, ui)
│   │   ├── api/                # Tauri invoke wrappers
│   │   └── types/              # TypeScript interfaces
│   ├── routes/                 # SvelteKit-style pages
│   │   ├── +page.svelte        # Home
│   │   ├── library/
│   │   ├── recent/
│   │   ├── stats/
│   │   └── settings/
│   ├── app.css                 # Global CSS, design tokens
│   └── app.html
├── docs/                       # This documentation
└── README.md
```

### 2.5 IPC Command Map

All communication between frontend and backend goes through Tauri `invoke`. The following table is the complete IPC surface.

| Command | Direction | Input | Output | Module |
|---|---|---|---|---|
| `get_all_games` | FE → BE | — | `Game[]` | library |
| `add_custom_game` | FE → BE | `AddGamePayload` | `Game` | library |
| `remove_game` | FE → BE | `game_id: string` | `void` | library |
| `launch_game` | FE → BE | `game_id: string` | `SessionId` | launcher |
| `kill_game` | FE → BE | `game_id: string` | `void` | launcher |
| `sync_steam` | FE → BE | — | `SyncResult` | steam |
| `get_sessions` | FE → BE | `game_id?: string` | `Session[]` | stats |
| `get_playtime` | FE → BE | `game_id: string` | `number` (seconds) | stats |
| `game_launched` | BE → FE | `GameEvent` | — | event |
| `game_exited` | BE → FE | `GameEvent` | — | event |
| `sync_completed` | BE → FE | `SyncResult` | — | event |

---

## 3. Module 02 — Rust Backend

### 3.1 Crate Dependencies

```toml
# src-tauri/Cargo.toml (relevant dependencies)

[dependencies]
tauri = { version = "2", features = ["tray-icon", "system-tray"] }
rusqlite = { version = "0.31", features = ["bundled"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
sysinfo = "0.30"          # Process monitoring
notify = "6"              # File system watching
keyring = "2"             # Secure credential storage
log = "0.4"
env_logger = "0.11"
```

### 3.2 Process Manager

**File:** `src-tauri/src/process/launcher.rs`

The process launcher is responsible for starting game executables and returning a PID for monitoring.

```rust
// Conceptual structure — not final implementation

pub enum LaunchMethod {
    DirectExecutable { path: PathBuf, args: Vec<String> },
    SteamProtocol { app_id: u32 },
}

pub struct LaunchResult {
    pub pid: Option<u32>,       // None if launched via steam:// protocol
    pub session_id: String,
    pub started_at: u64,        // Unix timestamp
}

pub fn launch_game(method: LaunchMethod) -> Result<LaunchResult, LaunchError>;
```

**Launch decision logic:**

```
launch_game(game_id)
     │
     ▼
query DB: does game have drm_type = 'steam'?
     │
  ┌──┴──┐
 Yes    No
  │      │
steam://  std::process::Command::new(executable_path)
rungame    .args(launch_args)
id/{id}    .spawn()
  │      │
  └──┬───┘
     │
spawn watcher thread with PID (or Steam child scan)
     │
     ▼
insert session row in DB (started_at = now, ended_at = NULL)
```

### 3.3 Process Watcher

**File:** `src-tauri/src/process/watcher.rs`

The watcher runs as a lightweight Tokio async task per active game session.

```rust
// Polling interval
const POLL_INTERVAL_SECS: u64 = 5;

// Conceptual watcher loop
pub async fn watch_process(pid: u32, session_id: String, db: Arc<Mutex<Connection>>) {
    let mut system = System::new();
    let start = SystemTime::now();

    loop {
        tokio::time::sleep(Duration::from_secs(POLL_INTERVAL_SECS)).await;
        system.refresh_process(Pid::from(pid as usize));

        if system.process(Pid::from(pid as usize)).is_none() {
            // Process ended — finalize session
            let duration = start.elapsed().unwrap_or_default().as_secs();
            finalize_session(&db, &session_id, duration).await;
            break;
        }
    }
}
```

**Edge cases handled:**

| Case | Handling |
|---|---|
| Normal game exit | Watcher detects PID gone, finalizes session |
| Game crash | Same as normal exit — next poll detects absence |
| Task manager kill | Same as crash |
| System sleep during game | Listen to OS power events, pause timer on sleep, resume on wake |
| Steam child process | Scan children of Steam process, match by executable name from `.acf` |

**Power event handling (per platform):**

- **Windows:** `WM_POWERBROADCAST` via `windows` crate
- **macOS:** `NSWorkspaceWillSleepNotification` via `objc` crate
- **Linux:** `logind` D-Bus signals via `zbus` crate

### 3.4 Steam Sync Module

**File:** `src-tauri/src/steam/`

**Step 1 — Read local `.acf` files (no auth required)**

Steam stores app manifests at:
- Windows: `C:\Program Files (x86)\Steam\steamapps\`
- macOS: `~/Library/Application Support/Steam/steamapps/`
- Linux: `~/.steam/steam/steamapps/`

Each installed game has a file like `appmanifest_570.acf` containing:

```
"AppState"
{
  "appid"     "570"
  "name"      "Dota 2"
  "installdir" "dota 2 beta"
  "SizeOnDisk" "28450938880"
  "StateFlags" "4"
}
```

The parser reads all `.acf` files in all configured Steam library folders (including additional drives, read from `libraryfolders.vdf`).

**Step 2 — Enrich with Steam Web API**

For each AppID found locally, fetch metadata from:

```
GET https://store.steampowered.com/api/appdetails?appids={app_id}
```

Response includes: full name, genres, developer, release date, header image URL, background image URL.

**API key usage:**
```
GET https://api.steampowered.com/IPlayerService/GetOwnedGames/v1/
    ?key={API_KEY}
    &steamid={STEAM_ID}
    &include_appinfo=1
    &include_played_free_games=1
```

Returns total playtime per game from Steam's records. This is used as a **seed value** — Wither then tracks additional time independently.

**Step 3 — Merge into local DB**

For each game from Steam:
1. Check if `game_id = 'steam_{app_id}'` already exists in `games` table
2. If not: insert new row with `source = 'steam'`
3. If yes: update `name`, `cover_url`, `background_url` if changed
4. Never overwrite locally tracked `playtime_seconds`

**Sync schedule:**
- On app launch: full sync (background, non-blocking)
- Every 10 minutes while tray is active: incremental sync (check for new `.acf` files only)
- On Steam library folder change (via file watcher): immediate incremental sync

### 3.5 File Watcher

**File:** `src-tauri/src/watcher/file_watcher.rs`

Uses the `notify` crate to watch Steam's `steamapps/` directory for new or removed `.acf` files. When a change is detected, it triggers an incremental sync and emits a `sync_completed` event to the frontend.

```rust
// Watched paths (resolved at runtime)
// - {steam_path}/steamapps/*.acf  (game installs/uninstalls)
// - {steam_path}/steamapps/libraryfolders.vdf  (new library drives)
```

---

## 4. Module 03 — Local Database (SQLite)

**File:** `src-tauri/src/db/schema.sql`  
**Engine:** SQLite via `rusqlite` (bundled feature — no system SQLite dependency)  
**Location:** Tauri app data directory (`$APPDATA/wither/wither.db` on Windows, `~/.local/share/wither/` on Linux, `~/Library/Application Support/wither/` on macOS)

### 4.1 Full Schema

```sql
-- ─────────────────────────────────────────
-- GAMES
-- Central registry of all games known to Wither.
-- source: 'steam' | 'custom'
-- drm_type: 'steam' | 'none'
-- launch_method: 'steam_protocol' | 'executable'
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS games (
  id                TEXT PRIMARY KEY,   -- 'steam_570' or 'custom_{uuid}'
  name              TEXT NOT NULL,
  source            TEXT NOT NULL,      -- 'steam' | 'custom'
  drm_type          TEXT NOT NULL DEFAULT 'none',
  launch_method     TEXT NOT NULL,
  executable_path   TEXT,               -- NULL if launch_method = 'steam_protocol'
  launch_args       TEXT,               -- JSON array of strings
  steam_app_id      INTEGER,            -- NULL if source != 'steam'
  cover_url         TEXT,               -- Remote URL or local cache path
  background_url    TEXT,
  genre             TEXT,
  developer         TEXT,
  release_year      INTEGER,
  steam_playtime_s  INTEGER DEFAULT 0,  -- Seed from Steam Web API
  is_installed      INTEGER DEFAULT 1,  -- 0 = uninstalled but tracked
  is_favourite      INTEGER DEFAULT 0,
  added_at          INTEGER NOT NULL,   -- Unix timestamp
  last_synced_at    INTEGER
);

-- ─────────────────────────────────────────
-- SESSIONS
-- Every individual play session tracked by Wither.
-- duration_s is the authoritative playtime source.
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS sessions (
  id            TEXT PRIMARY KEY,       -- UUID
  game_id       TEXT NOT NULL REFERENCES games(id) ON DELETE CASCADE,
  started_at    INTEGER NOT NULL,       -- Unix timestamp
  ended_at      INTEGER,                -- NULL if session still active
  duration_s    INTEGER,                -- NULL until session ends
  was_crashed   INTEGER DEFAULT 0       -- 1 if process vanished unexpectedly
);

-- ─────────────────────────────────────────
-- COLLECTIONS
-- User-created playlists / groups of games.
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS collections (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  created_at  INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS collection_games (
  collection_id  TEXT REFERENCES collections(id) ON DELETE CASCADE,
  game_id        TEXT REFERENCES games(id) ON DELETE CASCADE,
  added_at       INTEGER NOT NULL,
  PRIMARY KEY (collection_id, game_id)
);

-- ─────────────────────────────────────────
-- SETTINGS
-- Key-value store for user preferences.
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS settings (
  key    TEXT PRIMARY KEY,
  value  TEXT NOT NULL
);

-- Default settings
INSERT OR IGNORE INTO settings (key, value) VALUES
  ('steam_api_key', ''),
  ('steam_user_id', ''),
  ('steam_path', ''),
  ('sync_interval_minutes', '10'),
  ('launch_on_startup', 'false'),
  ('close_behavior', 'tray'),     -- 'tray' | 'quit'
  ('accent_color', '#D62828');

-- ─────────────────────────────────────────
-- IMAGE CACHE
-- Tracks locally cached cover/background images.
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS image_cache (
  url         TEXT PRIMARY KEY,
  local_path  TEXT NOT NULL,
  cached_at   INTEGER NOT NULL
);

-- ─────────────────────────────────────────
-- INDEXES
-- ─────────────────────────────────────────
CREATE INDEX IF NOT EXISTS idx_sessions_game_id ON sessions(game_id);
CREATE INDEX IF NOT EXISTS idx_sessions_started_at ON sessions(started_at);
CREATE INDEX IF NOT EXISTS idx_games_source ON games(source);
CREATE INDEX IF NOT EXISTS idx_games_last_played ON sessions(started_at DESC);
```

### 4.2 Computed Views

```sql
-- Total playtime per game (Wither-tracked only)
CREATE VIEW IF NOT EXISTS v_playtime AS
SELECT
  game_id,
  COALESCE(SUM(duration_s), 0) AS total_s,
  MAX(started_at)               AS last_played_at,
  COUNT(*)                      AS session_count
FROM sessions
WHERE ended_at IS NOT NULL
GROUP BY game_id;

-- Games enriched with playtime (used by frontend list)
CREATE VIEW IF NOT EXISTS v_games_full AS
SELECT
  g.*,
  COALESCE(p.total_s, 0)        AS wither_playtime_s,
  COALESCE(p.last_played_at, 0) AS last_played_at,
  COALESCE(p.session_count, 0)  AS session_count
FROM games g
LEFT JOIN v_playtime p ON p.game_id = g.id;
```

### 4.3 Migration Strategy

Wither uses a simple integer version migration system. On every app launch, the backend checks `PRAGMA user_version` and runs pending migration files sequentially.

```
src-tauri/src/db/migrations/
  001_initial.sql
  002_add_collections.sql
  003_add_image_cache.sql
  ...
```

---

## 5. Module 04 — Design System

### 5.1 Typography

| Role | Font | Weight | Usage |
|---|---|---|---|
| Wordmark only (`Wither` logo) | JetBrains Mono | 600 | `font-family: 'JetBrains Mono', monospace` |
| Nav counts, playtime stats, monospace data | JetBrains Mono | 500 | `font-family: 'JetBrains Mono', monospace` |
| All UI text — nav labels, body, titles, buttons | Neue Haas Grotesk (Inter as fallback) | 300, 400, 500, 600 | `font-family: 'Neue Haas Grotesk', 'Inter', sans-serif` |

**Font loading in Tauri:** All fonts are bundled as local `.woff2` files in `src/assets/fonts/`. No CDN dependency at runtime.

- **JetBrains Mono** — open source (SIL OFL 1.1), freely distributable. Files: `JetBrainsMono-Medium.woff2`, `JetBrainsMono-SemiBold.woff2`.
- **Neue Haas Grotesk** — commercial license required. Files: `NHaasGroteskTXPro-45Lt.woff2`, `NHaasGroteskTXPro-55Rg.woff2`, `NHaasGroteskTXPro-65Md.woff2`. Fallback: **Inter** (open source, identical Neo-Grotesk structure).
- **Inter** (fallback) — open source (SIL OFL 1.1). Used when Neue Haas Grotesk is not licensed. Files: `Inter-Regular.woff2`, `Inter-Medium.woff2`, `Inter-SemiBold.woff2`.

**Type scale:**

```css
--text-xs:   10px;   /* JetBrains Mono: badges, timestamps, nav counts */
--text-sm:   11px;   /* Sans: filter chips, section labels, status text */
--text-base: 12px;   /* Sans: card titles, nav items, button labels */
--text-md:   13px;   /* Sans: body, list rows */
--text-logo: 16px;   /* JetBrains Mono: wordmark only */
--text-xl:   26px;   /* JetBrains Mono: card placeholder glyphs */
```

**Font smoothing (always applied globally):**
```css
-webkit-font-smoothing: antialiased;
-moz-osx-font-smoothing: grayscale;
```

---

### 5.2 Color Tokens

The color system is monochromatic white-on-black. No color accents — hierarchy is achieved through opacity levels only.

```css
/* ── Backgrounds ── */
--bg-root:       #000000;                    /* App root */
--bg-sidebar:    rgba(8, 8, 8, 0.96);        /* Sidebar — glassmorphism base */
--bg-topbar:     rgba(0, 0, 0, 0.82);        /* Topbar — glassmorphism base */
--bg-s1:         rgba(255, 255, 255, 0.04);  /* Cards, resting inputs */
--bg-s2:         rgba(255, 255, 255, 0.07);  /* Hover states, active nav */
--bg-s3:         rgba(255, 255, 255, 0.11);  /* Active pill tabs, focus states */

/* ── Borders ── */
--border-1:      rgba(255, 255, 255, 0.07);  /* Default dividers, card borders */
--border-2:      rgba(255, 255, 255, 0.13);  /* Emphasis borders, focused inputs */

/* ── Text ── */
--text-1:        #ffffff;                    /* Primary — headings, active items */
--text-2:        rgba(255, 255, 255, 0.50);  /* Secondary — nav labels, metadata */
--text-3:        rgba(255, 255, 255, 0.22);  /* Tertiary — placeholders, hints */

/* ── Semantic (status only) ── */
--status-synced:      #32d74b;               /* Steam sync OK */
--status-syncing:     #f5a623;               /* Sync in progress */
--status-error:       #E53935;               /* Sync failed */
--status-disconnected: rgba(255,255,255,0.25); /* Steam not connected */
```

**Glassmorphism rule:** `backdrop-filter: blur(24px)` is applied **only** to sidebar and topbar. Never to cards, modals, or dropdowns — those use solid `--bg-s1/s2` fills.

---

### 5.3 Spacing and Radius

```css
/* ── Spacing ── */
--space-1:  4px;
--space-2:  8px;
--space-3:  12px;
--space-4:  16px;
--space-5:  20px;
--space-6:  24px;

/* ── Border radius ── */
--radius-sm:   4px;    /* Badges, source chips */
--radius-md:   8px;    /* Dropdown panels, nav hover */
--radius-lg:   10px;   /* Game cards */
--radius-pill: 980px;  /* Search bar, filter chips, Play button, Add game button */
```

**Pill radius (`980px`) is the signature shape of Wither's interactive elements.** Search bar, filter chips, play overlay button, and "Add game" CTA all use `border-radius: 980px`. This creates a consistent, friendly roundness that contrasts with the sharp edges of the cards.

---

### 5.4 Layout Structure

```
Window (min: 960×640)
│
├── Sidebar (220px expanded / 54px collapsed)
│   ├── Header: Wordmark "Wither" (JetBrains Mono) + collapse toggle (< / >)
│   ├── Nav section: Library
│   │   ├── All games (active indicator: 2px white left border)
│   │   ├── Recent
│   │   ├── Favourites
│   │   └── Collections
│   ├── Divider
│   ├── Nav section: Discover
│   │   ├── Steam Store
│   │   └── Statistics
│   ├── Divider
│   ├── Settings (gear icon)
│   └── User row: avatar + username + sync status dot + label
│
└── Main area (flex: 1)
    ├── Topbar (54px)
    │   ├── Search bar (pill, full-radius, black, flex: 1, max 380px)
    │   ├── Filter chips group (multi-select dropdown)
    │   ├── Divider
    │   └── Add game (pill CTA button)
    │
    └── Content (scrollable, padding 20px)
        ├── Section header (title + sort control)
        └── Game grid (5 columns, gap 10px, card aspect-ratio 3/4)
```

---

### 5.5 Sidebar Collapse Behavior

The sidebar collapses from 220px to 54px (icon-only mode) via CSS transition.

```css
.sidebar {
  width: 220px;
  transition: width 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
}
.sidebar.collapsed { width: 54px; }
```

**Toggle icon:** The collapse button shows a `<` chevron when expanded and `>` when collapsed. This is implemented via `transform: scaleX(-1)` on the SVG — no separate icons needed.

```css
.sb-toggle svg {
  transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}
.sidebar.collapsed .sb-toggle svg {
  transform: scaleX(-1);
}
```

**Elements that fade on collapse:** Logo, nav labels, counts, user info — all use `opacity` transition (not `display: none`) to avoid layout flicker.

```css
.sidebar.collapsed .sb-label,
.sidebar.collapsed .sb-logo,
.sidebar.collapsed .sb-uinfo,
.sidebar.collapsed .sb-count,
.sidebar.collapsed .sb-section {
  opacity: 0;
  pointer-events: none;
}
```

**Active indicator:** 2px white left border on the active nav item.

```css
.sb-item.active::before {
  content: '';
  position: absolute;
  left: 0; top: 5px; bottom: 5px;
  width: 2px;
  background: #ffffff;
  border-radius: 0 2px 2px 0;
}
```

---

### 5.6 Sync Status Indicator

The sync status is shown in the sidebar bottom user row as a colored dot + spinning icon + label. Three states exist.

```
States:
  syncing      → dot: #f5a623 (pulse animation) + spinning refresh icon + "Syncing..."
  synced       → dot: #32d74b (solid)            + static icon              + "Steam synced"
  error        → dot: #E53935 (solid)            + static icon              + "Sync failed"
  disconnected → dot: rgba(255,255,255,0.25)     + no icon                  + "Not connected"
```

**CSS animations:**

```css
/* Dot pulse during sync */
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.3; }
}

/* Spinner icon during sync */
@keyframes spin {
  from { transform: rotate(0deg); }
  to   { transform: rotate(360deg); }
}

.sb-dot.syncing    { animation: pulse 1s ease-in-out infinite; }
.sb-sync-icon.spinning { animation: spin 1s linear infinite; }
```

**Svelte store integration:**

```typescript
// src/lib/stores/syncStore.ts
export type SyncState = 'idle' | 'syncing' | 'synced' | 'error' | 'disconnected'

export const syncState = writable<SyncState>('disconnected')
export const lastSyncedAt = writable<number | null>(null)

// On app launch — trigger sync and update state
export async function runStartupSync() {
  syncState.set('syncing')
  try {
    const result = await invoke<SyncResult>('sync_steam')
    lastSyncedAt.set(result.synced_at)
    syncState.set('synced')
  } catch {
    syncState.set('error')
  }
}
```

**`onMount` in `App.svelte`:**

```typescript
import { onMount } from 'svelte'
import { runStartupSync } from '$lib/stores/syncStore'

onMount(() => {
  runStartupSync()
})
```

---

### 5.7 Search Bar

The search bar is a full-radius pill shape, black background, positioned at the far left of the topbar before the filters.

```css
.search-bar {
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.07);
  border-radius: 980px;           /* full pill */
  padding: 7px 16px;
  flex: 1;
  max-width: 380px;
  transition: border-color 0.15s, background 0.15s;
}

.search-bar:focus-within {
  border-color: rgba(255, 255, 255, 0.13);
  background: rgba(255, 255, 255, 0.07);
}

.search-bar input {
  background: transparent;
  border: none;
  outline: none;
  font-family: 'Neue Haas Grotesk', 'Inter', sans-serif;
  font-size: 12px;
  color: #ffffff;
  width: 100%;
}

.search-bar input::placeholder {
  color: rgba(255, 255, 255, 0.22);
}
```

**Svelte component (`SearchBar.svelte`):**

```svelte
<script lang="ts">
  import { createEventDispatcher } from 'svelte'
  const dispatch = createEventDispatcher()
  let value = ''

  function handleInput() {
    dispatch('search', { query: value })
  }
</script>

<div class="search-bar">
  <svg width="13" height="13" viewBox="0 0 13 13" fill="none">
    <circle cx="5.5" cy="5.5" r="4" stroke="rgba(255,255,255,0.22)" stroke-width="1.3"/>
    <path d="M9 9l2.8 2.8" stroke="rgba(255,255,255,0.22)" stroke-width="1.3" stroke-linecap="round"/>
  </svg>
  <input
    bind:value
    on:input={handleInput}
    placeholder="Search games…"
    type="text"
  />
</div>
```

---

### 5.8 Filter System (Multi-Select Dropdown)

Filters are accessed via a single "Filter" pill button in the topbar. Clicking it opens a dropdown panel with checkboxes. Multiple filters can be active simultaneously.

**Available filters:**

| Filter key | Label | Description |
|---|---|---|
| `owned` | Owned | Games in Steam library (including uninstalled) |
| `installed` | Installed | Games currently installed on disk |
| `custom` | Custom | Manually added executables |
| `recent` | Recently played | Played in the last 30 days |
| `favourite` | Favourites | Starred games |

**Filter button states:**

```
No filters active  → "Filter" label, --bg-s1, no border emphasis
1+ filters active  → "Filter · 2" label (count badge), --bg-s3, --border-2
```

**Dropdown panel CSS:**

```css
.filter-dropdown {
  position: absolute;
  top: calc(100% + 8px);
  left: 0;
  background: rgba(18, 18, 18, 0.96);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(255, 255, 255, 0.10);
  border-radius: 12px;
  padding: 8px;
  min-width: 180px;
  z-index: 100;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.6);
}

.filter-option {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 10px;
  border-radius: 7px;
  cursor: pointer;
  font-size: 12px;
  font-weight: 400;
  color: rgba(255, 255, 255, 0.5);
  transition: background 0.12s, color 0.12s;
}

.filter-option:hover  { background: rgba(255,255,255,0.05); color: #fff; }
.filter-option.active { background: rgba(255,255,255,0.08); color: #fff; }

/* Custom checkbox */
.filter-checkbox {
  width: 14px; height: 14px;
  border: 1px solid rgba(255,255,255,0.2);
  border-radius: 4px;
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0;
  transition: background 0.12s, border-color 0.12s;
}
.filter-option.active .filter-checkbox {
  background: #ffffff;
  border-color: #ffffff;
}
```

**Svelte component (`FilterDropdown.svelte`):**

```svelte
<script lang="ts">
  import { createEventDispatcher, onMount } from 'svelte'
  const dispatch = createEventDispatcher()

  export let activeFilters: string[] = []

  const OPTIONS = [
    { key: 'owned',     label: 'Owned' },
    { key: 'installed', label: 'Installed' },
    { key: 'custom',    label: 'Custom' },
    { key: 'recent',    label: 'Recently played' },
    { key: 'favourite', label: 'Favourites' },
  ]

  let open = false

  function toggle(key: string) {
    if (activeFilters.includes(key)) {
      activeFilters = activeFilters.filter(f => f !== key)
    } else {
      activeFilters = [...activeFilters, key]
    }
    dispatch('change', { filters: activeFilters })
  }

  function handleOutsideClick(e: MouseEvent) {
    if (!(e.target as Element).closest('.filter-wrapper')) open = false
  }

  onMount(() => {
    document.addEventListener('click', handleOutsideClick)
    return () => document.removeEventListener('click', handleOutsideClick)
  })
</script>

<div class="filter-wrapper" style="position: relative;">
  <button class="tb-btn" class:active={activeFilters.length > 0} on:click={() => open = !open}>
    <svg width="12" height="12" viewBox="0 0 12 12" fill="none" stroke="currentColor" stroke-width="1.3">
      <path d="M1 3h10M3 6h6M5 9h2"/>
    </svg>
    Filter{activeFilters.length > 0 ? ` · ${activeFilters.length}` : ''}
  </button>

  {#if open}
    <div class="filter-dropdown">
      {#each OPTIONS as opt}
        <div
          class="filter-option"
          class:active={activeFilters.includes(opt.key)}
          on:click={() => toggle(opt.key)}
        >
          <div class="filter-checkbox">
            {#if activeFilters.includes(opt.key)}
              <svg width="9" height="9" viewBox="0 0 9 9" fill="none" stroke="#000" stroke-width="1.6">
                <path d="M1.5 4.5l2 2 4-4" stroke-linecap="round" stroke-linejoin="round"/>
              </svg>
            {/if}
          </div>
          {opt.label}
        </div>
      {/each}
    </div>
  {/if}
</div>
```

**Filter logic (applied in the games store):**

```typescript
// src/lib/stores/libraryStore.ts

export function applyFilters(games: Game[], filters: string[]): Game[] {
  if (filters.length === 0) return games

  return games.filter(game => {
    return filters.every(f => {
      switch (f) {
        case 'owned':     return true                         // all library games are owned
        case 'installed': return game.is_installed
        case 'custom':    return game.source === 'custom'
        case 'recent':    return game.last_played_at > (Date.now()/1000) - 30*24*3600
        case 'favourite': return game.is_favourite
        default:          return true
      }
    })
  })
}
```

Filters use AND logic — a game must satisfy **all** active filters to appear.

---

### 5.9 Component Inventory (Updated)

| Component | File | Description |
|---|---|---|
| `GameCard` | `GameCard.svelte` | 3/4 aspect ratio, cover, title, genre, stats, hover Play pill |
| `NavItem` | `NavItem.svelte` | Sidebar row: icon + label + count, active left-border indicator |
| `SidebarToggle` | `SidebarToggle.svelte` | `<` / `>` chevron, scaleX(-1) flip animation |
| `SearchBar` | `SearchBar.svelte` | Full-radius pill, black bg, magnifier icon |
| `FilterDropdown` | `FilterDropdown.svelte` | Multi-select dropdown with custom checkboxes |
| `SyncStatusDot` | `SyncStatusDot.svelte` | Colored dot: synced/syncing/error/disconnected states |
| `SyncIcon` | `SyncIcon.svelte` | Spinning refresh icon during sync |
| `SourceBadge` | `SourceBadge.svelte` | Glassmorphism badge top-right of card ("Steam", "Custom") |
| `PlayOverlay` | `PlayOverlay.svelte` | Frosted glass pill overlay on card hover |
| `SectionHeader` | `SectionHeader.svelte` | Title (Inter 600) + sort/view control |
| `AddGameButton` | `AddGameButton.svelte` | Pill CTA: `+ Add game`, triggers file picker |

---

### 5.10 Animation Principles

All animations in Wither are functional — they communicate state, not decoration.

| Animation | Property | Duration | Easing | Trigger |
|---|---|---|---|---|
| Sidebar collapse/expand | `width` | `300ms` | `cubic-bezier(0.4,0,0.2,1)` | Toggle click |
| Sidebar label fade | `opacity` | `180ms` | `ease` | Sidebar collapse |
| Toggle icon flip | `transform: scaleX(-1)` | `300ms` | `cubic-bezier(0.4,0,0.2,1)` | Sidebar collapse |
| Card hover scale | `transform: scale(1.04)` | `220ms` | `ease` | Mouse enter |
| Play overlay | `opacity: 0 → 1` | `180ms` | `ease` | Mouse enter |
| Sync dot pulse | `opacity` | `1000ms` | `ease-in-out` | Syncing state |
| Sync icon spin | `transform: rotate(360deg)` | `1000ms` | `linear` | Syncing state |
| Filter dropdown | `opacity + translateY(-4px → 0)` | `150ms` | `ease-out` | Filter button click |
| Search focus glow | `border-color + background` | `150ms` | `ease` | Input focus |

**No entrance animations, no page transitions, no loading skeletons.** Data either exists or shows a placeholder glyph. Speed is the feature.

---

## 6. Module 05 — Hybrid Launch System

### 6.1 Decision Logic

Every game in the database has a `launch_method` field set at import time.

```
launch_game(game_id)
        │
        ▼
  query DB for game row
        │
   ┌────┴────┐
steam_protocol  executable
        │           │
open steam://    std::process::Command
rungameid/{id}   ::new(&game.executable_path)
        │        .args(&launch_args)
        │        .spawn()
        │           │
        │        get PID directly
        │           │
   scan Steam    ───┘
   child procs
   for game exe
        │
        ▼
  spawn watcher thread (Module 02.3)
        │
        ▼
  insert sessions row (started_at = now, ended_at = NULL)
        │
        ▼
  emit game_launched event to frontend
```

### 6.2 DRM Detection

When importing a Steam game, Wither sets `drm_type` based on the following heuristic:

1. Check Steam Web API `appdetails` response for `"drm_notice"` field
2. Check if the executable exists as a standalone (DRM-free games can be launched directly)
3. Default to `'steam'` DRM if uncertain — safer behavior

Games added manually via "Add custom game" always get `drm_type = 'none'` and `launch_method = 'executable'`.

### 6.3 Playtime Tracking for Non-Steam Games

Non-Steam games are launched via direct executable. The PID is obtained immediately from `Child::id()` after `Command::spawn()`.

**Watcher behavior for direct executables:**

```
PID obtained at spawn
        │
loop every 5 seconds:
        │
  sysinfo::System::refresh_process(pid)
        │
  process exists?
  ┌─────┴─────┐
 Yes          No
  │            │
sleep 5s    calculate duration:
             ended_at - started_at (Unix timestamps)
             │
            update sessions row:
             ended_at = now()
             duration_s = calculated
             │
            emit game_exited to frontend
```

**Sleep/wake handling:**

```
OS power event: SLEEP
        │
save checkpoint: {session_id, elapsed_so_far_s}
pause timer

OS power event: WAKE
        │
resume timer from checkpoint
do NOT count sleep time as playtime
```

### 6.4 Adding a Custom Game (UX Flow)

See Module 08 for the complete "Adding Games" specification, which covers all three addition modes: Steam auto-sync, custom executable, and third-party launcher auto-discovery.

---

## 7. Module 06 — Steam Integration

### 7.1 What is Required from the User

| Requirement | Reason | Optional? |
|---|---|---|
| Steam installed on machine | To read `.acf` files and use `steam://` protocol | No |
| Steam API Key | To fetch metadata (covers, genres) | Yes — library works without it, no metadata |
| Steam User ID (SteamID64) | To fetch owned games list | Yes — only installed games shown without it |

Steam API Key is free: `https://steamcommunity.com/dev/apikey`
Steam ID lookup: `https://steamid.io`

Both values are stored in the `settings` table (not in environment variables, not in plaintext config files).

### 7.2 Data Retrieved per Game

| Field | Source | Notes |
|---|---|---|
| App ID | `.acf` file | Always available |
| Install path | `.acf` file | Always available |
| Game name | `.acf` file + Web API | ACF is faster, API is more accurate |
| Cover image | Steam CDN | `https://cdn.cloudflare.steamstatic.com/steam/apps/{id}/library_600x900.jpg` |
| Background image | Steam CDN | `https://cdn.cloudflare.steamstatic.com/steam/apps/{id}/library_hero.jpg` |
| Genre | Steam Web API | Requires API key |
| Developer | Steam Web API | Requires API key |
| Steam playtime | Steam Web API | Used as seed only — Wither tracks independently |
| Is installed | `.acf` StateFlags field | `4` = fully installed |

### 7.3 Image Caching

All remote images are downloaded once and cached locally:

```
$APP_DATA/wither/image_cache/
  steam_570_cover.jpg
  steam_570_hero.jpg
  custom_{uuid}_cover.jpg
```

The `image_cache` DB table maps remote URL → local path. Before fetching, always check the cache. Cache is never invalidated automatically — user can clear it in Settings.

### 7.4 Sync Result Type

```typescript
// TypeScript interface returned to frontend after sync
interface SyncResult {
  added: number;       // New games found
  updated: number;     // Existing games with updated metadata
  removed: number;     // Games whose .acf was deleted (uninstalled)
  errors: string[];    // Non-fatal errors (e.g. API rate limit)
  synced_at: number;   // Unix timestamp
}
```

---

## 8. Module 07 — Build and Distribution

### 8.1 Build Requirements

| Tool | Version | Purpose |
|---|---|---|
| Rust | stable (1.78+) | Backend compilation |
| Node.js | 20 LTS | Frontend build toolchain |
| pnpm | 9+ | Package manager |
| Tauri CLI | 2.x | App bundling |

### 8.2 Build Commands

```bash
# Development
pnpm tauri dev

# Production build (current platform)
pnpm tauri build

# Production build outputs:
# Windows:  src-tauri/target/release/bundle/msi/Wither_x.x.x_x64.msi
#           src-tauri/target/release/bundle/nsis/Wither_x.x.x_x64-setup.exe
# macOS:    src-tauri/target/release/bundle/dmg/Wither_x.x.x_x64.dmg
# Linux:    src-tauri/target/release/bundle/deb/wither_x.x.x_amd64.deb
#           src-tauri/target/release/bundle/appimage/wither_x.x.x_amd64.AppImage
```

### 8.3 Bundle Size Targets

| Platform | Target binary size | RAM idle (tray) | RAM active (UI open) |
|---|---|---|---|
| Windows | < 8 MB | < 20 MB | < 40 MB |
| macOS | < 8 MB | < 20 MB | < 40 MB |
| Linux | < 6 MB | < 15 MB | < 35 MB |

### 8.4 CI/CD (GitHub Actions)

Three workflow files:

```
.github/workflows/
  ci.yml          # Run on every PR: cargo test, cargo clippy, svelte-check
  build.yml       # Run on tag push: build all platforms, upload artifacts
  release.yml     # Run on GitHub release publish: attach binaries
```

Cross-compilation strategy:
- Windows build: runs on `windows-latest` runner
- macOS build: runs on `macos-latest` runner (universal binary: x64 + arm64)
- Linux build: runs on `ubuntu-22.04` runner

### 8.5 Auto-Update

Tauri's built-in updater is used. Update manifest hosted on GitHub Releases.

```json
// tauri.conf.json (relevant section)
{
  "updater": {
    "active": true,
    "endpoints": [
      "https://github.com/wither-app/wither/releases/latest/download/latest.json"
    ],
    "dialog": true,
    "pubkey": "{{ TAURI_PUBLIC_KEY }}"
  }
}
```

Update check happens once on app launch (background, non-blocking). User is notified via a subtle badge in the Settings nav item — no forced modal dialogs.

---

## 9. Module 08 — Adding Games

This module covers all three methods by which games enter the Wither library. Every method produces a row in the `games` table with consistent fields — the source of the game is transparent to the launch and tracking systems.

### 8.1 Method 1 — Steam Auto-Sync (automatic)

No user action required beyond initial setup.

**Trigger:** On app launch and every 10 minutes while tray is active (see Module 02.4, Module 06).

**Flow:**
```
App starts
     │
     ▼
Rust reads all .acf files from steamapps/ directories
     │
     ▼
For each AppID found:
  - game already in DB? → skip or update metadata
  - game not in DB?     → fetch metadata from Steam Web API
                          → download and cache cover + background image
                          → insert row with source = 'steam'
     │
     ▼
emit sync_completed → frontend re-renders library
```

**What the user sees:** Games appear automatically in the library with cover art, genre, and developer info. No interaction needed.

**Failure modes:**
- No API key set → games added with name only, no cover art, no genre
- Steam not installed → Steam sync skipped entirely, user notified in Settings
- API rate limit (100k requests/day, practically never hit) → logged, retried on next sync cycle

---

### 8.2 Method 2 — Custom Executable (manual, single game)

Used for: DRM-free games, emulators, indie games not on Steam, any `.exe` or binary the user wants to track.

**Entry point:** "+ Add game" button in the sidebar.

**Full UX flow:**

```
User clicks "+ Add game"
     │
     ▼
Native file picker opens (Tauri dialog::open())
Filter: *.exe (Windows), no filter (macOS/Linux)
     │
     ▼
User selects executable
     │
     ▼
Frontend sends add_custom_game IPC command:
  { executable_path: "/path/to/game.exe" }
     │
     ▼
Backend infers game name from filename:
  "TheWitcher3.exe"     → "The Witcher 3"
  "hollow_knight.exe"   → "Hollow Knight"
  "DOOM_Eternal.exe"    → "Doom Eternal"
  (strip extension, replace _ and -, title case)
     │
     ▼
Backend queries SteamGridDB API by inferred name
  → returns list of candidate artworks
  → automatically picks highest-rated result
  → downloads and caches cover + background
     │
     ▼
Frontend shows "Add Game" confirmation modal:
  - Editable name field (pre-filled with inferred name)
  - Cover art preview (from SteamGridDB or placeholder)
  - "Change cover" button (opens image picker for manual override)
  - Optional: launch arguments field
     │
     ▼
User confirms → backend inserts game row:
  source = 'custom'
  drm_type = 'none'
  launch_method = 'executable'
  id = 'custom_{uuid}'
     │
     ▼
emit sync_completed → game appears in library
```

**Name inference rules (Rust):**

```rust
fn infer_name_from_path(path: &Path) -> String {
    let stem = path.file_stem()
        .unwrap_or_default()
        .to_string_lossy();

    // Replace separators with spaces
    let spaced = stem.replace(['_', '-', '.'], " ");

    // Title case each word
    spaced.split_whitespace()
        .map(|w| {
            let mut c = w.chars();
            match c.next() {
                None => String::new(),
                Some(f) => f.to_uppercase().to_string() + c.as_str(),
            }
        })
        .collect::<Vec<_>>()
        .join(" ")
}
```

---

### 8.3 Method 3 — Third-Party Launcher Auto-Discovery (semi-automatic)

Used for: Epic Games, GOG, EA App, Ubisoft Connect — launchers without a full Wither integration yet.

Wither scans known installation directories for these launchers and proposes found games to the user in bulk.

**Scanned paths per platform:**

| Launcher | Windows | macOS | Linux |
|---|---|---|---|
| Epic Games | `C:\Program Files\Epic Games\` | `~/Library/Application Support/Epic/` | `~/.local/share/Epic/` |
| GOG | `C:\GOG Games\` | `/Applications/` (GOG prefix) | `~/.local/share/GOG/` |
| EA App | `C:\Program Files\EA Games\` | — | — |
| Ubisoft Connect | `C:\Program Files (x86)\Ubisoft\Ubisoft Game Launcher\games\` | — | — |

Additional paths can be added by the user in Settings → "Custom scan paths".

**Discovery flow:**

```
User clicks "Scan for games" in Settings
  (or triggered automatically on first launch if no games found)
     │
     ▼
Backend scans all known paths
  For each subdirectory found:
    - look for main executable (largest .exe, or .sh on Linux)
    - infer game name from directory name
     │
     ▼
Frontend shows "Games found" modal:
  List of discovered games with checkboxes
  Each row: [checkbox] [inferred name] [path] [source badge]
  "Select all" / "Deselect all" controls
     │
     ▼
User checks desired games → clicks "Add selected"
     │
     ▼
For each selected game:
  - query SteamGridDB for cover art by name
  - insert row in games table (source = 'custom', drm_type = 'none')
     │
     ▼
emit sync_completed → games appear in library
```

**Executable detection heuristic (per discovered directory):**

```rust
fn find_main_executable(dir: &Path) -> Option<PathBuf> {
    // 1. Look for exe matching directory name (most reliable)
    // 2. Look for exe in root of directory (not in subdirs)
    // 3. Pick largest exe by file size (usually the main binary)
    // 4. Exclude: uninstall*.exe, setup*.exe, redist*, vc_redist*
}
```

---

### 8.4 SteamGridDB Integration

SteamGridDB (`https://www.steamgriddb.com`) is a community-maintained database of game artwork for any game — not just Steam titles. It is the primary source of cover art for custom and third-party games.

**API key:** Free with registration at `https://www.steamgriddb.com/api/v2`. Stored in `settings` table under key `steamgriddb_api_key`.

**Endpoints used:**

```
# Search game by name → returns game ID candidates
GET https://www.steamgriddb.com/api/v2/search/autocomplete/{name}

# Get grids (cover art, portrait 600×900) for a game
GET https://www.steamgriddb.com/api/v2/grids/game/{game_id}
    ?dimensions=600x900
    &styles=material,alternate
    &limit=5

# Get heroes (background art, 1920×620) for a game
GET https://www.steamgriddb.com/api/v2/heroes/game/{game_id}
    &limit=3
```

**Selection strategy:** Always pick the image with the highest `score` field (community upvotes). Download it, store in `image_cache`, record path in `games.cover_url`.

**Fallback chain for cover art:**

```
1. SteamGridDB (by game name search)          — works for ~95% of known games
2. Steam CDN (if game has a known Steam AppID) — exact match
3. IGDB (future integration)                  — broader database
4. User-provided image (manual override)       — always available
5. Placeholder (IBM Plex Mono initials + color gradient) — always fallback
```

**Rust client structure:**

```rust
// src-tauri/src/steamgriddb/client.rs

pub struct SteamGridDbClient {
    api_key: String,
    http: reqwest::Client,
    cache: Arc<Mutex<Connection>>,
}

impl SteamGridDbClient {
    pub async fn search_game(&self, name: &str) -> Result<Vec<SgdbGame>>;
    pub async fn get_grid(&self, game_id: u64) -> Result<Option<SgdbImage>>;
    pub async fn get_hero(&self, game_id: u64) -> Result<Option<SgdbImage>>;
    pub async fn download_and_cache(&self, image: &SgdbImage, local_key: &str) -> Result<PathBuf>;
}
```

**Rate limits:** SteamGridDB free tier allows 50 requests/minute. Wither batches artwork fetches and adds a 100ms delay between requests to stay within limits.

---

### 8.5 IPC Commands for Game Addition

| Command | Input | Output | Description |
|---|---|---|---|
| `add_custom_game` | `AddGamePayload` | `Game` | Add single executable manually |
| `scan_third_party` | `ScanPayload?` | `DiscoveredGame[]` | Scan known directories for games |
| `confirm_discovered_games` | `string[]` (IDs) | `Game[]` | Bulk-add selected discovered games |
| `search_artwork` | `{ name: string }` | `ArtworkCandidate[]` | Search SteamGridDB for cover options |
| `set_custom_cover` | `{ game_id, image_path }` | `void` | Override cover with user image |

**TypeScript types:**

```typescript
export interface AddGamePayload {
  executable_path: string;
  name?: string;              // If omitted, inferred from filename
  cover_path?: string;        // If omitted, fetched from SteamGridDB
  launch_args?: string[];
}

export interface DiscoveredGame {
  temp_id: string;            // Temporary ID before user confirms
  inferred_name: string;
  executable_path: string;
  source_hint: 'epic' | 'gog' | 'ea' | 'ubisoft' | 'unknown';
  cover_url: string | null;   // Pre-fetched from SteamGridDB if found
}

export interface ArtworkCandidate {
  url: string;
  width: number;
  height: number;
  score: number;
  style: string;
}

export interface ScanPayload {
  custom_paths?: string[];    // Additional paths to scan beyond defaults
}
```

---

### 8.6 Settings Required for Game Addition

| Setting key | Description | Required for |
|---|---|---|
| `steamgriddb_api_key` | SteamGridDB API key | Cover art for custom/discovered games |
| `steam_api_key` | Steam Web API key | Cover art and metadata for Steam games |
| `custom_scan_paths` | JSON array of extra directories to scan | Third-party discovery |
| `auto_scan_on_launch` | Boolean — run discovery scan on every launch | Third-party discovery |

All configurable in Settings → "Library" section.

---

## Appendix A — TypeScript Type Definitions

```typescript
// src/lib/types/index.ts

export interface Game {
  id: string;
  name: string;
  source: 'steam' | 'custom';
  drm_type: 'steam' | 'none';
  launch_method: 'steam_protocol' | 'executable';
  executable_path: string | null;
  steam_app_id: number | null;
  cover_url: string | null;
  background_url: string | null;
  genre: string | null;
  developer: string | null;
  release_year: number | null;
  is_installed: boolean;
  is_favourite: boolean;
  added_at: number;
  // From v_games_full view
  wither_playtime_s: number;
  last_played_at: number;
  session_count: number;
}

export interface Session {
  id: string;
  game_id: string;
  started_at: number;
  ended_at: number | null;
  duration_s: number | null;
  was_crashed: boolean;
}

export interface AddGamePayload {
  name: string;
  executable_path: string;
  cover_path?: string;
  launch_args?: string[];
}

export interface SyncResult {
  added: number;
  updated: number;
  removed: number;
  errors: string[];
  synced_at: number;
}

export interface GameEvent {
  game_id: string;
  session_id: string;
  timestamp: number;
}
```

---

## Appendix B — Key Design Decisions Log

| Decision | Rationale |
|---|---|
| Tauri over Electron | ~10x less RAM, no bundled Chromium, Rust backend for native performance |
| Svelte over React/Vue | No virtual DOM, smallest bundle, fastest render for a tool app |
| SQLite over JSON files | Structured queries, views, indexes; handles libraries of 1000+ games efficiently |
| System tray (not always-on) | Zero RAM when UI closed; background sync via lightweight Rust threads only |
| IBM Plex Mono for all display text | Monospace conveys precision and tooling identity; free and open source |
| Hybrid launch (not executable-only) | Most Steam games require Steamworks DRM; bypassing it causes crashes |
| Local playtime tracking (not Steam-only) | Enables tracking for custom/DRM-free games; works offline; user owns their data |
| Image cache on disk | Eliminates redundant network requests; app works fully offline after first sync |
| No account system | Reduces complexity; privacy-first; local-first philosophy |
| SteamGridDB for custom cover art | Only public database with community artwork for any game, not just Steam titles; free API |
| Name inference from filename | Eliminates need for user to type game name manually; handles 90% of cases correctly |
| Third-party launcher auto-discovery | Scanning known paths is faster and less error-prone than asking the user to navigate manually |
| Bulk confirmation UI for discovered games | User retains control — Wither never silently adds games without explicit approval |
| JetBrains Mono for wordmark only | Creates strong identity anchor without making the entire UI feel like a terminal |
| Neue Haas Grotesk (Inter fallback) for UI | Premium Neo-Grotesk with excellent screen rendering; Inter is metrically identical for fallback |
| Pill radius (980px) for interactive elements | Consistent roundness signature for search bar, filters, buttons — contrasts with sharp card edges |
| Multi-select filter dropdown over tab row | Tabs consume horizontal space and allow only single selection; dropdown supports AND-logic multi-filter |
| Glassmorphism on sidebar/topbar only | Blur on cards causes rendering jank during scroll; reserved for fixed chrome elements only |
| Sync status dot in sidebar (not toast/banner) | Persistent ambient indicator — never interrupts the user, always visible at a glance |
| `<` / `>` chevron for sidebar toggle | More legible than hamburger at small sizes; direction communicates state unambiguously |

---

## 10. Module 09 — Steam Integration: Implementation Guide

This module is the complete technical reference for implementing Steam login, library sync, and metadata fetching inside Wither. It is written to be directly usable by a coding LLM to implement the feature from scratch.

---

### 9.1 Overview: Three Separate Systems

Steam integration in Wither consists of three independent systems that work together:

| System | Purpose | Auth required |
|---|---|---|
| **OpenID 2.0 Login** | Authenticate the user, obtain their SteamID64 | No API key — browser redirect |
| **Steam Web API** | Fetch owned games list, playtime seed, metadata | API key + SteamID64 |
| **Local .acf Parser** | Read installed games directly from filesystem | None |

These three systems are independent. The `.acf` parser works with zero credentials. The Web API enriches data. OpenID is only needed if the user wants to log in with their Steam account to get their full owned-games list (including uninstalled games).

---

### 9.2 System 1 — Steam OpenID 2.0 Login

#### 9.2.1 How Steam OpenID Works

Steam implements **OpenID 2.0** (not OpenID Connect — they are incompatible). The flow is:

```
Wither opens a local HTTP server on port 14069 (or any free port)
     │
     ▼
Wither constructs the Steam OpenID redirect URL
     │
     ▼
Tauri opens the URL in the system browser (shell::open)
     │
     ▼
User logs in on steamcommunity.com
     │
     ▼
Steam redirects browser to http://localhost:14069/callback?openid.*=...
     │
     ▼
Wither's local server receives the callback
     │
     ▼
Wither verifies the response with Steam (direct server-to-server call)
     │
     ▼
Wither extracts SteamID64 from the Claimed ID
     │
     ▼
Local server shuts down
Wither saves SteamID64 to settings table
     │
     ▼
Proceed to Web API sync (Section 9.3)
```

#### 9.2.2 Constructing the OpenID Redirect URL

The redirect URL must be constructed with exact OpenID 2.0 parameters.

```rust
// src-tauri/src/steam/openid.rs

const STEAM_OPENID_ENDPOINT: &str = "https://steamcommunity.com/openid/login";
const LOCAL_CALLBACK_PORT: u16 = 14069;

pub fn build_auth_url() -> String {
    let callback = format!("http://localhost:{}/auth/steam/callback", LOCAL_CALLBACK_PORT);

    let params = [
        ("openid.ns",         "http://specs.openid.net/auth/2.0"),
        ("openid.mode",       "checkid_setup"),
        ("openid.return_to",  &callback),
        ("openid.realm",      &format!("http://localhost:{}", LOCAL_CALLBACK_PORT)),
        ("openid.identity",   "http://specs.openid.net/auth/2.0/identifier_select"),
        ("openid.claimed_id", "http://specs.openid.net/auth/2.0/identifier_select"),
    ];

    let query = params.iter()
        .map(|(k, v)| format!("{}={}", k, urlencoding::encode(v)))
        .collect::<Vec<_>>()
        .join("&");

    format!("{}?{}", STEAM_OPENID_ENDPOINT, query)
}
```

**Required Rust crates:**
```toml
urlencoding = "2"
tiny_http = "0.12"   # lightweight local HTTP server
```

#### 9.2.3 Receiving and Verifying the Callback

When Steam redirects back to `localhost:14069/auth/steam/callback`, the URL contains OpenID parameters including the claimed identity. You must **verify** this with Steam before trusting it.

```rust
pub async fn handle_callback(query_params: HashMap<String, String>) -> Result<u64, AuthError> {
    // 1. Check openid.mode == "id_res"
    if query_params.get("openid.mode") != Some(&"id_res".to_string()) {
        return Err(AuthError::Cancelled);
    }

    // 2. Verify with Steam (required — never skip this step)
    let mut verify_params = query_params.clone();
    verify_params.insert("openid.mode".to_string(), "check_authentication".to_string());

    let client = reqwest::Client::new();
    let response = client
        .post("https://steamcommunity.com/openid/login")
        .form(&verify_params)
        .send()
        .await?
        .text()
        .await?;

    // Steam returns "is_valid:true" in the response body if valid
    if !response.contains("is_valid:true") {
        return Err(AuthError::InvalidSignature);
    }

    // 3. Extract SteamID64 from claimed_id
    // Format: https://steamcommunity.com/openid/id/76561198XXXXXXXXX
    let claimed_id = query_params
        .get("openid.claimed_id")
        .ok_or(AuthError::MissingClaimedId)?;

    let steam_id: u64 = claimed_id
        .split('/')
        .last()
        .ok_or(AuthError::InvalidClaimedId)?
        .parse()
        .map_err(|_| AuthError::InvalidClaimedId)?;

    Ok(steam_id)
}
```

#### 9.2.4 Local HTTP Server Lifecycle

The local server must start before the browser is opened and shut down after the callback is received (or after a 2-minute timeout).

```rust
pub async fn run_auth_flow(app_handle: tauri::AppHandle) -> Result<u64, AuthError> {
    // 1. Start local server
    let server = tiny_http::Server::http(format!("0.0.0.0:{}", LOCAL_CALLBACK_PORT))
        .map_err(|_| AuthError::PortInUse)?;

    // 2. Build and open auth URL in browser
    let auth_url = build_auth_url();
    tauri::api::shell::open(&app_handle.shell_scope(), auth_url, None)?;

    // 3. Wait for callback (timeout: 120 seconds)
    let request = tokio::time::timeout(
        Duration::from_secs(120),
        tokio::task::spawn_blocking(move || server.recv())
    ).await
        .map_err(|_| AuthError::Timeout)??
        .map_err(|_| AuthError::ServerError)?;

    // 4. Parse query params from request URL
    let query_params = parse_query(request.url());

    // 5. Send 200 OK back to browser ("You can close this tab")
    let response = tiny_http::Response::from_string(
        "<html><body><h2>Logged in. You can close this tab.</h2></body></html>"
    ).with_header("Content-Type: text/html".parse().unwrap());
    request.respond(response)?;

    // 6. Verify and return SteamID64
    handle_callback(query_params).await
}
```

#### 9.2.5 Tauri Command Exposed to Frontend

```rust
// src-tauri/src/commands/steam.rs

#[tauri::command]
pub async fn steam_login(app_handle: tauri::AppHandle, db: tauri::State<'_, DbPool>) -> Result<SteamUser, String> {
    let steam_id = run_auth_flow(app_handle).await
        .map_err(|e| e.to_string())?;

    // Fetch basic profile info
    let user = fetch_player_summary(steam_id).await
        .map_err(|e| e.to_string())?;

    // Save to settings
    db.save_setting("steam_user_id", &steam_id.to_string()).await?;
    db.save_setting("steam_username", &user.personaname).await?;
    db.save_setting("steam_avatar_url", &user.avatar).await?;

    Ok(user)
}
```

**Frontend call (Svelte):**
```typescript
import { invoke } from '@tauri-apps/api/core';

async function loginWithSteam() {
  try {
    const user = await invoke<SteamUser>('steam_login');
    // user.personaname, user.steamid, user.avatar
    steamUser.set(user);
    await invoke('sync_steam'); // trigger library sync
  } catch (e) {
    console.error('Steam login failed:', e);
  }
}
```

---

### 9.3 System 2 — Steam Web API

#### 9.3.1 Base URL and Authentication

All requests go to:
```
https://api.steampowered.com/{Interface}/{Method}/v{Version}/
```

Authentication is via query parameter — the API key is **never** sent as a header.

```
?key=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

The API key is read from the `settings` table at runtime. It is never hardcoded.

#### 9.3.2 Endpoint: GetPlayerSummaries

Fetches basic profile info for one or more SteamIDs.

```
GET https://api.steampowered.com/ISteamUser/GetPlayerSummaries/v2/
    ?key={API_KEY}
    &steamids={STEAM_ID64}
```

**Response fields used by Wither:**

```json
{
  "response": {
    "players": [{
      "steamid":     "76561198XXXXXXXXX",
      "personaname": "username",
      "avatar":      "https://...jpg",        // 32x32
      "avatarmedium":"https://...jpg",        // 64x64
      "avatarfull":  "https://...jpg",        // 184x184
      "profileurl":  "https://steamcommunity.com/id/...",
      "communityvisibilitystate": 3           // 3 = public
    }]
  }
}
```

`communityvisibilitystate: 3` means the profile is public. If it is not 3, the library will be inaccessible via API — inform the user.

#### 9.3.3 Endpoint: GetOwnedGames

Returns the full list of games owned by the user.

```
GET https://api.steampowered.com/IPlayerService/GetOwnedGames/v1/
    ?key={API_KEY}
    &steamid={STEAM_ID64}
    &include_appinfo=1
    &include_played_free_games=1
    &skip_unvetted_apps=false
    &format=json
```

**Parameters:**

| Parameter | Value | Description |
|---|---|---|
| `include_appinfo` | `1` | Include game name and icon URL in response |
| `include_played_free_games` | `1` | Include free-to-play games the user has played |
| `skip_unvetted_apps` | `false` | Include games not yet reviewed by Steam (e.g. some regional titles) |

**Response fields used per game:**

```json
{
  "response": {
    "game_count": 42,
    "games": [{
      "appid":             570,
      "name":              "Dota 2",
      "playtime_forever":  12400,    // minutes — used as seed value
      "playtime_2weeks":   240,      // minutes played in last 2 weeks
      "img_icon_url":      "abc123"  // icon hash — build URL manually
    }]
  }
}
```

**Icon URL construction:**
```
https://media.steampowered.com/steamcommunity/public/images/apps/{appid}/{img_icon_url}.jpg
```

**Cover and background image URLs (not from this endpoint — constructed by AppID):**
```
Cover (portrait 600×900):
https://cdn.cloudflare.steamstatic.com/steam/apps/{appid}/library_600x900.jpg

Background (hero 1920×620):
https://cdn.cloudflare.steamstatic.com/steam/apps/{appid}/library_hero.jpg

Header (landscape 460×215):
https://cdn.cloudflare.steamstatic.com/steam/apps/{appid}/header.jpg
```

These URLs are deterministic — no API call needed to construct them. Always try the `library_600x900.jpg` first; if it 404s, fall back to `header.jpg`.

#### 9.3.4 Endpoint: GetAppDetails (Store API)

Fetches rich metadata for a single game. Note: this is a **Store API**, not the Web API — different base URL, no key required.

```
GET https://store.steampowered.com/api/appdetails
    ?appids={APPID}
    &filters=basic,genres,release_date,developers
```

**Response:**
```json
{
  "570": {
    "success": true,
    "data": {
      "name":         "Dota 2",
      "genres":       [{"id": "1", "description": "Action"}],
      "developers":   ["Valve"],
      "release_date": {"date": "9 Jul, 2013"},
      "is_free":      true
    }
  }
}
```

**Important:** This endpoint is **rate-limited aggressively** — roughly 200 requests per 5 minutes. Wither must cache this data permanently and never re-fetch unless the user explicitly requests a metadata refresh.

#### 9.3.5 Endpoint: GetRecentlyPlayedGames

Returns up to 10 games played in the last 2 weeks.

```
GET https://api.steampowered.com/IPlayerService/GetRecentlyPlayedGames/v1/
    ?key={API_KEY}
    &steamid={STEAM_ID64}
    &count=10
```

Used by Wither to populate the "Recently played" section on the Home view, cross-referenced with local session data.

#### 9.3.6 Privacy Settings — Critical Edge Cases

| Profile state | `GetOwnedGames` result | Wither behavior |
|---|---|---|
| Public profile, public library | Full game list returned | Normal sync |
| Public profile, private library | Empty response `{}` | Show warning: "Set your game details to Public in Steam Privacy Settings" |
| Private profile | Empty response `{}` | Show warning: "Your Steam profile is set to Private" |
| Public profile, private playtime | Games returned, all `playtime_forever = 0` | Sync games without playtime seed |

Wither must detect the empty response case and distinguish it from a network error. Check: if `response.game_count` is missing or 0 after a successful HTTP 200, it is a privacy issue, not an error.

---

### 9.4 System 3 — Local .acf File Parser

This system requires zero credentials and works completely offline. It is always run first, before any API call.

#### 9.4.1 Locating Steam Library Directories

Steam's main library path is platform-dependent:

```rust
pub fn get_steam_root() -> Option<PathBuf> {
    #[cfg(target_os = "windows")]
    {
        // Check registry first
        // HKEY_LOCAL_MACHINE\SOFTWARE\WOW6432Node\Valve\Steam → InstallPath
        // Fallback: default path
        Some(PathBuf::from("C:/Program Files (x86)/Steam"))
    }
    #[cfg(target_os = "macos")]
    {
        dirs::home_dir().map(|h| h.join("Library/Application Support/Steam"))
    }
    #[cfg(target_os = "linux")]
    {
        // Check multiple locations in order
        let candidates = vec![
            dirs::home_dir().map(|h| h.join(".steam/steam")),
            dirs::home_dir().map(|h| h.join(".local/share/Steam")),
        ];
        candidates.into_iter().flatten().find(|p| p.exists())
    }
}
```

Steam allows **multiple library folders** (e.g. on additional drives). These are listed in:
```
{steam_root}/steamapps/libraryfolders.vdf
```

Parse this file to get all library paths — each contains its own `steamapps/` directory.

```
// libraryfolders.vdf format (simplified)
"libraryfolders"
{
    "0"  "C:\\Program Files (x86)\\Steam"
    "1"  "D:\\SteamLibrary"
    "2"  "E:\\Games\\Steam"
}
```

#### 9.4.2 Parsing .acf Manifest Files

Each installed game has a file named `appmanifest_{appid}.acf` in the `steamapps/` directory.

```
// appmanifest_570.acf
"AppState"
{
    "appid"           "570"
    "Universe"        "1"
    "name"            "Dota 2"
    "StateFlags"      "4"
    "installdir"      "dota 2 beta"
    "LastUpdated"     "1710000000"
    "SizeOnDisk"      "28450938880"
    "buildid"         "13450000"
}
```

**StateFlags values:**

| Value | Meaning |
|---|---|
| `4` | Fully installed |
| `6` | Update required |
| `1026` | Downloading |
| other | Partially installed / broken |

Only import games with `StateFlags = 4` or `6`.

**Executable path construction:**
```
{library_path}/steamapps/common/{installdir}/
```

The actual executable name is not in the `.acf` file. Wither finds it by:
1. Looking for an exe matching the game name
2. Falling back to the largest `.exe` in the root directory
3. For Steam DRM games, using `steam://rungameid/{appid}` regardless

```rust
// src-tauri/src/steam/acf_parser.rs

pub struct AcfGame {
    pub app_id:     u32,
    pub name:       String,
    pub install_dir: PathBuf,
    pub state_flags: u32,
    pub last_updated: u64,
    pub size_on_disk: u64,
}

pub fn parse_acf_file(path: &Path) -> Result<AcfGame, ParseError> {
    let content = std::fs::read_to_string(path)?;
    // Simple key-value extraction — no full VDF parser needed
    // for the fields Wither needs
    let app_id     = extract_value(&content, "appid")?.parse()?;
    let name       = extract_value(&content, "name")?;
    let install_dir = extract_value(&content, "installdir")?;
    let state_flags = extract_value(&content, "StateFlags")?.parse()?;
    // ...
    Ok(AcfGame { app_id, name, install_dir: PathBuf::from(install_dir), state_flags, .. })
}

pub fn scan_library(steam_root: &Path) -> Vec<AcfGame> {
    let steamapps = steam_root.join("steamapps");
    std::fs::read_dir(&steamapps)
        .into_iter()
        .flatten()
        .flatten()
        .filter(|e| e.file_name().to_string_lossy().starts_with("appmanifest_"))
        .filter_map(|e| parse_acf_file(&e.path()).ok())
        .filter(|g| g.state_flags == 4 || g.state_flags == 6)
        .collect()
}
```

---

### 9.5 Full Sync Flow

This is the complete sync sequence that runs on app launch and every 10 minutes.

```
sync_steam()
     │
     ├─ Step 1: Scan all .acf files → get list of installed AppIDs
     │
     ├─ Step 2: For each AppID:
     │    ├─ Already in DB?
     │    │   Yes → check if name/installdir changed → update if needed
     │    │   No  → mark as "needs enrichment"
     │    │
     │    └─ Is game no longer in .acf (uninstalled)?
     │        → set is_installed = 0 in DB (do NOT delete — preserve playtime)
     │
     ├─ Step 3: If API key + SteamID available:
     │    ├─ Call GetOwnedGames → get full owned list (including uninstalled)
     │    ├─ For each owned game not yet in DB → add with is_installed = 0
     │    └─ Update steam_playtime_s seed for all games
     │
     ├─ Step 4: For each game needing enrichment (new games only):
     │    ├─ Fetch cover: try library_600x900.jpg → fallback header.jpg
     │    ├─ Fetch hero: library_hero.jpg
     │    ├─ Call GetAppDetails for genre + developer (rate-limited: add 300ms delay)
     │    └─ Save to image_cache table + update games row
     │
     └─ Step 5: Emit sync_completed event to frontend
          → { added, updated, removed, errors, synced_at }
```

**Rust module structure for sync:**

```rust
// src-tauri/src/steam/sync.rs

pub struct SyncResult {
    pub added:   u32,
    pub updated: u32,
    pub removed: u32,
    pub errors:  Vec<String>,
}

pub async fn run_full_sync(
    db: Arc<Mutex<Connection>>,
    settings: &Settings,
    app_handle: &tauri::AppHandle,
) -> SyncResult {
    let mut result = SyncResult::default();

    // Step 1+2: Local scan (always runs)
    if let Some(steam_root) = get_steam_root() {
        let local_games = scan_all_libraries(&steam_root);
        result += merge_local_games(&db, &local_games).await;
    }

    // Step 3+4: API enrichment (only if credentials available)
    if !settings.steam_api_key.is_empty() && !settings.steam_user_id.is_empty() {
        let owned = fetch_owned_games(&settings.steam_api_key, &settings.steam_user_id).await;
        match owned {
            Ok(games) => result += merge_owned_games(&db, &games, &settings).await,
            Err(e)    => result.errors.push(e.to_string()),
        }
    }

    // Emit event to frontend
    app_handle.emit_all("sync_completed", &result).ok();

    result
}
```

---

### 9.6 Credentials Storage

Steam credentials (API key, SteamID64, username) are stored in the `settings` SQLite table, not in environment variables or plaintext config files.

For the API key specifically, use the OS keychain via the `keyring` crate for additional security:

```rust
// Store API key in OS keychain
keyring::Entry::new("wither", "steam_api_key")?.set_password(&api_key)?;

// Read API key from OS keychain
let api_key = keyring::Entry::new("wither", "steam_api_key")?.get_password()?;
```

OS keychain locations:
- **Windows:** Windows Credential Manager
- **macOS:** Keychain Access
- **Linux:** Secret Service (GNOME Keyring / KWallet)

Non-sensitive settings (SteamID64, username, avatar URL) are stored directly in the `settings` SQLite table.

---

### 9.7 Settings UI — Steam Section

The Settings page "Steam" section must expose:

| Setting | Input type | Description |
|---|---|---|
| Steam Web API Key | Password input + link to `steamcommunity.com/dev/apikey` | Required for metadata and owned games list |
| Steam Account | Read-only display (avatar + username) + "Login / Change account" button | Populated after OpenID login |
| Profile visibility warning | Inline warning if `communityvisibilitystate != 3` | Guides user to fix privacy settings |
| Manual SteamID | Text input (fallback if OpenID not used) | For advanced users who know their SteamID64 |
| Sync now | Button | Triggers immediate full sync |
| Last synced | Timestamp display | Shows when last sync completed |

---

### 9.8 Error Handling Reference

| Error condition | HTTP status | Wither response |
|---|---|---|
| Invalid API key | 403 | Show "Invalid Steam API key" in Settings with link to get a new one |
| Private profile / library | 200 + empty `{}` | Show "Your Steam library is set to private" with instructions |
| Rate limited (Store API) | 429 | Log error, skip metadata for this sync cycle, retry next cycle |
| Steam servers down | 5xx | Log error, show "Steam sync failed — will retry" in status bar |
| OpenID timeout (user didn't log in) | — | Close local server, show "Login cancelled" |
| Port 14069 in use | — | Try ports 14070–14079 before failing |
| `.acf` parse error | — | Skip that game, log warning, continue with others |

---

### 9.9 New IPC Commands (this module)

| Command | Input | Output | Description |
|---|---|---|---|
| `steam_login` | — | `SteamUser` | Run OpenID flow, return user profile |
| `steam_logout` | — | `void` | Clear SteamID and credentials from DB |
| `get_steam_user` | — | `SteamUser \| null` | Return currently logged-in Steam user |
| `sync_steam` | — | `SyncResult` | Run full sync (local + API) |
| `get_sync_status` | — | `SyncStatus` | Return last sync time and result |

**New TypeScript types:**

```typescript
export interface SteamUser {
  steamid:      string;     // SteamID64 as string
  personaname:  string;     // Display name
  avatar:       string;     // 32x32 avatar URL
  avatarfull:   string;     // 184x184 avatar URL
  profileurl:   string;
  is_public:    boolean;    // communityvisibilitystate === 3
}

export interface SyncStatus {
  last_synced_at: number | null;   // Unix timestamp, null if never synced
  is_syncing:     boolean;
  last_result:    SyncResult | null;
}
```

---

*Document version: 1.1.0 — Added Module 09: Steam Integration Implementation.*  
*Last updated: March 2026*

---

## 11. Module 10 — Silent Steam Launch

This module specifies how Wither launches Steam invisibly, monitors the game, and shuts Steam down automatically when the game exits.

---

### 10.1 Pre-Launch Check: Was Steam Already Running?

Before doing anything, Wither records whether Steam was already running. This determines whether Wither is responsible for closing it afterward.

```rust
pub fn steam_is_running() -> bool {
    let mut system = System::new();
    system.refresh_processes();
    system.processes_by_name("steam")
        .any(|p| !p.name().to_lowercase().contains("steamwebhelper"))
}
```

This check is done **once** at launch time and stored in the session context. Never re-checked mid-session.

---

### 10.2 Launching Steam Silently

```rust
pub async fn launch_steam_silent() -> Result<Child, LaunchError> {
    let steam_exe = find_steam_executable()?;

    let child = std::process::Command::new(&steam_exe)
        .arg("-silent")        // No main window on startup
        .arg("-nochatui")      // No chat/friends popup
        .arg("-nofriendsui")   // No friends list window
        .arg("-noreactlogin")  // Skip login UI if already logged in
        .spawn()
        .map_err(LaunchError::SpawnFailed)?;

    Ok(child)
}
```

**Flag reference:**

| Flag | Effect |
|---|---|
| `-silent` | Starts Steam minimized to tray, no main window |
| `-nochatui` | Disables the chat/messaging popup |
| `-nofriendsui` | Disables the friends list overlay |
| `-noreactlogin` | Skips the new React-based login screen if already logged in |

---

### 10.3 Waiting for Steam to be Ready

Steam is ready when `steamwebhelper` process appears (signals IPC layer is up).

```rust
pub async fn wait_for_steam_ready(timeout: Duration) -> Result<(), LaunchError> {
    let deadline = std::time::Instant::now() + timeout;

    loop {
        if std::time::Instant::now() > deadline {
            return Err(LaunchError::SteamTimeout);
        }

        let mut system = System::new();
        system.refresh_processes();

        let ready = system.processes_by_name("steamwebhelper").count() > 0;

        if ready {
            // Extra buffer: Steam needs ~1.5s after webhelper appears
            tokio::time::sleep(Duration::from_millis(1500)).await;
            return Ok(());
        }

        tokio::time::sleep(Duration::from_millis(500)).await;
    }
}
```

**Timeout:** 20 seconds. Report error to frontend if exceeded.

---

### 10.4 Complete Launch Flow with Frontend Events

```rust
pub async fn launch_steam_game(
    game: &Game,
    db: Arc<Mutex<Connection>>,
    app: &tauri::AppHandle,
) -> Result<String, LaunchError> {

    let steam_was_running = steam_is_running();

    // Step 1: Start Steam if needed
    if !steam_was_running {
        app.emit_all("game_launch_state", json!({ "status": "starting_steam", "game_id": game.id })).ok();
        launch_steam_silent().await?;
        app.emit_all("game_launch_state", json!({ "status": "waiting_for_steam", "game_id": game.id })).ok();
        wait_for_steam_ready(Duration::from_secs(20)).await?;
    }

    // Step 2: Launch game via steam:// protocol
    app.emit_all("game_launch_state", json!({ "status": "launching_game", "game_id": game.id })).ok();
    let app_id = game.steam_app_id.ok_or(LaunchError::MissingAppId)?;
    open::that(format!("steam://rungameid/{}", app_id)).map_err(LaunchError::ProtocolFailed)?;

    // Step 3: Detect game PID (scan child processes, match exe name)
    let game_pid = wait_for_game_process(game, Duration::from_secs(30)).await?;

    // Step 4: Insert session row
    let session_id = uuid::Uuid::new_v4().to_string();
    let started_at = unix_now();
    db.lock().unwrap().execute(
        "INSERT INTO sessions (id, game_id, started_at) VALUES (?1, ?2, ?3)",
        rusqlite::params![session_id, game.id, started_at],
    )?;

    app.emit_all("game_launch_state", json!({
        "status": "running",
        "game_id": game.id,
        "session_id": session_id,
        "pid": game_pid,
        "started_at": started_at
    })).ok();

    // Step 5: Spawn background watcher (non-blocking)
    let app_c = app.clone(); let db_c = db.clone();
    let sid = session_id.clone(); let gid = game.id.clone();
    tokio::spawn(async move {
        watch_and_cleanup(game_pid, sid, gid, steam_was_running, started_at, db_c, app_c).await;
    });

    Ok(session_id)
}
```

---

### 10.5 Game Process Detection

```rust
async fn wait_for_game_process(game: &Game, timeout: Duration) -> Result<u32, LaunchError> {
    let deadline = std::time::Instant::now() + timeout;
    let exe_hints = get_exe_hints(game); // from .acf installdir

    loop {
        if std::time::Instant::now() > deadline {
            return Err(LaunchError::GameProcessNotFound);
        }
        let mut system = System::new();
        system.refresh_processes();
        for (pid, process) in system.processes() {
            let name = process.name().to_lowercase();
            if exe_hints.iter().any(|h| name.contains(h)) {
                return Ok(pid.as_u32());
            }
        }
        tokio::time::sleep(Duration::from_millis(1000)).await;
    }
}
```

---

### 10.6 Watcher and Automatic Steam Shutdown

```rust
async fn watch_and_cleanup(
    game_pid: u32, session_id: String, game_id: String,
    steam_was_running: bool, started_at: u64,
    db: Arc<Mutex<Connection>>, app: tauri::AppHandle,
) {
    loop {
        tokio::time::sleep(Duration::from_secs(5)).await;
        let mut system = System::new();
        system.refresh_process(sysinfo::Pid::from(game_pid as usize));

        if system.process(sysinfo::Pid::from(game_pid as usize)).is_none() {
            let ended_at = unix_now();
            let duration_s = ended_at - started_at;

            db.lock().unwrap().execute(
                "UPDATE sessions SET ended_at=?1, duration_s=?2 WHERE id=?3",
                rusqlite::params![ended_at, duration_s, session_id],
            ).ok();

            app.emit_all("game_launch_state", json!({
                "status": "exited",
                "game_id": game_id,
                "duration_s": duration_s
            })).ok();

            // Only shut down Steam if Wither was the one that opened it
            if !steam_was_running {
                // Use Steam's own shutdown command — clean exit
                let _ = std::process::Command::new("steam").arg("-shutdown").spawn();
                tokio::time::sleep(Duration::from_secs(10)).await;
            }

            break;
        }
    }
}
```

---

### 10.7 Frontend Launch State Machine

```typescript
export type LaunchStatus =
  | 'idle'
  | 'starting_steam'
  | 'waiting_for_steam'
  | 'launching_game'
  | 'running'
  | 'exited'
  | 'error'

export const STATUS_LABELS: Record<LaunchStatus, string> = {
  idle:              '',
  starting_steam:    'Starting Steam...',
  waiting_for_steam: 'Initializing...',
  launching_game:    'Loading game...',
  running:           'Running',
  exited:            'Session saved',
  error:             'Launch failed',
}
```

**GameCard.svelte rendering logic:**

```
game.id === launchState.game_id ?
  ├─ 'starting_steam'    → spinner + "Starting Steam..."
  ├─ 'waiting_for_steam' → spinner + "Initializing..."
  ├─ 'launching_game'    → spinner + "Loading game..."
  ├─ 'running'           → green dot + live elapsed timer (client-side from started_at)
  ├─ 'exited'            → "Session saved" flash (2s) → reset to idle
  └─ 'error'             → red icon + error message
```

---

### 10.8 Sleep/Wake Handling During Active Session

```
OS power event: SLEEP
  → checkpoint = { session_id, elapsed_s: unix_now() - started_at }
  → pause watcher loop

OS power event: WAKE
  → session.started_at += sleep_duration
  → resume watcher loop
  → effect: sleep time excluded from duration_s
```

| OS | API | Crate |
|---|---|---|
| Windows | `WM_POWERBROADCAST` (`PBT_APMSUSPEND` / `PBT_APMRESUMEAUTOMATIC`) | `windows` |
| macOS | `NSWorkspaceWillSleepNotification` / `NSWorkspaceDidWakeNotification` | `objc2` |
| Linux | `logind` D-Bus `PrepareForSleep` signal | `zbus` |

---

## 12. Module 11 — Steam Store (Custom UI)

This module implements a native Wither-style Steam Store using the unofficial Steam Storefront API — the same endpoints used by the Steam client itself (Big Picture mode). No API key required.

**Disclaimer:** These endpoints are unofficial and undocumented by Valve. They are stable but may change without notice. All responses must be cached aggressively to handle potential downtime.

---

### 11.1 Common Query Parameters

All endpoints are on `https://store.steampowered.com/api/`. No authentication required.

| Parameter | Example | Description |
|---|---|---|
| `cc` | `it`, `us`, `de` | Country code — determines currency |
| `l` | `english`, `italian` | Language for text fields |

Country code is auto-detected from system locale at first launch, stored in `settings`, with manual override in Settings → Store.

---

### 11.2 Endpoint: `featured`

Returns games currently featured on the Steam homepage.

```
GET https://store.steampowered.com/api/featured/?cc={cc}&l={l}
```

**Key response fields:**
```json
{
  "large_capsules": [{
    "id": 2767030,
    "name": "Game Name",
    "discounted": true,
    "discount_percent": 20,
    "original_price": 5999,
    "final_price": 4799,
    "currency": "EUR",
    "large_capsule_image": "https://...jpg",
    "header_image": "https://...jpg"
  }],
  "featured_win": [ /* same structure */ ],
  "specials": { "items": [ /* discounted games */ ] }
}
```

Used for: **Store home hero carousel**.

---

### 11.3 Endpoint: `featuredcategories`

Returns curated lists: top sellers, new releases, specials, coming soon.

```
GET https://store.steampowered.com/api/featuredcategories/?cc={cc}&l={l}
```

```json
{
  "top_sellers":  { "items": [ /* app objects */ ] },
  "new_releases": { "items": [ /* app objects */ ] },
  "specials":     { "items": [ /* discounted games */ ] },
  "coming_soon":  { "items": [ /* unreleased games */ ] }
}
```

Used for: **Store category tabs**.

---

### 11.4 Endpoint: `appdetails`

Full metadata for a single game. One AppID per request.
Rate limit: ~200 requests / 5 minutes — always check cache first.

```
GET https://store.steampowered.com/api/appdetails/
    ?appids={APPID}&cc={cc}&l={l}
    &filters=basic,price_overview,genres,developers,release_date,screenshots,movies
```

**Key fields:**
```json
{
  "570": {
    "success": true,
    "data": {
      "steam_appid": 570,
      "name": "Dota 2",
      "short_description": "...",
      "header_image": "https://...jpg",
      "developers": ["Valve"],
      "is_free": true,
      "price_overview": {
        "initial_formatted": "29,99€",
        "final_formatted":   "23,99€",
        "discount_percent":  20
      },
      "genres": [{ "description": "Action" }],
      "release_date": { "coming_soon": false, "date": "9 Jul, 2013" },
      "screenshots": [{ "path_thumbnail": "https://...jpg", "path_full": "https://...jpg" }],
      "movies": [{ "mp4": { "480": "https://...mp4", "max": "https://...mp4" } }]
    }
  }
}
```

Used for: **Game detail page**.

---

### 11.5 Endpoint: `search/results` (Search + Browse)

```
# Search by query
GET https://store.steampowered.com/search/results/
    ?term={query}&json=1&cc={cc}&l={l}&count=20&start=0

# Browse by category
GET https://store.steampowered.com/search/results/
    ?filter={filter}&json=1&cc={cc}&l={l}&count=20&start=0
```

| `filter` value | Tab |
|---|---|
| `topsellers` | Top Sellers |
| `newreleases` | New Releases |
| `specials` | Specials / Sales |
| `popularupcoming` | Popular Upcoming |

**Response:**
```json
{
  "total_count": 5000,
  "items": [{
    "name": "Elden Ring",
    "logo": "https://...jpg",
    "price": "49,99€",
    "sale_price": "39,99€",
    "id": 1245620
  }]
}
```

Used for: **Search bar** (300ms debounce) and **category browsing** with pagination.

---

### 11.6 Store UI Structure

```
Store view
├── Search bar (top, always visible)
├── Tab: Featured       ← /api/featured
├── Tab: Top Sellers    ← /api/search?filter=topsellers
├── Tab: New Releases   ← /api/search?filter=newreleases
└── Tab: Specials       ← /api/search?filter=specials
```

**Store card layout:**
```
┌──────────────────┐
│  [cover 3:4]     │
├──────────────────┤
│  Game Name       │  IBM Plex Mono 12px 500
│  Action RPG      │  IBM Plex Sans 10px --text2
│                  │
│  49,99€          │  normal price
│  ~~59,99€~~ -20% │  if discounted: strikethrough + red badge
│  [In Library]    │  if already owned: badge instead of price
└──────────────────┘
```

---

### 11.7 Caching Strategy

| Data | TTL | Storage |
|---|---|---|
| `featured` | 1 hour | SQLite `store_cache` |
| `featuredcategories` | 1 hour | SQLite `store_cache` |
| `appdetails` | 24 hours | SQLite `store_cache` |
| Search results | 15 minutes | In-memory only |
| Cover images | Forever | `image_cache` table |

```sql
CREATE TABLE IF NOT EXISTS store_cache (
  key        TEXT PRIMARY KEY,  -- e.g. 'featured:it', 'appdetails:570:it'
  value      TEXT NOT NULL,     -- JSON blob
  cached_at  INTEGER NOT NULL
);
```

---

### 11.8 "Buy" Button Behavior

Purchasing never happens inside Wither. The "Buy" button opens the game's Steam store page in the system browser.

```typescript
async function openInSteamStore(appId: number) {
  await invoke('open_url', { url: `https://store.steampowered.com/app/${appId}` });
}
```

Button label: **"View in Steam Store →"** — same pattern used by Playnite and Heroic Launcher.

---

### 11.9 New IPC Commands (this module)

| Command | Input | Output | Description |
|---|---|---|---|
| `launch_steam_game` | `{ game_id: string }` | `{ session_id: string }` | Silent Steam launch with events |
| `store_get_featured` | — | `FeaturedResponse` | Home page featured games |
| `store_get_categories` | — | `FeaturedCategoriesResponse` | Category lists |
| `store_get_app` | `{ app_id: number }` | `AppDetails` | Full game detail |
| `store_search` | `{ query: string, page: number }` | `SearchResults` | Search |
| `store_browse` | `{ filter: string, page: number }` | `SearchResults` | Browse by category |
| `open_url` | `{ url: string }` | `void` | Open URL in system browser |

---

---

## 13. Module 12 — Frontend Stores & App Bootstrap

This module is the complete specification for the Svelte frontend state layer. It defines every store, how they are initialized on app launch, and how they connect to the IPC commands defined in previous modules.

---

### 12.1 Store Architecture Overview

```
App.svelte (root)
     │
     onMount()
     │
     ├── syncStore.runStartupSync()     → triggers Steam sync on every launch
     ├── libraryStore.loadGames()       → fetches all games from SQLite
     └── settingsStore.load()           → loads user preferences
          │
          ▼
     Reactive derived stores
     ├── filteredGames    ← libraryStore.games + filterStore.active + searchStore.query
     ├── syncStatusLabel  ← syncStore.state + syncStore.lastSyncedAt
     └── storeGames       ← storeStore.featured (lazy, loaded on Store tab open)
```

All stores are in `src/lib/stores/`. Every store exposes a typed writable or derived. No global mutable state outside of stores.

---

### 12.2 `syncStore.ts` — Steam Sync State

Manages the full lifecycle of Steam sync: startup trigger, progress state, result, and error.

```typescript
// src/lib/stores/syncStore.ts
import { writable, derived } from 'svelte/store'
import { invoke } from '@tauri-apps/api/core'
import { listen } from '@tauri-apps/api/event'
import type { SyncResult } from '$lib/types'

export type SyncState = 'disconnected' | 'syncing' | 'synced' | 'error'

export const syncState     = writable<SyncState>('disconnected')
export const lastSyncedAt  = writable<number | null>(null)
export const syncError     = writable<string | null>(null)
export const syncResult    = writable<SyncResult | null>(null)

// Human-readable status label for sidebar display
export const syncLabel = derived(
  [syncState, lastSyncedAt],
  ([$state, $last]) => {
    switch ($state) {
      case 'syncing':      return 'Syncing...'
      case 'synced':       return $last ? `Synced` : 'Synced'
      case 'error':        return 'Sync failed'
      case 'disconnected': return 'Not connected'
    }
  }
)

// Dot color token for SyncStatusDot component
export const syncDotClass = derived(syncState, $state => ({
  disconnected: 'disconnected',
  syncing:      'syncing',
  synced:       'synced',
  error:        'error',
}[$state]))

// Called once on app mount
export async function runStartupSync(): Promise<void> {
  // Check if Steam credentials exist
  const apiKey = await invoke<string>('get_setting', { key: 'steam_api_key' })
  if (!apiKey) {
    syncState.set('disconnected')
    return
  }

  syncState.set('syncing')
  syncError.set(null)

  try {
    const result = await invoke<SyncResult>('sync_steam')
    syncResult.set(result)
    lastSyncedAt.set(result.synced_at)
    syncState.set('synced')

    // If sync had non-fatal errors, still show synced but log them
    if (result.errors.length > 0) {
      console.warn('[wither] Sync completed with warnings:', result.errors)
    }
  } catch (e: unknown) {
    syncState.set('error')
    syncError.set(String(e))
  }
}

// Listen for backend-emitted sync events (from file watcher / periodic sync)
export async function initSyncListener(): Promise<void> {
  await listen<SyncResult>('sync_completed', (event) => {
    syncResult.set(event.payload)
    lastSyncedAt.set(event.payload.synced_at)
    syncState.set('synced')
  })
}
```

---

### 12.3 `libraryStore.ts` — Game Library

Manages the full games list, search query, active filters, and derived filtered view.

```typescript
// src/lib/stores/libraryStore.ts
import { writable, derived, get } from 'svelte/store'
import { invoke } from '@tauri-apps/api/core'
import type { Game } from '$lib/types'

// ── Raw data ──────────────────────────────────────────────────
export const games         = writable<Game[]>([])
export const isLoading     = writable<boolean>(false)

// ── Search ────────────────────────────────────────────────────
export const searchQuery   = writable<string>('')

// ── Filters ───────────────────────────────────────────────────
export const activeFilters = writable<string[]>([])

// ── Sort ──────────────────────────────────────────────────────
export type SortKey = 'name' | 'hours' | 'last_played' | 'added'
export const sortKey       = writable<SortKey>('name')
export const sortDir       = writable<'asc' | 'desc'>('asc')

// ── Derived: filtered + sorted view ───────────────────────────
export const filteredGames = derived(
  [games, searchQuery, activeFilters, sortKey, sortDir],
  ([$games, $query, $filters, $sort, $dir]) => {
    let result = $games

    // 1. Search filter (name match, case-insensitive)
    if ($query.trim()) {
      const q = $query.toLowerCase()
      result = result.filter(g => g.name.toLowerCase().includes(q))
    }

    // 2. Active filters (AND logic — game must satisfy all)
    if ($filters.length > 0) {
      const now = Math.floor(Date.now() / 1000)
      result = result.filter(g =>
        $filters.every(f => {
          switch (f) {
            case 'owned':     return true
            case 'installed': return g.is_installed
            case 'custom':    return g.source === 'custom'
            case 'recent':    return g.last_played_at > now - 30 * 24 * 3600
            case 'favourite': return g.is_favourite
            default:          return true
          }
        })
      )
    }

    // 3. Sort
    result = [...result].sort((a, b) => {
      let cmp = 0
      switch ($sort) {
        case 'name':        cmp = a.name.localeCompare(b.name); break
        case 'hours':       cmp = a.wither_playtime_s - b.wither_playtime_s; break
        case 'last_played': cmp = a.last_played_at - b.last_played_at; break
        case 'added':       cmp = a.added_at - b.added_at; break
      }
      return $dir === 'asc' ? cmp : -cmp
    })

    return result
  }
)

// ── Actions ───────────────────────────────────────────────────

export async function loadGames(): Promise<void> {
  isLoading.set(true)
  try {
    const result = await invoke<Game[]>('get_all_games')
    games.set(result)
  } finally {
    isLoading.set(false)
  }
}

export async function addCustomGame(payload: AddGamePayload): Promise<void> {
  const game = await invoke<Game>('add_custom_game', payload)
  games.update(gs => [...gs, game])
}

export async function removeGame(gameId: string): Promise<void> {
  await invoke('remove_game', { game_id: gameId })
  games.update(gs => gs.filter(g => g.id !== gameId))
}

export async function toggleFavourite(gameId: string): Promise<void> {
  await invoke('toggle_favourite', { game_id: gameId })
  games.update(gs =>
    gs.map(g => g.id === gameId ? { ...g, is_favourite: !g.is_favourite } : g)
  )
}

// Reload library after a sync completes
export async function refreshAfterSync(): Promise<void> {
  await loadGames()
}
```

---

### 12.4 `launchStore.ts` — Game Launch State

Tracks the active game session and exposes the launch state machine defined in Module 10.

```typescript
// src/lib/stores/launchStore.ts
import { writable, derived } from 'svelte/store'
import { invoke } from '@tauri-apps/api/core'
import { listen } from '@tauri-apps/api/event'

export type LaunchStatus =
  | 'idle'
  | 'starting_steam'
  | 'waiting_for_steam'
  | 'launching_game'
  | 'running'
  | 'exited'
  | 'error'

export interface LaunchState {
  status:     LaunchStatus
  game_id:    string | null
  session_id: string | null
  started_at: number | null
  duration_s: number | null
  error:      string | null
}

const INITIAL: LaunchState = {
  status: 'idle', game_id: null, session_id: null,
  started_at: null, duration_s: null, error: null
}

export const launchState = writable<LaunchState>(INITIAL)

// Human-readable status messages for the UI spinner
export const STATUS_LABELS: Record<LaunchStatus, string> = {
  idle:              '',
  starting_steam:    'Starting Steam...',
  waiting_for_steam: 'Initializing...',
  launching_game:    'Loading game...',
  running:           'Running',
  exited:            'Session saved',
  error:             'Launch failed',
}

// Derived: is any game currently running?
export const isGameRunning = derived(
  launchState,
  $s => $s.status === 'running'
)

// Subscribe to backend launch state events
export async function initLaunchListener(): Promise<void> {
  await listen<LaunchState>('game_launch_state', (event) => {
    launchState.set({ ...INITIAL, ...event.payload })

    // Auto-reset to idle 2s after game exits
    if (event.payload.status === 'exited') {
      setTimeout(() => launchState.set(INITIAL), 2000)
    }
  })
}

export async function launchGame(gameId: string): Promise<void> {
  launchState.set({ ...INITIAL, status: 'starting_steam', game_id: gameId })
  try {
    await invoke('launch_steam_game', { game_id: gameId })
    // Further state updates come from backend events via initLaunchListener
  } catch (e) {
    launchState.set({ ...INITIAL, status: 'error', game_id: gameId, error: String(e) })
  }
}
```

---

### 12.5 `storeStore.ts` — Steam Store State

Manages all Steam Store data: featured games, category tabs, search, and individual app detail pages.

```typescript
// src/lib/stores/storeStore.ts
import { writable, derived } from 'svelte/store'
import { invoke } from '@tauri-apps/api/core'
import type { FeaturedResponse, FeaturedCategoriesResponse, AppDetails, SearchResults } from '$lib/types'

// ── Featured / home ───────────────────────────────────────────
export const featured           = writable<FeaturedResponse | null>(null)
export const featuredCategories = writable<FeaturedCategoriesResponse | null>(null)
export const storeLoading       = writable<boolean>(false)
export const storeError         = writable<string | null>(null)

// ── Current tab ───────────────────────────────────────────────
export type StoreTab = 'featured' | 'top_sellers' | 'new_releases' | 'specials'
export const activeStoreTab = writable<StoreTab>('featured')

// ── Search ────────────────────────────────────────────────────
export const storeSearchQuery   = writable<string>('')
export const storeSearchResults = writable<SearchResults | null>(null)
export const storeSearchLoading = writable<boolean>(false)

// ── App detail page ───────────────────────────────────────────
export const selectedAppId      = writable<number | null>(null)
export const appDetail          = writable<AppDetails | null>(null)
export const appDetailLoading   = writable<boolean>(false)

// Lazy-loaded: called when the user first navigates to Store tab
export async function loadStoreFeatured(): Promise<void> {
  storeLoading.set(true)
  storeError.set(null)
  try {
    const [feat, cats] = await Promise.all([
      invoke<FeaturedResponse>('store_get_featured'),
      invoke<FeaturedCategoriesResponse>('store_get_categories'),
    ])
    featured.set(feat)
    featuredCategories.set(cats)
  } catch (e) {
    storeError.set(String(e))
  } finally {
    storeLoading.set(false)
  }
}

// Debounced search — call after 300ms of no typing
let searchTimeout: ReturnType<typeof setTimeout>
export function debouncedSearch(query: string): void {
  storeSearchQuery.set(query)
  clearTimeout(searchTimeout)
  if (!query.trim()) { storeSearchResults.set(null); return }
  searchTimeout = setTimeout(async () => {
    storeSearchLoading.set(true)
    try {
      const results = await invoke<SearchResults>('store_search', { query, page: 0 })
      storeSearchResults.set(results)
    } finally {
      storeSearchLoading.set(false)
    }
  }, 300)
}

export async function browseCategory(filter: string, page = 0): Promise<void> {
  storeLoading.set(true)
  try {
    const results = await invoke<SearchResults>('store_browse', { filter, page })
    storeSearchResults.set(results)
  } finally {
    storeLoading.set(false)
  }
}

export async function openAppDetail(appId: number): Promise<void> {
  selectedAppId.set(appId)
  appDetail.set(null)
  appDetailLoading.set(true)
  try {
    const detail = await invoke<AppDetails>('store_get_app', { app_id: appId })
    appDetail.set(detail)
  } finally {
    appDetailLoading.set(false)
  }
}

export async function openInBrowser(appId: number): Promise<void> {
  await invoke('open_url', { url: `https://store.steampowered.com/app/${appId}` })
}
```

---

### 12.6 `settingsStore.ts` — User Preferences

```typescript
// src/lib/stores/settingsStore.ts
import { writable } from 'svelte/store'
import { invoke } from '@tauri-apps/api/core'

export interface Settings {
  steam_api_key:         string
  steam_user_id:         string
  steam_username:        string
  steam_avatar_url:      string
  steamgriddb_api_key:   string
  sync_interval_minutes: number
  close_behavior:        'tray' | 'quit'
  auto_scan_on_launch:   boolean
  custom_scan_paths:     string[]
  store_country_code:    string   // e.g. 'it', 'us'
  store_language:        string   // e.g. 'english', 'italian'
}

export const settings = writable<Settings | null>(null)
export const settingsLoading = writable<boolean>(false)

export async function loadSettings(): Promise<void> {
  settingsLoading.set(true)
  try {
    const raw = await invoke<Record<string, string>>('get_all_settings')
    settings.set({
      steam_api_key:         raw.steam_api_key         ?? '',
      steam_user_id:         raw.steam_user_id         ?? '',
      steam_username:        raw.steam_username        ?? '',
      steam_avatar_url:      raw.steam_avatar_url      ?? '',
      steamgriddb_api_key:   raw.steamgriddb_api_key   ?? '',
      sync_interval_minutes: Number(raw.sync_interval_minutes ?? 10),
      close_behavior:        (raw.close_behavior ?? 'tray') as 'tray' | 'quit',
      auto_scan_on_launch:   raw.auto_scan_on_launch === 'true',
      custom_scan_paths:     JSON.parse(raw.custom_scan_paths ?? '[]'),
      store_country_code:    raw.store_country_code    ?? 'us',
      store_language:        raw.store_language        ?? 'english',
    })
  } finally {
    settingsLoading.set(false)
  }
}

export async function saveSetting(key: string, value: string): Promise<void> {
  await invoke('save_setting', { key, value })
  await loadSettings() // reload to keep store in sync
}
```

---

### 12.7 `App.svelte` — Root Bootstrap

This is the root component. It initializes all stores on mount in the correct order.

```svelte
<!-- src/App.svelte -->
<script lang="ts">
  import { onMount } from 'svelte'
  import { runStartupSync, initSyncListener } from '$lib/stores/syncStore'
  import { loadGames, refreshAfterSync }      from '$lib/stores/libraryStore'
  import { initLaunchListener }               from '$lib/stores/launchStore'
  import { loadSettings }                     from '$lib/stores/settingsStore'
  import { syncResult }                       from '$lib/stores/syncStore'
  import Sidebar   from '$lib/components/Sidebar.svelte'
  import MainArea  from '$lib/components/MainArea.svelte'

  onMount(async () => {
    // 1. Load settings first (other stores may depend on them)
    await loadSettings()

    // 2. Load local library immediately — don't wait for sync
    await loadGames()

    // 3. Start background listeners
    await initLaunchListener()
    await initSyncListener()

    // 4. Trigger Steam sync (non-blocking — UI shows spinner via syncState)
    runStartupSync()
  })

  // When sync completes, refresh the library to pick up new games
  $: if ($syncResult) {
    refreshAfterSync()
  }
</script>

<div class="app">
  <Sidebar />
  <MainArea />
</div>
```

**Bootstrap order is critical:**
1. `loadSettings` — must be first; API keys are needed by sync
2. `loadGames` — shows existing library immediately, before sync completes
3. `initLaunchListener` + `initSyncListener` — register event listeners
4. `runStartupSync` — fires async in background; UI updates reactively

This means the user sees their library immediately on launch, with a spinning sync indicator in the sidebar. New games appear automatically when sync finishes.

---

### 12.8 Complete IPC Command Reference

This table consolidates every `invoke` command used across all frontend stores.

| Store | Command | Input | Output |
|---|---|---|---|
| syncStore | `sync_steam` | — | `SyncResult` |
| syncStore | `get_setting` | `{ key }` | `string` |
| libraryStore | `get_all_games` | — | `Game[]` |
| libraryStore | `add_custom_game` | `AddGamePayload` | `Game` |
| libraryStore | `remove_game` | `{ game_id }` | `void` |
| libraryStore | `toggle_favourite` | `{ game_id }` | `void` |
| launchStore | `launch_steam_game` | `{ game_id }` | `{ session_id }` |
| storeStore | `store_get_featured` | — | `FeaturedResponse` |
| storeStore | `store_get_categories` | — | `FeaturedCategoriesResponse` |
| storeStore | `store_get_app` | `{ app_id }` | `AppDetails` |
| storeStore | `store_search` | `{ query, page }` | `SearchResults` |
| storeStore | `store_browse` | `{ filter, page }` | `SearchResults` |
| storeStore | `open_url` | `{ url }` | `void` |
| settingsStore | `get_all_settings` | — | `Record<string, string>` |
| settingsStore | `save_setting` | `{ key, value }` | `void` |
| steamAuth | `steam_login` | — | `SteamUser` |
| steamAuth | `steam_logout` | — | `void` |
| steamAuth | `get_steam_user` | — | `SteamUser \| null` |

---

### 12.9 Backend IPC Additions Required

The following new Tauri commands must be implemented in Rust to support this module (not yet listed in previous modules):

```rust
// src-tauri/src/commands/library.rs

#[tauri::command]
pub async fn toggle_favourite(game_id: String, db: State<DbPool>) -> Result<(), String>;

// src-tauri/src/commands/settings.rs

#[tauri::command]
pub async fn get_all_settings(db: State<DbPool>) -> Result<HashMap<String, String>, String>;

#[tauri::command]
pub async fn save_setting(key: String, value: String, db: State<DbPool>) -> Result<(), String>;

#[tauri::command]
pub async fn get_setting(key: String, db: State<DbPool>) -> Result<String, String>;
```

---

*Document version: 1.4.0 — Added Module 12: Frontend Stores & App Bootstrap.*
*Last updated: March 2026*
