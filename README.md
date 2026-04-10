# Shipwright

> "Build the whole ship."

An opinionated development methodology for Claude Code that optimises for **shipping products**, not just writing code.

## What It Does

Shipwright routes your development tasks through the right level of process:

- **Quick** — Single file, clear intent. Just do it.
- **Standard** — Multi-file, some ambiguity. Plan first, then build.
- **Major** — New capability, vague requirements. Discover what to build, plan it, then build it.

## Installation

```bash
# From a plugin registry (when published)
claude plugin add shipwright

# Or from local directory
claude --plugin-dir /path/to/shipwright
```

## Usage

```bash
# Let the orchestrator classify and route
/sw-orchestrate "add user authentication"

# Jump directly to a phase
/sw-discover "we need better onboarding"
/sw-plan "add pagination to the users endpoint"
```

## Phases

| Phase | Lead Role | Output |
|-------|-----------|--------|
| Discover | Product Manager | Problem statement, user flows, scoped MVP |
| Plan | Architect | Architecture decisions, ADRs, task sequence |
| Build | Developer | Working code with tests (P2) |
| Ship | QA | PR-ready branch with changelog (P2) |

## Artifacts

Shipwright saves artifacts to your project repo:

```
docs/shipwright/
├── discover/    # Discovery documents
└── plan/        # Implementation plans with ADRs
```

## Design

See [docs/design.md](docs/design.md) for the full design specification.

## License

MIT
