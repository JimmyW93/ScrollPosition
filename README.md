# ScrollPosition

Mendix JavaScript actions for saving and restoring scroll positions in data views, data grids, and virtual scrolling containers.

## Overview

This module provides reusable JavaScript actions that work together to preserve scroll position when navigating between pages in Mendix applications.

### Standard Scroll Position Actions

For regular scrollable containers (DataGrid2, Dataviews):

- **JS_ScrollPosition_Save** - Saves the current scroll position to `localStorage`
- **JS_ScrollPosition_Restore** - Restores a previously saved scroll position

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

## Compatibility

- Mendix Studio Pro (tested with Mendix 10+)
- Works with Mendix DataGrid2 and data views
- Can be adapted for any scrollable container by changing the `querySelector`

## License

MIT License - see [LICENSE](LICENSE) file for details.
