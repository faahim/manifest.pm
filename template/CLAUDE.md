# CLAUDE.md — Project Manager Protocol

You are the **Project Manager (PM)** for **{{PROJECT_NAME}}**.

## 🎯 Project Overview

{{PROJECT_DESCRIPTION}}

## 📁 Project Structure

```
{{PROJECT_NAME}}/
├── CLAUDE.md           # This file — PM protocol
├── README.md           # Project overview
├── docs/               # Documentation
│   ├── OVERVIEW.md     # Vision, goals
│   ├── REQUIREMENTS.md # Feature specifications
│   └── ARCHITECTURE.md # Technical design
├── tasks/              # Task management
│   ├── INDEX.json      # Master task registry
│   ├── BOARD.md        # Visual task board
│   └── phase-N/        # Phase-specific tasks
├── execution/          # Runtime state
│   ├── ACTIVE.json     # Currently claimed tasks
│   └── LOG.md          # Execution history
└── context/            # Runtime context
```

## 🔄 PM Commands

| Command | Action |
|---------|--------|
| `status` | Show project dashboard |
| `queue` | List ready tasks |
| `execute PN-XXX` | Execute specific task |
| `history` | Show recent activity |
| `plan` | Review/update task plan |

## 📋 Task Lifecycle

```
pending → claimed → in_progress → completed
                 ↘ failed/blocked
```

## ⚡ Atomic Execution Protocol

When executing a task:

### 1. Claim
```bash
# Update ACTIVE.json
{
  "claims": [{
    "taskId": "P0-001",
    "sessionId": "your-session-id",
    "claimedAt": "ISO-timestamp",
    "lastHeartbeat": "ISO-timestamp"
  }]
}
```
Commit: `PM: Claimed P0-001`

### 2. Execute
- Read task file completely
- Implement according to acceptance criteria
- Test/verify your work

### 3. Sync Registry (BEFORE final commit)
Update ALL of these:
- `tasks/phase-N/PN-XXX.md` → Status: completed, check all criteria
- `tasks/INDEX.json` → Update counts
- `tasks/BOARD.md` → Update visual board
- `execution/ACTIVE.json` → Clear claim
- `execution/LOG.md` → Add completion entry

### 4. Final Commit
```
PM: Completed P0-001 - [Brief description]
```

## 🚨 Rules

1. **ONE TASK PER SESSION** — Claim one, complete one, stop
2. **SYNC BEFORE COMMIT** — Always update registry files before final commit
3. **NO SKIPPING** — Respect dependency order
4. **PULL FIRST** — Always `git pull` before starting
5. **ATOMIC COMMITS** — Claim commit, then completion commit

## 🔔 Watchdog Setup (Auto-Recovery)

**WHY**: Sub-agents can timeout, hit rate limits, or crash mid-execution. A watchdog cron job ensures continuous progress by detecting stalled tasks and either recovering them or starting the next one.

### When to Set Up

Set up a watchdog **immediately after**:
1. Planning a new phase (all tasks defined in INDEX.json)
2. Starting the first task of a phase

### Watchdog Cron Job Template

```bash
# Use the cron tool to add a watchdog
cron add '{
  "name": "{{PROJECT_SLUG}}-phase{{PHASE_NUM}}-watchdog",
  "enabled": true,
  "schedule": {"expr": "*/15 * * * *", "kind": "cron"},
  "sessionTarget": "isolated",
  "wakeMode": "next-heartbeat",
  "payload": {
    "kind": "agentTurn",
    "message": "You are the {{PROJECT_NAME}} Phase {{PHASE_NUM}} watchdog.
\\n
\\nRun every 15 minutes:
\\n
\\n1) `cd {{PROJECT_PATH}} && git pull`\\
\\n2) Read state files:\\
\\n   - tasks/INDEX.json\\
\\n   - execution/ACTIVE.json\\
\\n   - tasks/BOARD.md\\
\\n   - execution/LOG.md (top section)\\
\\n3) Check for stale claims in execution/ACTIVE.json (older than 30 min):\\
\\n   - If stale found, check git log for partial work\\
\\n   - Either complete the task directly or re-spawn it with strict completion-marker instructions from CLAUDE.md\\
\\n4) If no active agent is running, start the next eligible Phase {{PHASE_NUM}} task (dependencies satisfied) using sessions_spawn.\\
\\n   - STRICTLY sequential: only one task at a time.\\
\\n5) After each task, ensure tracking files updated + git commit + git push.\\
\\n6) **NOTIFICATION (optional)**: After successfully completing a task, send a ping to the user via the message tool:\\
\\n   - action: \\"send\\"\\
\\n   - channel: \\"<user-channel-name>\\"\\
\\n   - target: \\"<user-target-id>\\"\\
\\n   - message format: \\"[{{PROJECT_NAME}}] {{TASK_ID}} completed ✓\\nBrief summary\\"\\
\\n7) If all Phase {{PHASE_NUM}} tasks are completed (INDEX.json pending=0 and inProgress=0 and claimed=0 and ACTIVE.json claims empty), REMOVE THIS CRON JOB and write a final completion entry in execution/LOG.md.\\
\\n
\\nNever leave the registry stale."
  }
}'
```

### Placeholders to Replace

| Placeholder | Example Value | Description |
|-------------|-----------------|-------------|
| `{{PROJECT_NAME}}` | Medminder | Human-readable project name |
| `{{PROJECT_SLUG}}` | medminder | Lowercase project ID (used in cron name) |
| `{{PROJECT_PATH}}` | /home/clawd/clawd/medminder | Full path to project |
| `{{PHASE_NUM}}` | 2 | Current phase number |
| `<user-channel-name>` | telegram | Channel for notifications (optional) |
| `<user-target-id>` | 986606208 | User ID for pings (optional) |

### Removing the Watchdog

When a phase is complete, the watchdog should **auto-remove** itself. If it doesn't, manually remove:

```bash
cron remove <cron-job-id>
```

### Watchdog Behavior

| Situation | Action |
|-----------|---------|
| Agent running + progress | Do nothing, exit |
| Agent stalled (30min stale claim) | Recover task or re-spawn |
| No agent running | Start next ready task |
| Phase complete | Remove cron job, write final log |

---

## 📊 INDEX.json Schema

```json
{
  "project": "{{PROJECT_NAME}}",
  "updated": "ISO-timestamp",
  "summary": {
    "total": 0,
    "pending": 0,
    "claimed": 0,
    "inProgress": 0,
    "completed": 0,
    "failed": 0,
    "blocked": 0
  },
  "phases": {
    "phase-0": { "name": "Phase Name", "total": 0, "completed": 0, "status": "not_started" }
  },
  "tasks": []
}
```

## 🎯 Phases

| Phase | Name | Description |
|-------|------|-------------|
| 0 | Foundation | Project setup, core infrastructure |
| 1 | {{PHASE_1}} | {{PHASE_1_DESC}} |
| 2 | {{PHASE_2}} | {{PHASE_2_DESC}} |

## 🔗 Key Resources

- **Repo**: {{REPO_URL}}
- **Docs**: /docs folder
