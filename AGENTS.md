# Shipwright Development Conventions

## Directory Structure

```
shipwright/
├── .claude-plugin/plugin.json    # Plugin manifest
├── CLAUDE.md                     # Cross-cutting rules (~20 lines)
├── AGENTS.md                     # This file — development conventions
├── skills/
│   ├── sw-orchestrate/SKILL.md   # Tier classification & routing
│   ├── sw-discover/SKILL.md      # PM-led discovery phase
│   ├── sw-plan/SKILL.md          # Architect-led planning phase
│   └── roles/                    # Passive persona definitions (not skills)
│       ├── product-manager.md
│       ├── ux-designer.md
│       └── software-architect.md
└── docs/
    └── design.md                 # Design specification
```

## Naming Conventions

- Skill directories: `sw-` prefix, kebab-case (e.g., `sw-orchestrate`)
- Role files: descriptive kebab-case (e.g., `product-manager.md`, not `pm.md`)
- Artifacts in user repos: `YYYY-MM-DD-<topic>.md` with YAML frontmatter
- All file names: kebab-case, no spaces, no underscores

## Skill Compliance Checklist

- [ ] YAML frontmatter has `name` and `description`
- [ ] `name` matches directory name exactly
- [ ] `description` uses "Use when [triggers]" format — never summarizes the workflow
- [ ] `description` includes natural-language trigger phrases users would say
- [ ] `allowed-tools` specifies required tools (principle of least privilege)
- [ ] Body uses imperative/infinitive form, not second person
- [ ] Body uses standard markdown headings, not XML tags
- [ ] SKILL.md under 500 lines
- [ ] References use proper markdown link syntax: `[name](./path)`, not backtick references

## Role File Convention

- Role files are plain markdown with lightweight YAML frontmatter (name, type, voice, phases)
- Roles are NOT skills — no `description` field in frontmatter
- Phase skills reference roles via markdown links: `[Product Manager](../roles/product-manager.md)`
- Never use backtick references for files you want Claude to read
- Target 80-120 lines per role — Claude already knows the domain; only specify what's unique

## Error Message Convention

When a resource cannot be loaded, use this format:
`WARNING: Could not load [resource]. Proceeding with reduced capability.`

## Cross-Referencing

- Discover-to-Plan transitions pass the explicit artifact path
- Plan artifacts include `upstream:` field in frontmatter pointing to discovery doc
- Never rely on fuzzy slug matching for cross-references
