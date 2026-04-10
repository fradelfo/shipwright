---
name: sw-status
description: >
  Use when the user wants to see what work is in progress, what's ready to
  continue, or what Shipwright has produced so far. Also use when the user
  says "what's in flight", "where did we leave off", "show me what's pending",
  "what have we shipped", or "status".
argument-hint: [nothing, or a topic name to filter]
allowed-tools: Read, Glob, Bash(ls *), Bash(find *)
---

# Shipwright Status

Survey `docs/shipwright/` and produce a clear picture of what's in flight, what's ready to continue, and what's complete.

## Context

If `$ARGUMENTS` is provided, filter the output to artifacts whose topic contains that string.

If `$ARGUMENTS` is empty, show all artifacts.

## How to Get There

### 1. Check for Artifacts

```bash
ls docs/shipwright/ 2>/dev/null
```

If the directory does not exist or is empty: report "No Shipwright artifacts found in this project yet. Run `/sw-orchestrate` to start." Stop here.

### 2. Collect Artifacts

Glob all markdown files across all phase directories:

```
docs/shipwright/audit/**/*.md
docs/shipwright/discover/**/*.md
docs/shipwright/design/**/*.md
docs/shipwright/plan/**/*.md
docs/shipwright/build/**/*.md
docs/shipwright/ship/**/*.md
docs/shipwright/learn/**/*.md
```

For each file found, read its YAML frontmatter: `type`, `topic`, `date`, `status`, `upstream`.

### 3. Group by Topic

Group artifacts by their `topic` field. A topic may have artifacts at multiple phases — that's a chain.

For each topic, determine:

- **Highest phase reached** — the furthest phase that has a complete artifact for this topic
- **Next step** — what command the user should run to continue

Next-step logic:

| Highest phase complete | Next step |
|------------------------|-----------|
| audit | `/sw-discover <topic>` or `/sw-plan docs/shipwright/audit/…` |
| discover | `/sw-design docs/shipwright/discover/…` or `/sw-plan docs/shipwright/discover/…` |
| design | `/sw-plan docs/shipwright/design/…` |
| plan | `/sw-build docs/shipwright/plan/…` |
| build | `/sw-ship docs/shipwright/build/…` |
| ship | Chain complete — retrospective saved to learn/ |

### 4. Classify Chains

Sort each topic chain into one of three buckets:

**In Flight** — has artifacts but the chain is not complete (no ship artifact yet). These are actionable: the user can continue right now.

**Complete** — has a ship artifact. The full lifecycle ran. Nothing pending.

**Orphaned** — has a plan or build artifact but the upstream discover/audit artifact is missing or was never created. Flag these — they may be manually started work that Shipwright doesn't have full context for.

### 5. Present Status

Output a concise status report. Use this format:

---

## Shipwright Status

### In Flight

**[topic]** — [date]
- Reached: [phase]
- Next: `[command with explicit artifact path]`

*(or "Nothing in flight." if empty)*

### Complete

**[topic]** — [date] — shipped [N] days ago

*(or "Nothing shipped yet." if empty)*

### Orphaned

**[topic]** — [date] — [which artifact exists, what's missing]

*(or omit this section entirely if no orphans)*

---

If `$ARGUMENTS` filtered the results and nothing matched: "No artifacts found for topic '[filter]'."

## Principles

- Show the next command with the explicit artifact path, not a fuzzy suggestion. The user should be able to copy and run it.
- Never show learn/ entries in the status output — they are outputs of ship, not actionable items.
- Orphaned artifacts are informational, not errors. Note them but don't alarm.
- If status is clean (nothing in flight, nothing orphaned), say so clearly — that's useful information.
