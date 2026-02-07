# bmad-init

Scaffold a structured AI-assisted development workflow for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## What is this?

`bmad-init` bootstraps a project scaffold that turns Claude Code into a disciplined development partner. Instead of dumping entire features into a single prompt, the workflow breaks work into **short, focused sessions** — research, plan, implement — using markdown files as persistent memory between sessions. Each session starts with a fresh context window and reads only the files it needs, keeping token usage low and output quality high.

The approach combines three methodologies:
- **BMAD** (Build Measure Analyze Design) — architecture from a PRD
- **FIC** (Focused Intelligence Cycle) — research → plan → implement per task
- **CCPM-light** — sprint board with task tracking

## Install

```bash
# Clone the repo
git clone https://github.com/al0x0508/bmad-init.git

# Symlink into PATH
ln -s "$(pwd)/bmad-init/bmad-init" /usr/local/bin/bmad-init
```

Verify:
```bash
bmad-init --help  # Should print usage info (or run on a test dir)
```

## Quick Start

```bash
# 1. Create and init your project
mkdir my-project && cd my-project && git init
bmad-init

# 2. Write your requirements
#    Edit docs/prd.md with your project vision, features, constraints

# 3. Start Claude Code
claude

# 4. Generate architecture (inside Claude Code)
/architect

# 5. Review architecture.md, then generate all tasks
/breakdown

# 6. Research (then start a new session)
/research TASK-001

# 7. Plan (new session — fresh context)
/plan TASK-001

# 8. Review plan.md yourself (10-15 min)

# 9. Implement
/implement TASK-001

# 10. Check sprint status and move to next task
/sprint
```

Each `/research`, `/plan`, and `/implement` should run in a **separate Claude Code session** to keep context fresh.

## How It Works

### Task Lifecycle

Each task follows a strict state machine. Commands transition between states:

```
         ┌──────────┐         ┌─────────────┐
         │ (create) │         │ (create N)  │
         └─────┬────┘         └──────┬──────┘
               │ /new-task           │ /breakdown
               └──────────┬──────────┘
                          ▼
                    ┌──────────┐
                    │   todo   │
                    └─────┬────┘
                          │ /research
                          ▼
                 ┌────────────────┐
                 │ research-done  │
                 └───────┬────────┘
                         │ /plan
                         ▼
                  ┌─────────────┐
            ┌────▶│  plan-ready  │◀─────────────────┐
            │     └──────┬───────┘                  │
            │            │                    /review (read-only)
  /research │            │ /implement               │
  or /plan  │            ▼                          │
  (re-entry)│   ┌────────────────┐                  │
            │   │  implementing  │──────────────────┘
            │   └───────┬────────┘
            │           │
            │     ┌─────┴──────┐
            │     │            │
            │all phases    (manual)
            │completed     edit status
            │     │            │
            │     ▼            ▼
            │┌──────────┐ ┌──────────┐
            ││   done   │ │ blocked  │
            │└──────────┘ └──────────┘
            │
            └── /review REJECTED path
```

### Session Workflow

Each session starts fresh. Markdown files on disk act as the memory bridge:

```
Session 1 (fresh context)         Session 2 (fresh context)
┌───────────────────────┐         ┌───────────────────────┐
│ Claude reads CLAUDE.md│         │ Claude reads CLAUDE.md│
│ /research TASK-001    │         │ /plan TASK-001        │
│ → explores codebase   │         │ → reads research.md   │
│ → writes research.md  │         │ → writes plan.md      │
│ Context used: ~40%    │         │ Context used: ~30%    │
└───────────┬───────────┘         └───────────┬───────────┘
            │                                 │
            ▼                                 ▼
      research.md                         plan.md
      (on disk)                          (on disk)

Session 3 (fresh context)
┌───────────────────────┐
│ Claude reads CLAUDE.md│
│ /implement TASK-001   │
│ → reads plan.md       │
│ → codes phase by phase│
│ → commits each phase  │
│ Context used: ~60%    │
└───────────────────────┘
```

### Files Produced at Each Step

