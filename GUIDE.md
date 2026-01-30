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

**When to use parallel:**
- Tasks have NO dependencies between them
- Tasks work on COMPLETELY SEPARATE code areas
- Phase requires acceleration (e.g., tight deadline)

**Sequential by default:**
- Simpler, no race conditions on tracking files
- Easier recovery when things go wrong
- Use this for most projects

**Watchdog parallel mode:**
- Set `parallelMode: true` in watchdog to enable multi-agent execution
- Watchdog spawns up to 3 agents for eligible tasks
- ACTIVE.json tracks multiple claims (one per task)

**Conflict Resolution:**
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

## Phase Review Protocol

After completing a phase, run a lightweight QA review before starting the next phase.

### When to Trigger
- All tasks in a phase are marked `completed`
- Before starting the first task of the next phase

### QA Agent Input (Pre-Summarized)

Keep the input concise to minimize tokens:

```markdown
## Phase N Review Request

**Phase:** N - Phase Name
**Tasks Completed:** X tasks
**Key Features:**
- Feature 1 (from task titles)
- Feature 2
- Feature 3

**Main Files Created/Modified:**
- src/path/to/file1.ts
- src/path/to/file2.ts
- (list ~5-10 core files only)

**Acceptance Criteria Summary:**
- Criterion 1
- Criterion 2
- (condensed, not full task files)
```

### QA Agent Actions (Single Pass, ~5 min)

1. ✅ **Build Check:** `npm run build` passes
2. 👀 **Spot-Read:** Main files only (not everything)
3. 🔍 **Red Flag Scan:**
   - Hardcoded values / TODOs left behind
   - Unused imports / dead code
   - Missing error handling
   - Auth/security bypasses
   - Console.logs left in production code
4. 📋 **Spot Verification:** Check 2-3 key acceptance criteria
5. 📝 **Report:** Output findings

### QA Report Format

```markdown
## Phase N Review

**Build:** ✅ Passes | ❌ Fails (reason)
**Code Quality:** ✅ No major issues | ⚠️ Issues found
**Feature Verification:** ✅ Spot-checked X/Y tasks

### Issues Found (if any)
- P1-005: Consider adding loading state for slow connections
- P1-009: Hardcoded "20" should be config value

### Red Flags (if any)
- ❌ Auth bypass in /api/admin (CRITICAL)
- ⚠️ Console.log in production code

### Recommendation
✅ Proceed to Phase N+1
⚠️ Fix issues before proceeding
❌ Major issues — do not proceed
```

### Spawn Template

```javascript
sessions_spawn({
  task: `You are a QA reviewer for {{PROJECT}}.

## Phase {{N}} Review Request

**Tasks Completed:** {{X}} tasks
**Key Features:**
{{FEATURE_LIST}}

**Main Files:**
{{FILE_LIST}}

**YOUR JOB:**
1. Run \`cd {{PROJECT_PATH}} && npm run build\`
2. Spot-read the main files listed above
3. Check for red flags (hardcoded values, TODOs, console.logs, auth issues)
4. Verify 2-3 acceptance criteria actually work
5. Output a Phase Review report

Be concise. Single pass. ~5 minutes max.`,
  label: "{{project}}-qa-phase-{{n}}",
  runTimeoutSeconds: 600
})
```

### Token Budget
- **Target:** ~20-30k tokens
- **How:** Pre-summarize input, single pass, no back-and-forth

### What It Catches
- Build failures
- Obvious bugs
- Incomplete features
- Security issues
- Code quality problems

### What It Doesn't Replace
- Human QA (you should still test manually)
- Comprehensive testing (unit, integration, e2e)
- Security audits

---

## Agent-Native Design

If your project is meant to be **run by AI agents** (not just built by them), consider these principles:

### Use Native Tools Over CLI Wrappers

| Task | ❌ Generic | ✅ Agent-Native |
|------|-----------|-----------------|
| Web scraping | subprocess to CLI | `browser` tool directly |
| HTTP requests | curl/fetch CLI | `web_fetch` tool |
| File ops | shell commands | `read`/`write` tools |

### Design for Agent Execution

1. **Scripts are agent tasks** — Written to be run by spawned sub-agents
2. **Error messages for agents** — Clear, actionable, not cryptic
3. **Leverage reasoning** — Agents can decide, not just execute
4. **Context is maintained** — Within session, everything remembered

### Add Agent Instructions

Document how the executing agent should use each module:

```typescript
/**
 * ## Agent Instructions
 * 
 * To use this module:
 * 1. Call web_fetch({ url, extractMode: 'text' })
 * 2. Pass result to extractContacts(content)
 */
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
- [ ] Agent-native design checklist

---

*Built with 🤍 for AI agents who deserve proper project management.*
