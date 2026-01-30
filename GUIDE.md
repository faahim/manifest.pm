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

### 3. Canonical State + Generated Views
Only `tasks/MANIFEST.json` (canonical) must be updated by hand. `tasks/INDEX.json` and `tasks/BOARD.md` are generated from it (render before commit).

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

### tasks/MANIFEST.json
Canonical registry of all tasks (statuses, deps, priority, estimates).

**When to update**: This is the *only* required task state file to edit.

### tasks/INDEX.json (generated)
Compatibility/legacy view generated from MANIFEST.json.

**When to update**: Never manually. Regenerate via `node tools/manifest/render.mjs`.

### tasks/BOARD.md (generated)
Human-readable board generated from MANIFEST.json.

**When to update**: Never manually. Regenerate via `node tools/manifest/render.mjs`.

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

---

## Phase Planning

When starting a project or completing a phase, use a systematic approach:

### Phase Planning Process

1. **Phase Definition** (tasks/PHASE-PLAN-TEMPLATE.md)
   - Phase name and objective
   - Success criteria (how we know it's complete)

2. **Task Breakdown**
   - List all features for the phase
   - Break features into tasks (15-45 min each)
   - Combine small tasks, split large ones

3. **Dependencies**
   - Define what tasks depend on what
   - Draw dependency graph if helpful
   - Ensure no circular dependencies

4. **Prioritization**
   - P0: Critical for phase completion
   - P1: Important but can wait
   - P2: Nice to have

5. **Create Task Files**
   - Use task template from PHASE-PLAN-TEMPLATE.md
   - One file per task in `tasks/phase-N/`

6. **Update Canonical Plan**
   - tasks/MANIFEST.json: Add all new tasks (canonical)
   - docs/ROADMAP.md: Update phase status
   - Run: `node tools/manifest/validate.mjs` then `node tools/manifest/render.mjs` (regenerates BOARD.md + INDEX.json)

### Planning Session

Use the sessions_spawn template to have a sub-agent do the planning:

```bash
sessions_spawn({
  task: `You are planning Phase {{PHASE_NUM}} for {{PROJECT_NAME}}.
  Follow tasks/PHASE-PLAN-TEMPLATE.md...`,
  label: "{{project}}-plan-phase-{{n}}",
  runTimeoutSeconds: 1200
})
```

### Task Size Guidelines

| Size | Time | Strategy |
|-------|-------|----------|
| Small | <15 min | Combine with related task |
| Ideal | 15-45 min | Perfect |
| Large | 45-90 min | Split into 2-3 tasks |
| Too Large | >90 min | Break down into sub-features |

**Signs a task is too large:**
- Multiple distinct features in one task
- Involves multiple files/systems
- Unclear what "done" means
- Acceptance criteria are vague

### After Planning

- Update tasks/MANIFEST.json with new tasks
- Run `node tools/manifest/render.mjs` to regenerate BOARD.md + INDEX.json
- Set up watchdog for the phase
- Start first task or let watchdog handle it

---

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
cat tasks/MANIFEST.json | jq '.tasks[] | select(.status=="pending")'
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

Minimum required edits (canonical state):

**MANIFEST.json** (`tasks/MANIFEST.json`):
- Set task status to `completed`
- Update `updated` timestamp
- (optional) fill `touches/domain/risk` if missing (helps safe parallel)

**ACTIVE.json** (`execution/ACTIVE.json`):
- Remove your claim entry from `claims`

**Task spec file** (`tasks/phase-N/PN-XXX.md`) — optional:
- Only update if the task has a long checklist/spec

**LOG.md** (`execution/LOG.md`):
- Add ONE short completion entry (keep it small)

Then run generators:
```bash
node tools/manifest/validate.mjs
node tools/manifest/render.mjs   # regenerates tasks/INDEX.json + tasks/BOARD.md
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

**ADAPTIVE — watchdog decides based on task analysis**

BIAS: Sequential by default. Only parallel when CLEARLY SAFE.

**Safe parallel conditions:**
- Tasks have NO dependencies between them
- Tasks work on COMPLETELY SEPARATE code areas
- No shared files, modules, or database tables
- No overlapping concerns (e.g., two tasks touching the same API endpoint)

**Examples:**

SAFE (can run parallel):
- Task A: Creates UI component X
- Task B: Implements API endpoint Y
- Task C: Writes tests for module Z

UNSAFE (must run sequential):
- Task A: Modifies src/api/users.ts
- Task B: Updates user schema in src/db/users.sql
- Task C: Changes user-related UI components

**Watchdog logic:**
1. Find all ready tasks
2. Read each task file to understand what it touches
3. Check safety conditions
4. If safe → spawn up to 3 agents
5. If unsafe → run one task sequentially

**Conflict Resolution:**
If two agents commit at the same time:
1. Second push will fail
2. Pull and rebase
3. Re-verify registry state
4. Push again

## Common Pitfalls

### ❌ Forgetting to render generated artifacts
**Problem**: MANIFEST.json is correct, but BOARD.md / INDEX.json look stale
**Solution**: Run `node tools/manifest/render.mjs` before final commit (and after pulls)

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
