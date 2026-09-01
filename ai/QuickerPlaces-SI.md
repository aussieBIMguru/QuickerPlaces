---
title: QuickerPlaces — System Instructions (SI)
status: implemented (v1) — see BUILD_SUMMARY.md
last_updated: 2026-08-31
---

# QuickerPlaces — System Instructions

## 0. How to use this document

This SI is the requirements/spec handoff for building **QuickerPlaces**, a Windows desktop utility for storing and quickly opening remembered "places" (folder paths and URLs). It was produced in a planning conversation, before the implementation template was attached.

**Before writing any code**, inspect the attached project template (zip) to determine:
- The application framework in use (WPF, WinForms, or other) and target .NET version
- Existing project structure, namespaces, and file organization
- Existing UI patterns, styles/themes, and control conventions already established in the template
- Any existing dialog patterns (this determines the open question in §7 — combined vs. wizard add-flow)
- Existing data-access, settings, or serialization scaffolding, if any

Where the template already establishes a convention, **follow the template's convention** over inventing a new one, as long as it doesn't conflict with a hard requirement stated below. Where this SI is silent or the template offers no precedent, use the assumptions in §8 and the coding conventions in §9.

## 1. Overview

QuickerPlaces is a lightweight, always-foreground desktop app (no system tray / minimize-to-tray behavior — closing the window exits the app). Its purpose is to let the user store folder paths and URLs against a memorable Alias, browse/search them in a grid, favourite the most-used ones as one-click bubbles, and back the whole thing with a JSON file that's always up to date and can be exported/imported.

Parent project context: QuickerLinks ("a better path launcher than Quick Links"). QuickerPlaces is the concrete application being built under that umbrella.

## 2. Goals

- Fast entry of a new folder or URL "place" under a unique Alias, with validation that prevents duplicate aliases and duplicate resources.
- A DataGrid of all stored places with right-click actions (Open / Rename Alias / Edit Path/URL / Toggle Favourite / Remove) and double-click-to-open.
- A row of favourite "bubbles" above the grid for one-click opening of the user's most important places; the grid itself can be collapsed to get it out of the way.
- Continuous persistence to a JSON settings file (write-through on every change, not just on exit).
- Export of user-selected places to a shareable JSON file, and import of places from such a file, with automatic exclusion of anything that would collide with what's already stored.

## 3. Non-Goals (for this pass)

- No system tray / background running — this is explicitly out of scope per the user's requirement.
- No requirement (unless the template already provides it) for: favicon/folder-icon rendering on bubbles or grid rows, full-text search/filter box, keyboard-shortcut scheme, or multi-window support. These are reasonable future enhancements but are not required for the base build — call them out separately if the template happens to already support them cheaply.
- No reachability/liveness checking of URLs or folders at add-time beyond basic format validation (see §6.2) — QuickerPlaces does not ping URLs or block on network calls during entry.

## 4. Core Concepts / Data Model

A **Place** is the fundamental record:

| Field | Type | Notes |
|---|---|---|
| `Alias` | string | User-facing unique name. Comparison is case-insensitive ("Docs" and "docs" collide). |
| `Type` | enum: `Folder` \| `Url` | Determines validation rules and the Open action. |
| `Resource` | string | Absolute folder path, or a URL. |
| `IsFavourite` | bool | Drives bubble visibility. |
| `FavouriteOrder` | int (nullable) | User-controlled manual ordering for bubbles when favourited (see §6.4). |
| `DateAdded` | DateTime | For internal bookkeeping; not necessarily a visible column unless template convention favors showing it. |

