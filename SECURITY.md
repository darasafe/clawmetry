# ClawMetry Hardened Mode — Security Documentation

## What Changed

This fork applies a **hardened mode** to the ClawMetry observability dashboard, making it **read-only** and **auth-gated**.

### Changes Made

1. **All write/control endpoints disabled** — POST/PUT/DELETE/PATCH requests return `403` with `"Write endpoints disabled in hardened mode"`. Exception: OTEL ingest endpoints (`/v1/metrics`, `/v1/traces`) remain open for telemetry ingestion.

2. **Sensitive data redacted** — All JSON API responses are filtered: keys matching `token`, `key`, `secret`, `password`, `credential`, `api_key`, `apiKey`, `bot_token` (case-insensitive) have their values replaced with `"[REDACTED]"`.

3. **Process environment snooping removed** — The code that read `/proc/{pid}/environ` to extract `OPENCLAW_GATEWAY_TOKEN` from running processes has been disabled.

4. **Phone-home removed** — The `api.ipify.org` call (which leaked the server's public IP) now returns `"disabled"`.

5. **Forced localhost binding** — The dashboard always binds to `127.0.0.1` regardless of `--host` argument, preventing network exposure.

6. **Dashboard auth token** — A random token is generated on first startup and saved to `~/.clawmetry-auth-token`. All requests (except `/` and `/auth`) require `?token=VALUE` or `Authorization: Bearer VALUE`.

## Threat Model

**Who are we protecting against?**

- **Network attackers** who could reach the dashboard if it were bound to `0.0.0.0` — mitigated by forced localhost binding.
- **Unauthorized local users** who could use the dashboard to invoke gateway tools, modify cron jobs, pause/resume the agent, or exfiltrate API keys — mitigated by disabling write endpoints and redacting secrets.
- **Supply chain / phone-home risks** — mitigated by removing the `ipify.org` call.

**What we're NOT protecting against:**
- A user with shell access to the machine (they can read files directly).
- Compromised dependencies in the Python environment.

## What's Still Exposed (Read-Only)

Even in hardened mode, the following data is visible to authenticated users:

- **Session transcripts** — full conversation history between user and agent
- **Memory files** — agent's long-term memory contents
- **Log files** — gateway and agent logs
- **Flow visualization** — tool calls, reasoning traces
- **System metrics** — CPU, memory, disk usage
- **Budget/cost data** — token usage and spending

All of this is **read-only**. No modifications can be made through the dashboard.

## How to Rotate the Auth Token

```bash
rm ~/.clawmetry-auth-token
# Restart the dashboard — a new token will be generated
```

The new token will be printed in the startup banner.
