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
                  ┌─────────────────────┐
                  │  /breakdown         │
                  │  (bulk or single)   │
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
                            │ /plan [--review]
                            ▼
                     ┌─────────────┐
            ┌───────▶│  plan-ready  │
            │        └──────┬───────┘
            │               │
  /research │               │ /implement
  or /plan  │               ▼
  (re-entry)│      ┌────────────────┐
            │      │  implementing  │
            │      └───────┬────────┘
            │              │
            │        ┌─────┴──────┐
            │        │            │
            │   all phases    (manual)
            │   completed     edit status
            │        │            │
            │        ▼            ▼
            │   ┌──────────┐ ┌──────────┐
            │   │   done   │ │ blocked  │
            │   └──────────┘ └──────────┘
            │
            └── /plan --review REJECTED path
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
├── /breakdown (bulk mode)
│   ├── reads:  docs/architecture.md, docs/prd.md
│   └── writes: tasks/TASK-*/status.md       (all tasks)
│                tasks/TASK-*/research.md     (templates)
│                tasks/TASK-*/plan.md         (templates)
│                docs/current-sprint.md       (all entries)
│
├── /breakdown TASK-001 "Title" (single-task mode)
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
├── /plan --review TASK-001 (optional, adds QA review after planning)
│   ├── reads:  (same as /plan) + docs/prd.md
│   └── writes: tasks/TASK-001/plan.md, status.md (+ displays review verdict)
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
    ├── reads:  docs/prd.md, docs/architecture.md
    │            docs/current-sprint.md, tasks/*/status.md
    └── writes: nothing (read-only, detects bootstrap vs sprint mode)
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
| `/sprint` | — | `prd.md`, `architecture.md`, `current-sprint.md`, `tasks/*/status.md` | nothing | — |
| `/architect` | — | `docs/prd.md` | `docs/architecture.md`, `docs/components/*.md` | PRD is filled (not template) |
| `/breakdown` | — or `TASK-XXX "Title" [P0\|P1\|P2]` | `docs/architecture.md`, `docs/prd.md` (bulk) or — (single) | `tasks/TASK-*/{status,research,plan}.md`, `current-sprint.md` | Architecture filled (bulk) or task doesn't exist (single) |
| `/research` | `TASK-XXX` | `status.md`, `prd.md`, `architecture.md`, `src/` | `research.md`, `status.md` | status = `todo` or `plan-ready` |
| `/plan` | `[--review] TASK-XXX` | `research.md`, `architecture.md` (+ `prd.md` with `--review`) | `plan.md`, `status.md` | status = `research-done` or `plan-ready` |
| `/implement` | `TASK-XXX` | `plan.md`, `status.md` | `src/`, `tests/`, `status.md`, `current-sprint.md` | status = `plan-ready` or `implementing` |
| `/fix` | `"bug description"` | `architecture.md`, `src/` | `src/`, `tests/` | — |
| `/worktree` | `TASK-XXX` | `status.md` | git worktree + branch | Task exists, inside git repo |
| `/swarm` | `TASK-A,TASK-B,...` | `tasks/*/status.md` | worktrees, branches, tmux-cli coordination | tmux + tmux-cli, clean main branch |
| `/merge-task` | `TASK-XXX` | `status.md`, branch diff | merge commit, cleanup worktree + branch | status = `done`, branch exists |

## File Structure

After running `bmad-init`, your project looks like this:

```
my-project/
├── .claude/
│   ├── CLAUDE.md                ← Project instructions for Claude
│   ├── commands/
│   │   ├── architect.md         ← /architect command
│   │   ├── breakdown.md         ← /breakdown command (bulk + single-task)
│   │   ├── fix.md               ← /fix command
│   │   ├── implement.md         ← /implement command
│   │   ├── merge-task.md        ← /merge-task command
│   │   ├── plan.md              ← /plan command (supports --review flag)
│   │   ├── research.md          ← /research command
│   │   ├── sprint.md            ← /sprint command (project status + dashboard)
│   │   ├── swarm.md             ← /swarm command (tmux-cli orchestration)
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

bmad-init supports two modes of parallel development: manual worktrees and fully automated tmux-cli orchestration.

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

### Complete lifecycle (before /swarm)

Before running `/swarm`, the project must be bootstrapped:

```
┌──────────────────┐
│  1. Write PRD    │
│  docs/prd.md     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  2. /architect   │
│  → architecture  │
│  → components/   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  3. Review       │
│  architecture.md │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  4. /breakdown   │
│  → tasks/TASK-*  │
│  → sprint board  │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  5. /swarm       │
│  TASK-A,TASK-B   │
│  (parallel FIC)  │
└──────────────────┘
```

Steps 1-4 produce the task definitions that `/swarm` needs. Without them, there are no `tasks/TASK-XXX/status.md` files to drive the workflow.

### Automated Swarm (tmux-cli)

For fully automated parallel work, use `/swarm`. A master Claude orchestrates workers via tmux-cli — each worker is a real `claude` process in its own tmux pane, visible and accessible:

```bash
# Inside Claude Code (must be running in tmux):
/swarm TASK-002,TASK-003,TASK-004
```

#### Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  TERMINAL (tmux session)                                             │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  PANE MASTER (your current Claude Opus session)                │  │
│  │                                                                │  │
│  │  > /swarm TASK-032,TASK-033,TASK-034                           │  │
│  │                                                                │  │
│  │  Role: orchestrate only, does NOT write code                   │  │
│  │  Tools: tmux-cli launch/send/wait_idle/list_panes              │  │
│  │  Coordination: reads status.md on disk (polling)               │  │
│  └───────────────────────────┬────────────────────────────────────┘  │
│                              │                                       │
│                tmux-cli launch "zsh" (×3)                            │
│                              │                                       │
│            ┌─────────────────┼─────────────────┐                     │
│            │                 │                 │                     │
│            ▼                 ▼                 ▼                     │
│    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│    │  PANE 0:6.2  │  │  PANE 0:6.3  │  │  PANE 0:6.4  │             │
│    │              │  │              │  │              │             │
│    │  worktree:   │  │  worktree:   │  │  worktree:   │             │
│    │  ../proj-032 │  │  ../proj-033 │  │  ../proj-034 │             │
│    │              │  │              │  │              │             │
│    │  claude      │  │  claude      │  │  claude      │             │
│    │  --model     │  │  --model     │  │  --model     │             │
│    │  sonnet      │  │  sonnet      │  │  sonnet      │             │
│    │              │  │              │  │              │             │
│    │  VISIBLE     │  │  VISIBLE     │  │  VISIBLE     │             │
│    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│           │                 │                 │                     │
│           ▼                 ▼                 ▼                     │
│      status.md         status.md         status.md                  │
│      (polled by         (polled by        (polled by                │
│       master)            master)           master)                  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

#### Workflow sequence

```
Master (Opus)            Worker 032 (Sonnet)  Worker 033 (Sonnet)  Worker 034 (Sonnet)
    │                         │                    │                    │
    │── git worktree add ────▶│                    │                    │
    │── git worktree add ─────┼───────────────────▶│                    │
    │── git worktree add ─────┼────────────────────┼───────────────────▶│
    │                         │                    │                    │
    │── tmux-cli launch ─────▶│ claude ready       │                    │
    │── tmux-cli launch ──────┼───────────────────▶│ claude ready       │
    │── tmux-cli launch ──────┼────────────────────┼───────────────────▶│ claude ready
    │                         │                    │                    │
    │                         │                    │                    │
    │── send /research ──────▶│                    │                    │
    │      (3s delay)         │                    │                    │
    │── send /research ───────┼───────────────────▶│                    │
    │      (3s delay)         │                    │                    │
    │── send /research ───────┼────────────────────┼───────────────────▶│
    │                         │                    │                    │
    │                         │ ▓▓▓ exploring ▓▓▓  │ ▓▓▓ exploring ▓▓▓  │ ▓▓▓ exploring ▓▓▓
    │                         │                    │                    │
    │  ┌─ POLL LOOP 30s ──┐  │                    │                    │
    │  │ Read status.md    │  │                    │                    │
    │  │ Display dashboard │  │                    │                    │
    │  │ Detect stalls     │  │                    │                    │
    │  └───────────────────┘  │                    │                    │
    │                         │                    │                    │
    │── Read: research-done ─▶│                    │                    │
    │── send /plan ──────────▶│ ▓▓▓ planning ▓▓▓   │                    │
    │                         │                    │                    │
    │── Read: research-done ──┼───────────────────▶│                    │
    │── send /plan ───────────┼───────────────────▶│ ▓▓▓ planning ▓▓▓   │
    │                         │                    │                    │
    │── Read: plan-ready ────▶│ ✓                  │                    │
    │── Read: plan-ready ─────┼───────────────────▶│ ✓                  │
    │── Read: plan-ready ─────┼────────────────────┼───────────────────▶│ ✓
    │                         │                    │                    │
    │                         │                    │                    │
    │══ BATCH REVIEW GATE ════│════════════════════│════════════════════│
    │                         │                    │                    │
    │  "All plans ready —     │ (idle)             │ (idle)             │ (idle)
    │   review each plan.md"  │                    │                    │
    │                         │                    │                    │
    │◀── YOU: "approve all"   │                    │                    │
    │                         │                    │                    │
    │── send /implement ─────▶│ ▓▓▓ coding ▓▓▓     │                    │
    │── send /implement ──────┼───────────────────▶│ ▓▓▓ coding ▓▓▓     │
    │── send /implement ──────┼────────────────────┼───────────────────▶│ ▓▓▓ coding ▓▓▓
    │                         │                    │                    │
    │  ┌─ POLL LOOP 60s ──┐  │                    │                    │
    │  │ Read status.md    │  │                    │                    │
    │  │ Track Phase X/Y   │  │                    │                    │
    │  └───────────────────┘  │                    │                    │
    │                         │                    │                    │
    │── Read: done ───────────┼───────────────────▶│ ✓                  │
    │── Read: done ──────────▶│ ✓                  │                    │
    │── Read: done ───────────┼────────────────────┼───────────────────▶│ ✓
    │                         │                    │                    │
    │                         │                    │                    │
    │══ SEQUENTIAL MERGE ═════│════════════════════│════════════════════│
    │                         │                    │                    │
    │── merge TASK-033 ───────┼───────────────────▶│ (smallest first)   │
    │── verify ── PASS ✓      │                    │                    │
    │── rebase remaining ────▶│                    │                   ▶│
    │                         │                    │                    │
    │── merge TASK-032 ──────▶│                    │                    │
    │── verify ── PASS ✓      │                    │                    │
    │── rebase remaining ─────┼────────────────────┼───────────────────▶│
    │                         │                    │                    │
    │── merge TASK-034 ───────┼────────────────────┼───────────────────▶│
    │── verify ── PASS ✓      │                    │                    │
    │                         │                    │                    │
    │                         │                    │                    │
    │══ CLEANUP ══════════════│════════════════════│════════════════════│
    │                         │                    │                    │
    │── tmux-cli kill ───────▶│ ✕                  │                    │
    │── tmux-cli kill ────────┼───────────────────▶│ ✕                  │
    │── tmux-cli kill ────────┼────────────────────┼───────────────────▶│ ✕
    │── worktree remove       │                    │                    │
    │── branch delete         │                    │                    │
    │                         │                    │                    │
    │── FINAL SUMMARY         │                    │                    │
    ▼                         ▼                    ▼                    ▼
```

#### Decision flow

```
                      ┌──────────────────┐
                      │  /swarm TASK-A,B  │
                      └────────┬─────────┘
                               │
                               ▼
                      ┌──────────────────┐
                      │ Inside tmux?     │
                      │ tmux-cli dispo?  │
                      └────────┬─────────┘
                               │
                       ┌───────┴───────┐
                       │               │
                      Yes              No
                       │               │
                       ▼               ▼
               ┌──────────────┐  ┌─────────────────┐
               │ Read each    │  │ STOP: "Run from │
               │ status.md    │  │ inside tmux"    │
               └───────┬──────┘  └─────────────────┘
                       │
         ┌─────────────┼─────────────┐
         │             │             │
      todo      research-done    plan-ready
         │             │             │
         ▼             ▼             ▼
    needs R+P     needs P only   skip to gate
         │             │          (no pane)
         └──────┬──────┘
                │
                ▼
        ┌──────────────┐
        │ Create       │
        │ worktrees    │
        │ (or reuse    │
        │ existing)    │
        └───────┬──────┘
                │
                ▼
        ┌──────────────┐
        │ Launch panes │
        │ tmux-cli     │
        │ + claude     │
        │ --model      │
        │ sonnet       │
        └───────┬──────┘
                │
                ▼
        ┌──────────────┐
        │ Send FIC     │◀───────────────────────┐
        │ commands     │                        │
        └───────┬──────┘                        │
                │                               │
                ▼                               │
        ┌──────────────┐                        │
        │ Poll 30s     │                        │
        │ status.md    │                        │
        │              │                        │
        │ research-done│                        │
        │ → send /plan │                        │
        │              │                        │
        │ 15min stall? │                        │
        │ → WARN user  │                        │
        └───────┬──────┘                        │
                │ all plan-ready                │
                ▼                               │
   ┌────────────────────────┐                   │
   │  BATCH REVIEW GATE     │                   │
   │  You review all plans  │                   │
   │  "approve" / "reject"  │                   │
   └────────────┬───────────┘                   │
                │                               │
        ┌───────┴───────┐                       │
        │               │                       │
     approve         reject                     │
        │               │                       │
        ▼               └───────────────────────┘
 ┌──────────────┐         (re-poll after revision)
 │ Send         │
 │ /implement   │
 └───────┬──────┘
         │
         ▼
 ┌──────────────┐
 │ Poll 60s     │
 │ status.md    │
 │ Phase X/Y    │
 │              │
 │ 20min stall? │
 │ → WARN user  │
 └───────┬──────┘
         │ all done
         ▼
 ┌──────────────┐
 │ Sequential   │───────┐
 │ merge        │       │
 └───────┬──────┘   conflict?
         │              │
         ▼              ▼
 ┌──────────────┐ ┌──────────────┐
 │ Quality gate │ │ STOP: user   │
 │ (auto-detect)│ │ decides      │
 └───────┬──────┘ └──────────────┘
         │ pass
         ▼
 ┌──────────────┐
 │ Rebase       │
 │ remaining    │
 │ branches     │
 └───────┬──────┘
         │ next task or done
         ▼
 ┌──────────────┐
 │ Cleanup      │
 │ panes +      │
 │ worktrees +  │
 │ branches     │
 └───────┬──────┘
         │
         ▼
 ┌──────────────┐
 │ Final        │
 │ summary      │
 └──────────────┘
```

**Prerequisites:**
- Running inside tmux
- `tmux-cli` available in PATH
- Clean main branch (no uncommitted changes)
- Max 5 tasks per swarm

**Error recovery:**
- Worker pane dies → master detects via `tmux-cli list_panes`, offers relaunch/skip/abort
- Worker stuck → stall timeout (15 min research/plan, 20 min implement), warns user
- Master crash → workers continue in their tmux panes; re-run `/swarm` with same tasks to reconnect

**Quality gate hook:**
bmad-init includes `.claude/hooks/on-task-completed.sh` which auto-detects your project's toolchain (pnpm/yarn/npm/make/cargo/pytest) and runs the appropriate verify command before any task is marked complete.

## FAQ

**Can I skip `/research`?**
Not recommended. `/plan` relies on the context gathered during research. Without it, the plan will be generic and miss project-specific patterns, reusable code, and dependencies.

**Do I have to use `--review` with `/plan`?**
No, the `--review` flag is optional. You can run `/plan TASK-XXX` alone and review plan.md yourself. Use `/plan --review TASK-XXX` when you want Claude to do an extra critical QA pass (completeness, architectural consistency, risks) after generating the plan.

**How do I block a task?**
Edit `tasks/TASK-XXX/status.md` manually and set status = `blocked`. To unblock, set it back to `implementing` or `plan-ready` as appropriate. There is no `/block` command — this is intentional to keep things simple.

**Does the PRD need to be perfect?**
No. Start with a rough version and iterate. You can re-run `/architect` at any time to regenerate the architecture from an updated PRD.

**Can I re-run a command?**
`/architect` can be re-run freely. For task commands, the precondition checks enforce the correct order. If you need to redo research or planning, manually reset the status in `status.md`.

**Can I still create tasks manually?**
Yes. Use `/breakdown TASK-XXX "Title"` to create a single ad-hoc task. Without arguments, `/breakdown` generates all tasks from the architecture in one shot.

**When should I use `/fix` vs the full cycle?**
Use `/fix` for small, well-understood bugs that affect 1-3 files. Use the full FIC cycle (`/breakdown TASK-XXX "Title"` → `/research` → `/plan` → `/implement`) for anything larger, any new feature, or any change that requires architectural understanding. `/fix` will tell you to escalate if the bug is too big.

**What if `/implement` gets interrupted mid-task?**
Re-run `/implement TASK-XXX` in a new session. The resumption algorithm reads `status.md` to find the last completed phase and picks up from the next one.

**What is `/swarm` and when should I use it?**
`/swarm` orchestrates multiple tasks in parallel using tmux-cli. A master Claude launches worker `claude` processes in tmux panes (one per task), each working in its own git worktree. Use it when you have 2-5 independent tasks that can be researched, planned, and implemented concurrently. It requires running inside tmux with `tmux-cli` available.

**What's the difference between `/worktree` and `/swarm`?**
`/worktree` creates a single worktree for manual parallel work — you manage each terminal and Claude session yourself. `/swarm` automates the entire process: worktree creation, tmux pane launching, FIC workflow coordination, review gating, sequential merge, and cleanup. Use `/worktree` for fine-grained control, `/swarm` for hands-off orchestration.

**What does `/merge-task` do exactly?**
`/merge-task` merges a completed task's branch into main with `--no-ff`, runs the quality gate, cleans up the worktree and branch, and produces a summary. If the quality gate fails, it offers to revert the merge. If there are conflicts, it stops and lets you resolve them manually.

**Can I add tasks to a running swarm?**
Yes. The master can run `/breakdown TASK-XXX "Title"` at any point during a swarm to create a new task, then set up its worktree and pane, and add it to the polling loop. This works during the review gate wait or while other workers are implementing. The max 5 workers limit still applies.

**What is the quality gate hook?**
The hook at `.claude/hooks/on-task-completed.sh` runs automatically when a task is completed. It auto-detects your project's toolchain (pnpm, yarn, npm, make, cargo, or pytest) and runs the appropriate verify/test command. If it fails, the task stays incomplete until the errors are fixed.

## License

[MIT](LICENSE)
