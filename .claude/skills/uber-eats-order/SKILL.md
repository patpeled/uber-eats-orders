---
name: uber-eats-order
description: Read lunch orders from a configured Slack channel via Chrome browser automation, parse free-form German/English messages into structured items, match them to a configured Uber Eats restaurant's menu using best-guess matching, build a cart, configure delivery/payment/tax, and halt at checkout for manual confirmation. Use when the user says "place lunch order", "uber eats order", "order lunch", or runs /uber-eats-order.
metadata:
  version: "0.1.0"
---

# Skill: uber-eats-order

Triggered by `/uber-eats-order` in Claude Code.

Reads lunch orders from a Slack channel, maps them to Uber Eats menu items, builds a cart
at the configured restaurant, configures delivery/payment/tax, then halts at checkout for
the user to place the order manually.

---

## Step 0: Config Check (always run first)

1. Check whether `.claude/skills/uber-eats-order/config.json` exists.

2. **File missing → full onboarding:**
   Tell the user:
   > "No config found. Let's set up the skill before placing your first order."
   Follow the full procedure in `references/ONBOARDING.md`, then continue to Step 1.

3. **File exists → validate fields:**
   Load `config.json` and check that all required fields are present and non-empty:
   `slackChannel`, `restaurantUrl`, `deliveryAddressLabel`, `paymentMethodLabel`,
   `taxProfileLabel`, `timeWindow.start`, `timeWindow.end`.

   - If one or more fields are missing/empty, list them to the user and collect only those
     fields using the matching section(s) of `references/ONBOARDING.md`. Then continue to Step 1.
   - If all fields are present, proceed directly to Step 1.

---

## Step 1: Read Slack Orders

**Goal:** Collect all order messages from the configured channel within the time window.

**Time window:**
- Start: `timeWindow.start` (HH:MM) on the *previous calendar day*
- End: `timeWindow.end` (HH:MM) on the *current calendar day*

**Procedure:**
1. Open `https://app.slack.com` in Chrome via `mcp__Claude_in_Chrome__navigate`.
2. Detect whether the user is logged in:
   - If redirected to a login/SSO page, pause and tell the user:
     > "Please log into Slack in the browser window, then let me know when you're ready."
   - Wait for user confirmation before continuing.
3. Navigate to the configured channel using Slack's search or sidebar.
4. Scroll up to load messages going back to `timeWindow.start` on the previous day.
   Keep scrolling until messages older than that timestamp are visible. After
   scrolling appears complete, wait 1–2 seconds and re-check that the oldest
   visible message timestamp is genuinely before the window start. If Slack is
   still loading older messages, retry the scroll once before concluding.
5. Extract all messages within the window: capture `{author, timestamp, text}` for each.
6. Ignore system/join/leave/topic-change messages and pure emoji reactions. Do NOT
   skip messages from Slack Workflow output or other integration bots that may carry
   real order text — if a "bot" message contains plausible order content, include it.
7. If no messages are found in the window, tell the user:
   > "No orders found in #channel-name between [start] and [end]. Nothing to order."
   Stop cleanly.

---

## Step 2: Parse Orders

**Goal:** Convert raw Slack messages into a structured order list.

Use the prompt in [`assets/parser-prompt.md`](assets/parser-prompt.md) as
the system instruction for an LLM reasoning step. Feed in the list of messages
collected in Step 1 (`[{author, text}]`) and capture the JSON array returned.

After parsing, proceed directly to Step 3 (no inline user confirmation —
the user reviews the cart at the final handoff in Step 7).

---

## Step 3: Scrape Restaurant Menu

**Goal:** Get the current menu from the configured Uber Eats restaurant page.

**Procedure:**
1. Open `config.restaurantUrl` in Chrome via `mcp__Claude_in_Chrome__navigate`.
2. Detect whether the user is logged in to Uber Eats:
   - If redirected to login, pause and prompt:
     > "Please log into Uber Eats in the browser window, then let me know when ready."
   - Wait for user confirmation.
3. Confirm the page is the expected restaurant: check page title / restaurant name header
   matches what the URL implies. If it looks wrong, warn the user and stop.
4. Extract menu items from the page. For each visible item capture:
   `{name, price, description (if visible)}`.
5. Scroll through all menu sections to ensure full coverage.

---

## Step 4: Match Orders to Menu Items

**Goal:** Map each parsed order item to the closest item on the restaurant menu.

Use the prompt in [`assets/matcher-prompt.md`](assets/matcher-prompt.md)
as the system instruction for an LLM reasoning step. Feed in:

