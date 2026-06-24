# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

A shell-script scheduler that runs Claude Code's `/insights` slash command headlessly across multiple local git repositories on a cron or systemd timer. The core constraint: Claude Code slash commands are user-invoked only, so this wraps OS-level schedulers to call `claude -p "/insights"` unattended.

## Commands

No build system. All scripts are plain Bash.

```bash
# Install cron entry (idempotent — removes old entry first)
bin/claude-insights-install

# Remove cron entry (preserves state and logs)
bin/claude-insights-uninstall

# Run manually (smoke test or force run)
bin/claude-insights-run

# View logs
tail -f ~/.local/state/claude-insights/cron.log
```

Alternatively, use systemd timers instead of cron:
```bash
cp systemd/claude-insights.{service,timer} ~/.config/systemd/user/
systemctl --user enable --now claude-insights.timer
```

## Architecture

**Flow:** `cron/systemd → bin/claude-insights-run → per repo: check 7-day marker → claude -p "/insights" → capture output`

### Key files

| File | Role |
|------|------|
| `bin/claude-insights-run` | Main orchestrator: discover repos, check cadence, run headlessly, log output |
| `bin/claude-insights-install` | Installs cron entry tagged `# claude-insights` |
| `bin/claude-insights-uninstall` | Removes by tag; preserves state |
| `systemd/claude-insights.{service,timer}` | Systemd alternative; `Persistent=true` handles missed runs |
| `examples/repos.conf.example` | Config template (one path per line; supports tilde; supports parent dirs) |

### Runtime paths (outside repo, not tracked)

- `~/.config/claude-insights/repos.conf` — repo list
- `~/.local/state/claude-insights/repos/<slug>/last-run` — per-repo cadence marker (mtime = last success timestamp)
- `~/.local/state/claude-insights/repos/<slug>/output/<timestamp>.md` — captured output
- `~/.local/state/claude-insights/repos/<slug>/output/latest.md` — symlink to most recent output
- `~/.local/state/claude-insights/runs/<timestamp>.log` — run summary log

### Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `INSIGHTS_CONFIG` | `~/.config/claude-insights/repos.conf` | Repo list |
| `INSIGHTS_STATE_DIR` | `~/.local/state/claude-insights` | Output, logs, markers |
| `INSIGHTS_MIN_INTERVAL_DAYS` | `7` | Per-repo cadence |
| `INSIGHTS_COMMAND` | `/insights` | Slash command |
| `INSIGHTS_TIMEOUT_SECONDS` | `1800` | Per-repo timeout |
| `INSIGHTS_CLAUDE_FLAGS` | `--permission-mode bypassPermissions` | Claude CLI flags |
| `INSIGHTS_SCAN_DEPTH` | `4` | Max repo discovery depth |
| `CLAUDE_BIN` | `claude` | Path to claude binary |
| `INSIGHTS_HOUR` / `INSIGHTS_MINUTE` | `3` / `15` | Cron schedule (install time only) |

### Design decisions worth knowing

- **Daily cron + 7-day marker, not weekly cron**: resilient to missed runs — if the machine is off, the next day it will catch up rather than skip a week.
- **Cadence tracked by `last-run` file mtime**: atomic, no separate database. The file's presence + mtime is the entire state.
- **Slug-based state dirs**: repo paths are normalized (`/path/to/repo` → `path-to-repo`) for safe filesystem names. macOS/Linux `stat` differences are handled via `uname` detection.
- **Output goes to state dir, not in-repo**: avoids dirtying working trees.
- **No locking**: deliberate simplicity tradeoff. Acknowledged risk: if a run hangs >24h, the next cron tick may double-run a repo.
- **`--permission-mode bypassPermissions` default**: required for unattended runs; assumes `/insights` is read-only-ish.

### Known sharp edges

- Auth in cron context: `ANTHROPIC_API_KEY` must be available in the cron environment (not just shell profile).
- Headless `/insights` behavior: not all Claude skills work non-interactively; verify `/insights` produces output without prompts.
- Discovery uses `find -prune` to stop at the first `.git`, so submodule trees are not traversed.
