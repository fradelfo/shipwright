---
type: learn
topic: assign-task
date: 2026-04-10
upstream: examples/ship-assign-task.md
tags: [notifications, optimistic-ui, authorisation, pundit, react-query]
---

# Learning: Assign Task Feature

## Retrospective

The discovery and plan phases caught the two main risks early: the notification delivery question (polling vs. ActionCable) and the lack of an established optimistic update pattern on the frontend. Having those as explicit unknowns in the discovery doc meant the planning ADRs resolved them before implementation started, which made the build phase unusually clean. The Correctness agent finding (missing `authorize` call) was the most valuable catch — easy to miss in review, real security hole. That one finding alone justified the review step. Next time: add a "did you call `authorize`?" item to the plan's verification column for any controller endpoint task.

## Key Insights

- Any plan task that adds a controller endpoint should include "verify `authorize` call" as an explicit verification criterion — the Correctness agent caught a missing Pundit authorization that was easy to overlook.
- When the frontend has no established rollback pattern, make the decision explicit in an ADR before build starts — leaving it implicit means the developer invents a pattern mid-implementation, which is harder to review and test.
- Notification infrastructure decisions (polling vs. push) have downstream UX implications that users notice; surface the trade-off in discovery so the PM can set expectations, not just in the ADR.
