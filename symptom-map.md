# Symptom Map — Differential Diagnosis

Format per entry: **Symptom** → ranked likely causes → confirming check → fix. Check causes in order — cheapest/most-likely first. Don't apply a fix without confirming, if a confirming check is available.

---

## 1. App crashes immediately on launch

**Likely causes, ranked:**
1. Device is not ARM64 (x86/x86_64 emulator, or an unsupported architecture) — the app ships only `aarch64` native binaries.
2. Android version below 7.0 (API 24).
3. `targetSdk` in the installed build was changed/bumped to 29+ (only relevant if Tony built from source himself).
4. Corrupted install (partial APK download/sideload).

**Confirm:** `adb logcat` around the crash, filtered to the app's PID or package name. A `UnsatisfiedLinkError` or `ClassNotFoundException` right at startup points to (1) or (4). Check device arch with `adb shell getprop ro.product.cpu.abi` — must be `arm64-v8a`.

**Fix:**
- Wrong arch (emulator): use a real ARM64 device, or an emulator image with ARM64 system image + Play Store's arm translation (unreliable) — the project has no x86 build path.
- Old Android: not fixable; device doesn't meet the API 24 floor.
- targetSdk drift (self-built): check `android/app/build.gradle.kts`, confirm `targetSdk = 28`, rebuild.
- Corrupted install: uninstall fully, re-download the APK fresh (`https://friuns2.github.io/openclaw-android-assistant/` or Google Play), reinstall.

---

## 2. Hangs/stuck during bootstrap extraction (startup step 2)

**Likely causes, ranked:**
1. Insufficient storage — the extraction needs room for the ~500MB total footprint plus temporary extraction overhead.
2. Extraction interrupted (app backgrounded/killed mid-extraction before the foreground service was granted, or before battery optimization exemption was accepted).
3. Corrupted `bootstrap-aarch64.zip` bundled in a bad build/sideload.

**Confirm:** Check available storage (Settings → Storage). Check `adb logcat` for `BootstrapInstaller` tag — it will log extraction progress/failure explicitly.

**Fix:**
- Free up storage (need comfortably more than 500MB free — extraction needs headroom beyond the final footprint).
- Force-stop the app, clear app data (Settings → Apps → AnyClaw → Storage → Clear Data — this wipes the extracted userland and any auth, so OAuth login will be needed again), relaunch and let it complete without switching apps or letting the screen lock during first-run setup.
- If clearing data and retrying still fails at the same step, the APK itself may be bad — re-download.

---

## 3. "Permission denied" executing binaries

