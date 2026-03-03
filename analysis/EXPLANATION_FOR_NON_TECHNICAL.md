# Can We Add These Features? — Plain Language Guide

## What Are We Adding?

Two small improvements to the shopping list in RecipeSage:

1. **"Copy to clipboard" button** — lets you copy your whole shopping list as plain text so you can paste it into WhatsApp, a text message, or anywhere else.
2. **+ / − buttons on each item** — lets you quickly increase or decrease the quantity of an item (e.g., change "2 cups flour" to "3 cups flour" with one tap).

---

## Feature 1 — Copy Shopping List as Plain Text

### What would it look like?

A new button in the shopping list options menu (same place as the current "Print" button) labeled something like **"Copy list"**. Tapping it copies all the items to your clipboard as simple text, one item per line:

```
2 cups flour
1 tsp salt
milk
3 apples
```

### Is it doable?

**Yes.** This is the simplest possible feature.

- No changes to the server or database are needed.
- The button just reads the items already loaded on screen and copies them.
- Estimated effort: **1–2 hours**, touching only **2 files**.

### Any risks?

Essentially none. The clipboard feature works on all modern browsers and phones. The only minor consideration is filtering out already-checked items (so you don't copy things you already have).

---

## Feature 2 — Increment / Decrement Quantity Buttons

### What would it look like?

Small **+** and **−** buttons next to each shopping list item. Tapping **+** increases the quantity by one step (e.g., `1 cup` → `2 cups`, or `1/2 tsp` → `1 tsp`). Tapping **−** decreases it.

### Is it doable?

**Yes**, but with a flag: there is a complication.

RecipeSage stores item quantities as part of the item name in a single text field. For example, `"2 cups flour"` is stored as the string `"2 cups flour"` — there is no separate "quantity" field in the database.

This means the + / − buttons have to be smart enough to **read the number from the text**, change it, and write the updated text back. This works fine for the common case (`"2 cups flour"` → `"3 cups flour"`), but requires careful handling of:
- Fractions (`1/2`, `3/4`, `1 1/2`)
- Items with no number at the start (like just `"milk"`)
- Making sure the number is at the beginning of the text and not elsewhere

No database changes are needed, and the existing "update item" API already supports updating item text.

- Estimated effort: **half a day**, touching **2–3 files**.

### Any risks?

Low-to-medium. The text parsing can get tricky at edge cases (weird formats, fractions, items without numbers). The logic should be tested with a range of realistic item titles. The underlying mechanism (updating the item text) is already in place and safe.

---

## Summary

| Feature | Feasible? | Effort | Risk |
|---|---|---|---|
| Copy as plain text | ✅ Yes | ~1–2 hours | Very low |
| +/− quantity buttons | ✅ Yes | ~half a day | Low-medium (text parsing) |

Both features are **safe to implement**, require **no database migrations**, and do **not affect anything else in the app**.
