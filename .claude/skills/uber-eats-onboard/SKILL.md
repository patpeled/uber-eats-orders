---
name: uber-eats-onboard
description: Set up or update the per-user configuration for the uber-eats-order skill (Slack channel, restaurant URL, delivery address, payment method, tax profile, time window). Re-runnable for field-level updates. Use when the user says "update my uber eats config", "change uber eats restaurant", "set up uber eats", or runs /uber-eats-onboard.
version: 0.1.0
---

# Skill: uber-eats-onboard

Triggered by `/uber-eats-onboard` in Claude Code.

This skill delegates to the existing onboarding procedure used by the main
`uber-eats-order` skill for first-run setup. It is exposed as its own slash
command so the user can update configuration explicitly without having to
trigger a full order flow.

---

## Procedure

Follow the full procedure in [`../uber-eats-order/onboarding.md`](../uber-eats-order/onboarding.md).

The procedure handles three modes automatically:

1. **First-time setup** — no `config.json` exists; walk through every field.
2. **Field-level update** — `config.json` exists; show current values and ask
   which field(s) to update (or `all`); re-collect only the chosen fields.
3. **Repair** — some required fields are missing/empty; prompt only for those.

After completing the procedure, the user can run `/uber-eats-order` to place
their next order with the updated configuration.
