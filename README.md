# Manifest.pm 📋

**A file-based project management framework for AI agents.**

Manifest.pm enables AI agents to manage projects across context resets with atomic task execution, dependency tracking, and persistent state—all stored in plain files.

## ✨ Why Manifest.pm?

AI agents lose context between sessions. Manifest.pm solves this by:

- **File-based state** — Everything in git, nothing in memory
- **Atomic execution** — One task per session, always completable
- **Self-documenting** — Any agent can pick up where another left off
- **Battle-tested** — Born from real projects (Insaf ERP, Ari)

## 🚀 Quick Start

```bash
# Clone the template
npx degit faahim/manifest.pm/template my-project

# Or manually copy the structure
```

## 📁 Project Structure

```
my-project/
├── CLAUDE.md           # PM protocol (the brain)
├── README.md           # Project overview
├── docs/
│   ├── OVERVIEW.md     # Vision, goals
│   ├── REQUIREMENTS.md # What to build
│   └── ARCHITECTURE.md # How to build it
├── tasks/
│   ├── INDEX.json      # Master task registry
│   ├── BOARD.md        # Visual task board
│   └── phase-N/        # Phase folders
│       └── PN-XXX.md   # Individual task files
├── execution/
│   ├── ACTIVE.json     # Currently claimed tasks
│   └── LOG.md          # Execution history
└── context/            # Runtime context files
```

## 🔄 The Protocol

### Task Lifecycle

```
pending → claimed → in_progress → completed
                 ↘ failed/blocked
```

### Atomic Execution (One Task Per Session)

1. **Pull** — Always `git pull` first
2. **Claim** — Update ACTIVE.json, commit
3. **Execute** — Do the work
4. **Sync** — Update ALL registry files
5. **Commit** — Push completion

### The Golden Rule

> **Sync before commit.** Always update these files before your final commit:
> - Task file (status → completed)
> - INDEX.json (update counts)
> - BOARD.md (update status)
> - ACTIVE.json (clear claim)
> - LOG.md (add entry)

## 📋 Task File Template

```markdown
# PN-XXX: Task Title

## Metadata
| Field | Value |
|-------|-------|
| Phase | N |
| Status | pending |
| Priority | P0/P1/P2 |
| Estimate | Xm |
| Dependencies | PN-YYY |
| Blocks | PN-ZZZ |

## Objective
What this task accomplishes.

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

## Context
Links to docs, existing code.

## Notes
Implementation notes.
```

## 📊 INDEX.json Schema

```json
{
  "project": "my-project",
  "updated": "ISO-timestamp",
  "summary": {
    "total": 10,
    "pending": 5,
    "claimed": 1,
    "inProgress": 1,
    "completed": 3,
    "failed": 0,
    "blocked": 0
  },
  "phases": {
    "phase-0": {
      "name": "Foundation",
      "total": 5,
      "completed": 3,
      "status": "in_progress"
    }
  },
  "tasks": [
    {
      "id": "P0-001",
      "title": "Task title",
      "phase": 0,
      "status": "completed",
      "priority": "P0",
      "deps": [],
      "blocks": ["P0-002"]
    }
  ]
}
```

## 🎯 PM Commands

When working with an agent, use these commands:

| Command | Action |
|---------|--------|
| `status` | Show project dashboard |
| `queue` | List ready tasks (deps satisfied) |
| `execute P0-001` | Execute specific task |
| `history` | Show recent activity |
| `plan` | Review/update roadmap |

## 🧠 For AI Agents

### Starting a Session

```
1. cd /path/to/project
2. git pull
3. Read CLAUDE.md
4. Check execution/ACTIVE.json for stale claims
5. Run `queue` to see ready tasks
6. Claim and execute ONE task
7. Sync all files, commit, push
8. STOP
```

### Handling Stale Claims

If ACTIVE.json has a claim older than 30 minutes:
1. Check if work was partially done
2. Either complete it or reset to pending
3. Clear the claim
4. Proceed with fresh claim

## 📈 Improvements Roadmap

- [ ] CLI tool: `manifest init`, `manifest claim`, `manifest complete`
- [ ] Validation script: Check registry sync
- [ ] Stale claim auto-detection
- [ ] Metrics tracking (velocity, task times)
- [ ] Dashboard generator

## 🏗️ Projects Using Manifest.pm

- **Insaf ERP** — Agricultural ERP system
- **Ari** — Automated outreach system

## 📄 License

MIT

---

*Manifest.pm — Because AI agents deserve proper project management too.* 🤍
