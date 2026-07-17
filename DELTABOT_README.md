# ReaClaw on DELTABOT (Windows)

Notes for this Windows checkout at `D:\reaclaw`. Upstream CI is Linux-only (`build-linux` job on a self-hosted k3s runner), so MSVC-only warnings slip through to `main`. This branch commits the two fixes needed to build on Windows.

## One-time setup

1. **Fetch vendor deps** (httplib, json, sqlite, reaper-sdk, WDL/swell):
   ```bash
   bash scripts/fetch-vendor-deps.sh
   ```
   Run from Git Bash (the script uses `curl` + `unzip`). Takes ~30s.

2. **Install OpenSSL via vcpkg.** Not bundled — CMake's `find_package(OpenSSL)` needs it:
   ```powershell
   C:\dev\vcpkg\vcpkg.exe install openssl:x64-windows
   ```
   ~4 min the first time. Only needed once per vcpkg root.

3. **(Optional) Install git hooks:**
   ```bash
   bash scripts/install-git-hooks.sh
   ```
   Sets `core.hooksPath` → `.githooks/`. `pre-commit` runs clang-format, `pre-push` runs full build+ctest. clang-format is not installed on DELTABOT — install with `winget install LLVM.LLVM` or `--no-verify` the commits.

## Build

```powershell
cd D:\reaclaw
cmake -B build -DCMAKE_TOOLCHAIN_FILE=C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
```

CMake auto-detects Visual Studio 18 BuildTools and uses the MSBuild generator — no `vcvarsall.bat` needed. First build ~10 min (SQLite amalgamation is slow under MSVC); incremental builds are seconds.

Output: `D:\reaclaw\build\Release\reaper_reaclaw.dll` (~3.4 MB) plus `libcrypto-3-x64.dll` and `libssl-3-x64.dll` copied beside it.

## Install into REAPER

```powershell
$plugins = "$env:APPDATA\REAPER\UserPlugins"
Copy-Item D:\reaclaw\build\Release\reaper_reaclaw.dll $plugins
Copy-Item D:\reaclaw\build\Release\libcrypto-3-x64.dll,D:\reaclaw\build\Release\libssl-3-x64.dll $plugins
```

Minimal config at `$env:APPDATA\REAPER\reaclaw\config.json`:
```json
{
  "server": { "port": 9091 },
  "tls":    { "enabled": true, "generate_if_missing": true },
  "auth":   { "type": "api_key", "key": "sk_change_me" }
}
```

Restart REAPER, then verify:
```powershell
curl.exe -sk -H "Authorization: Bearer sk_change_me" https://localhost:9091/health
```

## Windows portability fixes on this branch

Both are MSVC-only warnings promoted to errors by `/WX`. GCC/Clang (Linux CI) don't flag them.

1. **`src/handlers/hints.h`** — forward-declared `MediaTrack`/`MediaItem` as `struct`, but the REAPER SDK (`vendor/reaper-sdk/sdk/reaper_plugin.h:1454-55`) declares them as `class`. MSVC C4099. Fix: match the SDK.
2. **`src/reaper/catalog.cpp:159`** — `int indexed = db.scalar_int(...)` narrows `int64_t → int`. MSVC C4244. Fix: `static_cast<int>(...)` (matching the existing pattern at line 164).

Worth PR'ing upstream — the fixes are strict improvements, and enabling MSVC in CI (or at minimum a `-Wnarrowing`-equivalent check) would prevent regressions.

## Toolchain (DELTABOT state)

- **Visual Studio 18 BuildTools** with `Microsoft.VisualStudio.Component.VC.Tools.x86.x64` (MSVC 19.50.35730.0)
- **CMake 4.3.2** (winget)
- **vcpkg** at `C:\dev\vcpkg` — `openssl:x64-windows@3.6.2` installed
- **Python 3.12.3** (CMake finds it for asset gen scripts)
- REAPER install expected at default location; set `REAPER_USER_PLUGINS` if using a portable install to make `cmake --install` work.

## Remotes

- `origin` — `evolv3ai/reaclaw` (this fork)
- `upstream` — `braveness23/reaclaw` (canonical)

Sync from upstream: `git fetch upstream && git rebase upstream/main`.