**Likely causes, ranked:**
1. `targetSdk` is not 28 in the installed build (see architecture.md — Android 10+ W^X enforcement blocks exec from app-writable storage above API 28).
2. Device-specific stricter SELinux policy (some OEM ROMs even at targetSdk 28) — rare but real.
3. A binary lost its executable bit during extraction/copy (corrupted or partial extraction, see #2 above).

**Confirm:** If self-built, check `android/app/build.gradle.kts` for `targetSdk = 28`. `adb logcat` around the failing exec call will show the exact path and a bare "Permission denied" (not a missing-file error) — that pattern strongly indicates the W^X issue rather than a missing/corrupt binary.

**Fix:**
- Self-built with wrong targetSdk: fix the manifest/gradle config to `targetSdk = 28`, **rebuild**, **reinstall**, and reboot the device (targetSdk permission changes sometimes need a reboot to fully apply at the OS level).
- Official APK/Play Store build with this symptom on an otherwise-supported device: this points to cause #2 (OEM SELinux hardening) — there is no in-app fix; it's a known class of device-specific incompatibility (same root cause historically seen with Termux itself on certain devices). Confirm by checking whether Termux itself (installed separately, F-Droid build) also fails to start a shell on that device — if so, this device just isn't going to run any Termux-derived app reliably without root.
- Corrupted extraction: clear app data and retry (see #2 above).

---

## 4. koffi build failure (startup step 5, OpenClaw build deps)

**Likely causes, ranked:**
1. Missing or broken clang/cmake/make toolchain inside the extracted userland.
2. An old/incompatible koffi version bundled in a stale build (koffi's own changelog notes explicit "fix build problems on Android/Termux" and "use Clang to build Linux ARM64 prebuild" fixes in its 2.15.x line — if the bundled server-bundle pins an older koffi, this class of failure is exactly what you'd expect).
3. Corrupted/partial install of build deps at step 5 (network interruption during that install).

**Confirm:** `adb logcat` for the koffi build's compiler output — look for a clang/cmake invocation failing, vs. an npm-level "prebuilt binary not found, rebuilding from source" message (the latter is often recoverable if the toolchain is intact; a genuine compile error is not).

**Fix:**
- Verify clang/cmake/make are installed and binary-patched (per the project's own troubleshooting table) — this is usually a matter of retrying step 5's install cleanly: clear app data and let it redo the full install rather than trying to patch a half-finished toolchain in place.
- If self-building: check `package.json`/`pnpm-lock.yaml` for the pinned koffi version; bumping to a current 2.15.x+ release (which explicitly fixed Android/Termux build issues) may resolve it — verify current koffi version and Android-specific changelog notes at https://koffi.dev/changelog before pinning a specific version.
- If this persists after a clean reinstall, it's worth filing/searching a GitHub issue against the repo — koffi-on-Android is a known-fragile intersection and upstream fixes land in both koffi and the app's bundled deps over time.

---

## 5. DNS/proxy errors — "No address associated with hostname" (steps 5-9, proxy.js)

**Likely causes, ranked:**
1. CONNECT proxy (port 18924) never started or crashed — this is by far the most common cause; the error text itself is characteristic of a musl binary trying (and failing) to do its own DNS resolution because it isn't actually routed through the proxy.
2. Actual device network connectivity problem (rare — but rule it out first since it's the cheapest check).
3. Something else already bound to port 18924, so the proxy failed to start.

**Confirm:** First, trivially confirm actual internet works (open any browser tab on the device). Then check `adb logcat` for `CodexServerManager` around proxy startup — it will show whether `proxy.js` launched successfully on 18924. If shell access into `$PREFIX` exists, `netstat`/`ss` (if available) or checking for a listening process on 18924 confirms directly.

**Fix:**
- Internet is fine but proxy didn't start: force-stop and relaunch the app — this is usually transient (a startup race or the proxy process dying silently). If it recurs every launch, clear app data for a full clean re-provision.
- Port 18924 already held by something else: identify and kill the conflicting process (see #11, port conflicts, below), or reboot the device to clear any leftover process.
- Persistent failure after clean reinstall: this may indicate a genuinely new proxy.js bug — search GitHub issues for the exact error string before assuming device-specific network config (VPN, private DNS, or a firewall app) is interfering; those are also worth ruling out if Tony has any active.

---

## 6. OAuth login page doesn't open / login loop (startup step 10)

**Likely causes, ranked:**
1. No default browser set on the device (the project's own troubleshooting table names this explicitly).
2. Browser opened but the OAuth callback isn't reaching the app (deep link / intent filter issue, or the local callback server the OAuth flow depends on isn't reachable).
3. Stale/corrupted auth state from a previous partial login attempt.

**Confirm:** Check Settings → Apps → Default apps → Browser app — is one actually set? If a browser opens but login never completes / app doesn't detect success, that points to (2).

**Fix:**
- No default browser: set one in Android settings, retry.
- Callback not completing: force-stop the app, clear app data (this wipes any half-completed auth state) and retry a clean login. If it still fails, check whether the device has any browser-level extension/setting blocking redirects or intercepting the callback URL scheme.
- If login succeeds in-browser but the app never picks it up, this is worth a fresh GitHub issue search — deep-link/callback handling is a common source of platform-specific regressions after Android OS updates.

---

## 7. Health check failure / stuck at step 11

**Likely causes, ranked:**
1. One of the services from earlier steps (proxy, Codex CLI, OpenClaw) didn't actually finish starting, so the health check correctly fails.
2. Health check itself is timing out due to a genuinely slow device/first-run (large extraction + multiple installs can be slow on lower-end hardware).

**Confirm:** This step is downstream of steps 2-10 — walk backward through `adb logcat` to find the actual first failure, rather than treating "stuck at health check" as the root cause. It rarely is.

**Fix:** Resolve whatever upstream step actually failed (see the relevant entry above). If genuinely just slow hardware and nothing failed upstream, give it more time before assuming it's broken — first-run setup on this app does a lot of work (Linux extraction, multiple package installs, a native binary build).

---

## 8. OpenClaw gateway fails to start / Control UI unreachable (step 12, ports 18789/19001)

**Likely causes, ranked:**
1. `~/.openclaw/openclaw.json` config or auth-profiles missing/invalid (named explicitly in the project's own troubleshooting table).
2. Port 18789 or 19001 already held by a zombie process from a previous session.
3. OpenClaw's own install/build (step 5) didn't complete cleanly, so the gateway binary/service itself is broken.

**Confirm:** If shell access into `$PREFIX` exists, run `openclaw doctor` (read-only first — see `openclaw-cli-diagnostics.md`) before anything else; it directly checks config syntax, deprecated keys, and gateway connectivity and will usually name the exact problem. Otherwise check `~/.openclaw/openclaw.json` exists and is valid JSON, and check `~/.openclaw/` for an auth-profiles file. Check `adb logcat` for `CodexServerManager` gateway-start lines.

**Fix:**
- If shell access exists, try `openclaw doctor --fix --verbose` first — it backs up config automatically and handles the common cases (stale gateway token, invalid JSON5, deprecated keys) non-destructively, which is safer and faster than the manual routes below.
- Invalid/missing config with no shell access, or `doctor` itself can't repair it: this usually means step 12's auth-config write never completed — clear app data and let it re-provision fully rather than hand-editing JSON inside the sandboxed app storage (which is awkward without root and easy to make worse).
- Port conflict: see #11 below.
- Broken step-5 install: address the koffi/build-deps failure (#4 above) first; the gateway can't start reliably if OpenClaw's own install was incomplete.

---

## 9. codex-web-local / WebView blank or won't load (steps 13-14, port 18923)

**Likely causes, ranked:**
1. codex-web-local server (18923) never started because an earlier step failed silently.
2. WebView itself is stale/broken (Android System WebView needs an update, or is disabled).
3. Port 18923 conflict.
4. **Unconfirmed but mechanically plausible — cleartext traffic blocked.** The WebView loads plain `http://127.0.0.1:18923/`, not HTTPS. On Android 9+ (API 28+), `WebView` does NOT honor the app-level `android:usesCleartextTraffic` manifest flag the way most other HTTP clients do — it needs either an explicit `NetworkSecurityConfig` domain exception or in-code `WebViewClient` handling, or it will silently fail to load cleartext content. I have not directly inspected this repo's `AndroidManifest.xml`/network security config to confirm whether this is handled, so treat this as a hypothesis to check, not a known cause — but if #1-3 are all ruled out and the WebView is *specifically* blank (not crashing, not showing a browser-style error page) while `curl http://127.0.0.1:18923/` from a shell inside `$PREFIX` succeeds, this is next to check.

**Confirm:** Walk backward through the startup sequence in logcat, same as #7 — a blank WebView at the very end is almost always a downstream symptom, not an independent bug. Separately, check whether Android System WebView (a separate, updatable system component) is up to date via the Play Store. For cause #4 specifically: if shell access exists, confirm the server actually responds (`curl -v http://127.0.0.1:18923/` from inside `$PREFIX`) — if that succeeds but the WebView itself never renders anything and logcat shows a WebView-layer error (look for `net::ERR_CLEARTEXT_NOT_PERMITTED` specifically), that confirms #4 rather than #1.

**Fix:**
- Downstream failure: fix the actual upstream step that failed.
- Stale WebView: update "Android System WebView" via Play Store, reboot, retry.
- Port conflict: see #11 below.
- Confirmed `ERR_CLEARTEXT_NOT_PERMITTED` in logcat (self-built only): this needs a manifest/network-security-config fix (`android:networkSecurityConfig` pointing at a config that allows cleartext for `127.0.0.1`/`localhost`, or `android:usesCleartextTraffic="true"` at minimum, though note this alone isn't sufficient for WebView per Android's docs) — search current Android WebView + cleartext documentation before making this change, since exact requirements have shifted across API levels and this is worth verifying live rather than from memory. This is very unlikely to be the cause on the official APK/Play Store build (a shipping app wouldn't ship broken this way) — treat this primarily as a self-build/fork debugging lead.

---

## 10. App gets killed in background / agents stop mid-task

**Likely causes, ranked:**
1. Battery optimization is re-enabled for the app (very common — some OEMs, especially aggressive-battery-management ones like MIUI/OneUI/ColorOS variants, silently re-apply battery restrictions after OS updates even if the user granted an exemption once).
2. Foreground service permission was revoked.
3. Device is simply under heavy memory pressure and the OS is reclaiming background apps regardless of exemptions (some OEM "app hibernation"/"deep sleep" features override standard Android battery-exemption APIs entirely).

**Confirm:** Settings → Apps → AnyClaw → Battery — confirm it says "Unrestricted" or equivalent, not "Optimized"/"Restricted". On OEMs with an extra battery-management layer (common on Xiaomi, Oppo, Vivo, Samsung, Huawei), there is often a *second*, OEM-specific autostart/background-activity toggle beyond stock Android's battery settings — this is frequently the one that actually matters and is easy to miss.

**Fix:**
- Re-grant "Unrestricted" battery status.
- On OEM devices, also check the manufacturer-specific autostart/background-permission manager (naming varies by OEM — "Autostart," "App Auto-launch," "Protected apps," "Background app management," etc.) and allow AnyClaw there too, since stock-Android battery settings alone often aren't sufficient on these skins.
- If this is on a memory-constrained device, note that no in-app setting can fully guarantee background survival under OS-level low-memory kill decisions — this is a genuine device/OS limitation, not something the app can be "fixed" to overcome perfectly.

---

## 11. Port conflicts (18789 / 18923 / 18924 / 19001)

**Likely causes, ranked:**
1. A previous session's process wasn't fully terminated when the app was force-killed, and it's still holding the port.
2. A separate, unrelated app on the device is using the same port (rare on Android since these are high, uncommon ports, but not impossible if Tony runs other local-dev-server-style apps).

**Confirm:** With shell access into `$PREFIX`, check for a listening process on the relevant port. Without shell access, the practical test is: force-stop the app fully (not just backgrounding it — actually force-stop from Settings → Apps, or `adb shell am force-stop com.codex.mobile`), then relaunch — if that resolves it, it was a zombie process from this app itself.

**Fix:**
- Force-stop via Settings → Apps → AnyClaw → Force Stop, then relaunch. This is the standard fix and resolves the large majority of self-caused port conflicts.
- If force-stop doesn't clear it, a device reboot will definitively kill any lingering process.
- If some other app is genuinely squatting one of these ports persistently, identify and close it — but this is uncommon enough that it should be low on the list until self-caused zombie processes are ruled out.

---

## 12. Build-from-source failures

See `build-from-source.md` for the full command sequence and per-step failure modes. Quick pointers:

- `npm install && npm run build` failing → almost always a Node.js version mismatch or a lockfile drift issue; check the pinned Node version (24.13.0 at last check) matches what's actually installed, and prefer `pnpm` if the repo uses `pnpm-lock.yaml` (it does) over plain `npm` for anything beyond the top-level convenience script.
- `download-bootstrap.sh` failing → network/GitHub-release-asset access issue fetching the Termux bootstrap zip; check the script's target URL is still live (bootstrap asset URLs can move between Termux releases).
- `build-server-bundle.sh` failing → check it ran after `npm run build` completed successfully, not before (it packages already-built frontend output).
- `./gradlew assembleDebug` failing → standard Android/Gradle troubleshooting applies (JDK version, Android SDK/NDK installed, `local.properties` pointing at a valid SDK path) — this is generic Android build tooling, not specific to this project, so standard Gradle error messages should be trusted and searched directly.

---

## Adding new entries

When a genuinely new failure mode is resolved that isn't covered above, append it here in this same format (Symptom → ranked causes → confirming check → fix) so this doc stays useful across sessions. Ask Tony before appending — don't do it silently.
