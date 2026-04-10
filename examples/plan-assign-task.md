---
type: plan
topic: assign-task
tier: major
status: complete
date: 2026-04-10
upstream: examples/discover-assign-task.md
---

# Plan: Assign Task to Teammate

## Overview

Add the ability to assign a task to a workspace member, with an in-app notification to the assignee and an activity log entry on the task. Scoped per discovery: single assignee, in-app notifications only, no email, no bulk operations.

## Codebase Context

- **Stack:** Rails 7.2 API, React 18 frontend, PostgreSQL, ActionCable for real-time
- **Pattern:** Service objects in `app/services/`, thin controllers, React Query for data fetching
- **Tests:** RSpec (backend), Vitest + Testing Library (frontend)
- **Relevant files:**
  - `app/models/task.rb` — `belongs_to :workspace`; no assignee field yet
  - `app/models/user.rb` — `has_many :workspace_memberships`
  - `app/controllers/api/tasks_controller.rb` — standard CRUD
  - `app/services/notifications/` — `InAppNotificationService` already exists
  - `app/javascript/components/TaskDetail/` — where the UI change lands

## Architecture Decisions

### ADR: Optimistic Update Strategy

**Status:** Accepted

**Context:** The discovery phase flagged that the frontend has no established rollback pattern. Two options: optimistic updates (instant UI, rollback on failure) or pessimistic updates (wait for server confirmation).

**Options:**
- Optimistic: Better perceived performance; requires rollback logic
- Pessimistic: Simpler; noticeable lag on assign action

**Decision:** Optimistic update using React Query's `onMutate` / `onError` / `onSettled` pattern. The assign action is low-risk (no money, no irreversible state), making optimistic UX the right trade-off.

**Consequences:** Frontend must handle rollback. Test the error path explicitly.

---

### ADR: Notification Delivery

**Status:** Accepted

**Context:** Spike confirmed the existing `InAppNotificationService` uses on-load polling, not ActionCable push. Push would be better UX but requires wiring a new ActionCable channel.

**Options:**
- Polling (existing): works now, ~30s delay
- ActionCable push: real-time, requires new channel and frontend subscriber

**Decision:** Use polling for MVP. Defer ActionCable to Phase 2. The 30s delay is acceptable for assignment notifications — not a blocking workflow.

**Consequences:** Users may not see assignment notification instantly. Document this as a known limitation in the ship artifact.

## Implementation Sequence

1. **Database migration** — Add `assignee_id` foreign key to `tasks`
   - Files: `db/migrate/YYYYMMDD_add_assignee_to_tasks.rb` (create)
   - Depends on: nothing

2. **Task model** — Add `belongs_to :assignee, class_name: 'User', optional: true`
   - Files: `app/models/task.rb` (modify)
   - Depends on: task 1

3. **AssignTask service** — Encapsulate assignment logic, activity log write, and notification dispatch
   - Files: `app/services/assign_task_service.rb` (create)
   - Depends on: task 2

4. **Tasks controller** — Add `PATCH /api/tasks/:id/assign` endpoint
   - Files: `app/controllers/api/tasks_controller.rb` (modify), `config/routes.rb` (modify)
   - Depends on: task 3

5. **Assignee dropdown component** — React component: workspace member list, avatar + name
   - Files: `app/javascript/components/TaskDetail/AssigneeDropdown.tsx` (create)
   - Depends on: nothing (can parallel with tasks 1–4)

6. **TaskDetail integration** — Wire dropdown into task detail; add React Query mutation with optimistic update
   - Files: `app/javascript/components/TaskDetail/index.tsx` (modify)
   - Depends on: tasks 4, 5

7. **Notification display** — Show assignment notifications in the existing notification feed
   - Files: `app/javascript/components/NotificationFeed/index.tsx` (modify)
   - Depends on: task 4

## Verification Strategy

| Task | What proves it works | What could go wrong |
|------|---------------------|---------------------|
| 1 | `rails db:migrate` succeeds; schema shows `assignee_id` | FK constraint missing; nullable column needed for existing tasks |
| 2 | `Task.new.assignee` returns nil without error | Association name collision |
| 3 | `AssignTaskService.new(task, assignee, actor).call` writes activity log + enqueues notification | Double-notification on retry |
| 4 | `PATCH /api/tasks/1/assign` with valid params returns 200; invalid user ID returns 422 | Missing auth check on endpoint |
| 5 | Dropdown renders workspace members; keyboard navigation works | Member list not filtered to workspace |
| 6 | Assigning optimistically updates UI; network failure reverts and shows toast | Rollback leaves stale cache |
| 7 | Assignment notification appears in feed within polling interval | Notification not scoped to correct user |

## Risk Flags

- **Polling lag (30s):** Acceptable for MVP per ADR; document in ship artifact so reviewers aren't surprised
- **Existing tasks with no assignee:** Migration must set `assignee_id` nullable — confirmed in task 1
- **Activity log schema:** Verify `TaskActivity` model supports `assignee_id` metadata before task 3
