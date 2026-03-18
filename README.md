# Wither Launcher - GitHub Repository
# https://github.com/yourusername/wither-launcher

A cross-platform game launcher with Steam integration, custom game support, local playtime tracking, and a monochromatic Swiss editorial design aesthetic.

## Features

- **Steam Integration**: Full library sync with metadata enrichment
- **Custom Games**: Add any executable to your library
- **Playtime Tracking**: Local session tracking with Steam playtime import
- **Silent Steam Launch**: Auto-starts Steam when needed, auto-shutdown when done
- **Steam Store**: Browse Featured, Top Sellers, New Releases, and Specials
- **Modern UI**: Monochromatic design with glassmorphism effects

## Tech Stack

- **Frontend**: Svelte 5 + TypeScript
- **Backend**: Rust + Tauri 2.x
- **Database**: SQLite (rusqlite)
- **Styling**: Plain CSS (no framework)

## Prerequisites

- [Node.js](https://nodejs.org/) (v18 or later)
- [pnpm](https://pnpm.io/) (v8 or later)
- [Rust](https://www.rust-lang.org/tools/install) (latest stable)

## Installation

1. Clone the repository:
```bash
git clone https://github.com/yourusername/wither-launcher.git
cd wither-launcher
```

2. Install dependencies:
```bash
pnpm install
```

3. Run in development mode:
```bash
pnpm tauri dev
```

## Building for Production

```bash
pnpm tauri build
```

The installer will be created at:
- **Windows**: `src-tauri/target/release/bundle/nsis/Wither_0.1.0_x64-setup.exe`
- **macOS**: `src-tauri/target/release/bundle/dmg/Wither_0.1.0_x64.dmg`
- **Linux**: `src-tauri/target/release/bundle/deb/wither_0.1.0_amd64.deb`

## Project Structure

```
wither-launcher/
├── src/                        # Svelte frontend
│   ├── lib/
│   │   ├── components/         # UI components
│   │   ├── stores/             # Svelte stores
│   │   ├── api/                # Tauri IPC wrappers
│   │   └── types/              # TypeScript interfaces
│   ├── routes/                 # SvelteKit pages
│   │   ├── +page.svelte        # Home
│   │   ├── library/
│   │   ├── recent/
│   │   ├── stats/
│   │   ├── store/              # Steam Store
│   │   └── settings/
│   └── app.css                 # Global styles
├── src-tauri/                  # Rust backend
│   ├── src/
│   │   ├── commands/           # Tauri IPC handlers
│   │   ├── steam/              # Steam integration
│   │   ├── process/            # Game launching
│   │   ├── db/                 # SQLite schema
│   │   └── main.rs             # Entry point
│   ├── Cargo.toml
│   └── tauri.conf.json
├── package.json
├── tsconfig.json
├── vite.config.ts
└── wither-documentation.md     # Full project documentation
```

## Modules

| Module | Description |
|--------|-------------|
| 01 | General Architecture |
| 02 | Rust Backend |
| 03 | Local Database (SQLite) |
| 04 | Design System |
| 05 | Hybrid Launch System |
| 06 | Steam Integration |
| 07 | Build & Distribution |
| 08 | Adding Games |
| 09 | Steam Integration: Implementation |
| 10 | Silent Steam Launch |
| 11 | Steam Store (Custom UI) |
| 12 | Frontend Stores & App Bootstrap |

## Configuration

### Steam API Key

To enable Steam metadata (covers, genres, playtime):

1. Get an API key at: https://steamcommunity.com/dev/apikey
2. Open Wither → Settings → Steam Account
3. Enter your API key

### SteamGridDB (Optional)

For custom cover art:

1. Get an API key at: https://www.steamgriddb.com/api
2. Open Wither → Settings → Steam Account
3. Enter your SteamGridDB API key

## License

MIT License - See LICENSE file for details.

## Acknowledgments

- Steam is a trademark of Valve Corporation
- This project is not affiliated with Valve Corporation
