# Codex 5h Window Anchor

Make Codex's 5-hour quota window start at **the hour you choose**, not at whatever moment you happen to send your first request.

**English** | [简体中文](README.md)

## The problem

Codex usage on a ChatGPT subscription runs in a rolling 5-hour window: it starts at the **first request** of the current window and resets 5 hours later. In practice, if you start working at 11:23, your windows become 11:23–16:23, 16:23–21:23, and so on. The reset points drift all day, and you tend to hit the limit right in the middle of real work.

## The fix

Send one **tiny request** automatically at fixed times every day, while no window is active, so it becomes the first request of a new window:

| Anchor (Asia/Taipei) | Anchored window | Resets at |
|---|---|---|
| 08:00 | 08:00–13:00 | 13:00 |
| 13:00 | 13:00–18:00 | 18:00 |
| 18:00 | 18:00–23:00 | 23:00 |
| 22:00 | 22:00–03:00 next day (or catches a drifted window) | 03:00 |

An anchor that lands inside a still-running window is harmless — it just sits in that window. An anchor that arrives after a window ended pulls the next window back onto the hour. Multiple anchors back each other up: if daytime windows drift because of real usage, the evening anchors can still catch the next one.

Every anchor ping uses the cheapest combination we measured (**1,941 tokens** per ping):

```bash
mkdir -p /tmp/codex-ping && cd /tmp/codex-ping && timeout 180 codex exec \
  --ephemeral --skip-git-repo-check -s read-only \
  -m gpt-6-luna -c model_reasoning_effort="low" \
  "Say OK only" < /dev/null
```

Key details, all learned the hard way:

- **`< /dev/null` is required** — without it, `codex exec` waits on stdin and hangs.
- **Do not pipe** this command; a pipe masks its real exit code.
- The prompt itself is only a few characters. The fixed overhead of a Codex request (system instructions and tool definitions, measured at about 59k characters) is the bulk, so the **model tier** is what saves quota — the lightest Luna model at the lowest reasoning effort — not shaving words off the prompt.
- This does not apply to API-key pay-as-you-go billing: there is no 5-hour window there.

## How to use (for Muse users)

1. Open [PROMPT.en.md](PROMPT.en.md) and copy the full prompt.
2. Adjust the parameters at the top if you like (anchor times, timezone, reporting style).
3. Paste it into your Muse chat. Muse does the rest: sign Codex in on this machine (device-code authorization), verify the pipeline, create the 4 daily scheduled tasks, and report back the next run times.
4. Silent on success, failures get reported — nothing else to manage.

Chinese version: [PROMPT.md](PROMPT.md)

## Requirements

- A Muse that can run scheduled tasks, with the Codex CLI installed on the same machine
- A ChatGPT subscription plan that includes Codex with its 5-hour window (e.g. Plus / Pro)

## Field notes

Built and verified on 2026-10-27, then running on schedule: the 18:00 and 22:00 anchors on 2026-10-27 and the 08:00 anchor on 2026-10-28 all succeeded at 1,941 tokens each, with window resets aligned to the anchor times.

A side verification from build day: with the quota already exhausted, a test ping returned a usage-limit error whose "try again at" time matched the account's actual reset time exactly — proof the anchor pings hit the same account and the same quota as real usage.

## License

MIT
