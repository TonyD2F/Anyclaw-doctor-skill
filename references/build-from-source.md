# Build-from-Source Reference

Use this when the fix requires rebuilding the app rather than repairing a running install (e.g., Tony is developing against the repo, or a fix genuinely requires a code/config change like `targetSdk`).

## Full sequence

```bash
git clone https://github.com/friuns2/openclaw-android-assistant.git
cd openclaw-android-assistant

npm install && npm run build

cd android && bash scripts/download-bootstrap.sh
bash scripts/build-server-bundle.sh && ./gradlew assembleDebug

adb install -r app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n com.codex.mobile/.MainActivity
```

Note: the repo uses `pnpm` (there's a `pnpm-workspace.yaml` and `pnpm-lock.yaml` at the root) even though the README's quick-start shows plain `npm install`. If `npm install` produces lockfile-drift warnings or dependency resolution errors that a fresh clone shouldn't have, switch to `pnpm install` instead — it's the tool the lockfile was actually generated with, and mixing package managers on a `pnpm-lock.yaml` repo is a common, avoidable source of "works on one machine, not another" bugs.

## Step-by-step: what each step does and how it fails

### 1. `npm install && npm run build`
Builds the TypeScript/Vue frontend (`src/` — codex-web-local) and any core packages. Failure modes:
- Node version mismatch — check `node-version.mjs`/`node-version.d.mts` in repo root for the pinned/expected version, compare against `node --version`.
- Lockfile drift if `npm` was used against a `pnpm-lock.yaml` repo (see note above) — prefer `pnpm install`.
- Missing native toolchain for any native deps at this stage (uncommon at the frontend-build layer, more relevant later for koffi specifically inside the Android build).

### 2. `bash scripts/download-bootstrap.sh`
Fetches the Termux `bootstrap-aarch64.zip` userland archive that gets bundled into the APK. Failure modes:
- Network access to GitHub release assets — check the network configuration allows `github.com`, `codeload.github.com`, `raw.githubusercontent.com`, `release-assets.githubusercontent.com`.
- The specific bootstrap release version the script points at may have been superseded/removed upstream by Termux — check the script's target URL against current Termux bootstrap releases if this fails on an otherwise-healthy network.

### 3. `bash scripts/build-server-bundle.sh`
Packages the already-built frontend (`src/` output) plus Express server code into `android/app/src/main/assets/server-bundle/`. **Must run after** step 1 completes successfully — it has nothing to bundle otherwise. If this fails, first confirm `npm run build`'s output actually exists where this script expects it before debugging the script itself.

### 4. `./gradlew assembleDebug`
Standard Android/Gradle build. Failure modes here are generic Android tooling issues, not project-specific — trust and search the actual Gradle error message directly rather than assuming it's an AnyClaw-specific problem:
- JDK version mismatch (check `gradle.properties`/`build.gradle.kts` for required JDK).
- Android SDK/NDK not installed or `local.properties` not pointing at a valid `sdk.dir`.
- `targetSdk`/`compileSdk` version drift — if Tony is deliberately checking/fixing `targetSdk = 28` per the W^X workaround (see architecture.md), this is the file to check: `android/app/build.gradle.kts`.

### 5. `adb install -r app/build/outputs/apk/debug/app-debug.apk`
Installs the freshly built debug APK, replacing any existing install (`-r` = reinstall, keeps app data unless data was incompatible). If Tony wants a genuinely clean slate rather than an upgrade-in-place, uninstall first (`adb uninstall com.codex.mobile`) before this step.

### 6. `adb shell am start -n com.codex.mobile/.MainActivity`
Launches the app directly via adb instead of tapping the icon — useful because it lets logcat capture from the very first moment of the startup sequence, which is exactly what you want when diagnosing an early-startup failure. Pair this with a fresh `adb logcat -c` (clear buffer) immediately before running it, then `adb logcat` in a second window, to get a clean capture of just this launch.

## Recommended diagnostic launch sequence

When Tony has adb access and needs to capture a clean failure trace:

```bash
adb logcat -c                                          # clear old logs
adb shell am force-stop com.codex.mobile                # ensure no zombie process
adb shell am start -n com.codex.mobile/.MainActivity     # launch fresh
adb logcat | grep -E "CodexServerManager|BootstrapInstaller|CodexForegroundService|MainActivity"
```

This captures exactly the startup sequence from a clean state, filtered to the four key classes named in the architecture doc — enough to map any hang or crash to a specific numbered startup step.