- the scraped menu from Step 3 (`[{name, price}]`)
- the parsed orders from Step 2 (`[{item, quantity, assignee}]`)

Capture the JSON array returned.

**Best-guess resolution (no inline confirmation):**

- For every order item, always use the highest-confidence match — regardless of
  whether the score is above or below any threshold. Do not ask the user
  mid-flow.
- For items where `menuMatch` is `null` (no plausible match at all), exclude
  them from the cart and add them to the `excludedItems` list to surface in
  Step 7's handoff message.
- The user reviews and corrects (if needed) at the final checkout in Step 7
  before placing the order.

---

## Step 5: Build Cart

**Goal:** Add all confirmed items to the Uber Eats cart.

**Procedure:**
1. On the restaurant page, check for an existing cart indicator (item count badge or
   "View cart" button with items).
   - **Same restaurant:** if the cart belongs to the configured `restaurantUrl`, ask:
     > "The cart already has items from this restaurant. Clear and start fresh, append, or abort? (clear / append / abort)"
   - **Different restaurant:** if the cart belongs to a different restaurant (Uber Eats only allows one restaurant per cart), only `clear` and `abort` are valid. Ask:
     > "The cart has items from a different restaurant. Clearing is required to proceed. Clear and continue, or abort? (clear / abort)"
   - On `abort`, stop cleanly.
   - On `clear`, remove existing items before proceeding.
2. For each confirmed order item (in any order):
   a. Find the item on the restaurant page by name.
   b. Click it to open the item detail.
   c. If quantity > 1, set the quantity before adding.
   d. Check whether the "Add to order" button is enabled.
      - **Disabled:** the item has required option groups (size, sides, drink choice,
        etc.). List the required option groups to the user, present the available
        choices for each, ask the user to pick, apply the selections, then continue.
      - **Enabled:** click "Add to order".
   e. Confirm the item appears in the cart count.
   f. **On failure** (item sold out, page error, button still disabled after retries,
      or any unexpected state), ask the user how to proceed:
      > "Couldn't add `<item>` (`<reason>`). Skip it / retry / abort?"
      Continue per the user's choice. If the user aborts, stop cleanly and report
      the partial state.
3. After all items are added, open the cart and verify the item list and quantities match
   the order. Report any discrepancies to the user before proceeding.

---

## Step 6: Configure Checkout

**Goal:** Set delivery address, payment method, and tax profile before handing off.

**Procedure:**
1. Proceed to checkout (click "View cart" → "Checkout" or equivalent).
2. **Delivery address:** Open the address selector at checkout (usually clicking the
   currently-shown address opens a dropdown of the user's saved addresses). Select the
   entry whose label matches `config.deliveryAddressLabel`. Confirm the selection took
   effect — the checkout should now show that address. Do NOT type an address; only
   pick from the saved-address dropdown.
3. **Payment method:** In the payment section, scan all listed options. Select the one
   whose label matches `config.paymentMethodLabel`. Explicitly verify Uber Cash / Uber
   Credits is NOT selected — if it is, deselect it.
4. **Tax profile (Invoice Details):** Locate the "Invoice Details" section on the
   checkout page (English UI assumed — see CLAUDE.md requirements). Select the profile
   matching `config.taxProfileLabel`.
   - **If the Invoice Details section is not visible**, treat this as a hard error
     (the user configured a tax profile and the order would otherwise lack it). Halt
     and ask the user explicitly:
     > "The Invoice Details section isn't available on this checkout page, so the
     > configured tax profile (`<taxProfileLabel>`) can't be applied. Proceed without
     > a tax profile, or abort? (proceed / abort)"
     - On `abort`, stop cleanly without proceeding to handoff.
     - On `proceed`, continue and surface the missing-tax-profile warning in the
       Step 7 handoff.
   - **If the section is visible but the configured profile is not in the list**,
     halt with the same hard-error prompt and let the user decide.

---

## Step 7: Handoff to User

**Goal:** Stop the skill and give the user everything needed to place the order.

Render the message using the template in
[`assets/handoff-output.md`](assets/handoff-output.md). The template self-documents
all placeholders and substitution rules; in summary you'll need:

- The current browser URL (checkout page) and the cart summary scraped from the page
  (item names, quantities, prices in whatever currency Uber displays — do not
  hardcode a symbol).
- The configured labels (`deliveryAddressLabel`, `paymentMethodLabel`,
  `taxProfileLabel`) and the resolved address Uber renders next to the saved
  address label.
- The `excludedItems` list built in Step 4 (omit the block entirely if empty).

The skill stops after rendering. The user clicks "Place Order" themselves.
