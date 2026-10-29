# Codex 5-Hour Quota Scheduled Refresh

Every 5 hours 3 minutes, your computer automatically says one tiny thing to Codex first, so your quota refills just 3 minutes after the previous round ends — no need to think about it yourself.

**English** | [简体中文](README.md)

## The problem

Codex's quota works like a stopwatch: the moment you send your first message, a 5-hour countdown starts, and only when it runs out does your quota refill.

The trouble is, it refills whenever you happened to start. Start working at 11:23 and your refills land at 16:23, 21:23… — never on a round hour, and the limit tends to hit right in the middle of real work.

## How it works

Since "whoever speaks first starts the countdown", let the computer speak first, the moment the previous round ends:

1. Every 5 hours 3 minutes, the computer automatically sends Codex the shortest possible message (just enough to get an "OK" back).
2. It arrives 3 minutes after the previous round ends, becomes the first message of the new round, and a fresh 5-hour countdown starts there.
3. Every round chains on the same way, so your quota is always on a "refills as soon as it ends" rhythm.

Why 5 hours 3 minutes instead of exactly 5 hours? Scheduled tasks can actually fire a few minutes *early*. On an exact 5-hour interval, a firing could land in the last few minutes of the previous round and be wasted. The extra 3 minutes is headroom for that early firing, so every run lands after the previous round has ended.

Two questions everyone asks:

- **If it speaks while I'm working, will it mess things up?** No. If the countdown is still running, its message just counts inside the current round — it doesn't restart anything and doesn't interrupt you. When the round ends, the next automatic message starts the new one.
- **Does that message itself cost quota?** A tiny bit. It uses the cheapest setup we found (lightest model, replies with a single OK), measured at about 1,941 tokens per run — negligible next to getting your quota back on time.

One small note: run times no longer line up with the clock. Because each interval is 3 minutes longer than 5 hours, the times drift 3 minutes later each round. That's inherent to a fixed interval and doesn't affect how it works.

<details>
<summary>Technical details (only if you like to tinker; safe to skip)</summary>

The command each automatic message actually runs:

```bash
mkdir -p /tmp/codex-ping && cd /tmp/codex-ping && timeout 180 codex exec \
  --ephemeral --skip-git-repo-check -s read-only \
  -m gpt-6-luna -c model_reasoning_effort="low" \
  "Say OK only" < /dev/null
```

Pitfalls found by testing:

- `< /dev/null` is required — without it, `codex exec` waits for input and hangs forever.
- Do not pipe this command; a pipe masks its real success/failure result.
- The prompt itself is only a few characters. The fixed overhead of one Codex request (system instructions, tool definitions, etc., measured at about 59k characters) is the bulk, so the model tier is what saves quota — the lightest Luna model at the lowest reasoning effort — not shaving a few more words off the prompt.
- Only applies to Codex on a ChatGPT subscription. API-key pay-as-you-go billing has no 5-hour countdown, so this doesn't apply.
- Don't use fixed daily clock times instead: in testing, scheduled tasks fired about 3–4 minutes *before* the set hour, so a run could land inside a still-running round and be wasted. The fixed interval plus 3 minutes of headroom exists precisely to solve that.

</details>

## How to use (for Muse users)

1. Open [PROMPT.en.md](PROMPT.en.md) and copy the full prompt.
2. Adjust the settings at the top if you like (refresh interval, timezone, whether to report every run).
3. Paste it into your Muse chat. Muse does the rest: sign Codex in, verify the pipeline once, create the task that runs every 5 hours 3 minutes, and tell you when it runs next.
4. Nothing to manage afterwards: silent on success, failures get reported.

Chinese version: [README.md](README.md) · [PROMPT.md](PROMPT.md)

## Requirements

- A Muse that can run scheduled tasks, with the Codex CLI installed on the same machine
- A ChatGPT subscription (Plus / Pro, or any plan whose Codex quota refills in 5-hour rounds)

## Field notes

This has been running on a real account: on 2026-10-27 at 18:00 and 22:00, and on 2026-10-28 at 08:00, each run successfully aligned the refill time to the hour, at 1,941 tokens each.

Build day also confirmed something important: with the quota already exhausted, a test message returned a usage-limit error whose reset time matched the account's actual reset time exactly — proof the automatic messages use the same account and the same quota as your own usage.

Update, 2026-10-29: the original setup ran at fixed times every day (08:00, 13:00, 18:00, 22:00). Because scheduled tasks actually fired about 3–4 minutes before the hour, a run sometimes landed inside a still-running round and was wasted. It has been replaced by the current fixed interval of every 5 hours 3 minutes.

## License

MIT
