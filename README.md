# swarm skill

Orchestrate parallel multi-agent pipelines for pi — Architect → Coder → Reviewer → QA → Commit.

## Install

```bash
pi install git:github.com/alfredkam/swarm-skill
```

Then install the dependencies:

```bash
pi install git:github.com/elpapi42/pi-fork
pi install git:github.com/tmdgusya/roach-pi
pi install npm:pi-observational-memory
```

## Usage

Invoke the orchestrator:

```
swarm [task_id]
swarm [task_1, task_2]
run pipeline for [task] in [service]
```

Each task flows through a pipeline: **Architect** (plan + AC refinement) → **Coder** (implement) → **Reviewer** (code review) → **QA** (test) → **Commit**. Independent tasks run in parallel (default 4, configurable).

## Dependencies

| Package | Why |
|---------|-----|
| [pi-fork](https://github.com/elpapi42/pi-fork) | Isolated child Pi processes |
| [roach-pi](https://github.com/tmdgusya/roach-pi) | Named agent dispatch (`planner`, `worker`, `qa-reviewer`, `security-reviewer`) |
| [pi-observational-memory](https://github.com/badlogic/pi-observational-memory) | Workspace memory for stuck-detection hints and cross-pipeline learning |

## License

MIT
