# Implementation Prompt — RecipeSage Shopping List Improvements

This prompt can be given directly to an AI coding assistant to implement both features. It references actual files in the repository.

---

## Context

RecipeSage is an Angular/Ionic monorepo. The shopping list UI lives in:
- `packages/frontend/src/app/pages/shopping-list-components/`
- `packages/frontend/src/app/components/shopping-list-item/`

**Critical data model fact:** `ShoppingListItem.title` is a plain text `String` that holds the entire item description including quantity (e.g., `"2 cups flour"`). There is **no separate quantity field**.

---

## Feature 1: Plain-text Copy to Clipboard

### Files to modify

1. `packages/frontend/src/app/pages/shopping-list-components/shopping-list-popover/shopping-list-popover.page.ts`
2. `packages/frontend/src/app/pages/shopping-list-components/shopping-list-popover/shopping-list-popover.page.html`

### Instructions

**Step 1 — Add `ToastController` to `shopping-list-popover.page.ts`**

The class already injects several services. Add `ToastController` from `@ionic/angular`:

```typescript
private toastCtrl = inject(ToastController);
```

**Step 2 — Add `copyPlainText()` method to `ShoppingListPopoverPage`**

Add this method to the class (e.g., after the `print()` method):

```typescript
async copyPlainText(): Promise<void> {
  const activeItems = this.shoppingListItems.filter((item) => !item.completed);
  const text = activeItems.map((item) => item.title).join('\n');

  try {
    await navigator.clipboard.writeText(text);
    const toast = await this.toastCtrl.create({
      message: 'Shopping list copied to clipboard',
      duration: 2000,
      position: 'bottom',
    });
    await toast.present();
  } catch (e) {
    // Clipboard unavailable — fallback to showing a prompt the user can manually copy
    const toast = await this.toastCtrl.create({
      message: 'Could not access clipboard',
      duration: 2000,
      color: 'danger',
    });
    await toast.present();
  }

  this.dismiss();
}
```

> Note: `this.shoppingListItems` is already available as an `@Input()` on this component. It receives the full item list from `shopping-list.page.ts` via `presentPopover()`.

**Step 3 — Add button to `shopping-list-popover.page.html`**

Add a new `<ion-button>` directly below or above the existing `print()` button (around line 98):

```html
<ion-button (click)="copyPlainText()" expand="block" size="small" class="ion-margin">
  <ion-icon name="copy-outline" slot="start"></ion-icon>
  Copy list
</ion-button>
```

**That's it for Feature 1.** No other files need to change.

---

## Feature 2: Increment / Decrement Quantity Controls

### Files to modify

1. `packages/frontend/src/app/components/shopping-list-item/shopping-list-item.component.ts`
2. `packages/frontend/src/app/components/shopping-list-item/shopping-list-item.component.html`
3. `packages/frontend/src/app/pages/shopping-list-components/shopping-list/shopping-list.page.ts` *(add tRPC call handler)*
4. `packages/frontend/src/app/pages/shopping-list-components/shopping-list/shopping-list.page.html` *(wire event)*

### Instructions

**Step 1 — Add a `titleChange` output event to `ShoppingListItemComponent`**

In `shopping-list-item.component.ts`, add a new `@Output()`:

```typescript
@Output() titleChange = new EventEmitter<string>();
```

**Step 2 — Add quantity parsing helpers to `ShoppingListItemComponent`**

Add these private methods to the class:

