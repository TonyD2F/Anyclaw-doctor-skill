# 🩺 AnyClaw Doctor

**A diagnostic & repair skill for [AnyClaw](https://github.com/OpenClawAndroid/openclaw-android-assistant)** — the Android app running 🦞 OpenClaw + 💻 Codex CLI + 🦀 Claw Code/OpenClaude, all in one APK, no root, no PC.

> Three AI agents crammed into a phone. Sometimes phones get sick. This is the house call.

---

## 🤕 What This Skill Does

AnyClaw does a *lot* under the hood — proot, an embedded Termux userland, a DNS/TLS proxy bridge, native Rust binaries, koffi builds, OAuth flows, a foreground service holding it all together in the background. When something breaks, it's rarely obvious *why* from the error alone.

This skill turns Claude into a systematic doctor for it:

```
🩺 Intake  →  🔍 Differential Diagnosis  →  ✅ Confirm  →  💊 Prescribe  →  🧪 Verify
```

No guessing. No "have you tried turning it off and on again" as a first move. Every fix traces back to a confirmed root cause.

---

## 📦 What's Inside

| File | What it covers |
|---|---|
| 🗂️ `SKILL.md` | The doctor workflow itself — how Claude triages, diagnoses, and treats |
| 🏗️ `references/architecture.md` | Full port map, the 14-step startup sequence, and *why* the proxy + `targetSdk 28` workarounds exist |
| 🧭 `references/symptom-map.md` | 12 symptom categories, each with ranked causes, a confirming check, and the fix |
| 🔧 `references/build-from-source.md` | Full build pipeline breakdown + a clean `adb logcat` capture recipe |

---

## 🚑 Symptoms It Diagnoses

- 💥 App crashes on launch
- 🐧 Bootstrap extraction hangs
- 🔒 "Permission denied" executing binaries (yep, it's the W^X thing)
- 🧱 koffi build failures
- 🌐 "No address associated with hostname" (spoiler: it's almost always the proxy)
- 🔑 OAuth login page not opening
- 🦞 OpenClaw gateway won't start
- 📱 Blank WebView
- 😴 App getting killed in the background
- 🚧 Port conflicts (18789 / 18923 / 18924 / 19001)
- 🛠️ Build-from-source failures

---

## 🧠 How It Thinks

Every fix in the symptom map follows the same shape:

1. **Ranked causes** — most likely first, cheapest check first
2. **A confirming check** — logcat pattern, file to inspect, setting to verify — *before* prescribing anything
3. **The fix** — concrete, numbered, verifiable
4. **What "fixed" looks like** — so you know when to stop

If a symptom isn't in the map, the skill searches GitHub Issues and the web *before* guessing — and if it still can't converge on a confident answer, it says so instead of making one up.

---

## 🚀 Using It

Drop `anyclaw-doctor/` into wherever your Claude skills live, then just describe what's going wrong with AnyClaw / OpenClaw / Codex / Claw Code on Android. The skill triggers automatically.

---

## 🦾 Built For

- 📱 Android 7.0+ (API 24), ARM64 only
- 🦞 [OpenClaw](https://openclaw.ai)
- 💻 [OpenAI Codex CLI](https://github.com/openai/codex)
- 🦀 [Claw Code / OpenClaude](https://claw-code.codes)
- 🐧 Termux-derived embedded Linux, via proot, no root required

---

## 🙏 Credit

Built against [`OpenClawAndroid/openclaw-android-assistant`](https://github.com/OpenClawAndroid/openclaw-android-assistant) — three agents, one APK, your pocket.

*They leaked Claude Code. We wrote it a doctor's note.* 😏
