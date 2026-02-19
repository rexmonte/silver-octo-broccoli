# silver-octo-broccoli
MacMini openclaw

## Discord bot troubleshooting (gateway supervision + invalid config + latency)

## What your latest logs confirm

You now have three overlapping issues:

1. **Gateway supervision conflict**
   - `Gateway failed to start: gateway already running (pid 10148)`
   - `Port 18789 is already in use`
2. **Config apply failure**
   - `config.apply ... INVALID_REQUEST`
   - `gateway failed: invalid config`
3. **Runtime latency**
   - `typing TTL reached (2m)`
   - `Slow listener detected` / 600s-class timeouts

So this is not one bug; it is a startup-control problem + config-validation problem + performance problem.

## Fix order (important)

### 1) Resolve gateway supervision conflict first

You cannot reliably debug config or latency while two startup modes conflict.

```bash
# stop supervised/manual gateway if running
openclaw gateway stop
launchctl bootout gui/$UID/ai.openclaw.gateway 2>/dev/null || true

# verify port is free
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

Then choose **one** mode only:

- Foreground/manual: `openclaw gateway`
- LaunchAgent-supervised: `launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.gateway.plist`

Do not run both.

### 2) If `launchctl bootstrap ... Input/output error` persists

That usually means stale/bad LaunchAgent state or plist mismatch. Reset it:

```bash
launchctl bootout gui/$UID/ai.openclaw.gateway 2>/dev/null || true
launchctl disable gui/$UID/ai.openclaw.gateway 2>/dev/null || true
launchctl enable gui/$UID/ai.openclaw.gateway 2>/dev/null || true
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.gateway.plist
launchctl print gui/$UID/ai.openclaw.gateway
```

If bootstrap still fails, use manual mode (`openclaw gateway`) while fixing LaunchAgent metadata.

### 3) Repair config pipeline

Once only one gateway is running:

```bash
openclaw doctor --fix
```

Then apply config in tiny increments (one field at a time):

1. model only
2. timeout only
3. token limit only

After each apply, verify logs do **not** show:

- `config.apply ... INVALID_REQUEST`
- `gateway failed: invalid config`

### 4) Only then tune latency

If config is clean but chat still stalls:

1. Move Discord profile to `qwen3:8b`.
2. Keep `thinking=off`.
3. Lower output token budget.
4. Lower timeout to fail fast.
5. Put Anthropic as first fallback.

## Focused smoke test

```bash
openclaw logs --follow
```

Send `ping` in Discord and require all of these:

1. no startup conflict errors (`already running`, `port 18789 in use` from duplicate start attempts)
2. no `config.apply ... INVALID_REQUEST`
3. no `gateway failed: invalid config`
4. run start + successful end
5. assistant message posted (not just typing)

## Practical diagnosis

Right now the most immediate blocker is startup control conflict (manual vs supervised gateway). Fix that first, then config apply validity, then latency.
