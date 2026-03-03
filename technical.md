# Technical Assessment: Plain-Text Grocery List Export

## Is it possible?
Yes, it is entirely possible and straightforward to implement a plain-text export of the grocery (shopping) list.

## Current Setup
Currently, RecipeSage handles shopping list exports via a "Print" button in the `ShoppingListPopoverPage` frontend. When clicked, it opens a URL (`/print/shoppingList/:id`) that hits an Express backend endpoint (`printShoppingList.ts`). This endpoint fetches the list from Prisma, translates category names, groups items according to user preferences, and renders an HTML page (`shoppinglist-default.ejs` or similar) relying on browser-native print-to-PDF functionality.

## How to Implement Plain-Text Export
There are two primary approaches to achieve this:

### Approach 1: Client-Side (Frontend Only) Export
The frontend (`shopping-list.page.ts`) already fetches the full list of `shoppingListItems` and the user's preferences.
1. **Add UI Button**: In `packages/frontend/src/app/pages/shopping-list-components/shopping-list-popover/shopping-list-popover.page.html`, add a new button (e.g., "Export as Text" or "Copy to Clipboard").
2. **Format Logic**: Reuse the `getShoppingListItemGroupings` utility function already present in the frontend to group and categorize the items. 
3. **String Builder**: Iterate over the grouped items and build a plain text string. For example:
   ```text
   RecipeSage Shopping List: [List Title]

   -- Produce --
   [ ] 2x Apples
   [ ] 1 bunch Bananas

   -- Dairy --
   [ ] Milk
   ```
4. **Trigger Export**: Use modern Web APIs to export the built string:
   - File Download: Create a `Blob` of type `text/plain`, attach it to an invisible `<a>` element, and trigger a programmatic click to download `shopping-list.txt`.
   - Copy to Clipboard: Use `navigator.clipboard.writeText(...)`.

### Approach 2: Server-Side (Backend) Export
Mirroring the existing `/print/shoppingList/:id` route.
1. **New Express Endpoint**: Add a new route in `packages/express/src/routes/export/` (e.g., `exportShoppingListText.ts`).
2. **Data Processing**: Reuse the logic from `printShoppingList.ts` to fetch items from Prisma, handle i18n category translations, and group them.
3. **Response Formatting**: Instead of `res.render(...)`, format the grouped items into a plain text string.
4. **Headers**: Set the HTTP headers to download a file:
   ```typescript
   res.setHeader('Content-disposition', 'attachment; filename="shopping-list.txt"');
   res.type('text/plain');
   res.send(formattedString);
   ```
5. **Frontend Update**: Add a button on the Popover that redirects the user to this new endpoint, triggering the download automatically.

## Complications / Edge Cases
1. **Localization (i18n)**: RecipeSage translates category names (e.g., "produce", "dairy"). The plain text export must fetch and resolve these translation keys (`translate.instant`) just as the frontend currently does.
2. **User Preferences**: The user can toggle options like `groupSimilar`, `groupCategories`, and `sortBy`. The export feature should respect these settings, meaning they need to be passed into the grouping utility correctly.
3. **Platform Differences (if copy to clipboard)**: If you choose to do a "Copy to Clipboard" approach on the frontend, be aware that Capacitor web-views (iOS/Android) behave slightly differently than standard desktop browsers. The Capacitor `Clipboard` API (`@capacitor/clipboard`) might be needed instead of the standard `navigator.clipboard`.
4. **Line Endings**: Generating plain text files should ideally use standard `\r\n` (CRLF) or `\n` line endings that play nicely with text editors on different OSes, though standard `\n` is usually fine for sharing via WhatsApp/Email.

## Recommendation
**Approach 1 (Client-Side "Copy to Clipboard" or "Share")** is highly recommended. It avoids hitting the backend with extra requests, directly uses the data already loaded on the page, and allows mobile users to easily paste the list into WhatsApp or Notes apps. For mobile deployments, injecting Capacitor's `Share` plugin (`@capacitor/share`) is the most native-feeling way to share plain text.

---

# Follow-up: Increase/Decrease Quantity to Nearest Whole Number

## Is it possible?
Yes, it is definitely possible. 

## How Shopping List Items Store Quantities
Currently, `ShoppingListItem` records in the database do **not** have separate columns for quantity (e.g., `1`), unit (`cup`), and ingredient name (`flour`). The entire text is stored as a single `title` string (e.g., `"1 1/2 cups flour"`). 

## How Quantity Manipulation Can Work
Despite being a single string, RecipeSage already contains an advanced ingredient parsing engine located in `packages/util/shared/src/parsers.ts`. This engine is currently used to scale recipes (e.g., doubling a recipe).
1. **The Parsing Logic**: The app uses Regular Expressions (`measurementRegexp`) to identify the fraction or decimal portion of the string, and a library called `fraction.js` to parse substrings like `"1 1/2"` into workable mathematical fractions.
2. **Rounding Logic**: A developer can adapt the existing `parseIngredients(ingredients, scale)` logic to instead find the extracted `FractionJS` object, and apply a rounding function (like `Math.ceil()` for increasing to the nearest whole number, or `Math.floor()` for decreasing).
3. **Updating the List**: 
   - Add "+/-" buttons or "Round Up" / "Round Down" buttons to the `ShoppingListItemComponent` UI.
   - When clicked, the frontend runs this string manipulation, updates the text (e.g. from `"1 1/2 cups flour"` to `"2 cups flour"`), and sends a standard update request to the backend (`updateShoppingListItems.mutate`) to save the new string.

## Complications
- **Ranges**: Some ingredients might be written as `"1 - 2 cups flour"`. The parser handles this, but the logic would need to decide whether to round both numbers or handle it gracefully.
- **Missing Units/Quantities**: If the user just adds "Apples" without a number, the rounding function should detect that there is no measurement and do nothing.
