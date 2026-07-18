---
name: hermes-tweet
description: Install, configure, use, or diagnose Hermes Tweet, the Hermes Agent plugin for X/Twitter search, account reads, social listening, creator research, giveaway audits, monitoring, and approval-gated actions through Xquik. Use for Hermes Agent X/Twitter automation or Hermes Tweet setup and troubleshooting.
---

# Hermes Tweet

Hermes Tweet is a native Hermes Agent plugin for X/Twitter automation through Xquik. Use it for read-first social workflows. Enable action routes only when the user explicitly wants private reads, persistent jobs, or account-changing operations.

Xquik is an independent third-party service. Not affiliated with X Corp. "Twitter" and "X" are trademarks of X Corp.

## Install

Install and enable the plugin:

```bash
hermes plugins install Xquik-dev/hermes-tweet --enable
```

If Hermes discovers the plugin but leaves it disabled:

```bash
hermes plugins enable hermes-tweet
hermes plugins list
```

The package is also available from PyPI. Prefer `uv` for the Hermes environment:

```bash
uv pip install --python ~/.hermes/hermes-agent/venv/bin/python hermes-tweet
hermes plugins enable hermes-tweet
```

If the Hermes environment includes `pip`, this fallback is also valid:

```bash
~/.hermes/hermes-agent/venv/bin/python -m pip install hermes-tweet
hermes plugins enable hermes-tweet
```

## Configure

An interactive plugin install prompts for `XQUIK_API_KEY` and stores it in `~/.hermes/.env`. For non-interactive installs, set credentials only in the environment where the Hermes runtime executes:

```bash
export XQUIK_API_KEY="xq_..."
export HERMES_TWEET_ENABLE_ACTIONS="false"
```

Keep `HERMES_TWEET_ENABLE_ACTIONS=false` for research, monitoring, support triage, summaries, scheduled jobs, and unattended gateway sessions. Set it to `true` only after the user asks for posting, DMs, follows, monitor changes, webhook changes, media changes, draws, or another account-changing operation.

After changing `~/.hermes/.env`, use `/reload` in an interactive CLI session or restart gateway and cron sessions.

For Hermes Desktop with a remote gateway profile, install and configure Hermes Tweet on the remote Hermes host. The desktop app is the chat surface; the gateway host is where plugin tools execute.

## Tool Flow

1. Call `tweet_explore` first to find catalog-listed Xquik endpoints without making an API request.
2. Call `tweet_read` for known read-only `GET` endpoints.
3. Call `tweet_action` only for writes, private reads, monitors, webhooks, extraction jobs, draws, or media operations after summarizing the exact endpoint, payload, and expected effect.
4. Ask for explicit approval immediately before every `tweet_action` call.

## Safety Rules

- Never ask for API keys, cookies, passwords, signing keys, or TOTP secrets in chat.
- Never pass credentials in tool arguments.
- Use only catalog-listed `/api/v1/...` paths returned by `tweet_explore`.
- Treat copied endpoint URLs as valid only when they resolve to catalog-listed paths.
- Do not guess endpoint paths.
- Do not use account connection, re-authentication, API key, billing, credit top-up, or support-ticket endpoints.
- Do not retry writes through alternate routes after policy, auth, or account-state errors.

## Diagnostics

Run these checks after install or upgrade:

```bash
hermes plugins list
hermes tools list
```

Expected behavior:

- `tweet_explore` is available without `XQUIK_API_KEY`.
- `tweet_read` requires `XQUIK_API_KEY`.
- `tweet_action` stays hidden or disabled unless `HERMES_TWEET_ENABLE_ACTIONS=true`.
- `hermes tools list` reports the `hermes-tweet` toolset rather than every individual tool name.

## References

- Hermes Tweet: https://github.com/Xquik-dev/hermes-tweet
- PyPI: https://pypi.org/project/hermes-tweet/
- Xquik API: https://docs.xquik.com/api-reference/overview
