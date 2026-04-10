# Shipwright

> "Build the whole ship."

An opinionated development methodology for Claude Code that optimises for **shipping products**, not just writing code.

## Why Shipwright

Existing Claude Code workflows optimise for *coding*. Shipwright optimises for *shipping*. Products need product thinking before the keyboard comes out, design thinking to map the right flows, and quality thinking to catch what the happy path misses. The gap between "Claude wrote the code" and "the feature shipped" is exactly where Shipwright operates.

## How It Works

Every request hits the orchestrator first. It classifies complexity and routes to the right lifecycle depth — no manual tier selection needed.

| Tier | Signal | Lifecycle |
|------|--------|-----------|
| **Rescue** | Inherited, messy, or chaotic existing codebase | Audit → Plan → Build → Ship |
| **Quick** | Single file, clear intent | Just do it. Developer role, Karpathy principles. |
| **Standard** | Multi-file, some ambiguity | Plan → Build → Ship |
| **Major** | New capability, vague requirements | Discover → Plan → Build → Ship |

## Phases

| Phase | Lead Role | Output |
|-------|-----------|--------|
| Audit | Architect + PM | Project map, health signals, triage roadmap |
| Discover | Product Manager | Problem statement, user flows, scoped MVP |
| Plan | Architect | Architecture decisions, ADRs, ordered task sequence |
| Build | Developer | Working code, TDD loop, QA companion per task |
| Ship | QA | Review findings, PR description, changelog entry |
| Status | — | In-flight work summary across all phases |

## Installation

Copy or symlink the repo into your Claude Code plugins directory:

```bash
git clone https://github.com/fradelfo/shipwright ~/.claude/plugins/shipwright
```

Then add it to your project or user config:

```json
{
  "plugins": ["~/.claude/plugins/shipwright"]
}
```

Or pass it on the command line:

```bash
claude --plugin-dir ~/.claude/plugins/shipwright
```

> **Note:** Plugin loading is a Claude Code beta feature. Check the [Claude Code docs](https://docs.anthropic.com/claude-code) for the current installation method.

## Usage

```bash
# Let the orchestrator classify and route
/sw-orchestrate "add user authentication"
/sw-orchestrate "I inherited this codebase and have no idea where to start"

# Jump directly to a phase
/sw-audit                                          # audit current directory
/sw-audit "messy Rails app I just took over"       # with context
/sw-discover "we need better onboarding"
/sw-plan "add pagination to the users endpoint"
/sw-build docs/shipwright/plan/2026-04-10-pagination.md
/sw-ship docs/shipwright/build/2026-04-10-pagination.md
/sw-status                                             # see what's in flight
/sw-status "pagination"                                # filter by topic
```

## Artifacts

Shipwright saves all artifacts into your project repo under `docs/shipwright/`. Each artifact has YAML frontmatter for traceability and links to its upstream source.

```
docs/shipwright/
├── audit/       # Audit reports — project map, health signals, triage roadmap
├── discover/    # Discovery documents — problem statements, user flows, scoped MVP
├── plan/        # Implementation plans — ADRs, task sequences, verification criteria
├── build/       # Build logs — task completion status, QA findings per task
├── ship/        # Ship artifacts — review findings, PR body, changelog entry
└── learn/       # Retrospectives — extracted insights from ship phase, searchable by discover
```

See [examples/](examples/) for representative samples of each artifact type.

## Agents

Shipwright dispatches specialised sub-agents during the Plan, Build, and Ship phases:

```
agents/
├── research/
│   ├── codebase-analyst.md    # Surveys existing patterns before planning
│   └── docs-researcher.md     # Fetches current external documentation
├── qa/
│   ├── edge-case-hunter.md    # Boundary/failure injection after each build task
│   └── exploratory-tester.md  # Adversarial flows during ship review
└── review/
    ├── correctness.md         # Logic errors, edge cases, test coverage
    ├── security.md            # OWASP-style vulnerability surface
    ├── simplicity.md          # YAGNI violations, over-engineering
    └── performance.md         # N+1, unbounded growth, blocking ops
```

## Roles

Five roles define how each phase thinks. They are passive persona definitions — not commands. Phase skills load and apply them automatically.

| Role | Lead Phase | Voice |
|------|------------|-------|
| Product Manager | Discover | Problem-first, outcome-focused |
| UX Designer | Discover | Journey-first, user-behaviour |
| Software Architect | Plan | Trade-off-first, boring-technology |
| Developer | Build | Pragmatic, surgical, test-driven |
| QA | Ship | Adversarial, release-disciplined |

## Learning Loop

Every ship phase writes a `docs/shipwright/learn/` entry — a distilled retrospective with key insights and tags. The discover phase reads these before problem framing and surfaces relevant past learnings. The loop compounds across features automatically.

## Design

See [docs/methodology.md](docs/methodology.md) for the full methodology explanation — the thinking behind the phases, roles, and design decisions.

See [docs/design.md](docs/design.md) for the original design specification.

## License

MIT
