# ScrollPosition

Mendix JavaScript actions for saving and restoring scroll positions in data views, data grids, and virtual scrolling containers.

## Overview

This module provides reusable JavaScript actions that work together to preserve scroll position when navigating between pages in Mendix applications.

### Standard Scroll Position Actions

For regular scrollable containers (DataGrid2, Dataviews):

- **JS_ScrollPosition_Save** - Saves the current scroll position to `localStorage`
- **JS_ScrollPosition_Restore** - Restores a previously saved scroll position
- **JS_ScrollPosition_RestoreOrFind** - Restores a saved scroll position and scrolls to the row of the given identifier when there is no saved position, or when the saved one doesn't show that row (see [Fallback: scroll to identifier](#fallback-scroll-to-identifier))

### Virtual Scrolling Actions

For virtual scrolling / infinite-load containers (e.g., VirtualListView, infinite-loading grids):

- **JS_VirtualScrollPosition_Save** - Saves scroll position for virtual scrolling containers to `localStorage`
- **JS_VirtualScrollPosition_Restore** - Restores scroll position with support for infinite-loading behavior

## Use Case

When users navigate from a list page (e.g., a data grid or data view with many items) to a detail page and return, they expect to see the same position they left off. These actions solve this by:

1. Saving the scroll position before navigating away
2. Restoring it when returning to the page

For **virtual scrolling containers** (infinite-load), the VirtualScrolling variants handle the additional complexity of dynamically-loaded content.

## Usage

### Saving Scroll Position

Call the appropriate **Save** action **before** navigating away from a page with scrollable content:

**Nanoflow:**

```Text
User clicks "Open Page" button -> Call JS_ScrollPosition_Save or JS_VirtualScrollPosition_Save (Pass the parameters) -> Show page
```

**Parameters (both actions):**

- `key` (string, required) - Unique identifier for this page/context. Use different keys for different pages.
- `querySelector` (string, required) - CSS selector for the scrollable container or its anchor element.

**Common querySelectors:**

| Container Type | Selector |
|----------------|----------|
| DataGrid2 | `".widget-datagrid-content"` or `".widget-datagrid-grid-body"` |
| Mendix Dataview | `".mx-dataview-content"` |

### Restoring Scroll Position

Call the appropriate **Restore** action **after** showing the page:

**Nanoflow:**

```Text
Return to List/DataGrid2 Page -> Show page -> Call JS_ScrollPosition_Restore or JS_VirtualScrollPosition_Restore (Pass the parameters)
```

**Events widget:**

