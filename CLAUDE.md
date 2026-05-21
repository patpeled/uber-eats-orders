# Uber Eats Auto-Order — Claude Code Project

This project automates building an Uber Eats cart from lunch orders posted in a Slack channel.
It uses Chrome browser automation (no IT-gated APIs) and stops before placing the order,
so the user can review and confirm before checkout.

---

## Available Skills

### `/uber-eats-order`

**Trigger:** Run this skill when you want to place the daily lunch order.

**What it does:**
1. Checks config (runs onboarding automatically if config is missing or incomplete)
2. Reads orders from the configured Slack channel (within the configured time window)
3. Parses free-form order messages into structured items
4. Scrapes the configured restaurant menu from Uber Eats
5. Matches order items to menu items using best-guess matching (no inline confirmation — final review happens at checkout)
6. Builds the cart with all matched items; prompts the user on cross-restaurant cart conflicts, items with required options, or per-item add failures
7. Configures delivery address, payment method, and tax profile (tax profile is treated as a hard requirement)
8. Stops at checkout and hands the URL to the user

**Skill file:** `.claude/skills/uber-eats-order/SKILL.md`

**First run:** The skill auto-detects missing or incomplete config and walks you through
setup before proceeding. No manual setup needed.

---

### `/uber-eats-onboard`

**Trigger:** Run this skill to update any part of your saved configuration.

**What it does:**
- Loads your existing config and displays current values
- Asks which field(s) you want to update (or "all" to redo everything)
- Re-collects and validates only the selected fields
- Saves the updated config

**Use when:**
- Switching to a different restaurant
- Updating your payment method
- Changing the delivery address
- First-time setup (also triggered automatically by `/uber-eats-order`)

**Skill file:** `.claude/skills/uber-eats-onboard/SKILL.md`

---

## Config

Your personal config is stored at `.claude/skills/uber-eats-order/config.json`.
This file is gitignored and never committed — it contains your personal Slack channel,
payment method, and delivery address.

See `assets/config.example.json` in the same skill directory for the expected format.

---

## Project Structure

Both skills follow the [agentskills.io specification](https://agentskills.io/specification)
directory conventions (`assets/` for templates and static resources, `references/` for
on-demand documentation).

```
.claude/
  skills/
    uber-eats-order/
      SKILL.md            ← main /uber-eats-order procedure
      config.json         ← your personal config (gitignored; created by onboarding)
      assets/
        config.example.json  ← config template
        parser-prompt.md     ← Step 2 LLM prompt
        matcher-prompt.md    ← Step 4 LLM prompt
        handoff-output.md    ← Step 7 user-facing output template
      references/
        ONBOARDING.md     ← detailed setup/update procedure
    uber-eats-onboard/
      SKILL.md            ← /uber-eats-onboard entry; reads its own references/ONBOARDING.md
      references/
        ONBOARDING.md     ← intentional duplicate of the one in uber-eats-order/
                            (keeps each skill self-contained per spec)
CLAUDE.md                 ← this file
```

---

## Requirements

- Chrome must be open and logged in to both `app.slack.com` and `ubereats.com`
  before running `/uber-eats-order`. The skill will prompt you to log in if needed.
- The `mcp__Claude_in_Chrome` MCP tool must be active in your Claude Code session.
  See the Claude in Chrome MCP installation docs to enable it if it's not already
  showing in `/doctor` or your settings.
- **Uber Eats UI language:** the skill currently assumes the Uber Eats web UI is set
  to English. Switch it via Uber Eats settings (Account → Settings → Language → English)
  before running. Localized variants are planned for v1 (see TICKET-006).

## Known Limitations (v0)

- **Monday lookback gap:** the skill reads messages from `timeWindow.start` on the
  *previous calendar day* to `timeWindow.end` on the *current day*. When run on a
  Monday, "previous day" = Sunday, and any orders posted on Friday afternoon (before
  the 16:00 window start) will be missed. For now, handle Mondays manually or
  temporarily widen the window via `/uber-eats-onboard`. A multi-day lookback is
  planned for v1.
