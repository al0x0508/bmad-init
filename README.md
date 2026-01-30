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

# 5. Create your first task
/new-task TASK-001 "Setup core module"

# 6. Research (then start a new session)
/research TASK-001

# 7. Plan (new session — fresh context)
/plan TASK-001

# 8. Review plan.md yourself (10-15 min)

# 9. Implement
/implement TASK-001

# 10. Check sprint status
/sprint
```

Each `/research`, `/plan`, and `/implement` should run in a **separate Claude Code session** to keep context fresh.

## How It Works

### Task Lifecycle

Each task follows a strict state machine. Commands transition between states:

```
                ┌──────────┐
                │ (create) │
                └─────┬────┘
                      │ /new-task
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
              │  plan-ready  │◀─────────────────┐
              └──────┬───────┘                  │
                     │                    /review (read-only)
                     │ /implement               │
                     ▼                          │
            ┌────────────────┐                  │
            │  implementing  │──────────────────┘
            └───────┬────────┘
                    │
              ┌─────┴──────┐
              │            │
         all phases    (manual)
         completed     edit status
              │            │
              ▼            ▼
        ┌──────────┐ ┌──────────┐
        │   done   │ │ blocked  │
        └──────────┘ └──────────┘
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
│                docs/current-sprint.md      (moves to Done)
│
└── /sprint
    ├── reads:  docs/current-sprint.md
    │            tasks/*/status.md
    └── writes: nothing (read-only)
```

## Commands Reference

| Command | Argument | Reads | Writes | Precondition |
|---------|----------|-------|--------|-------------|
| `/architect` | — | `docs/prd.md` | `docs/architecture.md`, `docs/components/*.md` | PRD is filled (not template) |
| `/new-task` | `TASK-XXX "Title"` | — | `tasks/TASK-XXX/{status,research,plan}.md` | Task does not exist |
| `/research` | `TASK-XXX` | `status.md`, `prd.md`, `architecture.md`, `src/` | `research.md`, `status.md` | status = `todo` |
| `/plan` | `TASK-XXX` | `research.md`, `architecture.md` | `plan.md`, `status.md` | status = `research-done` |
| `/review` | `TASK-XXX` | `plan.md`, `research.md`, `architecture.md`, `prd.md` | nothing | status = `plan-ready` |
| `/implement` | `TASK-XXX` | `plan.md`, `status.md` | `src/`, `tests/`, `status.md`, `current-sprint.md` | status = `plan-ready` or `implementing` |
| `/sprint` | — | `current-sprint.md`, `tasks/*/status.md` | nothing | — |

## File Structure

After running `bmad-init`, your project looks like this:

```
my-project/
├── .claude/
│   ├── CLAUDE.md                ← Project instructions for Claude
│   └── commands/
│       ├── architect.md         ← /architect command
│       ├── implement.md         ← /implement command
│       ├── new-task.md          ← /new-task command
│       ├── plan.md              ← /plan command
│       ├── research.md          ← /research command
│       ├── review.md            ← /review command
│       └── sprint.md            ← /sprint command
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

## Worktree Parallelism

For independent tasks, use git worktrees to implement in parallel:

```bash
# Create worktrees for parallel tasks
git worktree add ../my-project-task002 -b feat/task-002
git worktree add ../my-project-task003 -b feat/task-003

# Work on task 002 in one terminal
cd ../my-project-task002
claude
# /implement TASK-002

# Work on task 003 in another terminal
cd ../my-project-task003
claude
# /implement TASK-003

# When done, merge back
cd ../my-project
git merge feat/task-002
git merge feat/task-003

# Clean up worktrees
git worktree remove ../my-project-task002
git worktree remove ../my-project-task003
```

Each worktree gets its own Claude Code session with independent context.

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

**What if `/implement` gets interrupted mid-task?**
Re-run `/implement TASK-XXX` in a new session. The resumption algorithm reads `status.md` to find the last completed phase and picks up from the next one.

## License

[MIT](LICENSE)
