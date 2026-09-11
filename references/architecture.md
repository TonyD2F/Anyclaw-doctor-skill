# AnyClaw Architecture Reference

Source: `OpenClawAndroid/openclaw-android-assistant` (forked from `openclaw/openclaw`), package `com.codex.mobile`. Verify against the live repo if anything here seems stale — this project has 85k+ commits and moves fast.

## What it is

One APK bundling three agents on top of an embedded, Termux-derived Linux userland running under `proot` — no root, no server, no PC required:

| Agent | What it is |
|---|---|
| **OpenClaw** | Personal AI assistant — gateway, agent routing, skills, Canvas, Control UI dashboard |
| **OpenAI Codex CLI** | Terminal coding agent — native Rust binary (`aarch64-unknown-linux-musl`) |
| **Claw Code / OpenClaude** | Clean-room rewrite of the (leaked) Claude Code architecture — 19 tools, 15 slash commands, MCP support, multi-agent swarms, provider-agnostic |

## Directory layout (in-app)

```
android/app/src/main/
├── AndroidManifest.xml
├── assets/
│   ├── proxy.js                 # CONNECT proxy — DNS/TLS bridge for musl binaries
│   ├── bionic-compat.js         # patches process.platform, os.cpus(), os.networkInterfaces()
│   └── server-bundle/           # pre-built Vue + Express + deps
└── java/com/codex/mobile/
    ├── BootstrapInstaller.kt     # Linux environment extraction
    ├── CodexForegroundService.kt # background persistence (foreground service)
    ├── CodexServerManager.kt     # install, auth, proxy, OpenClaw, server orchestration
    └── MainActivity.kt           # WebView + setup orchestration
```

On-device runtime root: `$PREFIX = /data/user/0/com.codex.mobile/files/usr` (extracted from the bundled `bootstrap-aarch64.zip`, Termux's minimal userland).

OpenClaw config/auth lives at `~/.openclaw/openclaw.json` inside that userland.

## Ports (all localhost)

| Port | Service | Purpose |
|---|---|---|
| 18789 | OpenClaw Gateway | WebSocket control plane for agents, sessions, tools |
| 18923 | codex-web-local | HTTP server, Vue.js UI — this is what the in-app WebView actually loads (`http://127.0.0.1:18923/`) |
| 18924 | CONNECT Proxy | DNS/TLS bridge for musl-linked binaries (see "Why the proxy exists" below) |
| 19001 | Control UI Server | Static file server for the OpenClaw dashboard |

If any of these ports are already bound by a stale/zombie process (e.g., app was force-killed mid-session and a child process survived), the corresponding service will fail to (re)bind on next launch.

## Startup sequence (14 steps)

Useful for pinpointing exactly where a stuck launch is failing — ask Tony what's on screen / what the last logcat line before it hung was, and map it to a step:

1. Battery optimization exemption + foreground service request
2. Bootstrap extraction (Termux userland unpacked to `$PREFIX`)
3. proot + Node.js + Python installation
4. `bionic-compat.js` extraction
5. OpenClaw build deps + install + **koffi build** + path patching
6. Claw Code / OpenClaude agent installation
7. Codex CLI + native platform binary installation
8. Full-access config write (`approval_policy = "never"`)
9. CONNECT proxy startup (port 18924)
10. OAuth login (`codex login` via system browser)
11. Health check
12. OpenClaw auth config + gateway (18789) + Control UI server (19001) startup
13. codex-web-local server startup (18923)
14. WebView loads `http://127.0.0.1:18923/`

## Why the CONNECT proxy exists (root cause behind most network errors)

The Codex CLI's native Rust binary is compiled for `aarch64-unknown-linux-musl` and reads `/etc/resolv.conf` for DNS resolution — which doesn't exist in the Android/proot environment the way it does on real Linux. The bundled Node.js CONNECT proxy on port 18924 bridges this: Node.js uses Android's Bionic DNS resolver (which works fine), and the musl-linked native binary is pointed at `HTTPS_PROXY=http://127.0.0.1:18924` so all its outbound HTTPS goes through that proxy instead of trying (and failing) to resolve DNS itself. **Any "No address associated with hostname" or DNS-looking error from an agent almost always traces back to this proxy not running, not started yet, or crashed** — not to the phone's actual network connection.

This is the same category of glibc/musl-vs-Bionic ABI friction that trips up other native CLI tools ported to Termux/Android (e.g., the official Claude Code native installer is glibc-linked and fails outright on Bionic with a missing `libpthread.so.0` — a different symptom of the same underlying incompatibility class). It's worth knowing this pattern exists broadly, not just in this one proxy.

## Why targetSdk 28 matters (root cause behind most "permission denied" errors)

Android 10 (API 29) introduced a W^X (Write XOR Execute) enforcement: apps targeting API 29+ cannot `exec()` files living in their own writable app-data directory — Android considers that a W^X violation, since normally only code embedded in the APK itself (and thus read-only, verified) should be executable. Termux-derived apps solve this by pinning `targetSdk = 28` in the manifest, which keeps the OS from applying that restriction. This is a known, deliberate, and stable workaround — not a bug — but it means:

- If `targetSdk` in `android/app/build.gradle.kts` ever gets bumped to 29+ (e.g., during a dependency update or a Play Store compliance push), every binary exec inside `$PREFIX` will start failing with "Permission denied," even though nothing else changed.
- On some OEM Android 10+ builds, exec-from-home-dir can still fail even at targetSdk 28 due to stricter SELinux policies on that specific device/ROM — this is a known device-specific edge case, not something fixable from inside the app.
- A device reboot is sometimes required after `targetSdk` changes for the OS to fully apply/reapply the exec permission.

## Tech stack / versions (verify against live repo — moves fast)

| Layer | Tech | Version (at last check) |
|---|---|---|
| AI Gateway | OpenClaw | 2026.2.21-2 |
| AI Agent | OpenAI Codex CLI | 0.104.0 |
| AI Agent | Claw Code / OpenClaude | latest (rolling) |
| Model | gpt-5.3-codex (via Codex OAuth) | — |
| Runtime | Node.js (via Termux) | 24.13.0 |
| Build tools | Clang/LLVM, CMake, Make, LLD | 21.1.8 / 4.2.3 |
| Frontend | Vue.js 3 + Vite + TailwindCSS | 3.x |
| Android | Kotlin + WebView | 2.1.0 |
| Linux | Termux bootstrap (aarch64) | — |

## Requirements

- Android 7.0+ (API 24), **ARM64 only** (arm64-v8a) — no x86/x86_64 emulator support
- Internet connection for first-run setup and API calls
- OpenAI account, authenticated via OAuth browser flow (shared across all three agents)
- ~500MB storage for the Linux environment + Node.js + Codex + OpenClaw + Claw Code

## Build-from-source quick reference

```bash
git clone https://github.com/friuns2/openclaw-android-assistant.git
cd openclaw-android-assistant
npm install && npm run build
cd android && bash scripts/download-bootstrap.sh
bash scripts/build-server-bundle.sh && ./gradlew assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n com.codex.mobile/.MainActivity
```

See `build-from-source.md` for the full breakdown of what each step does and how it can fail.
