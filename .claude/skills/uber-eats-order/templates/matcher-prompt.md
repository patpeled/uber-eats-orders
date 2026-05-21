# Matcher Prompt — Parsed Orders → Menu Items

Used by `SKILL.md` Step 4 to map each parsed order item to the closest item
on the scraped restaurant menu.

---

You are matching food order requests to items on a restaurant menu.

Inputs:

- **Menu:** list of `{name, price}` from the menu scraped in Step 3.
- **Orders:** list of `{item, quantity, assignee}` from the parser in Step 2.

For each order item, find the best matching menu entry. Output a JSON array:

```json
[
  {
    "orderedItem": "as parsed",
    "assignee": "name",
    "quantity": 1,
    "menuMatch": "exact menu item name",
    "matchConfidence": 0.0,
    "alternatives": ["next best match 1", "next best match 2"]
  }
]
```

Confidence rules:

- A match is **high-confidence (≥0.85)** when the names are clearly the same item.
- A match is **low-confidence (<0.85)** when there is ambiguity (multiple similar
  items, partial name match, or the item may not be on the menu).
- If **no plausible match exists**, set `menuMatch` to `null`.

Resolution policy (used by SKILL.md Step 4):

- Always use the highest-confidence match — regardless of score. No inline user confirmation.
- Items with `menuMatch: null` are excluded from the cart and surfaced in the
  Step 7 handoff message's "items excluded" warning.
