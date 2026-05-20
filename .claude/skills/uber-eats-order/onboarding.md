# Uber Eats Order Skill — Onboarding

This procedure runs when `/uber-eats-onboard` is invoked, or automatically on the first call
to `/uber-eats-order` when `config.json` is missing or incomplete.

---

## How to Run

- **First time:** Triggered automatically by `/uber-eats-order` if `config.json` is absent.
- **Updating config:** Run `/uber-eats-onboard` explicitly at any time.
- **Partial update:** If `config.json` exists, load and display current values, then ask:
  > "Which field would you like to update? (slackChannel / restaurantUrl / deliveryAddress /
  > paymentMethodLabel / taxProfileLabel / timeWindow / all)"
  Only re-collect the chosen field(s); keep all others unchanged.

---

## Config File Location

`.claude/skills/uber-eats-order/config.json` (gitignored — never committed)

---

## Field Collection Procedure

Work through each field in order. For a partial update, skip fields not selected.

---

### 1. slackChannel

Ask:
> "What is the name of the Slack channel where lunch orders are posted?
> (e.g. `#lunch-orders`)"

After the user provides the name:
- Open `https://app.slack.com` in Chrome via `mcp__Claude_in_Chrome`.
- Navigate to the channel by name using the search bar.
- Confirm the channel loads and is accessible (not private/restricted beyond access).
- If the channel is not found, tell the user and ask them to re-enter.

Save: the channel name exactly as provided (e.g. `#lunch-orders`).

---

### 2. restaurantUrl

Ask:
> "Please paste the Uber Eats URL for the restaurant you order from.
> You can find it by opening the restaurant page on ubereats.com and copying the URL."

After the user provides the URL:
- Open the URL in Chrome via `mcp__Claude_in_Chrome`.
- Confirm the page loads as a valid Uber Eats restaurant page (check for menu items visible,
  restaurant name in page title or header).
- If it redirects to a login page, prompt the user to log in and then re-check.
- If the page is not a valid restaurant, tell the user and ask them to re-enter.

Save: the URL exactly as provided.

---

### 3. deliveryAddress

Ask:
> "What is the delivery address for orders?
> (e.g. `Jopestrasse 4, 72072 Tübingen`)"

After the user provides the address:
- On the Uber Eats restaurant page already open, locate the delivery address field
  (usually at the top of the page or in checkout).
- Enter the address and confirm Uber Eats resolves it to a valid delivery location
  (i.e., no "address not found" or "outside delivery zone" error).
- If unresolvable, tell the user and ask them to re-enter.

Save: the address string as provided.

---

### 4. paymentMethodLabel

- Open `https://www.ubereats.com/payment` in Chrome (or navigate via account menu →
  Payment).
- Read all saved payment methods listed on the page.
- Present the list to the user, explicitly excluding any Uber Cash or Uber Credits entries:
  > "I found these payment methods on your account:
  >   1. Visa •••• 4242
  >   2. Mastercard •••• 8891
  > Which should be used for orders? (enter the number)"
- Record the exact label as displayed on the page.

Save: the selected payment method label string.

---

### 5. taxProfileLabel

- Open `https://riders.uber.com/tax-profiles` in Chrome.
- Read the available tax profiles (Personal, Business, or named profiles).
- Present the list to the user:
  > "I found these tax profiles on your account:
  >   1. Personal
  >   2. Business — Acme GmbH
  > Which should be applied to orders? (enter the number)"

Save: the selected profile label string.

---

### 6. timeWindow

Show the default and ask whether to keep or override:
> "The default order window is: previous day 16:00 → current day 11:30.
> Press Enter to keep the default, or type new values as `HH:MM HH:MM` (start end)."

If the user provides new values, validate they are valid `HH:MM` strings.

Save: `{ "start": "HH:MM", "end": "HH:MM" }`.

---

## Saving Config

After all fields are collected, write/update `.claude/skills/uber-eats-order/config.json`
using the validated values. Show the user the final saved config for confirmation:

> "Config saved. Here's what will be used:
>
>   Channel:        #lunch-orders
>   Restaurant:     https://www.ubereats.com/...
>   Delivery to:    Jopestrasse 4, 72072 Tübingen
>   Payment:        Visa •••• 4242
>   Tax profile:    Business — Acme GmbH
>   Order window:   16:00 prev day → 11:30 today
>
> Run `/uber-eats-order` to place your first order, or `/uber-eats-onboard` to update any field."