```typescript
/**
 * Parses a leading numeric value from the item title.
 * Handles: integers ("2"), simple fractions ("1/2"), mixed numbers ("1 1/2").
 * Returns { value: number, step: number, rest: string } or null if no leading number.
 */
private parseLeadingQuantity(title: string): { value: number; step: number; rest: string } | null {
  // Match: optional whole number, optional fraction, followed by a space and more text
  const mixedFraction = title.match(/^(\d+)\s+(\d+)\/(\d+)\s*(.*)/);
  if (mixedFraction) {
    const whole = parseInt(mixedFraction[1], 10);
    const num = parseInt(mixedFraction[2], 10);
    const den = parseInt(mixedFraction[3], 10);
    return { value: whole + num / den, step: 1 / den, rest: mixedFraction[4] };
  }

  const simpleFraction = title.match(/^(\d+)\/(\d+)\s*(.*)/);
  if (simpleFraction) {
    const num = parseInt(simpleFraction[1], 10);
    const den = parseInt(simpleFraction[2], 10);
    return { value: num / den, step: 1 / den, rest: simpleFraction[3] };
  }

  const integer = title.match(/^(\d+)\s*(.*)/);
  if (integer) {
    return { value: parseInt(integer[1], 10), step: 1, rest: integer[2] };
  }

  return null;
}

/**
 * Formats a number back to a readable string (integer, fraction, or mixed number).
 */
private formatQuantity(value: number, step: number): string {
  if (step === 1 || Number.isInteger(value)) {
    return String(Math.round(value));
  }
  const den = Math.round(1 / step);
  const whole = Math.floor(value);
  const num = Math.round((value - whole) * den);
  if (whole === 0) return `${num}/${den}`;
  if (num === 0) return String(whole);
  return `${whole} ${num}/${den}`;
}

/** Builds an updated title string after changing quantity. */
private buildNewTitle(newValue: number, step: number, rest: string): string {
  const prefix = this.formatQuantity(newValue, step);
  return rest ? `${prefix} ${rest}` : prefix;
}

increment() {
  const parsed = this.parseLeadingQuantity(this.title);
  let newTitle: string;
  if (parsed) {
    const newValue = parsed.value + parsed.step;
    newTitle = this.buildNewTitle(newValue, parsed.step, parsed.rest);
  } else {
    // No leading number: treat current quantity as 1, new quantity is 2
    newTitle = `2 ${this.title}`;
  }
  this.titleChange.emit(newTitle);
}

decrement() {
  const parsed = this.parseLeadingQuantity(this.title);
  if (!parsed || parsed.value <= parsed.step) return; // Don't go below the minimum step
  const newValue = parsed.value - parsed.step;
  const newTitle = this.buildNewTitle(newValue, parsed.step, parsed.rest);
  this.titleChange.emit(newTitle);
}
```

**Step 3 — Add +/− buttons to `shopping-list-item.component.html`**

Inside the `<ion-item>`, add two buttons in a `slot="end"` before the existing recategorize button:

```html
<ion-button slot="end" fill="clear" (click)="decrement()">
  <ion-icon slot="icon-only" icon="remove"></ion-icon>
</ion-button>
<ion-button slot="end" fill="clear" (click)="increment()">
  <ion-icon slot="icon-only" icon="add"></ion-icon>
</ion-button>
```

**Step 4 — Handle `titleChange` in `shopping-list.page.ts`**

Add this method to `ShoppingListPage`:

```typescript
async updateItemTitle(item: ShoppingListItemSummary, newTitle: string) {
  item.title = newTitle; // Optimistic update

  const response = await this.trpcService.handle(
    this.trpcService.trpc.shoppingLists.updateShoppingListItem.mutate({
      id: item.id,
      title: newTitle,
    }),
  );
  if (!response) {
    await this.loadList(); // Revert on error
    return;
  }
  if (this.reference !== response.reference) {
    this.reference = response.reference;
  }
}
```

**Step 5 — Wire the event in `shopping-list.page.html`**

Find where `<shopping-list-item>` is rendered and add the `titleChange` event binding. Look for each `<shopping-list-item>` block and add:

```html
(titleChange)="updateItemTitle(item, $event)"
```

The existing bindings already pass `[id]`, `[title]`, `[completed]`, etc. — add this output alongside them.

> `updateShoppingListItem` (singular) tRPC endpoint is at `packages/trpc/src/procedures/shoppingLists/updateShoppingListItem.ts`. It already accepts `{ id, title }` and requires no changes.

---

## Things to Watch For

- **Completed items**: the +/− buttons are rendered per item. Hide or disable them when `completed === true` if desired.
- **Minimum quantity**: the `decrement()` method already guards against going below the step value. You may also want to prevent going below `1` for integers.
- **Floating point**: use `Math.round(value * 100) / 100` if floating-point drift appears in fraction math.
- **No backend changes required** for either feature.
- **No schema migration required** for either feature.
