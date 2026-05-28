## 2026-05-27 self-improving-agent association

- Associated skill: `/Users/liushan/.agents/skills/self-improving-agent/SKILL.md`
- Project: `/Users/liushan/Documents/zkw/mini_schedule/pds`
- Project-local memory: `/Users/liushan/Documents/zkw/mini_schedule/pds/.learnings`
- Rule: run skill-related work from `/pds` or its child paths so the hook helper can discover this `.learnings` directory by walking parent directories.

## 2026-05-28 PDS implementation workflow

- Added `CLAUDE.md` as the persistent agent instruction file for `/pds`.
- Added `PROGRESS.md` as the implementation ledger for Now / Next / Later delivery.
- Pattern: use `/pds` as source of truth, then execute one vertical batch at a time across `/backend` and `/web`.

## 2026-05-28 Batch 1 public signup order

- Public signup order endpoint is `POST /api/v1/public/signup/orders`.
- Batch 1 creates a `pending` Brand, an active BrandUser owner placeholder, and a `pending_payment` SaaSPlanOrder.
- The SaaS order amount must be derived inside the backend transaction from the active SaaSPlan monthly/yearly price; frontend amount is never accepted.
- SMS remains mock-first in development. Production without a real provider must not silently allow public signup.
