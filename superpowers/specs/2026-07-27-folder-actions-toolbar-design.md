# Spec — Folder actions moved into the folder view (web.admin)

**Date:** 2026-07-27
**Project:** web.admin (only)
**Status:** design, awaiting implementation plan

## Goal

Two changes to the shared folder sidebar (`FoldersPanel`), which is used by Media,
Products and Posts:

1. **Align the item counters.** The per-item count on real folder rows must sit at
   the same right edge as the counters on the `All` and `Unsorted` rows.
2. **Move per-folder actions (Rename / Delete) out of the row.** Remove the hover
   `Ellipsis` dropdown from each folder row and surface Rename/Delete as a small
   toolbar at the top of the folder's content area, next to the existing `Select`
   (multiselect) button — shown only when a real folder is open.

The two changes are linked: the hover `Ellipsis` button occupies horizontal space on
the right of every folder row (even at `opacity-0`), which pushes folder counters left
so they no longer line up with `All`/`Unsorted`. Removing it fixes the alignment for
free.

## Current state

- `src/features/folders/FoldersPanel.tsx`
  - `All` / `Unsorted` rows: rendered directly as `FolderRowButton`
    (`justify-between`), counter flush right.
  - Real folder rows (`folders.map`, lines ~146–215): wrapped in
    `<div class="group flex items-center gap-1">` = `FolderRowButton` (`flex-1`) **plus**
    a `DropdownMenu` trigger (`Ellipsis`, lines ~188–198) that is
    `opacity-0 group-hover:opacity-100`. The dropdown holds **Rename** (line ~200) and
    **Delete** (line ~208).
  - Inline rename editor: the other branch of the row ternary (lines ~147–178) — an
    autofocus `Input` + Save/Cancel, Enter/Escape handlers.
  - Delete confirmation: `AlertDialog` (lines ~268–295), message uses
    `deleteCount` = `counts.byFolder.get(id)` (how many items fall back to Unsorted).
  - Mutations `create` / `rename` / `remove` (lines ~71–103) call
    `createFolder` / `renameFolder` / `deleteFolder` from `src/lib/folders.ts`.
- Selection toolbar: `src/features/folders/SelectionBar.tsx` renders in the **page
  header row** (replacing search/`Select`/`Upload`), not inside `FoldersPanel`. Entered
  via `startSelection` from a `Select` button on each page (e.g.
  `MediaPage.tsx:181`, same in `ProductsPage.tsx`, `PostsPage.tsx`). Selection state
  lives in `useFolders` (`selectionMode`, `selectedIds`, `startSelection`,
  `exitSelection`, …).
- Active folder id is `useFolders().activeFolderId` (null for `all`/`unsorted`);
  `folders` list and `renameFolder`/`deleteFolder` are all reachable from the hook /
  lib already.

## Design

### 1. Counter alignment

Delete the `<div class="group …">` wrapper and the `DropdownMenu` from the folder-row
map. Render each folder row as a plain `FolderRowButton` (identical structure to
`All`/`Unsorted`). `justify-between` + `shrink-0` on the counter then aligns all rows on
the same right edge automatically. The inline rename editor branch stays (now triggered
from the new toolbar, see below).

### 2. Folder actions toolbar

Introduce a small **FolderActionsBar** shown in the page header area of each consuming
page, on the same row as / adjacent to the `Select` button, and **only when a real
folder is active** (`activeFolderId != null` — i.e. not `All`/`Unsorted`). It contains:

- **Rename** — puts the active folder into the existing inline-rename flow (or a small
  inline input in the bar; reuse the current `renameFolder` mutation + the same
  Enter/Escape/Save/Cancel affordances). The rename state (`editingId`/`editName`) moves
  up to where it can be driven from the bar.
- **Delete** — opens the existing `AlertDialog` for the active folder (same
  `deleteCount` message, same `remove` mutation).

Placement detail: the bar sits with the `Select`/search controls (the non-selection
header state). When `selectionMode` is on, `SelectionBar` already replaces that row —
folder actions are hidden during multiselect (they don't apply to a selection). When no
real folder is open, the bar is absent and only the usual search/`Select`/`Upload`
controls show.

Because `FoldersPanel` no longer owns Rename/Delete UI, the rename-state and
delete-target state, plus the `AlertDialog`, either (a) move into the shared bar
component, or (b) stay in a shared owner (the page or a small shared hook) that both the
panel-less rows and the bar read from. Preferred: extract a **`FolderActionsBar`**
component under `src/features/folders/` that takes `{ site, section, activeFolderId,
folders, onRenamed?, onDeleted? }`, owns the rename input + delete `AlertDialog`, and
reuses `renameFolder`/`deleteFolder` from `src/lib/folders.ts`. Pages render it next to
`Select`.

## Files affected (web.admin)

- `src/features/folders/FoldersPanel.tsx` — remove the dropdown + wrapper from folder
  rows; folder rows become plain `FolderRowButton`. Possibly move rename/delete state
  out.
- **New** `src/features/folders/FolderActionsBar.tsx` — the top toolbar (Rename +
  Delete for the active folder).
- `src/features/media/MediaPage.tsx`, `src/features/products/ProductsPage.tsx`,
  `src/features/posts/PostsPage.tsx` — render `FolderActionsBar` in the header row
  (non-selection state), when a real folder is active.
- `src/lib/folders.ts` — no change expected (mutations reused).
- `useFolders.ts` — expose whatever the bar needs (already exposes `activeFolderId`,
  `folders`, counts); add helpers only if required.

## Edge cases

- `All` / `Unsorted` active → no actions bar (they aren't deletable/renamable).
- Delete of the active folder → items fall back to Unsorted (FK `on delete set null`);
  after delete, reset the active folder to `all` (existing `effectiveFolder` fallback
  already handles a vanished id, but the bar should also exit to `all`).
- Multiselect active → bar hidden (SelectionBar owns the row).
- Rename to empty / duplicate name — keep current validation behavior.

## Testing (per platform-docs/methodology/testing.md)

- Component (Vitest + RTL) on `FolderActionsBar`: shows only for a real folder; Rename
  calls `renameFolder`; Delete opens confirmation and calls `deleteFolder`.
- Component on `FoldersPanel`: folder rows no longer render a dropdown; counters present
  on all rows with consistent right alignment (assert structure/classes).
- e2e (Playwright) optional / propose in report: open a folder → rename via bar → delete
  via bar → items move to Unsorted.

## Out of scope

- Folder creation flow (unchanged).
- Any change to how counts are computed.
- Drag-and-drop / reordering of folders.
