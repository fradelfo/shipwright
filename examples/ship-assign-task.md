---
type: ship
topic: assign-task
tier: major
status: complete
date: 2026-04-10
upstream: examples/build-assign-task.md
---

# Ship: Assign Task to Teammate

## Review Findings

### Exploratory Testing

Tested the happy path, error paths, and adversarial flows. Highlights:

- Assigning while offline: optimistic update applies, then reverts cleanly with toast — correct.
- Assigning to yourself: allowed and works — edge case not in the plan, but reasonable.
- Rapid re-assigns (clicking different members quickly): last write wins due to React Query mutation queueing. No duplicates observed.
- Workspace with 50 members: dropdown renders without visible lag.
- Notification appears within the polling interval (~28s in testing).

**Known limitation (per ADR):** Notification is polling-based, ~30s delay. Documented below.

### Agent Findings

| Agent | Severity | Finding | Location | Resolution |
|-------|----------|---------|----------|------------|
| Correctness | High | `assign` endpoint missing `authorize` call — any authenticated user can assign tasks in any workspace | `tasks_controller.rb:87` | Fixed — added `authorize @task, :assign?` and Pundit policy |
| Correctness | Low | `AssignTaskService` activity log entry uses `Time.now` instead of `Time.current` — timezone inconsistency possible | `assign_task_service.rb:31` | Fixed — changed to `Time.current` |
| Security | Medium | Workspace member list endpoint returns full user objects including `email` — over-exposure | `api/workspace_members_controller.rb:12` | Fixed — scoped serializer to `id`, `name`, `avatar_url` only |
| Simplicity | Low | `AssigneeDropdown` has a `useMemo` on a list of <10 items — premature optimisation | `AssigneeDropdown.tsx:44` | Advisory — removed `useMemo`; not worth the complexity |
| Performance | None | | | No issues found |

### Overall Recommendation

**Ship it** — two findings fixed (Correctness/High, Security/Medium). Low findings addressed or accepted as advisory.

## PR Title

```
feat(tasks): assign task to workspace member with in-app notification
```

## PR Body

### What

Adds the ability to assign a task to any workspace member. The assignee receives an in-app notification, and the task's activity log records the assignment with actor and timestamp. Re-assigning and unassigning are both supported.

This is the first phase of task ownership. Filter/sort by assignee and email notifications are deferred to Phase 2 (see `docs/shipwright/discover/2026-04-10-assign-task.md`).

**Known limitation:** Assignment notifications use polling (~30s delay). ActionCable push is deferred to Phase 2 per ADR in the plan artifact.

### How to Test

1. Open any task in the detail view
2. Click the "Assignee" field — a dropdown of workspace members should appear
3. Select a member — the field should update immediately (optimistic)
4. Log in as the selected member — an assignment notification should appear within ~30s
5. Re-assign to a different member — previous assignee no longer shown; new assignee notified
6. Disconnect from the network, attempt to assign — UI should revert and show an error toast

### Deployment Notes

- Run `rails db:migrate` — adds `assignee_id` column to `tasks` table (nullable, no backfill needed)
- No environment variable changes
- No cache invalidation required

## Changelog Entry

```markdown
## [Unreleased] — 2026-04-10
### Added
- Tasks can now be assigned to any workspace member
- Assignee receives an in-app notification when a task is assigned or reassigned
- Task activity log records all assignment changes with actor and timestamp
```

## Retrospective

The discovery and plan phases caught the two main risks early: the notification delivery question (polling vs. ActionCable) and the lack of an established optimistic update pattern on the frontend. Having those as explicit unknowns in the discovery doc meant the planning ADRs resolved them before implementation started, which made the build phase unusually clean. The Correctness agent finding (missing `authorize` call) was the most valuable catch — easy to miss in review, real security hole. That one finding alone justified the review step. Next time: add a "did you call `authorize`?" item to the plan's verification column for any controller endpoint task.
