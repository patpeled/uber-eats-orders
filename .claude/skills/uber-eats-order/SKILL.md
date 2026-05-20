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
   Follow the full procedure in `onboarding.md`, then continue to Step 1.

3. **File exists → validate fields:**
   Load `config.json` and check that all required fields are present and non-empty:
   `slackChannel`, `restaurantUrl`, `deliveryAddress`, `paymentMethodLabel`,
   `taxProfileLabel`, `timeWindow.start`, `timeWindow.end`.

   - If one or more fields are missing/empty, list them to the user and collect only those
     fields using the matching section(s) of `onboarding.md`. Then continue to Step 1.
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
   Keep scrolling until messages older than that timestamp are visible.
5. Extract all messages within the window: capture `{author, timestamp, text}` for each.
6. Ignore system messages, bot messages, and reactions — plain user text only.
7. If no messages are found in the window, tell the user:
   > "No orders found in #channel-name between [start] and [end]. Nothing to order."
   Stop cleanly.

---

## Step 2: Parse Orders

**Goal:** Convert raw Slack messages into a structured order list.

**LLM prompt (execute as a reasoning step):**

```
You are parsing lunch order messages from a Slack channel.
Messages are in German, English, or a mix of both.

Input: a list of Slack messages, each with {author, text}.

For each message, extract one or more order items. Output a JSON array:
[
  {
    "rawText": "original message text",
    "author": "Slack display name",
    "item": "food/drink item name (cleaned, in original language)",
    "quantity": 1,
    "assignee": "name of person the item is for (use author if not specified)",
    "confidence": 0.0–1.0
  }
]

Rules:
- "2x Chicken Bowl" → quantity: 2
- "Chicken Bowl für Patrick" → assignee: "Patrick"
- "Chicken Bowl für Patrick und Sarah" → two entries, one for each, quantity 1 each
- "Chicken salad bitte" → item: "Chicken salad", bitte is filler, ignore it
- Comma-separated items in one message → separate entries
- If item name is ambiguous or unclear, set confidence below 0.7
- Preserve item names as written (don't translate or normalise)
```

After parsing, show the user a summary table:

```
Parsed orders:
  Patrick     | Chicken Bowl         | qty 1
  Sarah       | Caesar Salad         | qty 1
  (author)    | Sparkling Water      | qty 2
  ...
Proceed? (yes / fix)
```

If the user says "fix", ask what to change before continuing.

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

**LLM prompt (execute as a reasoning step):**

```
You are matching food order requests to items on a restaurant menu.

Menu: [list of {name, price} from Step 3]
Orders: [list of {item, quantity, assignee} from Step 2]

For each order item, find the best matching menu entry. Output:
[
  {
    "orderedItem": "as parsed",
    "assignee": "name",
    "quantity": N,
    "menuMatch": "exact menu item name",
    "matchConfidence": 0.0–1.0,
    "alternatives": ["next best match 1", "next best match 2"]
  }
]

A match is high-confidence (≥0.85) when the names are clearly the same item.
A match is low-confidence (<0.85) when there is ambiguity (e.g. multiple similar items,
partial name match, or the item may not be on the menu).
If no plausible match exists, set menuMatch to null.
```

**Ambiguity resolution (batch):**

Collect all low-confidence matches and any null matches, then present them all at once:

```
Some items need confirmation:

1. "Chicken salad" (Patrick) — did you mean?
   a) Chicken Caesar Salad — €12.50
   b) Thai Chicken Salad — €13.00
   c) Grilled Chicken Bowl — €11.80
   [type 1a, 1b, 1c, or skip]

2. "Sparkling Water" (Sarah) — did you mean?
   a) San Pellegrino 0.5L — €3.50
   b) Laufen Still 0.5L — €2.80
   [type 2a, 2b, or skip]
```

- User responses update the match for that item.
- Items typed "skip" are excluded from the cart with a warning at the end.
- Items left unanswered fall back to the highest-confidence alternative automatically.

---

## Step 5: Build Cart

**Goal:** Add all confirmed items to the Uber Eats cart.

**Procedure:**
1. On the restaurant page, check for an existing cart indicator (item count badge or
   "View cart" button with items).
   - If the cart is not empty, ask the user:
     > "The cart already has items. Clear it and start fresh, or append? (clear / append / abort)"
   - If abort, stop cleanly.
   - If clear, remove existing items before proceeding.
2. For each confirmed order item (in any order):
   a. Find the item on the restaurant page by name.
   b. Click it to open the item detail.
   c. If quantity > 1, set the quantity before adding.
   d. Click "Add to order" / "In den Warenkorb" (handle German UI).
   e. Confirm the item appears in the cart count.
3. After all items are added, open the cart and verify the item list and quantities match
   the order. Report any discrepancies to the user before proceeding.

---

## Step 6: Configure Checkout

**Goal:** Set delivery address, payment method, and tax profile before handing off.

**Procedure:**
1. Proceed to checkout (click "View cart" → "Checkout" or equivalent).
2. **Delivery address:** Locate the address field. If it shows the wrong address, clear it
   and enter `config.deliveryAddress`. Confirm Uber Eats resolves it without errors.
3. **Payment method:** In the payment section, scan all listed options. Select the one
   whose label matches `config.paymentMethodLabel`. Explicitly verify Uber Cash / Uber
   Credits is NOT selected — if it is, deselect it.
4. **Tax profile (Invoice Details):** Locate the "Invoice Details" or "Rechnungsdetails"
   section on the checkout page. Select the profile matching `config.taxProfileLabel`.
   If the section is not visible, note this to the user (it may not be available for all
   accounts/regions).

---

## Step 7: Handoff to User

**Goal:** Stop the skill and give the user everything needed to place the order.

1. Capture the current browser URL (the checkout page URL).
2. Read the cart summary from the page: item names, quantities, subtotal, delivery fee,
   total.
3. Output to the user:

```
✅ Order ready for review.

Items in cart:
  Patrick     | Chicken Caesar Salad  | ×1 | €12.50
  Sarah       | San Pellegrino 0.5L   | ×1 | €3.50
  ...

Subtotal: €XX.XX   Delivery: €X.XX   Total: €XX.XX

Delivery to: Jopestrasse 4, 72072 Tübingen
Payment:     Visa •••• 4242
Tax profile: Business — Acme GmbH

⚠️  Items excluded (no match found):
  - "mystery dish" (Klaus) — could not be matched to any menu item

👉 Review and place your order: [checkout URL]
```

The skill stops here. The user clicks "Place Order" themselves.
