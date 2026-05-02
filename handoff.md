# Handoff — claude-insights weekly automation

**Date:** 2026-05-02
**Repo:** `PowerAppsDarren/claude-insights`
**Branch:** `claude/automate-weekly-insights-Fp7mq`
**Commit:** `66f20e9` — *Add cron/systemd runner for weekly /insights across local repos*
**Status:** Pushed. Not yet merged. Not yet exercised on a real machine with a real `claude` binary.

---

## TL;DR

Goal: run the `/insights` slash command once a week in every git repo on disk.

The deliverable is a small shell-script wrapper that a system scheduler (cron
or systemd) drives. It calls `claude -p "/insights"` headlessly inside each
configured repo and stores per-repo output centrally.

Eight files, ~366 lines total. Smoke-tested with a stubbed `claude`.

---

## The hard constraint (read this first)

Claude Code itself **cannot** schedule recurring jobs.

- Slash commands are user-invoked. Claude does not auto-fire them.
- `settings.json` hooks are event-triggered (PreToolUse, SessionStart, Stop, …),
  not time-based.
- The `update-config` skill, hooks, memory, and CLAUDE.md all operate inside a
  session — they can't start a session.

So this work lives **outside** Claude Code. It's a thin wrapper that the OS
scheduler invokes; the wrapper invokes Claude in headless mode.

If a future Claude Code release adds a time-based hook, this wrapper becomes
obsolete and should be replaced by that.

---

## Architecture

```
cron (daily 03:15)              [or systemd timer]
        │
        ▼
bin/claude-insights-run
        │
        ├── reads ~/.config/claude-insights/repos.conf
        │       (each line: a repo path OR a parent dir of repos)
        │
        ├── discovery: expands parent dirs to .git children (depth ≤ 4)
        │
        ├── for each discovered repo:
        │     - check ~/.local/state/claude-insights/repos/<slug>/last-run
        │     - if mtime < INSIGHTS_MIN_INTERVAL_DAYS (default 7), skip
        │     - else: cd <repo> && claude -p "/insights"
        │     - write stdout/stderr to <slug>/output/<timestamp>.{md,err}
        │     - on success, touch last-run, point latest.md → newest
        │
        └── append to runs/<timestamp>.log
```

**Why daily cron + per-repo 7-day marker** instead of weekly cron: weekly cron
silently drops a week if the machine is off at the scheduled minute. Daily +
marker retries the next day.

---

## File map

| Path | Purpose |
|---|---|
| `bin/claude-insights-run` | Main runner. Discovery → per-repo cadence check → headless `claude -p`. |
| `bin/claude-insights-install` | Adds the daily cron entry. Idempotent (replaces an existing entry tagged `# claude-insights`). |
| `bin/claude-insights-uninstall` | Removes the cron entry. State/logs preserved. |
| `examples/repos.conf.example` | Annotated config template. |
| `systemd/claude-insights.service` | Oneshot unit that calls the runner. |
| `systemd/claude-insights.timer` | Daily timer with 15-min randomization + `Persistent=true` (catch-up after downtime). |
| `README.md` | User-facing docs: install, env vars, troubleshooting, unattended auth. |
| `.gitignore` | `*.log`, `.DS_Store`, `state/`. |

---

## Configuration surface (env vars)

| Var | Default | Notes |
|---|---|---|
| `INSIGHTS_CONFIG` | `~/.config/claude-insights/repos.conf` | One path per line. |
| `INSIGHTS_STATE_DIR` | `~/.local/state/claude-insights` | Markers, output, logs. |
| `INSIGHTS_MIN_INTERVAL_DAYS` | `7` | Per-repo cadence. |
| `INSIGHTS_COMMAND` | `/insights` | Slash command to run. |
| `INSIGHTS_TIMEOUT_SECONDS` | `1800` | Per-repo wall clock. |
| `INSIGHTS_CLAUDE_FLAGS` | `--permission-mode bypassPermissions` | Required for unattended runs. |
| `INSIGHTS_SCAN_DEPTH` | `4` | Max depth when discovering nested repos. |
| `CLAUDE_BIN` | `claude` | Override if `claude` not on cron's PATH. |
| `INSIGHTS_HOUR` / `INSIGHTS_MINUTE` | `3` / `15` | Used only by the installer. |

---

## How a user adopts it

