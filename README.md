# claude-insights

Run the `/insights` slash command via Claude Code on every git repo on your
machine, automatically, once a week.

## Why this exists (the constraint)

Claude Code itself can't schedule recurring jobs.

- Slash commands are **user-invoked** — Claude can't auto-fire them.
- `settings.json` hooks are **event-triggered** (PreToolUse, SessionStart, Stop, …),
  not time-based.

So this repo is a thin shell wrapper around your OS scheduler (cron or systemd)
that calls Claude Code in headless mode (`claude -p "/insights"`) inside each
configured repo.

## How it works

```
cron (daily, 03:15)
  └── bin/claude-insights-run
        ├── reads ~/.config/claude-insights/repos.conf
        ├── discovers repos (each line: a repo, or a parent dir of repos)
        ├── for each repo not run in the last 7 days:
        │     cd <repo> && claude -p "/insights"
        └── writes output to ~/.local/state/claude-insights/repos/<slug>/output/
```

The script runs **daily** but only acts on repos whose last successful run is
older than `INSIGHTS_MIN_INTERVAL_DAYS` (default 7). That makes it idempotent
and resilient to missed runs — if your machine was off on Sunday, it runs
Monday instead of skipping the week.

## Install

```bash
git clone <this-repo> ~/claude-insights
chmod +x ~/claude-insights/bin/*

# 1. Tell it which repos to scan
mkdir -p ~/.config/claude-insights
cp ~/claude-insights/examples/repos.conf.example ~/.config/claude-insights/repos.conf
$EDITOR ~/.config/claude-insights/repos.conf

# 2. Install the daily cron entry
~/claude-insights/bin/claude-insights-install
```

Verify:

```bash
crontab -l | grep claude-insights
~/claude-insights/bin/claude-insights-run   # run once now to smoke-test
```

## Configuration

`~/.config/claude-insights/repos.conf` — one path per line. Each is either a
git repo (contains `.git`) or a parent directory whose `.git` children get
discovered (up to 4 levels deep).

Environment variables (set in the cron entry, your shell, or systemd unit):

| Var | Default | Purpose |
|---|---|---|
| `INSIGHTS_CONFIG` | `~/.config/claude-insights/repos.conf` | Repo list |
| `INSIGHTS_STATE_DIR` | `~/.local/state/claude-insights` | Logs + per-repo output |
| `INSIGHTS_MIN_INTERVAL_DAYS` | `7` | Per-repo cadence |
| `INSIGHTS_COMMAND` | `/insights` | Slash command to run |
| `INSIGHTS_TIMEOUT_SECONDS` | `1800` | Per-repo timeout |
| `INSIGHTS_CLAUDE_FLAGS` | `--permission-mode bypassPermissions` | Flags passed to `claude` |
| `INSIGHTS_SCAN_DEPTH` | `4` | Max depth when discovering nested repos |
| `CLAUDE_BIN` | `claude` | Path to the Claude Code binary |
| `INSIGHTS_HOUR` / `INSIGHTS_MINUTE` | `3` / `15` | Cron schedule (install only) |

## Output

Per repo:

```
~/.local/state/claude-insights/repos/<slug>/
  ├── last-run                  # mtime = last successful run
  └── output/
      ├── 20260428T031500.md    # raw stdout from Claude
      ├── 20260428T031500.err   # stderr (only if non-empty)
      └── latest.md             # symlink to the newest run
```

Run-level log: `~/.local/state/claude-insights/runs/<timestamp>.log`.

## Unattended auth — important

Cron has a minimal environment. For `claude` to run unattended:

- **API key path:** export `ANTHROPIC_API_KEY` in the cron entry, or in
  `~/.profile` if your `claude` reads it. Don't put keys in this repo.
- **Logged-in CLI path:** make sure the credentials file Claude Code persists
  (typically under `~/.claude/`) is readable by the user the cron runs as.
  Test by running `~/claude-insights/bin/claude-insights-run` manually first —
  if that works, cron will too.

`--permission-mode bypassPermissions` is the default in `INSIGHTS_CLAUDE_FLAGS`
because there's no human to approve tool calls. Override if `/insights` is
strictly read-only and you'd rather keep prompts on (it'll just hang on the
first prompt and time out, so leave it alone unless you know what you want).

## systemd alternative

If you prefer systemd timers over cron:

```bash
mkdir -p ~/.config/systemd/user
cp ~/claude-insights/systemd/claude-insights.{service,timer} ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now claude-insights.timer
systemctl --user list-timers claude-insights.timer
```

## Uninstall

```bash
~/claude-insights/bin/claude-insights-uninstall   # cron
# or
systemctl --user disable --now claude-insights.timer
```

State and logs are left in place; remove `~/.local/state/claude-insights/`
manually if you want them gone.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `'claude' not found in PATH` from cron | Cron's PATH doesn't include where `claude` lives. Set `CLAUDE_BIN=/full/path/to/claude` or extend PATH in the crontab. |
| Every repo logs `FAIL rc=124` | Hit the per-repo timeout. Raise `INSIGHTS_TIMEOUT_SECONDS`. |
| Output is empty but rc=0 | `/insights` skill not loaded for non-interactive sessions. Verify with `cd <repo> && claude -p "/insights"` interactively first. |
| Same repo runs every day | Marker write failing — check `INSIGHTS_STATE_DIR` is writable by the cron user. |
| Want to force a re-run | `rm ~/.local/state/claude-insights/repos/<slug>/last-run` |
