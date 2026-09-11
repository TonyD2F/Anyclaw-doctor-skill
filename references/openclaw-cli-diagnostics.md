# OpenClaw Native CLI Diagnostics

**Read this before working through `symptom-map.md` line-by-line if shell access into the app's userland (`$PREFIX`) is available.** OpenClaw ships its own built-in diagnostic/repair CLI, and running it first is almost always faster than manually chasing individual symptoms — it's the same tool the OpenClaw project itself points people to as "run this first."

Verify exact flag availability against the installed version (`openclaw --version`) since flags have been added over time — but the core commands below are stable and well-established as of current OpenClaw releases.

## The core command: `openclaw doctor`

Five postures:

| Posture | Command | Behavior |
|---|---|---|
| Inspect | `openclaw doctor` / `openclaw doctor --json` | Advisory checks, read-only, human or machine-readable |
| Repair | `openclaw doctor --fix` (alias `--repair`) | Applies supported repairs, prompting unless a repair is safe to do non-interactively |
| Lint | `openclaw doctor --lint [--json]` | Read-only findings with threshold-based exit codes (useful for scripting/CI, less relevant here) |
| Shared SQLite maintenance | `openclaw doctor --state-sqlite compact` | Checkpoints/compacts/verifies the canonical shared state DB |
| Session SQLite tools | `openclaw doctor --session-sqlite <mode>` | Inspects/maintains session SQLite, can import legacy history |

`doctor` checks, in sequence: Node.js version compatibility, config file syntax (JSON5, unknown/deprecated keys, type mismatches), model auth/token status, and Gateway connectivity — i.e. almost exactly the failure surface covered in `symptom-map.md` entries #7 and #8, but checked programmatically instead of by manual log-reading.

`--fix` is safe to run repeatedly — it won't touch things that are already correct — and it backs up the config first (typically to `~/.openclaw/openclaw.json.bak`) before making any change, so it's low-risk to run proactively.

**Recommended first move whenever shell access exists and the symptom involves OpenClaw config, auth, gateway, or "something's off after an update":**

```bash
openclaw doctor --fix --verbose
```

If that alone resolves the issue, no need to walk the rest of the symptom map for that entry. If `doctor` reports something it *can't* auto-fix, that output tells you exactly what's broken and is a much better starting point for a manual fix than reasoning from symptom text alone.

### What `--fix` commonly repairs

