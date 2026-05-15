# ScrollPosition

Mendix JavaScript actions for saving and restoring scroll positions in data views and data grids.

## Overview

This module provides two reusable JavaScript actions that work together to preserve scroll position when navigating between pages in Mendix applications:

- **JS_ScrollPosition_Save** - Saves the current scroll position to `localStorage`
- **JS_ScrollPosition_Restore** - Restores a previously saved scroll position

## Use Case

When users navigate from a list page (e.g., a data grid or data view with many items) to a detail page and return, they expect to see the same position they left off. These actions solve this by:

1. Saving the scroll position before navigating away
2. Restoring it when returning to the page

## Usage

### Saving Scroll Position

Call `JS_ScrollPosition_Save` **before** navigating away from a page with scrollable content:

**Nanoflow:**
```
User clicks "Open Page" button -> Call JS_ScrollPosition_Save (Pass the parameters) -> Show page
```

**Parameters:**
- `key` (string, required) - Unique identifier for this page/context. Use different keys for different pages.
- `querySelector` (string, required) - CSS selector for the scrollable container. Common values:
  - `".widget-datagrid-content"` - For DataGrid2
  - `".mx-dataview-content"` - For Mendix Dataviews

### Restoring Scroll Position

Call `JS_ScrollPosition_Restore` **after** showing the page:

**Nanoflow:**
```
Return to List/DataGrid2 Page -> Call JS_ScrollPosition_Restore (Pass the parameters) -> Show page
```

**Events widget:**

The [Events widget](https://marketplace.mendix.com/link/component/224259) can also be used.
Create a nanoflow and add the `JS_ScrollPosition_Restore` action to the nanoflow and fill in the `key` and `querySelector` parameters.
Note that the Events widget might require a delay of 100 milliseconds or more depending on how long it takes for the page to load.


**How it works:**
- Reads `scrollPos_<key>` from `localStorage`
- Polls every 100ms (up to 30 attempts / 3 seconds) for the scrollable element to appear in DOM
- On first match, sets `scrollTop` to the saved value
- Removes the saved value from `localStorage` (one-shot consumption)
- If no scrollable container appears within 3 seconds, silently abandons

## Implementation Details

### JS_ScrollPosition_Save

- Finds the first element matching `querySelector` with:
  - `overflow-y: auto` or `overflow-y: scroll`
  - `scrollHeight > clientHeight` (actually overflows)
- Writes `scrollTop` to `localStorage` as `scrollPos_<key>`
- If no scrollable container is found, nothing is written

### JS_ScrollPosition_Restore

- Reads `scrollPos_<key>` from `localStorage`
- If not found, returns immediately (no-op)
- Uses polling because the DOM may not be fully rendered when the nanoflow runs
- Stops polling on first match or after 30 attempts (3 seconds)
- Clears the saved position after restoring (prevents accidental reuse)

## Compatibility

- Mendix Studio Pro (tested with Mendix 10+)
- Works with Mendix DataGrid2 and data views
- Can be adapted for any scrollable container by changing the `querySelector`

## License

MIT License - see [LICENSE](LICENSE) file for details.
