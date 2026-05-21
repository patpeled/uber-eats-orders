---
name: uber-eats-onboard
description: Set up or update the per-user configuration for the uber-eats-order skill (Slack channel, restaurant URL, delivery address, payment method, tax profile, time window). Re-runnable for field-level updates. Use when the user says "update my uber eats config", "change uber eats restaurant", "set up uber eats", or runs /uber-eats-onboard.
metadata:
  version: "0.1.0"
---

# Skill: uber-eats-onboard

Triggered by `/uber-eats-onboard` in Claude Code.

This skill walks the user through configuring the per-user `config.json` consumed
by the `uber-eats-order` skill. It is self-contained per the agentskills.io spec:
the full procedure lives in this skill's own `references/ONBOARDING.md`.

> **Maintainer note:** `references/ONBOARDING.md` is intentionally duplicated
> between this skill and `uber-eats-order/references/ONBOARDING.md` so each
> skill stays self-contained (the spec recommends keeping file references one
> level deep and avoiding cross-skill paths). When updating the onboarding
> procedure, change both copies.

---

## Procedure

Read [`references/ONBOARDING.md`](references/ONBOARDING.md) and follow it step
by step. The procedure handles three modes automatically:

1. **First-time setup** — no `config.json` exists; walk through every field.
2. **Field-level update** — `config.json` exists; show current values and ask
   which field(s) to update (or `all`); re-collect only the chosen fields.
3. **Repair** — some required fields are missing/empty; prompt only for those.

`config.json` is written to `../uber-eats-order/config.json` (the runtime config
location consumed by the main skill — gitignored).

After completing the procedure, suggest the user run `/uber-eats-order` to place
their next order with the updated configuration.
