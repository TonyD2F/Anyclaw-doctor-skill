---
name: anyclaw-doctor
description: Diagnostic and repair skill for AnyClaw (the Android app that runs OpenClaw + OpenAI Codex CLI + Claw Code/OpenClaude inside an embedded Termux-derived Linux userland, no root). Use this whenever Tony reports AnyClaw, OpenClaw, OpenCode, Claw Code, or OpenClaude problems on Android — app crashes on launch, bootstrap/proot extraction failures, "permission denied" executing binaries, OpenClaw gateway not starting, koffi build failures, DNS/proxy errors ("No address associated with hostname"), OAuth login page not opening, the foreground service getting killed in the background, WebView not loading, port conflicts on 18789/18923/18924/19001, stuck startup sequence, or general "it's broken" / "won't start" / "help me fix" reports about this app. Also trigger for setup/build issues (gradlew assembleDebug failures, bootstrap download failures, server-bundle build issues) and for proactive health checks of an existing install. Do not use this for generic Android troubleshooting unrelated to AnyClaw/OpenClaw/Codex/Claw Code.
---

# AnyClaw / OpenClaw / OpenCode Doctor

A systematic diagnostic skill for the AnyClaw Android app (repo: `OpenClawAndroid/openclaw-android-assistant`, package `com.codex.mobile`). This app runs three agents — OpenClaw gateway, OpenAI Codex CLI, and Claw Code/OpenClaude — inside an embedded Termux-derived Linux userland via proot, no root required.

**This skill treats itself like a doctor treats a patient: intake → vitals → differential diagnosis → targeted fix → verify → discharge notes.** Never jump straight to "try this" — establish what's actually wrong first.

## 0. Before anything else

