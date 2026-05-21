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
5. Matches order items to menu items, asking for confirmation on ambiguous matches
6. Builds the cart with all confirmed items
7. Configures delivery address, payment method, and tax profile
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

**Skill file:** `.claude/skills/uber-eats-order/onboarding.md`

---

## Config

Your personal config is stored at `.claude/skills/uber-eats-order/config.json`.
This file is gitignored and never committed — it contains your personal Slack channel,
payment method, and delivery address.

See `config.example.json` in the same directory for the expected format.

---

## Project Structure

```
.claude/
  skills/
    uber-eats-order/
      SKILL.md            ← main skill procedure (read by Claude when running /uber-eats-order)
      onboarding.md       ← setup/update procedure (read by Claude when running /uber-eats-onboard)
      config.example.json ← template showing all required config fields
      config.json         ← your personal config (gitignored, created by onboarding)
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
