# Prompt for Muse (copy the full text and paste it into your Muse chat)

```text
Please set up a Codex 5-hour quota scheduled refresh on this machine for me: one tiny Codex request, sent automatically at a fixed interval, that starts a new round immediately after the previous one ends, so my quota refills as soon as possible.

## Parameters (use these defaults unless I changed them)
- Refresh interval: every 5 hours 3 minutes
- Timezone: Asia/Taipei
- Reporting: stay silent on success, do not message me; only report failures

## Background (act on this, don't lecture me about it)
With Codex on a ChatGPT subscription, the 5-hour usage window starts at the first request of the current window. The first tiny request sent after the previous round ends becomes the first request of the new round and starts a new 5-hour countdown. If the previous round is still running, the request simply lands inside it and does not restart it; that is normal.

Why the interval is 5 hours 3 minutes instead of exactly 5 hours: scheduled tasks can fire a few minutes earlier than the set time. On an exact 5-hour interval, a firing could land in the last few minutes of the previous round and be wasted. The extra 3 minutes is headroom for that early firing, so every run lands after the previous round has ended.

## What to do, in order, verifying each step
1. Check the Codex CLI: run `codex --version` and `codex login status`.
   - If the CLI is not installed, tell me first and suggest the right install method for this machine; install only after I confirm.
   - If not logged in: run `codex login --device-auth` (in the background), send me the sign-in link and the one-time device code from its output, and wait for me to authorize. After the login process exits successfully, run `codex login status` again and confirm it says Logged in using ChatGPT. If it shows API-key login instead, warn me that pay-as-you-go API billing has no 5-hour window and this setup does not apply; ask whether I want to switch to a ChatGPT subscription login, and do not continue.
2. After login is confirmed, run ONE pipeline verification (note: this sends a real tiny request; if quota is available it immediately starts a 5-hour window — tell me that before running it):
   In an empty directory, run this exact command (see the command spec below; do not drop the `< /dev/null`):
   mkdir -p /tmp/codex-ping && cd /tmp/codex-ping && timeout 180 codex exec --ephemeral --skip-git-repo-check -s read-only -m gpt-6-luna -c model_reasoning_effort="low" "Say OK only" < /dev/null
   - Exit code 0 and a reply containing OK: pipeline works, continue.
   - Usage-limit error: note the reset time it reports; if it matches the reset time I told you, treat the pipeline as verified (right account, right command, quota just happens to be exhausted) and continue to the next step. Do not retry.
   - Any other error: summarize the actual error for me and stop for my decision.
3. Create ONE scheduled task that runs every 5 hours 3 minutes (using the timezone parameter above). The key is its starting point: base it on the successful verification request from step 2 — that request started a window that ends about 5 hours later, so the task's first run must be 3 minutes after that, i.e. 5 hours 3 minutes after the verification succeeded; after that it runs every 5 hours 3 minutes. The task's instructions use the template below:

--- scheduled task template starts ---
Goal: send one tiny Codex request on this machine to start a new 5-hour Codex quota window after the previous round has ended. To use as little quota as possible, always use the lightest Luna model, the lowest reasoning effort, and the shortest prompt; never switch back to a flagship model.

Steps:
1. Run `codex login status`. If it shows not logged in, do not try to log in; report to the user that the login has expired and a new login is needed, then stop.
2. If logged in, run the following command exactly once, no retries (it must include `< /dev/null`, otherwise codex exec waits on stdin and hangs; do not pipe this command, a pipe masks its real exit code):
   mkdir -p /tmp/codex-ping && cd /tmp/codex-ping && timeout 180 codex exec --ephemeral --skip-git-repo-check -s read-only -m gpt-6-luna -c model_reasoning_effort="low" "Say OK only" < /dev/null
3. On success (exit code 0 and reply contains OK): do not message the user; just record the success internally.
4. On failure: send the user a short message with the key points of the actual error. Distinguish: (a) usage-limit / rate-limit error — the previous round may not have ended yet, or quota is exhausted, so no new window was started; (b) login expired — handle as in step 1. Do not retry automatically and do not try workarounds.
--- scheduled task template ends ---

4. After creating it, verify the task's next run time, then report back to me: the refresh interval, the next run date and time, and the silent-on-success / report-on-failure rule. After that, do not bring this up again unless a run fails.

## Command spec (hard-won details, do not change)
- `< /dev/null` is required: without it, codex exec waits for stdin and hangs.
- Do not pipe the command (e.g. `| tail`): a pipe masks the real exit code; judge success by the exit code.
- Model is pinned to `gpt-6-luna` with `model_reasoning_effort="low"`: the prompt itself is only a few characters; the fixed overhead of a Codex request (system instructions + tool definitions) is the bulk, so the model tier is what saves quota, not shortening the prompt by a few words. Measured usage for this combination is about 1,941 tokens per ping.
- Prompt is pinned to `Say OK only`, the reply is just OK; one request only, read-only sandbox (-s read-only), no persisted session (--ephemeral), working directory is the empty /tmp/codex-ping.
```

## Parameters you can change

- **Interval**: replace the refresh interval at the top of the prompt with your own. Note the interval must be slightly longer than 5 hours (this setup uses 5 hours 3 minutes) to leave headroom for tasks firing early; on exactly 5 hours, a firing could land in the last few minutes of the previous round and be wasted.
- **Timezone**: use your own IANA timezone, e.g. `America/New_York`, `Europe/London`, `Asia/Shanghai`.
- **Reporting**: change the reporting line if you want a confirmation after every run instead of silence.

## Extra notes (for readers; not part of the prompt)

- Run times no longer line up with the clock: because each interval is 3 minutes longer than 5 hours, firing times drift 3 minutes later each round. That is inherent to a fixed interval.
- If you open a new window yourself through heavy Codex use in between, one interval request may land inside that window and be wasted; the first request after that window expires picks the chain up again automatically.
