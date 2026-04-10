---
type: discover
topic: assign-task
tier: major
status: complete
date: 2026-04-10
upstream: null
---

# Discovery: Assign Task to Teammate

## Problem Statement

Team members in the task app have no way to delegate work — every task is implicitly owned by the person who created it. As teams grow beyond 2-3 people, this creates bottlenecks where one person becomes the de-facto owner of everything they created, even when others are better placed to handle it. The result: tasks stall, accountability blurs, and standups devolve into "who's actually doing this?"

## Success Criteria

- A task can be assigned to any workspace member
- The assignee receives a notification (in-app; email is out of scope for MVP)
- Task list views show the assigned avatar/name clearly
- Re-assigning or unassigning is possible without losing history
- Assignment is visible in the task's activity log

## Non-Goals

- Email notifications — adds infrastructure complexity; in-app first
- Assigning to multiple people simultaneously — introduces shared accountability ambiguity
- Assigning to people outside the workspace — permissions model not ready for that

## User Flows

### Happy Path

1. User opens a task detail view
2. User clicks the "Assignee" field (currently shows "Unassigned")
3. A dropdown shows all workspace members with avatars and names
4. User selects a teammate
5. Task updates immediately (optimistic UI); assignee field shows selected member
6. Selected teammate receives an in-app notification: "Alex assigned 'Fix login bug' to you"
7. Activity log on the task records: "Alex assigned to Jordan — Apr 10, 2026"

### Error Paths

1. **Network failure on assign**: Optimistic update reverts; toast shows "Couldn't save — try again". No notification sent.
2. **Assignee removed from workspace mid-flow**: Dropdown shows all current members at open time; if the user's session is stale, the save fails with "This user is no longer in the workspace" and the assignment is cleared.
3. **Task deleted while dropdown open**: Save returns 404; toast shows "This task no longer exists"; user is redirected to task list.

## Screen Inventory

| Screen | Purpose | Entry | Exit |
|--------|---------|-------|------|
| Task Detail | Assign/unassign; view activity log | Task list row click | Back to task list |
| Member Dropdown | Select assignee from workspace members | Clicking "Assignee" field | Select a member, or press Escape |
| Notification Feed | See incoming assignment notifications | Bell icon in nav | Dismiss or click to task |

## Feasibility Assessment

**Complexity:** Moderate

**Blockers:**
- None identified — workspace membership model already exists; notifications system has in-app primitives

**Unknowns:**
- Notification delivery guarantees: does the existing in-app notification system support background delivery, or only on-load polling? Needs spike before implementation.
- Optimistic update rollback: the current frontend has no established rollback pattern. Will need a decision (rollback vs. pessimistic update).

## Scoped MVP Feature List

| In | Out | Reason for exclusion |
|----|-----|---------------------|
| Assign to single workspace member | Assign to multiple people | Shared accountability too complex for MVP |
| In-app notification to assignee | Email notification | Requires email infrastructure work |
| Re-assign / unassign | Assignment approval workflow | Over-engineered for current team size |
| Activity log entry | Full audit export | Not requested; low value at current scale |
| Assignee visible in task list | Filter/sort by assignee | Phase 2 — need data first |

## Phase 2 Candidates

- Filter and sort task list by assignee — natural follow-on once assignment data exists
- Email notifications — once in-app is stable and users request it
- Assign during task creation — reduces friction; low effort once the component exists
- Bulk reassign — useful when a team member goes on leave
