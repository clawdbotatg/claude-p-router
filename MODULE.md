# router — subscription router for claude-p-agent

Routes every turn to the Claude subscription login with the most headroom.
The first module ever extracted from the engine, and the reference example of
the **env hook** attachment pattern.

## What it is

One executable, `env`, run by the engine before every spawn (see "modules" in
`agent.py`). It polls Claude's OAuth usage endpoint for every login on this
box — the plain `~/.claude` (`default`) plus one config dir per plan under
`~/.clawd-accounts/<name>` — and prints `CLAUDE_CONFIG_DIR=<best>` so the
child runs on the plan with the most headroom. Usage is TTL-cached on disk
(`.cache.json`, gitignored), so the common case adds ~0.2s to a turn and a
cold poll ~2s.

## What it needs

- Nothing, if you have one login: with fewer than two candidates it does
  nothing at all.
- To route: extra logins, each signed in once via
  `CLAUDE_CONFIG_DIR=~/.clawd-accounts/<name> claude /login`.
- macOS Keychain or `.credentials.json` per config dir (it reads the same
  credential store Claude Code writes).
- Knobs: `CLAUDE_P_ROUTER=0` disables; `CLAUDE_P_ROUTER_TTL` (default 600s)
  cache lifetime; `CLAWD_ACCOUNTS_DIR` overrides the accounts root.

## Wiring

None beyond `tools/module add` — the engine discovers `env` hooks by
convention. An operator-set `CLAUDE_CONFIG_DIR` in the environment always
wins (the hook prints nothing when a pin is present).

## How to verify

```bash
env -u CLAUDE_CONFIG_DIR ./env          # from this dir
# → prints CLAUDE_CONFIG_DIR=… (or nothing if <2 logins); stderr notes the pick
cat .cache.json                          # per-plan utilization it saw
```

## What can go wrong

- The usage endpoint is **undocumented**; any failure (endpoint change, auth,
  network) degrades to "don't route" — turns still run, on the ambient login.
- It shares the transcript store by symlinking each account dir's `projects/`
  to `~/.claude/projects` so sessions resume under any plan. If you don't
  want cross-plan resume, don't use this module.
- Wrong-plan surprises: the pick is per-turn and cache-driven; a plan can be
  chosen up to TTL seconds after it filled up. Lower the TTL if that bites.

## How to uninstall

`tools/module remove router` (or delete this folder). Turns immediately fall
back to the ambient `CLAUDE_CONFIG_DIR` / default login. Optionally remove
the `projects` symlinks in `~/.clawd-accounts/*/`.
