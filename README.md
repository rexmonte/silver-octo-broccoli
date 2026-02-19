# silver-octo-broccoli
MacMini openclaw

## Discord bot troubleshooting (online but not replying)

You are absolutely thinking about this the right way: if `qwen3-coder:30b` is stalling, you should switch to a smaller local model (like 14B) or use a cloud provider profile (Anthropic).

Your logs show the gateway/Discord side is working, but generation is timing out:

- `logged in to discord as ...` ✅ (Discord auth works)
- `embedded run start ... model=qwen3-coder:30b` ✅ (message reached agent)
- `typing TTL reached (2m)` ⚠️ (no timely model completion)

That means the bottleneck is **model/profile selection and latency**, not basic bot connectivity.

## Direct answer: "Why not just use other models?"

Usually one of these is happening:

1. OpenClaw is pinned to a specific profile/model (`ollama/qwen3-coder:30b`) for that channel/session.
2. Other local models exist on disk but are not selected as the active profile.
3. The fallback profile (like Anthropic) is not configured with a valid API key, so fallback cannot succeed.

So yes — your proposed fix is exactly the right move.

## Fastest recovery path

### 1) Keep one gateway process only

```bash
lsof -nP -iTCP:18789 -sTCP:LISTEN
openclaw logs --follow
```

If duplicate start conflicts return, cleanly restart one instance:

```bash
openclaw gateway stop
launchctl bootout gui/$UID/ai.openclaw.gateway 2>/dev/null || true
openclaw gateway
```

### 2) Switch from 30B to a 14B (or smaller) local model

1. List locally available models:

```bash
ollama list
```

2. Pick a faster model you already have (for example a 14B instruct model).
3. In OpenClaw, set the active Discord agent/profile model from `qwen3-coder:30b` to that faster model.
4. Send a tiny Discord test prompt: `ping`.

If you still get typing timeout, drop again to 7B/8B.

### 3) Enable Anthropic fallback (if desired)

If you want cloud fallback when local inference is slow:

1. Add a valid `ANTHROPIC_API_KEY` to the OpenClaw environment/profile that the gateway actually uses.
2. Enable/select an Anthropic profile in OpenClaw and make sure it is allowed as a fallback for Discord runs.
3. Restart gateway after key/profile changes.

Then test in Discord and confirm logs show the Anthropic profile being selected when local runs stall.

### 4) Clean stale session state (recommended once)

Your log had orphaned-turn cleanup warnings, so do a one-time session cleanup:

```bash
openclaw gateway stop
mkdir -p ~/.openclaw/backup
cp -a ~/.openclaw/agents/main/sessions ~/.openclaw/backup/sessions.$(date +%Y%m%d-%H%M%S)
rm -f ~/.openclaw/agents/main/sessions/*.lock
openclaw gateway
```

## Validation checklist

After changing models/profiles, this is what "fixed" looks like in logs:

- You still see `logged in to discord`.
- `embedded run start` uses your new chosen profile/model (14B or Anthropic), not `qwen3-coder:30b`.
- You no longer see `typing TTL reached (2m)` for short prompts.
- Discord receives a response within a few seconds.

## Practical recommendation

For reliability, start with this order:

1. Fast local model (7B/14B) as primary.
2. Anthropic as fallback if local generation exceeds latency budget.
3. Keep 30B for manual/high-quality tasks, not interactive Discord chat.
