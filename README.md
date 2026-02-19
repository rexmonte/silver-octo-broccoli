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

## 10-minute morning recovery checklist (copy/paste)

Use this every morning before you start deeper work.

### Step A — quick health snapshot (60 seconds)

```bash
echo "=== gateway listeners ==="
lsof -nP -iTCP:18789 -sTCP:LISTEN

echo "=== launch agent status ==="
launchctl print gui/$UID/ai.openclaw.gateway 2>/dev/null | head -40 || echo "launch agent not loaded"

echo "=== latest openclaw errors ==="
tail -200 /tmp/openclaw/openclaw-$(date +%F).log 2>/dev/null | rg -n "INVALID_REQUEST|gateway failed: invalid config|typing TTL|Slow listener|timeout|already running|Port 18789"
```

### Step B — enforce single startup mode (2 minutes)

```bash
openclaw gateway stop
launchctl bootout gui/$UID/ai.openclaw.gateway 2>/dev/null || true
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

Then choose one mode and stick with it for the day:

```bash
# manual mode
openclaw gateway

# OR supervised mode
# launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

### Step C — config validity gate (2 minutes)

```bash
openclaw doctor --fix
openclaw logs --follow
```

From your client, do one minimal config update only.

If you see either of these, stop and fix config payload before anything else:

- `config.apply ... INVALID_REQUEST`
- `gateway failed: invalid config`

### Step D — latency gate (3 minutes)

Send `ping` in Discord and watch logs.

Pass if all are true:

1. No startup conflict errors.
2. No invalid-config errors.
3. No `typing TTL reached (2m)` for the ping.
4. Reply is posted in Discord quickly.

If it fails, switch to fast chat profile immediately:

- `qwen3:8b`
- lower output token budget
- `thinking=off`
- shorter timeout
- Anthropic first fallback

### Step E — daily operating rule (always)

- Do not mix launchctl + manual gateway on same day.
- Do not tune latency while config is invalid.
- Do not debug old logs longer than 10 minutes; run a fresh `ping` test and branch from that result.
