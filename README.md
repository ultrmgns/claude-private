# Claude Private Edition

A patched build of Claude Code CLI (v2.1.88) with **all telemetry, analytics, and phone-home behavior removed**.

One binary. No telemetry. Drop-in replacement.

---

## Install

Download `claude-private-2.1.88.run` from [Releases](../../releases), then:

```bash
chmod +x claude-private-2.1.88.run
./claude-private-2.1.88.run
```

Pass `--prefix /custom/path` to install somewhere other than `~/.local/bin/`.

The binary is self-contained (Bun executable). No dependencies. Runs on any **Linux x86_64** system.

---

## Usage

```bash
claude-private                          # interactive session
claude-private -p "explain this code"   # non-interactive
```

All standard `claude` flags work (`-p`, `--model`, `--allowedTools`, `--add-dir`, etc.).

### Using with alternative backends

Claude Code speaks the **Anthropic Messages API** (`POST /v1/messages`). If your backend speaks a different protocol (OpenAI, vLLM, Ollama, etc.), use [claude-code-router](https://github.com/musistudio/claude-code-router) to translate, then point at it:

```bash
ANTHROPIC_BASE_URL=http://localhost:3456 claude-private
```

---

## What Was Removed

The stock Claude Code CLI contains 17+ phone-home mechanisms, many firing on background timers even when you're idle:

| Mechanism | How often it fires | Destination |
|---|---|---|
| Datadog event logging | Every **15 seconds** | `http-intake.logs.us5.datadoghq.com` |
| 1st-party event logging | Batched periodic | `api.anthropic.com/api/event_logging/batch` |
| BigQuery metrics export | Every **5 minutes** | `api.anthropic.com/api/claude_code/metrics` |
| GrowthBook feature flags | Periodic refresh | GrowthBook remote eval |
| Remote managed settings | Every **1 hour** | `api.anthropic.com/api/claude_code/managed_settings` |
| Policy limits polling | Every **1 hour** | `api.anthropic.com/api/claude_code/policy_limits` |
| User settings sync | Background | `api.anthropic.com/api/claude_code/user_settings` |
| Metrics opt-out check | Background (24h cache) | `api.anthropic.com/.../metrics_enabled` |
| Session transcript ingress | During conversations | `api.anthropic.com/v1/session_ingress/` |
| MCP registry prefetch | On startup | `api.anthropic.com/mcp-registry/` |
| Bootstrap API | On startup | `api.anthropic.com/api/claude_cli/bootstrap` |
| Grove API | Background | `api.anthropic.com/api/claude_code_grove` |
| Referral eligibility | Background | `api.anthropic.com/.../referral/eligibility` |
| Trusted device enrollment | After login | `api.anthropic.com/api/auth/trusted_devices` |
| API preconnect warmup | On startup | HEAD request to API |
| Plugin auto-updates | Background | Remote marketplaces |
| Background housekeeping | Various timers | Magic docs, skill improvement, etc. |

**All removed.** The Claude API endpoint for actual conversations (`POST /v1/messages`) is untouched.

### How it was done

**Layer 1 — Binary patching.** All telemetry URLs in the compiled executable were replaced with same-length dummy strings. The Datadog client token was zeroed out. Binary size unchanged, no structural modifications.

**Layer 2 — Environment overrides.** The wrapper script sets:
```
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
DISABLE_TELEMETRY=1
DISABLE_AUTOUPDATER=1
CLAUDE_CODE_ENABLE_TELEMETRY=0
OTEL_METRICS_EXPORTER=none
OTEL_LOGS_EXPORTER=none
OTEL_TRACES_EXPORTER=none
```

Source-level patches across 19 files

---

## Rebuilding from a newer version

```bash
# Patch a new binary
python3 patch_binary.py /path/to/new/claude ./claude-notelemetry

# Rebuild the .run installer
bash make-run.sh
```

`patch_binary.py` does same-length byte replacement on 15 URL patterns. If Anthropic changes their telemetry endpoints in a future version, you may need to update the patterns.

---

## Files

```
claude-private-2.1.88.run    Self-extracting installer 
claude-notelemetry            Patched binary
claude-private                Wrapper script
patch_binary.py               Reproducible patching script
make-run.sh                   Builds the .run installer
install.sh                    Alternative installer
```

---

## Limitations

- **Linux x86_64 only** — compiled Bun executable.
- **No auto-updates** — disabled by design. Re-patch when you want a new version.
- **Some features degraded** — Grove, referrals, team memory sync, and anything depending on remote feature flags won't work. Core functionality (conversations, tools, file editing, bash, MCP) is unaffected.

---
### MacOS support

#### Installation
```brew install node
npm install -g @anthropic-ai/claude-code
python3 patch_binary.py $(which claude) ./claude-notelemetry
```

After patching, run:
`bash install.sh` 
To put it in ~/.local/bin/ so you are able to run it from the terminal

Run 
```
sudo codesign --force --sign - --preserve-metadata=entitlements ~/.local/bin/claude-private
sudo codesign --force --sign - --preserve-metadata=entitlements ~/.local/bin/claude-notelemetry
```
To bypass code signing restrictions in MacOS 26. Will need to be done after every patch.

To run it (recommend LM Studio server (free) with endpoint support: `anthropic-compatible`
example command: `ANTHROPIC_BASE_URL=http://localhost:1234 claude-private`

## Based on

Claude Code CLI v2.1.88 by [Anthropic](https://anthropic.com). Source exposed 2026-03-31 via npm registry source map file.
