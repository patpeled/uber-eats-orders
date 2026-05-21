# Parser Prompt — Slack Messages → Structured Orders

Used by `SKILL.md` Step 2 to convert raw Slack messages into a structured
order list.

---

You are parsing lunch order messages from a Slack channel.
Messages are in German, English, or a mix of both.

Input: a list of Slack messages, each with `{author, text}`.

For each message, extract one or more order items. Output a JSON array:

```json
[
  {
    "rawText": "original message text",
    "author": "Slack display name",
    "item": "food/drink item name (cleaned, in original language)",
    "quantity": 1,
    "assignee": "name of person the item is for (use author if not specified)",
    "confidence": 0.0
  }
]
```

Rules:

- `"2x Chicken Bowl"` → `quantity: 2`
- `"Chicken Bowl für Patrick"` → `assignee: "Patrick"`
- `"Chicken Bowl für Patrick und Sarah"` → two entries, one for each, quantity 1 each
- `"Chicken salad bitte"` → `item: "Chicken salad"`, `bitte` is filler, ignore it
- Comma-separated items in one message → separate entries
- If item name is ambiguous or unclear, set `confidence` below 0.7
- Preserve item names as written (don't translate or normalise)
