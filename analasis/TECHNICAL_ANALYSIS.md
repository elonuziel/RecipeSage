# Technical Analysis — RecipeSage Feature Feasibility

## Architecture Overview

RecipeSage is an Angular (v17+) / Ionic monorepo targeting web, iOS (Capacitor), and PWA. The relevant layers are:

| Layer | Location | Notes |
|---|---|---|
| Frontend (Angular/Ionic) | `packages/frontend/src/app/` | Standalone components, tRPC client |
| API layer (tRPC) | `packages/trpc/src/procedures/` | Type-safe mutations & queries |
| Database schema | `packages/prisma/src/prisma/schema.prisma` | PostgreSQL via Prisma |

### Shopping-list data model (key fact)

`ShoppingListItem` has **no separate quantity field**. Items have a single `title: String` column. Everything — quantity, unit, and item name — lives together in that text string:

```
ShoppingListItem {
  id, userId, shoppingListId, recipeId
  title          String    // e.g. "2 cups flour" or "milk"
  completed      Boolean
  categoryTitle  String?
  createdAt, updatedAt
}
```

### Existing shopping-list files

| File | Role |
|---|---|
| `packages/frontend/src/app/pages/shopping-list-components/shopping-list/shopping-list.page.ts` | Main page; holds `items` array; calls tRPC |
| `packages/frontend/src/app/pages/shopping-list-components/shopping-list/shopping-list.page.html` | Template; renders `<shopping-list-item>` per item |
| `packages/frontend/src/app/pages/shopping-list-components/shopping-list-popover/shopping-list-popover.page.ts` | Options popover; already receives full `shoppingListItems` array; contains the `print()` method |
| `packages/frontend/src/app/pages/shopping-list-components/shopping-list-popover/shopping-list-popover.page.html` | Popover HTML; has existing "Print" `<ion-button>` |
| `packages/frontend/src/app/components/shopping-list-item/shopping-list-item.component.ts` | Per-item component; emits `completeToggle`, `recategorize`, `deleteClick` |
| `packages/frontend/src/app/components/shopping-list-item/shopping-list-item.component.html` | Item HTML template |
| `packages/frontend/src/app/services/util.service.ts` | Contains `generatePrintShoppingListURL()` — server-side print routing |
| `packages/trpc/src/procedures/shoppingLists/updateShoppingListItem.ts` | tRPC mutation; accepts `{ id, title?, completed?, categoryTitle? }` |

---

## Feature 1 — Plain-text Grocery List Export

### Minimal Change Strategy

The existing `print()` call in the popover goes to a **server-side route** (`/print/shoppingList/:id`). For a plain-text export intended for messaging/clipboard, no server changes are needed at all — it can be done entirely in the popover component with the `shoppingListItems` data already present as an `@Input()`.

**Approach (100% client-side):**

1. Add a `copyPlainText()` method to `ShoppingListPopoverPage` that:
   - Maps `shoppingListItems` to lines `item.title`
   - Joins with `\n`
   - Calls `navigator.clipboard.writeText(text)`
   - Shows a brief toast ("Copied to clipboard")
2. Add a "Copy list" `<ion-button>` to `shopping-list-popover.page.html` next to the existing Print button.

No tRPC changes. No new files. No schema changes.

### Files Affected

| File | Change |
|---|---|
| `shopping-list-popover.page.ts` | Add `copyPlainText()` method (~15 lines) |
| `shopping-list-popover.page.html` | Add one `<ion-button>` |

### Risk Assessment

- **Clipboard API availability**: `navigator.clipboard.writeText()` requires HTTPS and a user gesture. RecipeSage is a PWA served over HTTPS — this is fine. On native iOS (Capacitor) it may need the Capacitor Clipboard plugin if the Web API is unavailable, but Web API works on modern iOS Safari.
- **Completed items**: `shoppingListItems` includes both completed and uncompleted items. Filter to `!item.completed` to avoid exporting already-checked items.
- **Grouping ignored**: As specified, categories and grouping are skipped.
- **No i18n key needed** unless you want to internationalize the button label.

### Complexity: **Low**

---

## Feature 2 — Increment / Decrement Quantity Controls

### Key Constraint

Quantity is **not a separate field**. It is embedded inside the `title` string (e.g., `"2 cups flour"`, `"1/2 tsp salt"`, `"milk"`). There is no structured numeric field to increment. The only safe minimal approach is to **parse and re-serialize the title string**.

### Minimal Change Strategy

Add a simple numeric parser to the item component. The algorithm:

1. Try to parse a leading number (integer or simple fraction like `1/2`, `1 1/2`) from `this.title`.
2. If a number is found, on `+` increment it by the smallest step (1 if integer, the fractional denominator if fractional), rebuild the string.
3. If no number is found at the start, prepend `"2 "` on `+` and do nothing on `-`.
4. Call `trpcService.shoppingLists.updateShoppingListItem.mutate({ id, title: newTitle })` — the mutation already supports this.

The `ShoppingListItemComponent` already receives `id` and `title` as `@Input()` and emits events. The cleanest approach is to emit a new `titleChange` event from the item component, handled in `ShoppingListPage` which calls the tRPC mutation.

Alternatively, inject `TRPCService` directly in `ShoppingListItemComponent` (this is simpler — other services are already injected there).

**Recommended (simpler):** Inject `TRPCService` in `ShoppingListItemComponent`, and call the mutation directly, then emit the updated title to the parent so the UI stays in sync.

### Files Affected

| File | Change |
|---|---|
| `shopping-list-item.component.ts` | Inject `TRPCService`; add `increment()` / `decrement()` methods with title-parsing logic (~50 lines) |
| `shopping-list-item.component.html` | Add two `<ion-button>` slots (`+` / `-`) |
| `shopping-list.page.ts` | Optionally: add a `titleUpdate` output handler if you prefer event-driven approach |

### Risk Assessment

- **Parsing ambiguity**: Item titles are free text. A title like `"Add 2 packs of butter"` would incorrectly parse `2` as the quantity. **Mitigation**: only parse numbers at the very beginning of the string (leading digits optionally followed by a fraction).
- **Fractions**: `1/2`, `3/4`, etc. are common. The parser needs to detect the denominator to step correctly (e.g., step of `0.5` for halves).
- **Mixed numbers**: `1 1/2` is valid. The parser needs to handle whitespace between whole and fraction parts.
- **No quantity**: If the item title has no leading number, increment should prepend `"2 "` (inferring quantity was `1`). Decrement with no number is a no-op (or shows a toast).
- **Negative quantity**: Decrement when quantity is already 1 should either stop at 1 or ask to delete.
- **Optimistic UI**: For a snappy feel, update the displayed title immediately before the network call resolves, then reload on error.
- **Backend**: The existing `updateShoppingListItem` tRPC mutation already supports updating `title`. No backend changes needed.

### Complexity: **Medium**

The parsing logic for fractional quantities and edge cases adds non-trivial complexity, but the integration surface is very small.

---

## Summary Table

| Feature | New Files | Modified Files | Schema Changes | Backend Changes | Complexity |
|---|---|---|---|---|---|
| Plain-text export | 0 | 2 | None | None | Low |
| Increment/Decrement | 0 | 2–3 | None | None | Medium |
