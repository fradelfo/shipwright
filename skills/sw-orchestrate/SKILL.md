---
name: sw-orchestrate
description: >
  Use when the user describes a development task, feature request, or
  improvement, or explicitly invokes /sw-orchestrate. Also use when the
  user is unsure what to build, asks "where do I start", or needs help
  deciding how much planning a task requires.
argument-hint: [what to build or improve]
allowed-tools: Read, Glob, Bash(ls *), Bash(mkdir *)
---

# Shipwright Orchestrator

Classify task complexity from the user's description and route to the appropriate Shipwright phase.

## Tier Classification

Read the user's `$ARGUMENTS` description. Apply the following heuristics to determine the task tier:

| Signal | Tier |
|--------|------|
| User explicitly states scope ("quick fix", "small change", "one-liner") | Quick |
| User explicitly states scope ("big feature", "major refactor", "not sure yet") | Major |
| Request references a specific file and a specific change | Quick |
| Request mentions multiple systems, components, or cross-cutting concerns | Standard or Major |
| Request is vague about requirements ("better", "improve", "add support for", "not sure") | Major |
| Request describes a single well-scoped feature or enhancement | Standard |
| Ambiguous or unclear scope | Standard (default) |

Additional rules:

- Always trust the user's own assessment of scope when explicitly stated.
- When multiple signals conflict, prefer the higher tier.
- A request that names one file but describes an unclear change is Standard, not Quick.

## Announce Classification

After classifying, announce the result with reasoning. Always offer an override.

Format the announcement as:

> This looks like a **[tier]** task — [one-sentence reason]. I'd recommend [action]. Want to proceed, or adjust?

Where `[action]` maps to:

- **Quick:** "helping with it directly"
- **Standard:** "running sw-plan to create an implementation plan"
- **Major:** "running sw-discover to explore requirements first"

Accept override phrases from the user:

- "just do it" / "skip planning" — switch to Quick
- "let's think about this" / "let's discover first" — switch to Major
- "plan it" / "make a plan" — switch to Standard

Wait for the user to confirm or override before routing.

## Routing

After classification is confirmed, route to the appropriate phase.

### Quick Tier

Say: "This looks like a quick fix. I'll help with it directly."

Help the user immediately. Apply these principles:

- Make the smallest surgical change that solves the problem.
- Keep it simple — prefer the obvious solution over the clever one.
- Stay goal-driven — solve what was asked, nothing more.
- No phase ceremony, no plan documents, no discovery artifacts.

### Standard Tier

Say: "Invoking sw-plan for: [user's description]"

Invoke the `sw-plan` skill, passing the user's original description as the argument.

### Major Tier

Say: "Invoking sw-discover for: [user's description]"

Invoke the `sw-discover` skill, passing the user's original description as the argument.

## Principles

- Suggest, never block. If the user wants to skip to build, let them.
- Default to Standard when ambiguous — it is the safe middle ground.
- Quick tier is useful, not a dead end. Provide direct, competent help.
- Use directive language for routing ("Invoke the sw-discover skill") not suggestion language ("Maybe we should consider running discovery").
- Respect the user's time. Classification and announcement should take seconds, not paragraphs.
