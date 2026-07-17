# ReaClaw on DELTABOT (Windows)

Notes for this Windows checkout at `D:\reaclaw`. Upstream CI is Linux-only (`build-linux` job on a self-hosted k3s runner), so MSVC-only warnings slip through to `main`. This branch commits the two fixes needed to build on Windows.

## One-time setup

1. **Fetch vendor deps** (httplib, json, sqlite, reaper-sdk, WDL/swell):
   ```bash
   bash scripts/fetch-vendor-deps.sh
   ```
   Run from Git Bash (the script uses `curl` + `unzip`). Takes ~30s.

2. **Install OpenSSL via vcpkg — static-md triplet.** Not bundled. Must be the `x64-windows-static-md` variant so OpenSSL links statically into the DLL; the dynamic `x64-windows` variant produces a DLL that pulls in `libssl-3-x64.dll` + `libcrypto-3-x64.dll` at load time, and REAPER's DLL loader does not search `UserPlugins\` for a plugin's *dependent* DLLs, so the plugin silently fails to load. This matches how upstream ships the Windows binary (CI cross-compiles OpenSSL `no-shared` under MinGW). 
   ```powershell
   C:\dev\vcpkg\vcpkg.exe install openssl:x64-windows-static-md
   ```
   ~3 min the first time. Only needed once per vcpkg root.

3. **(Optional) Install git hooks:**
   ```bash
   bash scripts/install-git-hooks.sh
   ```
   Sets `core.hooksPath` → `.githooks/`. `pre-commit` runs clang-format, `pre-push` runs full build+ctest. clang-format is not installed on DELTABOT — install with `winget install LLVM.LLVM` or `--no-verify` the commits.

## Build

```powershell
cd D:\reaclaw
cmake -B build `
  -DCMAKE_TOOLCHAIN_FILE=C:/dev/vcpkg/scripts/buildsystems/vcpkg.cmake `
  -DVCPKG_TARGET_TRIPLET=x64-windows-static-md `
  -DOPENSSL_USE_STATIC_LIBS=ON `
  -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
```

CMake auto-detects Visual Studio 18 BuildTools and uses the MSBuild generator — no `vcvarsall.bat` needed. First build ~10 min (SQLite amalgamation is slow under MSVC); incremental builds are seconds.

Output: `D:\reaclaw\build\Release\reaper_reaclaw.dll` (~8.5 MB — OpenSSL baked in, no separate runtime DLLs).

## Install into REAPER

```powershell
$plugins = "$env:APPDATA\REAPER\UserPlugins"
Copy-Item D:\reaclaw\build\Release\reaper_reaclaw.dll $plugins
```

(No OpenSSL runtime DLLs to copy — they're statically linked in.)

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

## Verified working on DELTABOT — 2026-07-16

```
GET /health → 200
{
  "catalog_size": 4313, "db_ok": true, "queue_depth": 0,
  "reaper_version": "7.77/x64", "server_running": true,
  "status": "ok", "uptime_seconds": 9, "version": "1.18.1"
}
```

Full flow smoke test also passed: `POST /state/tracks` (create) → track_count 0→1 in `GET /state`, auth enforcement returns 401 without the API key, `/capabilities` and `/agent/guide` both serve.

Init diagnostic breadcrumb file: `%TEMP%\reaclaw-diag.txt` — reaclaw writes step-by-step init progress via raw Win32 handles before anything else runs. If the plugin fails to load, this is the first place to look. A missing file means Windows blocked LoadLibrary (usually a missing dependent DLL — see the static-md note above).

## Installing the agent skill on Windows

`scripts/install-agent-skill.sh` calls `ln -sfn` to link `.claude/skills/reaper` into `~/.claude/skills/`. On Git Bash without developer mode, `ln -s` silently falls back to a hard copy — the "install" appears to succeed, but later `LEARNED.md` edits in the repo don't reach agent sessions and new files in the repo skill dir (like `LOCAL.md`) aren't visible. Replace the copy with a directory junction (no admin required, works across drives):

```powershell
$dest = "$env:USERPROFILE\.claude\skills\reaper"
$src  = "D:\reaclaw\.claude\skills\reaper"
if (Test-Path $dest) { Remove-Item -Recurse -Force $dest }
cmd /c mklink /J $dest $src
```

Verify with `Get-Item $dest | Select-Object LinkType, Target` — expect `Junction` and the source path.

The skill also needs a `python3` on PATH and a `REACLAW_CONFIG` override pointing at the Windows config path. See `.claude/skills/reaper/LOCAL.md` (gitignored, device-specific) for the shim details and one-time setup on DELTABOT.

## Live event feed is disabled on Windows

Reaclaw's csurf-based event stream (issue #111 upstream) is currently a no-op on Windows — MSVC/MinGW C++ ABI mismatch on `IReaperControlSurface` means the extension logs a WARN at load and refuses to register the surface. In practice: **any Windows agent that needs to react to human GUI edits must poll `GET /state/changes` rather than subscribe.** The AGENT_GUIDE's "stay in sync with a human editing at the same time" section still works, but via polling only. Linux/macOS builds keep the live feed.

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