1. Read `references/architecture.md` once per session if you haven't already — it has the full port map, startup sequence, and file layout you'll need to reason about symptoms accurately.
2. Ask Tony (or infer from what he's pasted) which of these he has access to:
   - `adb logcat` output (device connected via USB/wireless debugging)
   - Shell access inside the app's environment (via a terminal feature in-app, or adb shell run-as)
   - Just the on-screen error / app behavior, no logs
   
   The diagnostic path differs a lot depending on this — don't assume adb access exists. If he only has "the app won't open," start with the on-device checklist in `references/symptom-map.md` before asking for logs he may not be able to get.

## 1. Intake — gather vitals

Ask only what you don't already know from context. Don't interrogate if Tony already described the symptom clearly — go straight to differential diagnosis.

Useful vitals, in priority order:
- **Exact symptom**: crash on launch vs. hangs at a specific startup step vs. starts but a specific agent (OpenClaw/Codex/Claw Code) doesn't respond vs. works then dies in background
- **Where in the startup sequence it stops** (see `references/architecture.md` §Startup Sequence — 14 numbered steps). If the WebView shows anything at all, note what step's UI it's stuck on.
- **Android version + device arch** — this app requires ARM64, Android 7.0+ (API 24), and specifically relies on `targetSdk = 28` for the W^X bypass. An x86 emulator or a device where that manifest setting got changed will fail differently than a real ARM64 phone.
- **First install vs. previously working** — fresh installs fail differently (bootstrap extraction, OAuth) than previously-working installs that broke (usually storage/permission drift, an OS update, or background kill).
- **Recent changes** — Android OS update, app update, storage cleared, battery optimization re-enabled, new device.

## 2. Differential diagnosis

Open `references/symptom-map.md` and match the symptom to its section. Each entry has: likely cause(s) ranked by probability, the specific check to confirm, and the fix. Don't guess — confirm before prescribing. If two causes are plausible, check the cheaper one first.

Categories covered in the symptom map:
- App crashes immediately on launch
- Stuck/hangs during bootstrap extraction (step 2)
- "Permission denied" executing binaries (steps 3, 6-8)
- koffi build failure (step 5, OpenClaw build deps)
- DNS/proxy errors, "No address associated with hostname" (steps 5-9, proxy.js)
- OAuth login page doesn't open / login loop (step 10)
- Health check failure / stuck at step 11
- OpenClaw gateway fails to start / Control UI unreachable (steps 12, ports 18789/19001)
- codex-web-local / WebView blank or won't load (step 13-14, port 18923)
- App gets killed in background / agents stop mid-task
- Port conflicts (another app or a previous zombie process holding 18789/18923/18924/19001)
- Build-from-source failures (`npm run build`, `download-bootstrap.sh`, `build-server-bundle.sh`, `gradlew assembleDebug`)

## 3. Confirm before fixing

For anything beyond a one-line settings toggle, get a confirming signal before prescribing the fix — don't pattern-match the symptom text alone:
- **If shell access into `$PREFIX` is available and the symptom looks OpenClaw-layer (config, auth, gateway, model auth, skills, "something's off after an update," agent hung/unresponsive) — run `openclaw doctor --fix --verbose` first.** See `references/openclaw-cli-diagnostics.md`. This is OpenClaw's own built-in diagnostic/repair tool and is faster and more reliable than manually chasing the same class of symptom through logs. It's safe to run repeatedly and backs up config before changing anything.
- If adb is available, pull the relevant `adb logcat` lines (grep for `CodexServerManager`, `BootstrapInstaller`, `CodexForegroundService` tags — these are the three key Kotlin classes per the architecture doc) rather than reasoning abstractly.
- If shell access into `$PREFIX` (`/data/user/0/com.codex.mobile/files/usr`) is available, check for the specific files/processes the symptom-map entry names (e.g., is `~/.openclaw/openclaw.json` present and valid JSON; is something already bound to port 18789).
- If neither is available, say so plainly and give the best on-screen/settings-level diagnostic Tony can run himself (e.g., "check Android Settings → Apps → AnyClaw → Battery → is 'Unrestricted' or exemption granted").

## 4. Prescribe and verify

- Give the fix as concrete numbered steps — exact adb commands, exact settings paths, exact files to check/edit.
- State what "fixed" looks like (e.g., "WebView should load http://127.0.0.1:18923/ and you'll see the chat composer") so Tony can self-verify.
- If the fix involves rebuilding from source, validate any script/config changes yourself first — don't hand over unverified shell commands. If you write or modify a script, dry-run or syntax-check it before presenting it, consistent with always validating code before delivery.
- If a fix is a genuine trade-off (e.g., disabling W^X-bypass workaround vs. accepting `targetSdk 28`'s security posture), say so — don't silently pick one.

## 5. If the symptom map doesn't cover it

Search first — this app is actively developed (85k+ commits, forked from `openclaw/openclaw`) and new failure modes appear in GitHub Issues faster than any static doc can track:
1. Web-search the exact error string + "AnyClaw" or "openclaw-android"
2. Check `https://github.com/OpenClawAndroid/openclaw-android-assistant/issues` for the error string
3. Check the upstream `https://github.com/openclaw/openclaw` issues if the symptom looks like an OpenClaw-core problem rather than an Android-porting problem
4. Only after searching, reason from architecture — don't fabricate a plausible-sounding cause. If you can't converge on a confident diagnosis, say so plainly and lay out what additional log/shell output would let you narrow it down, rather than guessing.

After resolving a genuinely new failure mode (not already in `references/symptom-map.md`), append it there in the same format so the skill improves over time — ask Tony first if he wants that.

## Reference files

- `references/architecture.md` — full architecture: 14-step startup sequence, port map, directory layout, tech stack versions, requirements. Read this before diagnosing anything you're not immediately sure about.
- `references/symptom-map.md` — the differential-diagnosis table: symptom → ranked causes → confirming check → fix. Primary working document for Android-app-layer symptoms (crashes, extraction hangs, permission-denied execs, WebView/port issues, background kills).
- `references/openclaw-cli-diagnostics.md` — OpenClaw's own built-in `openclaw doctor` / `gateway status` / `models auth` CLI diagnostic layer, plus Custodian skills (release-versioned auto-repair playbooks) and skill-loading precedence. **Check this first when shell access exists and the symptom is OpenClaw-layer** (config, auth, gateway, model auth, agent hangs, skills not loading) — it's usually faster than manually working the symptom map for that class of problem.
- `references/build-from-source.md` — full build/setup command reference for when the fix requires rebuilding rather than just fixing a running install.
