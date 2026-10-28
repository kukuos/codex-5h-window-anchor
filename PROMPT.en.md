# Prompt for Muse (copy the full text and paste it into your Muse chat)

```text
Please set up Codex 5-hour quota window anchors on this machine for me: a set of tiny scheduled Codex requests, sent automatically at fixed times every day, that pre-start the quota window so its reset times line up with my usage habits.

## Parameters (use these defaults unless I changed them above)
- Anchor times: 08:00, 13:00, 18:00, 22:00 every day
- Timezone: Asia/Taipei
- Reporting: stay silent on success, do not message me; only report failures

## Background (act on this, don't lecture me about it)
With Codex on a ChatGPT subscription, the 5-hour usage window starts at the first request of the current window. If an anchor time arrives after the previous window has ended, this tiny request becomes the first request of a new window and anchors it to the hour. If the previous window is still running, the request simply lands inside it and does not restart it; that is normal.

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
3. Create one daily scheduled task per anchor time (4 in total, using the timezone parameter above, exact times, no flexible shifting). Each task's instructions use the template below, replacing {TIME} with that anchor time and {RESET} with its reset time 5 hours later (for 22:00 that is 03:00 next day):

--- scheduled task template starts ---
Goal: send one tiny Codex request on this machine to pre-start the user's Codex 5-hour quota window (this anchor is {TIME}; the anchored window resets at about {RESET}). To use as little quota as possible, always use the lightest Luna model, the lowest reasoning effort, and the shortest prompt; never switch back to a flagship model.

Steps:
1. Run `codex login status`. If it shows not logged in, do not try to log in; report to the user that the login has expired and a new login is needed, then stop.
2. If logged in, run the following command exactly once, no retries (it must include `< /dev/null`, otherwise codex exec waits on stdin and hangs; do not pipe this command, a pipe masks its real exit code):
   mkdir -p /tmp/codex-ping && cd /tmp/codex-ping && timeout 180 codex exec --ephemeral --skip-git-repo-check -s read-only -m gpt-6-luna -c model_reasoning_effort="low" "Say OK only" < /dev/null
3. On success (exit code 0 and reply contains OK): do not message the user; just record the success internally.
4. On failure: send the user a short message with the key points of the actual error. Distinguish: (a) usage-limit / rate-limit error — the previous window may not have ended yet at this anchor time, or quota is exhausted, so no new window was anchored; (b) login expired — handle as in step 1. Do not retry automatically and do not try workarounds.
--- scheduled task template ends ---

4. After creating them, verify each of the four tasks' next run time, then report back to me: the four anchor times, their next run dates, and the silent-on-success / report-on-failure rule. After that, do not bring this up again unless an anchor run fails.

## Command spec (hard-won details, do not change)
- `< /dev/null` is required: without it, codex exec waits for stdin and hangs.
- Do not pipe the command (e.g. `| tail`): a pipe masks the real exit code; judge success by the exit code.
- Model is pinned to `gpt-6-luna` with `model_reasoning_effort="low"`: the prompt itself is only a few characters; the fixed overhead of a Codex request (system instructions + tool definitions) is the bulk, so the model tier is what saves quota, not shortening the prompt by a few words. Measured usage for this combination is about 1,941 tokens per ping.
- Prompt is pinned to `Say OK only`, the reply is just OK; one request only, read-only sandbox (-s read-only), no persisted session (--ephemeral), working directory is the empty /tmp/codex-ping.
```

## Parameters you can change

- **Times**: replace the anchor times at the top of the prompt with your own. Pick the first anchor before you usually start using Codex, and note that a window opened within 5 hours before an anchor will swallow that anchor (e.g. a window started after 03:00 runs past 08:00), so either avoid opening windows in that span or use several anchors as fallbacks, as this setup does.
- **Timezone**: use your own IANA timezone, e.g. `America/New_York`, `Europe/London`, `Asia/Shanghai`.
- **Reporting**: change the reporting line if you want a daily success confirmation instead of silence.
