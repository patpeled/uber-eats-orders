# Uber Eats Order Skill — Onboarding

This procedure runs when `/uber-eats-onboard` is invoked, or automatically on the first call
to `/uber-eats-order` when `config.json` is missing or incomplete.

---

## How to Run

- **First time:** Triggered automatically by `/uber-eats-order` if `config.json` is absent.
- **Updating config:** Run `/uber-eats-onboard` explicitly at any time.
- **Partial update:** If `config.json` exists, load and display current values, then ask:
  > "Which field would you like to update? (slackChannel / restaurantUrl / deliveryAddressLabel /
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

### 3. deliveryAddressLabel

This mirrors the payment method and tax profile pattern: pick from addresses
already saved in the user's Uber Eats account (no free-form typing, no
autocomplete handling required).

- Navigate to the saved-addresses page in Uber Eats. The reliable path is via
  the account menu: click your avatar/initials top-right → Account → Addresses.
  (Direct URLs like `ubereats.com/manage_delivery/addresses` or
  `ubereats.com/<region>/manage_delivery/addresses` work in some regions but
  may redirect — prefer the account-menu navigation for cross-region safety.)
- Read all saved addresses listed on the page.
- Present the list to the user with the label Uber Eats shows for each entry
  (e.g. `Home`, `Work`, or the address itself if no label):
  > "I found these saved addresses on your account:
  >   1. Work — Jopestrasse 4, 72072 Tübingen
  >   2. Home — Beispielstrasse 12, 70173 Stuttgart
  > Which should be used for orders? (enter the number)"
- Record the exact label as displayed on the page.

**No saved addresses on the account:** Halt and tell the user:
> "No saved addresses found. Please add one in your Uber Eats account at
> ubereats.com → Account → Addresses, then re-run /uber-eats-onboard."

Save: the selected address label string.

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

**No saved payment methods (excluding Uber Cash/Credits):** Halt and tell the user:
> "No suitable payment methods found. Please add a card or other non-credit method
> at ubereats.com/payment, then re-run /uber-eats-onboard."

Save: the selected payment method label string.

---

### 5. taxProfileLabel

Tax profile discovery has two paths — try the primary, fall back to the secondary
if the primary doesn't load the expected list (the primary URL is unverified for
German Uber accounts and may redirect).

> **Prerequisite for the secondary path:** `restaurantUrl` (Field 2) must already
> be set. If it isn't, complete Field 2 first or skip Field 5 until after.

**Primary path — global tax profiles portal:**
- Open `https://riders.uber.com/tax-profiles` in Chrome.
- If the page loads with a list of tax profiles, read them and skip to "Present
  the list" below.
- If the page redirects, 404s, or shows an empty state that doesn't match the
  user's expectation, fall back to the secondary path.

**Secondary path — discovery via Uber Eats checkout:**
1. **Pre-check existing cart state.** Open `config.restaurantUrl` and inspect the
   cart indicator. If the cart is **not empty** (regardless of restaurant), halt
   and ask the user:
   > "The tax-profile fallback path needs to temporarily add an item to your
   > cart and remove it. Your cart currently has items that will be touched.
   > Proceed and let me restore the cart afterwards, or abort? (proceed / abort)"
   On `abort`, stop. On `proceed`, snapshot the existing cart contents so they
   can be restored after discovery.
2. **Pick a placeholder item without required options** (e.g. a plain drink).
   If no such item exists on this menu (every item has required option groups),
   tell the user and ask them to pick one interactively — reuse the
   required-options handling from `SKILL.md` Step 5 to satisfy the options before
   adding.
3. **Add the placeholder** and proceed to checkout.
4. Locate the "Invoice Details" section and read the available tax profiles from
   the dropdown.
5. **Restore cart state:** remove the placeholder, then re-add any items from the
   pre-check snapshot. Confirm the cart matches the snapshot before continuing.

**Present the list (either path):**
> "I found these tax profiles on your account:
>   1. Personal
>   2. Business — Acme GmbH
> Which should be applied to orders? (enter the number)"

**No saved tax profiles exist:** Halt and direct the user:
> "No tax profiles found. Please create one at riders.uber.com/tax-profiles
> or via your Uber account settings, then re-run /uber-eats-onboard."

Save: the selected profile label string.

---

### 6. timeWindow

Show the default and ask whether to keep or override:
> "The default order window is: previous day 16:00 → current day 11:30.
> Reply `default` to keep, or paste two times separated by a space
> (start end) — for example: `16:00 11:30`."

Accept formats: two `HH:MM` strings separated by a space, comma, dash, or arrow
(e.g. `16:00 11:30`, `16:00, 11:30`, `16:00-11:30`, `16:00 → 11:30`). Reject anything
else and re-ask.

Save: `{ "start": "HH:MM", "end": "HH:MM" }`.

---

## Saving Config

After all fields are collected, write/update `.claude/skills/uber-eats-order/config.json`
using the validated values. Show the user the final saved config for confirmation:

> "Config saved. Here's what will be used:
>
>   Channel:           <slackChannel>
>   Restaurant:        <restaurantUrl>
>   Delivery address:  <deliveryAddressLabel>
>   Payment:           <paymentMethodLabel>
>   Tax profile:       <taxProfileLabel>
>   Order window:      <timeWindow.start> prev day → <timeWindow.end> today
>
> Run `/uber-eats-order` to place your first order, or `/uber-eats-onboard` to update any field."