- Regenerating a missing/stale Gateway Token (device-token mismatch — a `"unauthorized: gateway token mismatch"` type error clears each agent's stale token from `auth.json` and re-pairs)
- Fixing corrupted/invalid JSON5 config syntax
- Migrating deprecated config keys to current schema (common after an OpenClaw version bump)
- Removing unknown config keys (and reporting exactly what it removed)
- Rebuilding workspace/session indexes
- Refreshing expired OAuth tokens

### What `--fix` does NOT catch

`doctor` is thorough but not omniscient — it won't diagnose problems outside OpenClaw's own config/state layer. In particular it won't catch: the Android-specific W^X/`targetSdk` permission-denied class of failure, the CONNECT-proxy-not-running DNS failure, port conflicts from a zombie process, or anything in the Android app shell itself (foreground service, WebView, bootstrap extraction). Those are still `symptom-map.md` territory — `doctor` and the symptom map are complementary, not redundant.

## Companion commands

```bash
openclaw status --all          # complete diagnostic snapshot — gateway, channels, skills, plugins, memory
openclaw gateway status        # is the gateway (port 18789) actually up and healthy
openclaw gateway health        # more detailed health readout
openclaw gateway restart       # restart without a full stop/start cycle
openclaw gateway stop
openclaw gateway start
openclaw models auth setup-token   # re-auth a model provider if a 401 shows up
openclaw --version
openclaw docs                  # search official docs from the CLI directly
```

**"Agent frozen / unresponsive" quick sequence** (this is a distinct failure mode from anything in the Android-specific symptom map — it's an OpenClaw-layer hang, not a startup-sequence hang):
```bash
openclaw gateway restart
# if that doesn't come back cleanly:
openclaw gateway stop
openclaw gateway start
# if still stuck:
openclaw doctor --fix
```

## Before running `--fix`, when the config matters

`--fix` backs up automatically, but for anything beyond a routine check it's still worth a manual snapshot first, especially before a version-migration-triggered repair:

```bash
cp -r ~/.openclaw/ ~/.openclaw.backup-$(date +%Y%m%d)
openclaw doctor --fix
diff ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak   # review exactly what changed
```

## Custodian skills — a distinct, more powerful repair layer

Separately from `openclaw doctor`, OpenClaw ships **Custodian skills**: release-versioned operational playbooks that live under `custodian-skills/` in this repo (visible at the repo root). These are more elaborate than a single CLI command — each one follows a fixed five-step workflow contract:

1. **Gather** — reads redacted current config, probes live state
2. **Mutate** — validated non-interactive writes only (`openclaw config set` / `openclaw config patch` from a trusted shell), never a raw file edit
3. **Repair** — runs `openclaw doctor` itself and separates diagnosis from any approved repair
4. **Prove** — exercises one live end-to-end outcome to confirm the fix actually worked, not just that a command exited 0
5. **Report** — records what changed, what was observed, what remains

**Important scoping caveat:** Custodian skills only load for the specific agent OpenClaw resolves as the "system agent" (via `agents.defaults.systemAgent.agentId` in config; if unset, OpenClaw falls back to the sole configured agent, or legacy `main` when no explicit agent roster exists). If Tony's setup has multiple agents configured and none is designated as the system agent, **no agent gets the Custodian skill library at all** — it's silently absent, not degraded. If a Custodian-skill-related repair seems like the right move but doesn't seem to be available/triggering, check whether a system agent is actually configured before assuming the skill itself is broken.

This matters for diagnosis: if Tony describes OpenClaw doing something that sounds like an automated self-repair workflow (multi-step gather → fix → verify → report), that's Custodian skills at work, not a manually-run `doctor --fix`. Don't conflate the two when interpreting logs or behavior.

**Named first-wave Custodian skills** (verify current names/set against the live repo — this is a "first wave," expect more over time):

| Skill | Outcome |
|---|---|
| `configure-channel` | Configure and send a confirmed test message through a channel (Discord, Slack, Telegram, WhatsApp, etc.) |
| `add-model-provider` | Configure API-key or OAuth/subscription provider access and run one live Gateway inference to prove it |
| `diagnose-gateway` | Read-only(-first) gateway diagnosis — this is the one most relevant to AnyClaw troubleshooting specifically, since it targets the same gateway (port 18789) that shows up throughout `symptom-map.md` entry #8 |

If Tony's setup has a system agent configured and `diagnose-gateway` is available, it's worth trying as an alternative/complement to manual `openclaw doctor` — it's specifically scoped to gateway issues rather than the broader config/auth surface `doctor` covers.

## Skill-loading precedence (relevant if a skill isn't showing up / behaving as expected)

Separate from Custodian skills, OpenClaw's general skill-discovery precedence (highest to lowest) is:

1. `<workspace>/skills` — per-agent
2. `<workspace>/.agents/skills` — project-agent
3. `~/.agents/skills` — personal-agent (default state)
4. `<state-dir>/skills` — shared managed, all agents on that state
5. `<state-dir>/agents/<agentId>/agent/workshop-skills` — workshop, that agent only
6. Custodian skills — bundled tier, but only for the resolved system agent
7. `skills.load.extraDirs` (config) + plugin skills — lowest precedence

If a skill Tony expects isn't loading, or a different version of a same-named skill is loading than expected, this precedence order — not a bug — is usually the explanation. Check `skills.entries.<name>.enabled` in config too; an individually-disabled skill won't show up regardless of precedence.

## When to reach for this file vs. `symptom-map.md`

- **OpenClaw-layer symptom** (gateway auth, config schema, model auth, agent hang, skill not loading) → try `openclaw doctor --fix` first, this file
- **Android-app-layer symptom** (crash on launch, stuck extraction, permission denied executing binaries, WebView blank, background kill, port conflict from a zombie *Android process*) → `symptom-map.md`

In practice these overlap at the edges (e.g., a DNS/proxy error touches both the Android CONNECT-proxy layer and OpenClaw's own network calls) — when in doubt, run `openclaw doctor` first since it's cheap and fast, then fall back to the Android-specific symptom map if it comes back clean but the problem persists.
