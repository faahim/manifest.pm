# Manifest.pm — Complete Guide

## Why This Framework Exists

AI agents (Claude, GPT, etc.) lose context between sessions. When working on multi-session projects, this creates problems:

1. **Lost progress** — Agent doesn't know what was done
2. **Duplicate work** — Agent redoes completed tasks
3. **Inconsistent state** — Partial changes, broken builds
4. **No coordination** — Multiple agents step on each other

Manifest.pm solves this by storing ALL project state in git-tracked files.

## Core Principles

### 1. File-Based State
Everything is in files. Nothing in memory. When a session ends, git has the full picture.

### 2. Atomic Execution
One task per session. Each task is small enough to complete. No half-finished work.

### 3. Registry Sync
All tracking files (INDEX.json, BOARD.md, etc.) are updated BEFORE the final commit. Never drift.

### 4. Self-Documenting
Any agent can read CLAUDE.md and understand exactly how to work on this project.

## File Purposes

### CLAUDE.md
The "brain" of the project. Contains:
- Project overview
- Directory structure
- PM commands
- Execution protocol
- Phase definitions

**When to update**: When adding phases, changing protocol, or updating project scope.

### docs/OVERVIEW.md
Vision and goals. High-level "why" and "what".

**When to update**: When scope changes significantly.

### docs/REQUIREMENTS.md
Detailed feature specifications. The "what" in detail.

**When to update**: When requirements are clarified or added.

### docs/ARCHITECTURE.md
Technical design. The "how".

**When to update**: When making architectural decisions.

### tasks/INDEX.json
Master registry of all tasks. Single source of truth for counts and status.

**When to update**: Every task status change.

### tasks/BOARD.md
Human-readable task board. Visual representation of INDEX.json.

**When to update**: Every task status change.

### tasks/phase-N/PN-XXX.md
Individual task files with full details.

**When to update**: When task is claimed, completed, or blocked.

### execution/ACTIVE.json
Currently claimed tasks. Prevents multiple agents from working on the same task.

**When to update**: When claiming or completing a task.

### execution/LOG.md
Execution history. Audit trail of all work done.

**When to update**: After every task completion.

## Task Design Guidelines

### Size
- **15-45 minutes** of focused work
- If longer, break into subtasks
- If shorter, consider combining with related work

### Acceptance Criteria
- **Specific**: "Button triggers API call" not "Button works"
- **Testable**: Can verify with a command or inspection
- **Complete**: All criteria met = task done

### Dependencies
- Only list **direct** dependencies
- Don't list transitive deps (A→B→C, A only lists B)
- Use `—` if no dependencies

### Blocks
- List tasks that can't start until this completes
- Helps with pipeline visualization

## Execution Protocol (Detailed)

### 1. Session Start
```bash
cd /path/to/project
git pull origin main
```

Read CLAUDE.md to understand the project and protocol.

### 2. Check for Stale Claims
```bash
cat execution/ACTIVE.json
```

If claims exist older than 30 minutes:
- Check git log for partial work
- Either complete the task or reset it
- Clear the stale claim

### 3. Find Next Task
Look at BOARD.md "Ready Queue" or:
```bash
# Tasks with status=pending and all deps completed
cat tasks/INDEX.json | jq '.tasks[] | select(.status=="pending")'
```

### 4. Claim Task
Update `execution/ACTIVE.json`:
```json
{
  "claims": [{
    "taskId": "P0-001",
    "sessionId": "your-identifier",
    "claimedAt": "2025-01-28T16:00:00Z",
    "lastHeartbeat": "2025-01-28T16:00:00Z"
  }]
}
```

Commit immediately:
```bash
git add execution/ACTIVE.json
git commit -m "PM: Claimed P0-001"
git push
```

### 5. Execute Task
- Read the full task file
- Implement according to acceptance criteria
- Test your work

### 6. Sync Registry (CRITICAL)
Update ALL of these before final commit:

**Task file** (`tasks/phase-N/PN-XXX.md`):
```markdown
| Status | completed |
| Completed At | 2025-01-28T16:30:00Z |

## Acceptance Criteria
- [x] Criterion 1
- [x] Criterion 2
```

**INDEX.json**:
- Increment `summary.completed`
- Decrement `summary.pending`
- Update phase counts
- Update task entry status

**BOARD.md**:
- Move task to completed
- Update phase progress
- Update ready queue

**ACTIVE.json**:
```json
{"claims": []}
```

**LOG.md**:
```markdown
### 16:30 UTC — P0-001: Task Title
- **Status**: ✅ Completed
- **Session**: your-identifier
- **Duration**: 30 min
- **Notes**: What was done
```

### 7. Final Commit
```bash
git add -A
git commit -m "PM: Completed P0-001 - Brief description"
git push
```

### 8. Stop
Do not start another task. End the session.

## Multi-Agent Coordination

### Claim-Based Locking
ACTIVE.json acts as a lock. Before claiming:
1. Pull latest
2. Check if task already claimed
3. If claimed by another, pick different task

### Parallel Execution
Multiple agents can work simultaneously on **different tasks** with no dependencies between them.

### Conflict Resolution
If two agents commit at the same time:
1. Second push will fail
2. Pull and rebase
3. Re-verify registry state
4. Push again

## Common Pitfalls

### ❌ Forgetting to sync registry
**Problem**: Task file says "completed" but INDEX.json still says "pending"
**Solution**: Always update ALL files before final commit

### ❌ Claiming multiple tasks
**Problem**: Agent claims P0-001 and P0-002
**Solution**: ONE task per session, always

### ❌ Not pulling first
**Problem**: Working on outdated code, merge conflicts
**Solution**: Always `git pull` before any work

### ❌ Skipping dependencies
**Problem**: Building P0-003 before P0-001 is done
**Solution**: Respect dependency order strictly

### ❌ Tasks too large
**Problem**: Session ends mid-task, work is lost
**Solution**: Keep tasks 15-45 minutes

## Spawning Sub-Agents

When using Clawdbot or similar systems:

```javascript
sessions_spawn({
  task: `You are the PM for {{project}}. Execute task P0-001.
  
**FIRST ACTION**: Run \`cd /path/to/project && git pull\` then read CLAUDE.md

**STEPS**:
1. Read CLAUDE.md — follow the atomic protocol EXACTLY
2. Read tasks/phase-0/P0-001.md for requirements
3. Claim P0-001 in ACTIVE.json, commit + push
4. Implement the task
5. Update ALL tracking files
6. Commit + push completion
7. Report summary and STOP

Remember: ONE TASK ONLY.`,
  label: "project-pm-p0-001",
  runTimeoutSeconds: 1200
})
```

## Future Improvements

- [ ] `manifest init` — CLI to scaffold new project
- [ ] `manifest validate` — Check registry sync
- [ ] `manifest queue` — Show ready tasks
- [ ] `manifest claim P0-001` — Auto-update ACTIVE.json
- [ ] `manifest complete` — Auto-sync all files
- [ ] Stale claim auto-cleanup
- [ ] Velocity metrics
- [ ] Dashboard generator

---

*Built with 🤍 for AI agents who deserve proper project management.*
