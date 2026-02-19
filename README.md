# silver-octo-broccoli
MacMini openclaw

## Discord bot troubleshooting (connected, typing, but no final reply)

### What your newest logs prove

From the log sequence you posted:

- Discord is connected (`logged in to discord ...`).
- A run starts for each message (`embedded run start`).
- The selected model is still `provider=ollama model=qwen3-coder:30b`.
- Typing stops after 2 minutes (`typing TTL reached (2m)`).
- The run eventually hard-times out at 10 minutes (`timeoutMs=600000`).
- Discord listener is flagged slow (~601s).

So the failure mode is: **request enters the agent, but the active model/profile cannot complete quickly enough for chat.**

### New clue from your Ollama logs

You also posted:

- `WARN MLX dynamic library not available`
- Local models present: `qwen3:8b`, `qwen3:14b`, `qwen2.5-coder:14b`, `qwen3-coder:30b`

Interpretation:

- You **do** have faster models installed locally (8B/14B), so switching is possible now.
- Missing MLX acceleration can reduce local inference performance, which makes 30B even less suitable for realtime Discord replies.

## Direct answer to your question

> "Why can’t we use the other models on disk or the 14B one, or Anthropic?"

You can. The logs indicate OpenClaw is currently pinned to `ollama/qwen3-coder:30b` for this Discord path, and fallback did not rescue this run (`Profile ollama:manual timed out ... Trying next account...`).

Typical reasons:

1. Active Discord profile is explicitly set to 30B.
2. Other local models exist but are not selected for that profile.
3. Anthropic fallback is not configured/authorized in the runtime environment the gateway uses.
4. Fallback order still prioritizes another slow local account before Anthropic.

## Immediate fix plan (recommended)

### 1) Keep only one gateway process

```bash
lsof -nP -iTCP:18789 -sTCP:LISTEN
openclaw logs --follow
```

If duplicate gateway starts reappear:

```bash
openclaw gateway stop
launchctl bootout gui/$UID/ai.openclaw.gateway 2>/dev/null || true
openclaw gateway
```

### 2) Switch Discord profile model from 30B to an installed 14B/8B

You already have good candidates installed:

- `qwen3:14b`
- `qwen2.5-coder:14b`
- `qwen3:8b`

Confirm inventory:

```bash
ollama list
```

Then set Discord/OpenClaw active model to one of those (start with `qwen3:14b` or `qwen2.5-coder:14b`, then drop to `qwen3:8b` if latency is still high) and retest with `ping`.

### 3) Verify fallback chain actually includes Anthropic

Use the profile/settings OpenClaw gateway loads and confirm:

- `ANTHROPIC_API_KEY` is present in that runtime.
- Anthropic profile is enabled.
- Fallback order puts Anthropic ahead of slow/retrying local profiles.

Restart gateway after changes:

```bash
openclaw gateway stop
openclaw gateway
```

### 4) Clear stale session lock/state once

```bash
openclaw gateway stop
mkdir -p ~/.openclaw/backup
cp -a ~/.openclaw/agents/main/sessions ~/.openclaw/backup/sessions.$(date +%Y%m%d-%H%M%S)
rm -f ~/.openclaw/agents/main/sessions/*.lock
openclaw gateway
```

## What "fixed" should look like in logs

- `embedded run start ... model=<qwen3:14b|qwen2.5-coder:14b|qwen3:8b|anthropic/...>`
- No `typing TTL reached (2m)` for short prompts.
- No `embedded run timeout ... 600000` for normal chat.
- No repeated `Slow listener detected` around 600s.
- Discord receives replies in seconds.

## Practical operating mode

For stable Discord chat:

1. Primary: `qwen3:14b` or `qwen2.5-coder:14b`.
2. Fallback: Anthropic.
3. Emergency low-latency fallback: `qwen3:8b`.
4. Reserve `qwen3-coder:30b` for manual/non-realtime jobs.
