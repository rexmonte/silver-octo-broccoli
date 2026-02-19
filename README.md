# silver-octo-broccoli
MacMini openclaw

## Discord bot troubleshooting (gateway healthy, config apply failing)

## What your latest logs confirm

From your newest log block:

- Gateway startup is healthy (canvas/heartbeat/health-monitor running).
- Discord provider starts and logs in correctly.
- Active model is still `ollama/qwen3:14b`.
- `config.get` succeeds, but `config.apply` fails with `INVALID_REQUEST`.
- Tool layer then reports `gateway failed: invalid config`.

This means core connectivity is fine, but **a config payload being sent over WS is invalid**.

## Key interpretation

Because `config.get` works and `config.apply` fails immediately, the issue is likely not transport/auth. It is most likely one or more invalid fields/values in the apply request body.

Also, `typing TTL reached (2m)` indicates reply latency remains a second issue, but config validity must be fixed first.

## Fix order (do this in sequence)

### 1) Stabilize baseline gateway state

```bash
openclaw doctor --fix
openclaw gateway stop
launchctl bootout gui/$UID/ai.openclaw.gateway 2>/dev/null || true
openclaw gateway
```

### 2) Stop bulk config applies; use minimal known-good apply

Apply only minimal settings first (single change at a time), and verify logs after each apply.

Start with:

- model only
- then timeout only
- then token limit only

If one step triggers `INVALID_REQUEST`, that field/value is the offender.

### 3) Validate by log signatures

After each config change, confirm:

- no `res ✗ config.apply ... INVALID_REQUEST`
- no `[tools] gateway failed: invalid config`

Only once these are clean should you continue Discord latency tuning.

## Discord latency tuning (after config.apply is clean)

If replies still lag after valid config applies:

1. Switch Discord profile to `qwen3:8b`.
2. Keep `thinking=off`.
3. Lower max output tokens.
4. Lower run timeout to fail fast.
5. Put Anthropic as first fallback.

## Focused smoke test

```bash
openclaw logs --follow
```

Then send `ping` in Discord and verify:

1. `embedded run start ... model=<active model>`
2. no `INVALID_REQUEST` apply errors
3. no `gateway failed: invalid config`
4. assistant reply posted (not just typing)

## Notes on benign warnings

- Bonjour name/hostname conflict auto-rename (`... (2)`, `openclaw-(2)`) is usually harmless for Discord message delivery.