```
bmad-init
│
├── /architect
│   ├── reads:  docs/prd.md
│   └── writes: docs/architecture.md
│                docs/components/*.md
│
├── /breakdown
│   ├── reads:  docs/architecture.md, docs/prd.md
│   └── writes: tasks/TASK-*/status.md       (all tasks)
│                tasks/TASK-*/research.md     (templates)
│                tasks/TASK-*/plan.md         (templates)
│                docs/current-sprint.md       (all entries)
│
├── /new-task TASK-001
│   └── writes: tasks/TASK-001/status.md
│                tasks/TASK-001/research.md  (template)
│                tasks/TASK-001/plan.md      (template)
│                docs/current-sprint.md      (appends entry)
│
├── /research TASK-001
│   ├── reads:  tasks/TASK-001/status.md
│   │            docs/prd.md, docs/architecture.md
│   │            src/**  (explores)
│   └── writes: tasks/TASK-001/research.md
│                tasks/TASK-001/status.md    (→ research-done)
│
├── /plan TASK-001
│   ├── reads:  tasks/TASK-001/research.md
│   │            docs/architecture.md
│   └── writes: tasks/TASK-001/plan.md
│                tasks/TASK-001/status.md    (→ plan-ready)
│
├── /review TASK-001
│   ├── reads:  tasks/TASK-001/plan.md
│   │            tasks/TASK-001/research.md
│   │            docs/architecture.md, docs/prd.md
│   └── writes: nothing (read-only)
│
├── /implement TASK-001
│   ├── reads:  tasks/TASK-001/plan.md
│   │            tasks/TASK-001/status.md
│   └── writes: src/**  (code changes)
│                tests/**  (test files)
│                tasks/TASK-001/status.md    (→ implementing → done)
│                docs/current-sprint.md      (Todo → In Progress, then In Progress → Done)
│
├── /worktree TASK-002
│   ├── reads:  tasks/TASK-002/status.md
│   └── runs:   git worktree add (creates branch + directory)
│
├── /swarm TASK-002,TASK-003
│   ├── reads:  tasks/TASK-*/status.md
│   ├── runs:   git worktree add (per task)
│   │            spawns teammate agents (research → plan → implement)
│   │            sequential merge to main after user approval
│   └── writes: worktrees, branches, merge commits
│                tasks/TASK-*/status.md    (→ done)
│
├── /merge-task TASK-002
│   ├── reads:  tasks/TASK-002/status.md
│   │            branch diff (git log + git diff --stat)
│   ├── runs:   git merge --no-ff, quality gate, cleanup
│   └── writes: merge commit on main
│                removes worktree + branch
│
└── /sprint
    ├── reads:  docs/current-sprint.md
    │            tasks/*/status.md
    └── writes: nothing (read-only)
```

## The FIC Methodology

**FIC (Focused Intelligence Cycle)** is the core workflow pattern in bmad-init. It structures AI-assisted development into three focused sessions per task:

1. **Research** (`/research`) — Explore the codebase, identify relevant files, patterns, and dependencies. Output: `research.md`. No code written.
2. **Plan** (`/plan`) — Read the research output, design a phased implementation plan. Output: `plan.md`. Reviewed by a human before proceeding.
3. **Implement** (`/implement`) — Execute the plan phase by phase, committing after each phase. Output: working code + tests.

**Why three sessions instead of one?**

- **Fresh context**: Each session starts with a clean context window, reading only the files it needs. No accumulated noise from prior work.
- **Markdown as memory**: Files on disk (`research.md`, `plan.md`, `status.md`) persist between sessions, acting as structured handoff documents.
- **Human checkpoint**: The plan is reviewed before implementation begins, catching architectural mistakes early instead of after code is written.
- **Resumability**: If a session is interrupted, `status.md` tracks exactly where work stopped. The next session picks up seamlessly.
- **Low token usage**: Each session uses 30-60% of the context window instead of exhausting it in a single long session.

The result is higher-quality output with lower token costs and a built-in review gate.

## Commands Reference