Suggested JSON settings shape (adjust to match template's existing serialization approach if one exists):

```json
{
  "schemaVersion": 1,
  "places": [
    {
      "alias": "Downloads",
      "type": "Folder",
      "resource": "C:\\Users\\Gavin\\Downloads",
      "isFavourite": true,
      "favouriteOrder": 0,
      "dateAdded": "2026-08-31T10:15:00+09:30"
    },
    {
      "alias": "Company Wiki",
      "type": "Url",
      "resource": "https://wiki.example.com",
      "isFavourite": false,
      "favouriteOrder": null,
      "dateAdded": "2026-08-31T10:16:40+09:30"
    }
  ]
}
```

Default settings file location (unless the template dictates otherwise): `%AppData%\QuickerPlaces\settings.json`.

## 5. Persistence Behavior

- The in-memory place list and the JSON file must be kept in sync continuously: every add, rename, path/URL edit, favourite toggle, reorder, or removal writes through to disk immediately (not batched to app close).
- Writes should be resilient to a mid-write crash (e.g. write to a temp file and atomically replace, if the template's serialization layer doesn't already handle this) — avoid a pattern that can corrupt or truncate the settings file. This follows the "avoid exceptions in user-facing tools" / robustness preference in §9.
- On startup, load the JSON file; if it's missing, start with an empty list and create it on first write. If it's present but unreadable/corrupt, fail gracefully (do not crash) — surface a non-blocking notice and start from an empty list rather than losing the app to an unhandled exception.

## 6. Functional Requirements

### 6.1 Adding a Place

Two distinct entry points: **Add Folder** and **Add URL** (e.g. two buttons/menu items). Each triggers its own validation pipeline before the entry is committed to the list:

1. Prompt for an **Alias**.
   - Reject/block submission if the Alias (case-insensitively) already exists, with a clear inline message — do not silently overwrite.
2. Prompt for the **Path** (folder, ideally via a folder-browse dialog as well as free text entry) or **URL** (free text, with basic format validation — see §6.2).
   - Reject/block submission if the Resource already exists in the store, using exact case-insensitive string match (see §6.2 for the precise rule).
3. On success, the new Place is appended to the list, written to disk immediately, and appears in the DataGrid.

**Open question — resolve against the template:** whether Alias and Path/URL are captured in a single combined dialog (both fields, validated together on submit) or a two-step flow (Alias first, confirmed unique, then Path/URL). Follow whatever dialog convention the template already uses; if the template has no precedent, default to the single combined dialog for simplicity.

### 6.2 Validation Rules

- **Alias uniqueness:** case-insensitive exact match against all existing aliases.
- **Resource duplicate check:** case-insensitive exact string match against all existing resources of the same type. (Explicitly *not* normalized — `C:\Foo` and `C:\Foo\` are treated as different values, and `http://` vs `https://` variants of a URL are treated as different values. This is a deliberate simplicity choice, not an oversight.)
- **Folder validation:** the entered path should be a syntactically valid path; if the template's conventions call for it, a folder-browse dialog is preferred over free typing to reduce invalid entries. Whether a non-existent-on-disk folder is blocked or just allowed with a warning is left to whatever's simplest given the template — either is acceptable, but silently accepting garbage input with no feedback is not.
- **URL validation:** basic well-formed-URI check (e.g. `Uri.TryCreate` with `UriKind.Absolute`). No network reachability check is performed at add-time.
- Editing an existing Place's Alias or Path/URL (via the grid context menu) re-runs the exact same validation rules as adding a new one (excluding the entry being edited itself from the duplicate check against itself).

### 6.3 DataGrid

- One row per Place. Columns at minimum: Alias, Type, Resource/Path, Favourite indicator. Add other columns (e.g. Date Added) only if useful and if the template's grid conventions make it easy — not a hard requirement.
- **Right-click context menu**, in this order:
  1. **Open** (top item, effectively the default action)
  2. Rename Alias
  3. Edit Path/URL
  4. Toggle Favourite
  5. Remove
- **Double-click on a row = Open**, equivalent to the top context-menu item.
- **Open** behavior: for a `Folder` type, open the path in File Explorer; for a `Url` type, open in the system default browser. If the resource can no longer be opened (folder deleted, malformed URL, etc.), fail gracefully with a clear, non-crashing message — do not let an unhandled exception surface to the user (see §9).
- The grid must be collapsible/expandable (e.g. a toggle or splitter) so the user can hide it and rely on the favourite bubbles alone.

### 6.4 Favourites (bubbles)

- Any Place can be toggled as a favourite (from the grid context menu, or directly from a bubble's own right-click menu — see below).
- Favourited places render as bubbles in a row **above** the DataGrid. Each bubble shows the Alias; clicking a bubble performs the same Open action as double-clicking its grid row.
- **Ordering:** user-controlled — bubbles support drag-to-reorder, and that order (`FavouriteOrder`) persists to the JSON settings file.
- **Right-click on a bubble** offers at minimum "Remove from favourites" directly, without requiring the user to go back to the grid.

### 6.5 Export

- The user can trigger an export action that presents their full list of stored Places (e.g. via checkboxes) and lets them select any number of them to write out to a JSON file (same shape as §4, or a reasonable subset of it).

### 6.6 Import

- The user selects a previously-exported JSON file to import.
- Before anything is offered to the user, the app filters the incoming items down to only those that do **not** collide with anything already stored — i.e., an incoming item is excluded entirely from the candidate list if its Alias already exists (case-insensitive) **or** its Resource already exists (case-insensitive exact match), per the same rules as §6.2.
- The remaining (non-colliding) candidates are presented to the user (e.g. via checkboxes) so they can select any number of them to actually bring in — mirroring the export experience.
- After the user confirms, the selected items are added, written to disk, and a short confirmation summary is shown (e.g. "4 imported"). Items that were excluded up front due to collisions are never part of the selectable list — there's no need to also report them as "skipped," since the user never saw them as an option.

## 7. Open Decisions (resolve once template is reviewed)

1. Combined single dialog vs. two-step wizard for adding a Place (§6.1).
2. Exact framework/tooling implied by the template (WPF vs. WinForms vs. other; target .NET version).
3. Whether the template already has a settings/serialization pattern to reuse rather than building one from scratch.
4. Whether the template's existing style system dictates the visual treatment of bubbles (e.g. Chip/Tag control already available) rather than a custom control.

## 8. Assumptions (carried forward unless the template or user says otherwise)

- Alias comparison is case-insensitive.
- The app is a normal window; closing it exits the process fully (no tray, no background task).
- Editing an existing entry's Alias or Path/URL re-runs the same validation as adding a new entry.
- Settings JSON lives at `%AppData%\QuickerPlaces\settings.json` and is written on every change, not just on close.

## 9. Technical Conventions

Per the user's standing coding preferences, apply throughout:

- Readable, best-practice, domain-suited, robust code — favor robustness over shortcuts, keep it reasonably scalable and well organized.
- XML doc-comment headers on public types/members where practical.
- Use nullable reference/value types where they genuinely add clarity (e.g. `FavouriteOrder` as `int?`).
- Avoid throwing exceptions out to the user in normal operation — validation failures, missing files, unopenable resources, etc. should be handled as expected conditions with clear UI feedback, not unhandled exceptions or crash dialogs.
- Prefer explicit typing when it aids clarity; `var` is fine when the right-hand side already makes the type obvious (e.g. `new(...)`).
- Prefer LINQ where it doesn't meaningfully hurt performance for this scale of data (a personal place list — hundreds of rows at most, not a hot path).

## 10. Suggested Next Step

In the follow-up chat: attach this SI plus the template zip, and ask Claude to (a) inspect the template's structure/framework/conventions first, (b) resolve the open decisions in §7 against what it finds, and (c) implement the base app per §§4–6.

---

*Note: this document is preserved here as the original spec handoff. For what was actually built against it — including where the implementation deviated or made a judgment call — see [`BUILD_SUMMARY.md`](./BUILD_SUMMARY.md).*