The [Events widget](https://marketplace.mendix.com/link/component/224259) can also be used.
Create a nanoflow and add the appropriate restore action to the nanoflow and fill in the `key` and `querySelector` parameters.
Note that the Events widget might require a delay of 100 milliseconds or more depending on how long it takes for the page to load.

**How it works (Standard):**

- Reads `scrollPos_<key>` from `localStorage`
- Polls every 100ms (up to **60 attempts / 6 seconds**) for the scrollable element (the `querySelector` match or one of its children/grandchildren) to appear in DOM
- If the saved position isn't reachable yet (e.g. a Data Grid 2 with virtual scrolling has only rendered its first rows), scrolls to the bottom to load more rows and retries
- Once reachable, sets `scrollTop` to the saved value
- Removes the saved value from `localStorage` (one-shot consumption)
- If no scrollable container appears within 6 seconds, silently abandons

**How it works (Virtual Scrolling):**

- Same storage mechanism (`scrollPos_<key>` in `localStorage`)
- Polls every 100ms (up to **60 attempts / 6 seconds**) for virtual scrolling containers
- Walks up the DOM tree from the anchor element to find scrollable parents
- **Smart infinite-load handling:** If the container exists but hasn't loaded enough rows to reach the saved position, it scrolls to the bottom to force loading more content, then retries
- Clears the saved position after successfully restoring (one-shot consumption)

### Fallback: scroll to identifier

`JS_ScrollPosition_Restore` does nothing when no position was saved, for example when the user opened the page from an external app instead of from the original app itself.
**JS_ScrollPosition_RestoreOrFind** solves that case. It uses the identifier of the detail page's object to find the matching row on the overview.

**Nanoflow:**

```Text
Overview row button -> Call JS_ScrollPosition_Save('orders', '.planning-list') -> Show detail page
Detail page "Back" -> Show overview page -> Call JS_ScrollPosition_RestoreOrFind('orders', '.planning-list', $Order/OrderNumber)
```

The external app opens the detail page directly, so there is no saved context. Going back then still lands on the right row, because the identifier is used.

**Parameters:**

- `key` (string, required) - Same key as used with JS_ScrollPosition_Save.
- `querySelector` (string, required) - An element that contains the rows of the overview. See [Choosing the querySelector](#choosing-the-queryselector).
- `identifier` (string, optional) - The text shown on the overview for this object, such as an order number. Leave it empty to get the plain JS_ScrollPosition_Restore behavior.

**How it works:**

1. **Restore the saved position if there is one**
Same as JS_ScrollPosition_Restore: polls for up to 60 attempts / 6 seconds, loads more rows of a virtual-scrolling grid when needed, sets `scrollTop` and removes `scrollPos_<key>`.
2. **Find the row (if the identifier is not empty)**
Polls every 100ms, up to 60 attempts when a position was saved or 30 attempts / 3 seconds when not, because the list loads its data after the page opens. Inside the elements matching `querySelector` it looks for the element whose text **exactly equals** the identifier, then takes the row around it.
3. **Decide what to show**
   - A restored position that shows the row is kept as is, so the user returns to the exact pixel position they left.
   - In every other case the row is vertically centered: there was no saved position (e.g. coming from an external app), the saved position couldn't be applied, or it is stale. A stale position is one left over from an earlier visit where Save ran but the user left the detail page another way (menu, browser back, closed tab). Restoring it would show the wrong rows.
   - `scrollPos_<key>` is removed and polling stops.
4. **If no row matches in time**
Only the saved position (if any) is applied. With no saved position and an empty identifier, nothing happens.

#### Choosing the querySelector

The selector must match an element that **contains the rows the user sees**. It doesn't have to be the element that scrolls: RestoreOrFind finds the scroll containers itself.

| Overview widget | querySelector |
|-----------------|---------------|
| List view | `".mx-listview"` or the list view's own class |
| A table, or another widget built from rows | The container around it, e.g. its scroll container or layout grid |

Prefer a custom class that you add to that container in Studio Pro (e.g. `planning-list`). Generated names like `.mx-name-scrollContainer2` change when someone renames or copies a widget. Also check that the selector matches the list you mean: a page can contain other list views (even empty ones) that a generic selector like `.mx-listview` matches, while the visible rows are drawn by a different widget.

Pass the same selector to JS_ScrollPosition_Save. Save looks for the scrolling element in the matched element and two levels below it. If it finds none, it saves nothing and RestoreOrFind centers the row instead of restoring the exact pixel position.

#### Choosing the identifier

- Pass the value **exactly as the overview shows it**, character for character. Numbers can be formatted (`1,234`), enumerations show their caption, and dates have a display format.
- It must be the **whole text of its own element**, such as a Text widget or a column that shows only that value. `Order 123 - Smith` doesn't match `123`, and `12` doesn't match `123`.
- It must be **text, not the value of an input**. An editable text box renders its value inside an `<input>`, which isn't searched. Use a Text widget, or a text box with read-only style "Text".
- It should be **unique on the overview**. If the same text also appears in an earlier row (e.g. order number `57` and a quantity of `57`), the action scrolls to that earlier row.

**Limitation:** only rows that are in the page can be found. The action is meant for lists that render all their rows. A row beyond the first page of a list view with "Load more" or paging, or one that a virtual-scrolling grid hasn't rendered, isn't found.

#### Troubleshooting

If the overview doesn't scroll to the row, open it in the browser, paste this into the developer console with your own identifier, and check the output:

```js
(() => {
	const id = "identifier"; // the identifier your nanoflow passes
	const norm = s => s.replace(/\s+/g, " ").trim();
	const widgets = el => { const names = []; for (let p = el; p; p = p.parentElement) { const m = [...p.classList].find(c => c.startsWith("mx-name-")); if (m) names.push(m); } return names.join(" < "); };
	const walker = document.createTreeWalker(document.body, NodeFilter.SHOW_TEXT);
	for (let n = walker.nextNode(); n; n = walker.nextNode())
		if (norm(n.nodeValue).includes(id)) {
			const el = n.parentElement;
			console.log("element text:", JSON.stringify(norm(el.textContent)), "| exact match:", norm(el.textContent) === id, "| inside:", widgets(el), el);
		}
})();
```

- **Nothing printed:** the row isn't in the page (not loaded, or the value is formatted differently or shown in an input).
- **`exact match: false`:** the value is part of a longer text. Show it in its own Text widget.
- **`exact match: true`:** the action can find this row. Make sure `querySelector` matches one of the widgets in the `inside:` list (or a custom class on one of them).

## Implementation Details

### Standard Actions (JS_ScrollPosition_Save / JS_ScrollPosition_Restore)

For regular scrollable containers, including Data Grid 2 with virtual scrolling (DataWidgets 3.11+).

**JS_ScrollPosition_Save:**

- For each element matching `querySelector`, checks the element itself and its children/grandchildren for one with:
  - `overflow-y: auto` or `overflow-y: scroll`
  - `scrollHeight > clientHeight` (actually overflows)
- Since DataWidgets 3.11, a Data Grid 2 with virtual scrolling scrolls `.widget-datagrid-grid.infinite-loading`, which sits inside `.widget-datagrid-content`, so the children are checked too
- Writes the first match's `scrollTop` to `localStorage` as `scrollPos_<key>`
- If no scrollable container is found, nothing is written

**JS_ScrollPosition_Restore:**

- Reads `scrollPos_<key>` from `localStorage`
- If not found, returns immediately (no-op)
- Uses polling because the DOM may not be fully rendered when the nanoflow runs
- Finds the scrollable element the same way as Save
- If the saved position is beyond the currently loaded rows, scrolls to the bottom to trigger loading more, then continues polling
- Sets `scrollTop` once the position is reachable, or on the last attempt (60 attempts / 6 seconds) with whatever has loaded
- Clears the saved position after restoring (prevents accidental reuse)

### JS_ScrollPosition_RestoreOrFind

- Restores the saved position with the same code as JS_ScrollPosition_Restore, then runs the identifier search in the same polling loop
- **Finding the element:** walks the text nodes inside each `querySelector` match. From a text node that contains the identifier, it climbs to the smallest element whose trimmed, whitespace-collapsed text equals it exactly, and stops climbing once an ancestor's text gets longer. Hidden elements (no layout boxes, e.g. a copy shown only on another screen size) are skipped
- **Finding the row:** the closest `.mx-listview-item`, otherwise the closest `tr` or `[role=row]`, otherwise the matching element itself. The list view item comes first because its content is often laid out with a table, whose `tr` is only part of the item
- **Finding the scroll containers:** every ancestor of the row with `overflow-y: auto|scroll` and `scrollHeight > clientHeight`, innermost first, plus the page itself when it scrolls. A list usually doesn't scroll itself; the page's content area or a scroll container around it does. All ancestors are used because the first one can be a false positive: `overflow-x: hidden` makes the computed `overflow-y` `auto`, so a wrapper that overflows by a pixel looks scrollable while the real scroll area sits further up
- **Keeping a restored position:** it is kept when at least part of the row is visible in every scroll container
- **Centering:** sets `scrollTop` on each scroll container, innermost first, so the row ends up in the middle of the visible area. This is what `scrollIntoView({ block: "center" })` does, except that `scrollIntoView` also scrolls `overflow: hidden` layout wrappers, which makes Mendix layouts jump
- If the row is found but nothing can scroll yet, it keeps polling, because the list may still be laying out

### Virtual Scrolling Actions (JS_VirtualScrollPosition_Save / JS_VirtualScrollPosition_Restore)

For virtual scrolling containers where content loads dynamically (infinite-load).

**JS_VirtualScrollPosition_Save:**

- Starts from the element matching `querySelector` and walks up the DOM tree
- Finds the first ancestor with:
  - `overflow-y: auto` or `overflow-y: scroll`
  - `scrollHeight > clientHeight` (actually overflows)
- Writes `scrollTop` to `localStorage` as `scrollPos_<key>`
- If no scrollable ancestor is found, nothing is written

**JS_VirtualScrollPosition_Restore:**

- Reads `scrollPos_<key>` from `localStorage`
- If not found, returns immediately (no-op)
- Polls every 100ms (up to 60 attempts / **6 seconds**) for the scrollable ancestor to appear
- **Infinite-load support:** If the target scroll position is beyond the currently loaded content, it scrolls to the bottom to trigger loading more rows, then continues polling
- Only consumes the saved value (removes from `localStorage`) once the target position is actually reached
- Clears the saved position after successfully restoring

## Choosing the Right Actions

| Scenario | Use |
|----------|-----|
| Standard DataGrid2, Dataview | JS_ScrollPosition_Save / JS_ScrollPosition_Restore |
| Infinite-loading DataGrid2 | JS_VirtualScrollPosition_Save / JS_VirtualScrollPosition_Restore |
| Overview (list view, SortList, table, ...) whose detail page can also be opened directly, e.g. from an external app | JS_ScrollPosition_Save / JS_ScrollPosition_RestoreOrFind |

## Compatibility

- Mendix Studio Pro (tested with Mendix 10+)
- Works with Mendix DataGrid2 and data views
- JS_ScrollPosition_RestoreOrFind works with any widget that renders all its rows with the identifier as text, such as a list view or a SortList
- Can be adapted for any scrollable container by changing the `querySelector`

## License

MIT License - see [LICENSE](LICENSE) file for details.
