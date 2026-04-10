# Examples

These are representative Shipwright artifacts from a fictional project — a task management app adding an "assign task to teammate" feature. They show what each phase produces and how the chain links together.

| File | Phase | What it shows |
|------|-------|---------------|
| [discover-assign-task.md](discover-assign-task.md) | Discover | Problem statement, user flows, feasibility, scoped MVP |
| [plan-assign-task.md](plan-assign-task.md) | Plan | ADRs, implementation sequence, verification strategy |
| [build-assign-task.md](build-assign-task.md) | Build | Task completion log, QA findings per task |
| [ship-assign-task.md](ship-assign-task.md) | Ship | Review findings, PR body, changelog entry, retrospective |
| [learn-assign-task.md](learn-assign-task.md) | Learn | Extracted insights for future discover phases |

## Reading Order

Start with `discover-assign-task.md` → `plan-assign-task.md` → `build-assign-task.md` → `ship-assign-task.md` → `learn-assign-task.md`.

Each artifact's `upstream:` frontmatter field points to its predecessor.
