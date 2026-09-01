# QuickerPlaces — User Manual

QuickerPlaces is a small always-on-top-of-your-workflow window for storing folder paths and URLs under a short, memorable name (an **alias**), so you can get back to them in one click instead of digging through File Explorer or your bookmarks. This guide covers everything you can do in the app itself. For what's under the hood, see `ai/BUILD_SUMMARY.md`.

## The window, at a glance

When QuickerPlaces opens you'll see, top to bottom:

- A header with the app name and four buttons: **Import...**, **Export...**, **Add Folder**, **Add URL**.
- A row of **favourite bubbles** — your pinned places, one click away. Empty at first, with a hint telling you how to add one.
- A **Hide List** / **Show List** button that collapses or restores everything below it.
- The **All Places** grid — every place you've saved, one row each.

Nothing here needs a save button. Every add, edit, favourite toggle, reorder, or removal is written to disk the moment it happens.

## Adding a place

Click **Add Folder** or **Add URL** in the header. Either opens the same small dialog:

1. **Alias** — the name you'll use to recognize this place. Must be unique; "Docs" and "docs" count as the same alias, so you'll be blocked (with a clear message) if you try to reuse one.
2. **Path or URL** — for a folder, either type the path or use the **Browse...** button to pick it; for a URL, type it in (e.g. `https://wiki.example.com`). This also has to be unique — you can't save the same path or URL twice. Note that this check is exact: `C:\Projects` and `C:\Projects\` are treated as different values, as are `http://` and `https://` versions of the same site, so use whichever form you actually want to keep.

Click **Save**, and the new place appears immediately at the bottom of the grid.

If you enter a folder path, QuickerPlaces checks that it's a syntactically valid path — it does not require the folder to already exist on disk, so you can save a place for a folder you're about to create. A URL is checked for being well-formed but is never contacted or pinged when you save it.

## Working with a place in the grid

Every row in the **All Places** grid shows the Alias, Type (Folder or URL), the Path/URL, whether it's a Favourite, and the date it was added.

- **Double-click a row** to open it — a folder opens in File Explorer, a URL opens in your default browser.
- **Right-click a row** for the full menu:
  - **Open** — same as double-click.
  - **Rename Alias** — change just the name; the same uniqueness check from adding applies.
  - **Edit Path/URL** — change just the destination; the same duplicate check applies.
  - **Toggle Favourite** — pin it to (or unpin it from) the bubble row above the grid.
  - **Remove** — deletes it. There's no undo, so double-check before confirming.

If a place can no longer be opened — the folder's been deleted, or the URL is malformed — you'll get a clear message instead of the app crashing or silently doing nothing.

## Favourites

Any place can be a favourite. Toggling **Favourite** (from the grid's right-click menu, or from a bubble's own right-click menu) adds or removes it from the bubble row above the grid.

- **Click a bubble** to open that place — identical to double-clicking its row.
- **Drag a bubble** left or right to reorder the row. The order you leave them in is remembered.
- **Right-click a bubble** for a shortcut menu: **Open**, or **Remove from Favourites** — you don't need to go back to the grid just to unpin something.

## Hiding the grid

If you only want the favourite bubbles visible, click **Hide List**. The grid collapses and the window shrinks to make room; click **Show List** to bring it back. This state is remembered between sessions, along with the window's size and position.

## Exporting places

Click **Export...** to open a checklist of every place you've saved, all checked by default. Uncheck anything you don't want to include, then choose where to save the resulting `.json` file. This is the way to back up your list or hand a set of places to someone else running QuickerPlaces.

## Importing places

Click **Import...** and pick a `.json` file that was previously created with Export. QuickerPlaces compares every item in that file against what you already have and silently drops anything that would collide — same alias (case-insensitive) or same path/URL (exact match) as something you already saved. You're never shown or asked about those; there's nothing to decide.

What's left — the items that don't collide with anything — is presented as a checklist, all checked by default, exactly like Export. Uncheck anything you don't want, click **Import**, and the selected items are added and written to disk immediately. A short summary tells you how many were brought in.

If everything in the file collides with what you already have, you'll see an empty (or very short) list — that's expected, not an error.

## Where your data lives

QuickerPlaces keeps two small JSON files, both plain text and safe to open in a text editor if you're curious or want to back them up manually:

- **Your places:** `%AppData%\QuickerPlaces\QuickerPlaces\places.json` — written the instant anything changes.
- **Window layout** (size, position, whether the grid is collapsed): `%LocalAppData%\QuickerPlaces\QuickerPlaces\settings.json` — saved when the window closes.

You never need to touch either file by hand, but if you ever want to move your places to another machine, copying `places.json` across is all it takes.

## If something goes wrong

- **First launch, or a missing places file:** QuickerPlaces just starts with an empty list — this is normal, not an error, and your first **Add Folder**/**Add URL** creates the file.
- **A places file that can't be read** (corrupted, edited by hand and broken, etc.): QuickerPlaces tells you once, on startup, that it couldn't load your saved places, and starts you with an empty list rather than crashing. Your original file is left on disk untouched — it isn't overwritten until you save something new — so if you know your way around JSON, you can inspect and fix it before adding anything new in the app.
- **A place that won't open:** you'll get an on-screen message explaining why (folder no longer exists, URL is malformed, etc.) rather than the app freezing or closing.

If you hit anything not covered here, or something that looks like an actual crash, that's worth reporting rather than working around — see `ai/BUILD_SUMMARY.md` for the project's current known-issues status.
