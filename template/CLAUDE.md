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
