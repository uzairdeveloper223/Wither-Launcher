# Wither Launcher

A cross-platform game launcher with Steam integration, custom game support, local playtime tracking, and a monochromatic Swiss editorial design aesthetic.

![License](https://img.shields.io/badge/license-MIT-blue)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20Windows-lightgrey)
![Tauri](https://img.shields.io/badge/Tauri-2.x-orange)

## Features

- **Steam Integration**: Full library sync with metadata enrichment (covers, genres, playtime)
- **Custom Games**: Add any executable — Linux native, Windows .exe (via Proton), AppImage, shell scripts
- **Linux Proton Support**: Automatically launches Windows games via Proton on Linux
- **GE-Proton Manager**: Download and manage GE-Proton versions directly from Settings
- **Playtime Tracking**: Local session tracking with Steam playtime import
- **Silent Steam Launch**: Auto-starts Steam when needed, auto-shutdown when done
- **Steam Store**: Browse Featured, Top Sellers, New Releases, and Specials
- **Cover Art Fallback**: Multi-layer cover art fetching ensures images display for all games
- **Modern UI**: Monochromatic design with glassmorphism effects and collapsible sidebar
- **System Tray**: Minimize to tray with background game tracking
- **CI/CD**: Automated builds for Linux and Windows on tag push

## Tech Stack

- **Frontend**: Svelte 5 + TypeScript
- **Backend**: Rust + Tauri 2.x
- **Database**: SQLite (rusqlite)
- **Styling**: Plain CSS (no framework)

## Downloads

Check the [Releases](https://github.com/piskevalee-cpu/Wither-Launcher/releases) page for pre-built binaries:

- **Linux**: `.deb` (Debian/Ubuntu) or `.AppImage` (universal)
- **Windows**: `.exe` (NSIS installer)

## Prerequisites (Development)

- [Node.js](https://nodejs.org/) (v18 or later)
- [pnpm](https://pnpm.io/) (v9 or later)
- [Rust](https://www.rust-lang.org/tools/install) (latest stable)

### Linux Additional Dependencies

```bash
sudo apt-get install -y \
  build-essential \
  libssl-dev \
  libgtk-3-dev \
  libwebkit2gtk-4.1-dev \
  librsvg2-dev \
  patchelf
```

## Installation

1. Clone the repository:
```bash
git clone https://github.com/piskevalee-cpu/Wither-Launcher.git
cd Wither-Launcher
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
- **Linux .deb**: `src-tauri/target/release/bundle/deb/wither_0.1.0_amd64.deb`
- **Linux AppImage**: `src-tauri/target/release/bundle/appimage/wither_0.1.0_amd64.AppImage`

## Releasing

Releases are automated via GitHub Actions. To create a new release:

```bash
git tag v0.1.0
git push origin v0.1.0
```

This triggers builds for both Linux and Windows, and creates a GitHub Release with all artifacts.

## Project Structure

```
Wither-Launcher/
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
│   │   ├── process/            # Game launching & Proton
│   │   ├── db/                 # SQLite schema
│   │   └── main.rs             # Entry point
│   ├── Cargo.toml
│   └── tauri.conf.json
├── .github/workflows/          # CI/CD
│   └── release.yml             # Auto-build on tag push
├── package.json
├── tsconfig.json
└── vite.config.ts
```

## Configuration

### Steam API Key

To enable Steam metadata (covers, genres, playtime):

1. Get an API key at: https://steamcommunity.com/dev/apikey
2. Open Wither → Settings → API Keys
3. Enter your API key

### SteamGridDB (Optional)

For custom cover art:

1. Get an API key at: https://www.steamgriddb.com/api
2. Open Wither → Settings → API Keys
3. Enter your SteamGridDB API key

### GE-Proton (Linux)

To download GE-Proton versions directly:

1. Open Wither → Settings → Proton / Compatibility
2. Click "Fetch Releases" to get available versions from GitHub
3. Click "Install" on any version to download and extract it

## Known Issues

- Filter dropdown in search bar is not fully functional yet
- Playtime is only tracked when games are launched from Wither
- Steam may not close automatically when a Steam game exits on some systems
- Fonts may show 404 errors in dev mode (cosmetic, doesn't affect production build)

## Roadmap

- [ ] Epic Games Store integration
- [ ] GOG integration
- [ ] Cloud saves sync
- [ ] Universal achievement tracking
- [ ] Controller support (Steam Input)
- [ ] Game news feed
- [ ] Screenshot gallery
- [ ] macOS support

## Acknowledgments

- Steam is a trademark of Valve Corporation
- This project is not affiliated with Valve Corporation
- [GE-Proton](https://github.com/GloriousEggroll/proton-ge-custom) by GloriousEggroll

## License

MIT License — See [LICENSE](LICENSE) file for details.