| Command | Argument | Reads | Writes | Precondition |
|---------|----------|-------|--------|-------------|
| `/help` | — | `prd.md`, `architecture.md`, `current-sprint.md`, `tasks/*/status.md` | nothing | — |
| `/architect` | — | `docs/prd.md` | `docs/architecture.md`, `docs/components/*.md` | PRD is filled (not template) |
| `/breakdown` | — | `docs/architecture.md`, `docs/prd.md` | `tasks/TASK-*/{status,research,plan}.md`, `current-sprint.md` | Architecture is filled (not template) |
| `/new-task` | `TASK-XXX "Title" [P0\|P1\|P2]` | — | `tasks/TASK-XXX/{status,research,plan}.md` | Task does not exist |
| `/research` | `TASK-XXX` | `status.md`, `prd.md`, `architecture.md`, `src/` | `research.md`, `status.md` | status = `todo` or `plan-ready` |
| `/plan` | `TASK-XXX` | `research.md`, `architecture.md` | `plan.md`, `status.md` | status = `research-done` or `plan-ready` |
| `/review` | `TASK-XXX` | `plan.md`, `research.md`, `architecture.md`, `prd.md` | nothing | status = `plan-ready` |
| `/implement` | `TASK-XXX` | `plan.md`, `status.md` | `src/`, `tests/`, `status.md`, `current-sprint.md` | status = `plan-ready` or `implementing` |
| `/fix` | `"bug description"` | `architecture.md`, `src/` | `src/`, `tests/` | — |
| `/worktree` | `TASK-XXX` | `status.md` | git worktree + branch | Task exists, inside git repo |
| `/swarm` | `TASK-A,TASK-B,...` | `tasks/*/status.md` | worktrees, branches, agent coordination | Agent Teams enabled, clean main branch |
| `/merge-task` | `TASK-XXX` | `status.md`, branch diff | merge commit, cleanup worktree + branch | status = `done`, branch exists |
| `/sprint` | — | `current-sprint.md`, `tasks/*/status.md` | nothing | — |

## File Structure

After running `bmad-init`, your project looks like this:

```
my-project/
├── .claude/
│   ├── CLAUDE.md                ← Project instructions for Claude
│   ├── commands/
│   │   ├── architect.md         ← /architect command
│   │   ├── breakdown.md         ← /breakdown command
│   │   ├── fix.md               ← /fix command
│   │   ├── help.md              ← /help command
│   │   ├── implement.md         ← /implement command
│   │   ├── merge-task.md        ← /merge-task command
│   │   ├── new-task.md          ← /new-task command
│   │   ├── plan.md              ← /plan command
│   │   ├── research.md          ← /research command
│   │   ├── review.md            ← /review command
│   │   ├── sprint.md            ← /sprint command
│   │   ├── swarm.md             ← /swarm command (Agent Teams)
│   │   └── worktree.md          ← /worktree command
│   └── hooks/
│       └── on-task-completed.sh ← Quality gate hook (auto-detects toolchain)
├── docs/
│   ├── prd.md                   ← Your Product Requirements (fill this first)
│   ├── architecture.md          ← Generated by /architect
│   ├── components/              ← Per-component specs from /architect
│   └── current-sprint.md        ← Sprint board
├── tasks/                       ← One directory per task
│   └── TASK-001/
│       ├── status.md            ← Task state machine
│       ├── research.md          ← FIC Phase 1 output
│       └── plan.md              ← FIC Phase 2 output (you review this)
└── src/                         ← Your code goes here
```

## Parallel Work

bmad-init supports two modes of parallel development: manual worktrees and fully automated Agent Teams.

### Manual Worktrees

For independent tasks, use `/worktree` to set up parallel work:

```bash
# Inside Claude Code:
/worktree TASK-002
# → Creates branch feat/task-002-api-endpoints
# → Creates worktree at ../my-project-TASK-002
# → Prints: cd ../my-project-TASK-002 && claude

/worktree TASK-003
# → Same for TASK-003
```

Then in separate terminals:

```bash
# Terminal 1
cd ../my-project-TASK-002 && claude
/research TASK-002

# Terminal 2
cd ../my-project-TASK-003 && claude
/research TASK-003
```

When done, merge back with `/merge-task`:

```bash
# Inside Claude Code (from main worktree):
/merge-task TASK-002
# → Merges feat/task-002-api-endpoints into main
# → Runs quality gate
# → Cleans up worktree + branch

/merge-task TASK-003
# → Same for TASK-003
```

Each worktree gets its own Claude Code session with independent context.

### Agent Teams (Swarm)

For fully automated parallel work, use `/swarm`. A lead agent creates worktrees, spawns teammate agents, coordinates the full FIC cycle, and merges results — all in one session:

```bash
# Inside Claude Code:
/swarm TASK-002,TASK-003,TASK-004
```

The lead agent (you) orchestrates the following flow:

```
┌──────────────────────────────────────────────────────────────────┐
│  /swarm TASK-002,TASK-003,TASK-004                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Create worktrees (one per task)                              │
│  2. Spawn teammate agents (Sonnet, one per worktree)             │
│  3. Teammates run /research + /plan in parallel                  │
│  4. ── BATCH REVIEW GATE ── you review all plans                 │
│  5. Approve/reject → teammates run /implement                    │
│  6. Sequential merge to main (smallest first)                    │
│  7. Quality gate after each merge                                │
│  8. Cleanup worktrees + branches                                 │
│  9. Final summary with results table                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Prerequisites:**
- Claude Code [Agent Teams](https://docs.anthropic.com/en/docs/claude-code/agent-teams) enabled (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env var)
- Clean main branch (no uncommitted changes)
- Max 5 tasks per swarm

**Quality gate hook:**
bmad-init includes `.claude/hooks/on-task-completed.sh` which auto-detects your project's toolchain (pnpm/yarn/npm/make/cargo/pytest) and runs the appropriate verify command before any task is marked complete.

## FAQ

**Can I skip `/research`?**
Not recommended. `/plan` relies on the context gathered during research. Without it, the plan will be generic and miss project-specific patterns, reusable code, and dependencies.

**Do I have to run `/review`?**
No, `/review` is optional. You can go straight from `/plan` to `/implement` after reviewing plan.md yourself. The `/review` command is useful when you want Claude to do an extra critical pass before you review.

**How do I block a task?**
Edit `tasks/TASK-XXX/status.md` manually and set status = `blocked`. To unblock, set it back to `implementing` or `plan-ready` as appropriate. There is no `/block` command — this is intentional to keep things simple.

**Does the PRD need to be perfect?**
No. Start with a rough version and iterate. You can re-run `/architect` at any time to regenerate the architecture from an updated PRD.

**Can I re-run a command?**
`/architect` can be re-run freely. For task commands, the precondition checks enforce the correct order. If you need to redo research or planning, manually reset the status in `status.md`.

**Can I still create tasks manually?**
Yes. `/new-task` still works for ad-hoc tasks. `/breakdown` generates all tasks from the architecture in one shot, but you can add more tasks individually at any time.

**When should I use `/fix` vs the full cycle?**
Use `/fix` for small, well-understood bugs that affect 1-3 files. Use the full FIC cycle (`/new-task` → `/research` → `/plan` → `/implement`) for anything larger, any new feature, or any change that requires architectural understanding. `/fix` will tell you to escalate if the bug is too big.

**What if `/implement` gets interrupted mid-task?**
Re-run `/implement TASK-XXX` in a new session. The resumption algorithm reads `status.md` to find the last completed phase and picks up from the next one.

**What is `/swarm` and when should I use it?**
`/swarm` orchestrates multiple tasks in parallel using Claude Code Agent Teams. A lead agent spawns teammate agents (one per task), each working in its own git worktree. Use it when you have 2-5 independent tasks that can be researched, planned, and implemented concurrently. It requires the `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` env var to be set.

**What's the difference between `/worktree` and `/swarm`?**
`/worktree` creates a single worktree for manual parallel work — you manage each terminal and Claude session yourself. `/swarm` automates the entire process: worktree creation, agent spawning, FIC workflow coordination, review gating, sequential merge, and cleanup. Use `/worktree` for fine-grained control, `/swarm` for hands-off orchestration.

**What does `/merge-task` do exactly?**
`/merge-task` merges a completed task's branch into main with `--no-ff`, runs the quality gate, cleans up the worktree and branch, and produces a summary. If the quality gate fails, it offers to revert the merge. If there are conflicts, it stops and lets you resolve them manually.

**What is the quality gate hook?**
The hook at `.claude/hooks/on-task-completed.sh` runs automatically when an Agent Teams teammate marks a task as complete. It auto-detects your project's toolchain (pnpm, yarn, npm, make, cargo, or pytest) and runs the appropriate verify/test command. If it fails, the task stays incomplete until the teammate fixes the errors.

## License

[MIT](LICENSE)
