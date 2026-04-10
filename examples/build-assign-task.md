---
type: build
topic: assign-task
tier: major
status: complete
date: 2026-04-10
upstream: examples/plan-assign-task.md
---

# Build: Assign Task to Teammate

## Tasks

| # | Description | Status | Notes |
|---|-------------|--------|-------|
| 1 | Database migration — add `assignee_id` to tasks | ✓ pass | |
| 2 | Task model — `belongs_to :assignee` | ✓ pass | |
| 3 | AssignTask service — assignment, activity log, notification | ✓ pass | QA: Medium finding noted (see below) |
| 4 | Tasks controller — `PATCH /api/tasks/:id/assign` endpoint | ✓ pass | |
| 5 | AssigneeDropdown component | ✓ pass | |
| 6 | TaskDetail integration — optimistic update mutation | ✓ pass | QA: rollback path tested explicitly |
| 7 | Notification feed display | ✓ pass | |

## Modified Files

- `db/migrate/20260410120000_add_assignee_to_tasks.rb`
- `app/models/task.rb`
- `app/services/assign_task_service.rb`
- `app/controllers/api/tasks_controller.rb`
- `config/routes.rb`
- `app/javascript/components/TaskDetail/AssigneeDropdown.tsx`
- `app/javascript/components/TaskDetail/index.tsx`
- `app/javascript/components/NotificationFeed/index.tsx`
- `spec/services/assign_task_service_spec.rb`
- `spec/requests/api/tasks_spec.rb`
- `spec/javascript/components/TaskDetail/AssigneeDropdown.test.tsx`

## QA Findings (Build Phase)

| Severity | Finding | Addressed |
|----------|---------|-----------|
| Medium | `AssignTaskService` does not guard against re-assigning to the same user — idempotency not guaranteed; calling twice sends two notifications | Yes — added early return when `task.assignee_id == assignee.id`; added idempotency spec |
| Low | Dropdown does not show current assignee as pre-selected on open | Advisory — noted for Phase 2 polish; does not affect core function |
| Low | Edge-case hunter: empty workspace (no other members) — dropdown renders empty list with no message | Advisory — added "No other members" empty state to dropdown |