```bash
git clone <repo-url> ~/claude-insights
mkdir -p ~/.config/claude-insights
cp ~/claude-insights/examples/repos.conf.example ~/.config/claude-insights/repos.conf
$EDITOR ~/.config/claude-insights/repos.conf       # list ~/code, ~/projects, etc.

~/claude-insights/bin/claude-insights-run          # smoke-test interactively
~/claude-insights/bin/claude-insights-install      # then enable cron
crontab -l | grep claude-insights                  # verify
```

systemd alternative is documented in `README.md`.

---

## Testing performed

In-environment smoke test with a stubbed `claude` that just `echo`s pwd:

- 4-line config (1 parent dir with 2 repos, 1 explicit repo, 1 non-repo dir, 1
  missing path) → 3 repos discovered. Missing path correctly warns to stderr
  without polluting the repo list. The non-repo dir contributes nothing.
- First run: 3 repos run, 3 markers created.
- Second run with same config: 3 repos skipped (`skip (within 7d)`).
- After `rm last-run` and `INSIGHTS_MIN_INTERVAL_DAYS=1`: all 3 re-run.
- Output, `latest.md` symlinks, and run-level log all populate correctly.

Bug found and fixed during testing: `log()` originally wrote to stdout, which
collided with `discover()`'s stdout (consumed by `mapfile`). Log lines were
being captured as fake repo paths. Fix: `log()` writes to stderr.

**Not yet tested:**

- Real `claude` binary in headless mode against a real `/insights` skill.
- Cron environment behavior (PATH, auth, locale).
- Behavior when `/insights` produces interactive prompts despite `-p`.

---

## Known sharp edges (verify on first real run)

1. **Auth in cron.** Cron has a minimal env. The user's `claude` must either
   read `ANTHROPIC_API_KEY` from a file the cron user can access, or the
   credentials file Claude Code persists (typically `~/.claude/...`) must be
   readable by the cron user. The README warns about this; it's still the
   most likely thing to break.

2. **`--permission-mode bypassPermissions` is on by default.** Required for
   unattended runs (no human to click approve). If `/insights` does anything
   write-y, this matters. Confirm `/insights` is read-only-ish before
   trusting unattended runs across many repos.

3. **`claude -p "/insights"` assumes the slash command works in non-interactive
   print mode.** Not all skills behave the same way headlessly. Verify by
   running `cd <some-repo> && claude -p "/insights"` manually before enabling
   the cron.

4. **Output goes to a central state dir, not into each repo.** Deliberate —
   we don't want to dirty user repos. If the user wants insights committed
   per-repo, that's a follow-up (write `INSIGHTS.md` into the repo + optional
   auto-commit/push to a `claude/insights` branch).

5. **Discovery depth is 4.** Monorepos with deeply nested submodules may need
   `INSIGHTS_SCAN_DEPTH` raised. Submodules will currently be picked up as
   separate repos — probably not what the user wants.

6. **Concurrency.** No locking. If cron fires twice (it shouldn't, but if a
   prior run hangs past 24h), two runners can race on the same repo. Adding
   `flock /var/lock/claude-insights.lock` to the cron entry would fix it;
   left out for simplicity.

---

## Suggested follow-ups (in priority order)

1. **End-to-end test on a real machine.** One repo, real `claude`, real
   `/insights`. Confirm output is what the user expects.
2. **Decide where output lives.** Central state dir (current) vs. `INSIGHTS.md`
   in each repo vs. aggregated weekly digest. The current design supports
   adding any of these without rewriting the runner.
3. **Add `flock`** to the cron entry once #1 confirms run duration.
4. **Aggregation/summary step.** A second script that reads each repo's
   `latest.md` and produces a single weekly digest (email, Markdown file,
   posted to a Plane issue, etc.). Hook depends on the user's notification
   preference.
5. **Per-repo opt-out.** Honor a `.claudeinsightsignore` file or a marker in
   the repo to skip it without editing the central config.

---

## Branch & remote

- Branch: `claude/automate-weekly-insights-Fp7mq`
- Pushed to: `origin` (the local-proxy Forgejo at
  `http://127.0.0.1:37039/git/PowerAppsDarren/claude-insights`)
- PR: not opened (was not requested).

To pick this up:

```bash
git fetch origin claude/automate-weekly-insights-Fp7mq
git checkout claude/automate-weekly-insights-Fp7mq
```
