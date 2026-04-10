# Shipwright — The Methodology

> "Build the whole ship."

---

## The Problem Shipwright Solves

Claude Code is good at writing code. It is less good at building products.

The gap is not technical. It's structural. When you ask Claude to "add a payment system", it starts implementing immediately — picks a library, writes the models, wires up the routes. Two hours later you have working code for the wrong thing, because nobody stopped to ask: *which* payment system? For which users? Handling which edge cases? With what error behaviour?

Shipwright fills that gap. It adds the thinking that happens before and after the keyboard — product thinking, design thinking, quality thinking — and wraps it in a workflow that Claude can follow consistently.

---

## The Core Insight: Shipping vs. Coding

A feature is shipped when it:

1. Solves the right problem (not just a problem)
2. Works for the user, not just in tests
3. Can be reviewed, deployed, and maintained
4. Has been questioned adversarially before it reached production

Writing the code is step 3a of that list. Most AI coding tools optimise for step 3a. Shipwright optimises for all four.

---

## How It Works: Tiers and Phases

Not every task needs the same amount of process. "Fix the typo in the header" doesn't need a discovery doc. "Build the analytics dashboard" does. Shipwright classifies tasks by complexity and routes to the appropriate lifecycle depth.

### The Four Tiers

**Rescue** — The project is already messy or inherited. Before building anything new, understand what exists. Route: Audit → Plan → Build → Ship.

**Quick** — Single file, clear intent, <30 min. No ceremony. The Developer role applies Karpathy principles: smallest surgical change, no speculative abstractions, verify before declaring done.

**Standard** — Multi-file, some ambiguity, 1–4 hours. Plan first, then build, then review. Route: Plan → Build → Ship.

**Major** — New capability, vague requirements, >4 hours, or "I'm not sure what to build." Full lifecycle. Route: Discover → Plan → Build → Ship.

### The Phases

Each phase has a lead role — a persona that shapes how Claude thinks during that phase. Phases produce artifacts saved to `docs/shipwright/` in the user's repo. Each artifact links to its upstream source, creating a traceable chain from problem to shipped code.

**Audit** (`sw-audit`) — *Architect + PM lead.* For existing codebases. Scans what exists, asks targeted questions about what the code can't answer, produces a triage roadmap of what to fix and in what order. Hands off to Plan.

**Discover** (`sw-discover`) — *PM lead.* Frames the problem before touching the keyboard. What are we building? For whom? What does success look like? What are we explicitly NOT building? Produces a discovery doc with problem statement, user flows, scoped MVP, and non-goals.

**Plan** (`sw-plan`) — *Architect lead.* Reads the existing codebase, designs the implementation, writes ADRs for non-obvious choices, produces an ordered task sequence with verification criteria per step. Every task must have a way to prove it works before implementation begins.

**Build** (`sw-build`) — *Developer lead, QA companion.* Implements the plan task by task in a TDD loop: write the test, implement, verify it passes, QA check, mark done. QA reviews edge cases after each task. For independent tasks, the plan can flag parallel execution via worktrees.

**Ship** (`sw-ship`) — *QA lead.* Adversarial review before the PR goes up. Dispatches four parallel review agents (correctness, security, simplicity, performance) and an exploratory tester. Fixes blocking issues, generates the PR description and changelog entry, saves a retrospective to `docs/shipwright/learn/`.

---

## The Five Roles

Roles are passive definitions — not commands. You can't invoke `/shipwright pm`. You run `/sw-discover`, and the discover skill loads the PM role. This keeps roles focused: the PM thinks in problems, not code; the QA thinks adversarially, not architecturally.

| Role | Lead Phase | Voice |
|------|------------|-------|
| **Product Manager** | Discover | Problem-first. Asks "why" and "for whom" before "what". Pushes back on scope creep. |
| **UX Designer** | Discover (support) | Journey-first. Thinks in user flows, not components. |
| **Architect** | Plan | Trade-off-aware. Prefers boring technology. Every non-obvious choice gets an ADR. |
| **Developer** | Build | Karpathy-disciplined. Surgical changes. No abstractions without a concrete reason. Declares done only when the test passes. |
| **QA** | Ship | Adversarial. "Try to break this." Also owns release prep — changelog, PR description, retrospective. |

---

## The Learning Loop

Every ship phase writes a `docs/shipwright/learn/` entry — a distilled retrospective with key insights and tags derived from the topic. The discover phase reads these before problem framing, surfaces relevant past learnings, and folds them into the current discovery context.

This means Shipwright compounds. The third time you build a feature touching authentication, the discover phase surfaces what went wrong the first two times. The insights don't live in someone's memory — they live in the repo, version-controlled, readable by anyone.

---

## Key Design Decisions

### Suggests, never blocks

Shipwright is a guide, not a gate. If you want to skip discovery and build immediately, it lets you. If you want to skip the plan and go straight to code, it lets you. The methodology earns trust through results — it doesn't enforce compliance. Every transition is a suggestion with an explicit override path.

### Artifacts live in the user's repo

Discovery docs, plans, ADRs, build logs, ship reports, learnings — all saved to `docs/shipwright/` in the user's project. Version-controlled, reviewable, persistent across sessions. Shipwright itself is stateless: it produces documents, not state.

### Roles are passive, skills are active

A role defines how to think: what questions to ask, what voice to use, what outputs to produce. A skill defines what to do: which roles to load, what steps to follow, what artifact to write. This separation means updating the PM voice once propagates to every phase that uses it — discover, audit, triage — without touching any skill file.

### ADRs are inline and lightweight

No separate ADR directory, no tooling, no ceremony. When the Architect encounters a non-obvious choice during planning, it writes a four-field ADR block directly in the plan document: context, options, decision, consequences. One page maximum. If a choice is obvious, there's no ADR — that would be noise.

### QA is continuous, not a gate

The QA role doesn't wait for "build complete." During the build phase it's a companion — after each task, it asks its core questions: empty input handled? idempotent? error path visible? The edge-case-hunter agent follows for systematic depth. Issues caught here are cheap. Issues caught at ship are expensive. Issues reaching production are very expensive.

### Rescue mode (sw-audit)

Greenfield methodology assumes you know what you're building and have a codebase with followable patterns. Real projects often have neither. `sw-audit` handles the case where the first question isn't "what should we build?" but "what do we even have?" It scans inline (Architect-led), asks targeted questions the code can't answer, and produces a triage roadmap that feeds directly into the normal Plan → Build → Ship flow.

---

## What Shipwright Is Not

- **Not a project management tool.** It doesn't replace Linear, Jira, or GitHub Issues. It produces implementation artifacts — discovery docs, plans, build logs. Project tracking is out of scope.
- **Not a CI/CD system.** It generates the `gh pr create` command but doesn't execute it. It describes deployment notes but doesn't deploy. The release decision belongs to the user.
- **Not framework-specific.** The Developer and Architect roles read the existing codebase and follow its patterns. Shipwright never says "use React" or "use PostgreSQL." The methodology is about *how* to think, not *what* to use.
- **Not a replacement for judgement.** Shipwright structures the thinking; it doesn't replace it. The user is always in control — every critical decision surfaces for confirmation, and every phase can be skipped or overridden.
