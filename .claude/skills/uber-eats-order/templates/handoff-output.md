# Handoff Output Template — Step 7

Used by `SKILL.md` Step 7 to surface the prepared cart to the user before they
manually place the order. Substitute the placeholders with values scraped from
the cart page (currency symbol included — do not hardcode any currency).

---

```
✅ Order ready for review.

Items in cart:
  <assignee>     | <menu item name>      | ×<qty> | <price-as-rendered-by-Uber>
  <assignee>     | <menu item name>      | ×<qty> | <price-as-rendered-by-Uber>
  ...

Subtotal: <subtotal>   Delivery: <delivery fee>   Total: <total>

Delivery to: <deliveryAddressLabel> (<resolved address shown by Uber>)
Payment:     <paymentMethodLabel>
Tax profile: <taxProfileLabel>

⚠️  Items excluded (no match found):
  - "<original order text>" (<assignee>) — could not be matched to any menu item

👉 Review and place your order: <checkout URL>
```

Notes:

- Omit the "⚠️ Items excluded" block entirely if no items were excluded.
- Display whatever currency symbol Uber renders on the cart page; do not
  substitute a hardcoded `€` or `$`.
- After printing this message, the skill stops. The user clicks "Place Order"
  themselves on the Uber Eats page.
