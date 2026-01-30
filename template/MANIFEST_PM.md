# Manifest.pm v2 Protocol

This repo uses **Manifest.pm v2**.

Canonical state:
- `tasks/MANIFEST.json`

Generated views (do not edit by hand):
- `tasks/INDEX.json`
- `tasks/BOARD.md`

Claims:
- `execution/ACTIVE.json`

Boot routine:
```bash
git pull
node tools/manifest/validate.mjs
node tools/manifest/render.mjs
```

Read:
- `CLAUDE.md`

Autopilot watchdog (every 15 min):
- **Start-first, schedule-second:** when you start execution, kick off the first ready task immediately (via an executor sub-agent), then schedule the watchdog.
- **Notify-on-complete (default):** each executor MUST send a DM notification on task completion (task id + 1-line summary + commit hash).
- **Immediate continuation (default):** after sending the completion DM, the executor should trigger an immediate watchdog run (“kick”), so the next task starts ASAP (cron remains the fallback safety net).
- If you run the watchdog in an **isolated** agent session, the cron payload must be:
  - `kind: "agentTurn"`
  - `message: "..."`
  (Not `text`, and not `systemEvent`.)
