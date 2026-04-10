---
type: audit
topic: inherited-app
tier: major
status: complete
date: 2026-04-10
upstream: null
---

# Audit: Inherited E-Commerce App

*This is a fictional example showing what `sw-audit` produces on a real messy codebase.*

*Scenario: A developer just joined a small startup and inherited a 3-year-old Rails e-commerce app. No handoff. The previous developer is gone. They run `/sw-audit "Rails e-commerce app I just took over, no idea what works"`.*

---

## Project Map

**Stack:** Ruby 3.0, Rails 6.1, PostgreSQL, Sidekiq, Stripe (v2 API), Bootstrap 4, no frontend framework

**Structure:**

```
app/
├── controllers/       # 23 controllers, mixed concerns
├── models/            # 31 models, no clear domain grouping
├── views/             # ERB, heavily duplicated layout partials
├── services/          # 4 service objects, rest inline in controllers
├── workers/           # 7 Sidekiq workers
├── helpers/           # 19 helper files, many empty
lib/
├── tasks/             # 6 rake tasks, undocumented
legacy/                # 847 files — see Debt Hotspots
spec/
├── models/            # 11 model specs, mostly empty
├── (no controller or integration specs)
```

**Entry points:** `config/routes.rb` (412 lines), `app/controllers/application_controller.rb`

**Contributors:** 3 (primary author inactive since 8 months ago), 847 commits total, 2 in last 6 months (both dependency bumps)

---

## Health Signals

| Signal | Status | Detail |
|--------|--------|--------|
| Tests | ⚠ partial | RSpec present; 11 model specs, ~15% coverage signal; no integration tests |
| Documentation | ✗ absent | README is 18 months old and describes the dev setup incorrectly; no inline docs on key models |
| Dependencies | ⚠ stale | Gemfile.lock shows Rails 6.1 (EOL), Stripe gem v5 (current is v12, API version mismatch) |
| Commit cadence | ✗ stalled | 2 commits in 6 months, both automated dependency PRs |
| Dead code | ✗ significant | `legacy/` folder: 847 files, no commits in 14 months; 19 empty helper files |
| CI / tooling | ⚠ partial | GitHub Actions present but only runs `bundle install`; no test run, no linting |

---

## Interview (3 questions asked)

**Q: The `legacy/` folder has 847 files and no commits in 14 months. Is this intentionally frozen, or abandoned mid-way?**

> "I think it's the old storefront before they rewrote it. Pretty sure nothing uses it — but I'm not 100% sure."

→ Marked `status: unknown` — needs verification before deletion.

**Q: There are two payment flows: `PaymentsController` using Stripe v2 API and `CheckoutsController` using the newer Stripe gem. Which is the live one?**

> "Oh god. I think Checkout is live but Payments might still handle subscriptions. I'm not sure."

→ Two live payment paths, possibly both active. Critical finding — marked as Priority 1.

**Q: Test coverage is very low. Is adding tests a goal for this project, or has it been intentionally skipped?**

> "Yes, it's a goal. We kept saying we'd add them later and never did."

→ Stabilisation is a goal. Tests included in roadmap.

---

## Debt Hotspots

- **`app/controllers/orders_controller.rb`** (423 lines) — business logic, payment calls, email sending, and inventory updates all inline. No service objects. The single most dangerous file in the codebase.
- **Two active payment flows** — `PaymentsController` (Stripe v2, deprecated API) and `CheckoutsController` (Stripe v9 gem). Unknown which handles subscriptions. Risk of silent payment failures.
- **`legacy/` folder** — 847 files, status unknown. If nothing references it, deleting it removes ~40% of the codebase. If something does, that's a hidden dependency.
- **Stripe gem at v5** — current is v12. Seven major versions behind. Webhook signature verification changed in v7; if this is still on v5, webhook validation may be broken.
- **`config/routes.rb` at 412 lines** — no namespacing, no concerns, everything flat. Makes it hard to understand what's actually exposed.

*Areas not inspected (file count 2,847 — shallow scan applied):* `app/views/` subtree (ERB templates), `lib/tasks/` internals, all worker implementations.

---

## Triage Roadmap

### Priority 1 — Infrastructure

**Verify and consolidate payment flows**
- Rationale: Two active payment paths with unknown ownership is an active revenue risk. Silent failures possible.
- Effort: High
- Start: `/sw-plan docs/shipwright/audit/2026-04-10-inherited-app.md`

---

### Priority 2 — Infrastructure

**Upgrade Stripe gem and migrate to current API**
- Rationale: v5 → v12 gap means webhook signature verification is likely broken. Fraud exposure.
- Effort: High (breaking changes across 7 major versions; needs a spike first)
- Start: `/sw-plan docs/shipwright/audit/2026-04-10-inherited-app.md`

---

### Priority 3 — Stabilisation

**Add integration tests for checkout and order flows**
- Rationale: Before touching payment or order code, a safety net is essential. No integration tests means every change is a gamble.
- Effort: Medium
- Start: `/sw-plan docs/shipwright/audit/2026-04-10-inherited-app.md`

---

### Priority 4 — Cleanup

**Verify and delete `legacy/` folder**
- Rationale: If nothing references it, deleting it removes ~40% of the file count and eliminates a major source of confusion. If something does reference it, that's an undocumented dependency that needs surfacing.
- Effort: Low (grep for references, then delete or document)
- Start: `/sw-plan docs/shipwright/audit/2026-04-10-inherited-app.md`

---

### Priority 5 — Cleanup

**Extract `OrdersController` business logic into service objects**
- Rationale: The 423-line controller is the highest-risk file for any future changes. Extracting it makes testing, debugging, and modification safe.
- Effort: Medium
- Start: After Priority 3 (tests first, then refactor safely)

---

### Priority 6 — Infrastructure

**Fix CI to run the test suite and a linter**
- Effort: Low
- Rationale: CI currently only runs `bundle install`. This gives false confidence — a broken test suite still passes CI.

---

*Suggested starting point:* **Priority 3 — Add integration tests for checkout and order flows.** Counterintuitively, don't fix the payment consolidation first — without tests, you can't safely verify the fix. Build the safety net before the repair.
