# Folder Actions Toolbar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move per-folder Rename/Delete out of the folder-list rows into a toolbar shown above the section content (next to the `Select` button), so folder rows become plain buttons whose item counters align with the `All` / `Unsorted` rows.

**Architecture:** A new `FolderActionsBar` component owns the rename input, the delete confirmation dialog, and the rename/delete mutations for the currently-open folder. Each section page (Media, Products, Posts) renders it in the header row when a real folder is active. `FoldersPanel` loses all rename/delete UI and state, keeping only folder selection and the create flow; its folder rows become identical in structure to `All`/`Unsorted`, which fixes counter alignment for free.

**Tech Stack:** React + TypeScript, Vite, TanStack Query, shadcn/ui (Radix), Tailwind v4, Vitest + React Testing Library. Project: `web.admin` only (no DB, no other repo).

**Spec:** `platform-docs/superpowers/specs/2026-07-27-folder-actions-toolbar-design.md`

**Repo rule reminder:** Do not commit or push without the user's explicit go-ahead. The commit steps below are part of the workflow; confirm with the user before running them (subagent-driven review happens between tasks).

---

## File Structure

- **Create** `web.admin/src/features/folders/FolderActionsBar.tsx` — toolbar for the active folder: Rename (inline input) + Delete (confirmation `AlertDialog`); owns rename/delete mutations. One clear responsibility: mutate the one folder the page hands it.
- **Create** `web.admin/src/features/folders/FolderActionsBar.test.tsx` — component tests.
- **Create** `web.admin/src/features/folders/FoldersPanel.test.tsx` — regression tests for the stripped panel (no per-row menu, counters render, selection works).
- **Modify** `web.admin/src/features/folders/FoldersPanel.tsx` — remove per-row dropdown, inline-rename branch, delete dialog, rename/delete state + mutations, and the now-unused `itemNoun`/`contentKey` props; folder rows become plain `FolderRowButton`.
- **Modify** `web.admin/src/features/media/MediaPage.tsx`, `web.admin/src/features/products/ProductsPage.tsx`, `web.admin/src/features/posts/PostsPage.tsx` — render `FolderActionsBar` in the non-selection header when a real folder is active; drop `itemNoun`/`contentKey` from the `FoldersPanel` call; (Products/Posts) add `activeFolderId` to the `useFolders` destructure.

---

## Task 1: FolderActionsBar component

**Files:**
- Create: `web.admin/src/features/folders/FolderActionsBar.tsx`
- Test: `web.admin/src/features/folders/FolderActionsBar.test.tsx`

- [ ] **Step 1: Write the failing test**

Create `web.admin/src/features/folders/FolderActionsBar.test.tsx`:

```tsx
import { beforeEach, describe, expect, it, vi } from 'vitest'
import { fireEvent, render, screen, waitFor, within } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { SiteConfig } from '@/config/sites'
import { deleteFolder, renameFolder, type FolderRow } from '@/lib/folders'
import { FolderActionsBar } from './FolderActionsBar'

// sonner дергает браузерные API/маунтит Toaster — мокаем toast, чтобы тест был тихим.
vi.mock('sonner', () => ({ toast: { success: vi.fn(), error: vi.fn() } }))
// Мокаем сеть папок: проверяем, что бар зовёт правильные функции с правильными аргументами.
vi.mock('@/lib/folders', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@/lib/folders')>()
  return {
    ...actual,
    renameFolder: vi.fn().mockResolvedValue(undefined),
    deleteFolder: vi.fn().mockResolvedValue(undefined),
  }
})

const site: SiteConfig = {
  slug: 'cozycorner',
  label: 'CozyCorner',
  projectUrl: 'https://demo.supabase.co',
  anonKey: 'sb_publishable_test',
  schema: 'cozycorner',
  bucket: 'cozycorner-photos',
}

const folder: FolderRow = {
  id: 'folder-1',
  created_at: '2026-01-01T00:00:00Z',
  section: 'media',
  name: 'Vases',
}

function renderBar(overrides: Partial<React.ComponentProps<typeof FolderActionsBar>> = {}) {
  const onDeleted = overrides.onDeleted ?? vi.fn()
  const qc = new QueryClient()
  render(
    <QueryClientProvider client={qc}>
      <FolderActionsBar
        site={site}
        section="media"
        folder={folder}
        itemCount={4}
        itemNoun="image"
        contentKey={['media', site.slug]}
        onDeleted={onDeleted}
        {...overrides}
      />
    </QueryClientProvider>,
  )
  return { onDeleted }
}

describe('<FolderActionsBar />', () => {
  beforeEach(() => vi.clearAllMocks())

  it('shows the folder name with Rename and Delete actions', () => {
    renderBar()
    expect(screen.getByText('Vases')).toBeInTheDocument()
    expect(screen.getByRole('button', { name: 'Rename' })).toBeInTheDocument()
    expect(screen.getByRole('button', { name: 'Delete' })).toBeInTheDocument()
  })

  it('renames the folder via the inline input', async () => {
    renderBar()
    fireEvent.click(screen.getByRole('button', { name: 'Rename' }))
    const input = screen.getByLabelText('Folder name')
    expect(input).toHaveValue('Vases')
    fireEvent.change(input, { target: { value: 'Lamps' } })
    fireEvent.keyDown(input, { key: 'Enter' })
    await waitFor(() => expect(renameFolder).toHaveBeenCalledWith(site, 'folder-1', 'Lamps'))
  })

  it('deletes the folder after confirmation and calls onDeleted', async () => {
    const { onDeleted } = renderBar()
    fireEvent.click(screen.getByRole('button', { name: 'Delete' }))
    const dialog = await screen.findByRole('alertdialog')
    expect(within(dialog).getByText(/4 images will be moved to Unsorted/)).toBeInTheDocument()
    fireEvent.click(within(dialog).getByRole('button', { name: 'Delete' }))
    await waitFor(() => expect(deleteFolder).toHaveBeenCalledWith(site, 'folder-1'))
    await waitFor(() => expect(onDeleted).toHaveBeenCalled())
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm --prefix web.admin test -- src/features/folders/FolderActionsBar.test.tsx`
Expected: FAIL — cannot resolve `./FolderActionsBar` (module does not exist yet).

- [ ] **Step 3: Write minimal implementation**

Create `web.admin/src/features/folders/FolderActionsBar.tsx`:

```tsx
import { useState } from 'react'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { Check, Loader2, Pencil, Trash2, X } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import {
  deleteFolder,
  foldersKey,
  renameFolder,
  type FolderRow,
  type FolderSection,
} from '@/lib/folders'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from '@/components/ui/alert-dialog'

// Тулбар действий над активной папкой: страница раздела рендерит его рядом с
// кнопкой Select и только когда открыта реальная папка (не All/Unsorted). Раньше
// эти действия жили в ellipsis-меню строки папки (FoldersPanel) — вынесли сюда,
// а строки папок стали простыми (их счётчики выровнялись с All/Unsorted).
export function FolderActionsBar({
  site,
  section,
  folder,
  itemCount,
  itemNoun,
  contentKey,
  onDeleted,
}: {
  site: SiteConfig
  section: FolderSection
  folder: FolderRow
  itemCount: number
  itemNoun: string
  contentKey: readonly unknown[]
  onDeleted: () => void
}) {
  const queryClient = useQueryClient()
  const key = foldersKey(site, section)

  const [editing, setEditing] = useState(false)
  const [editName, setEditName] = useState(folder.name)
  const [confirmingDelete, setConfirmingDelete] = useState(false)

  const rename = useMutation({
    mutationFn: (name: string) => renameFolder(site, folder.id, name),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: key })
      setEditing(false)
      toast.success('Folder renamed')
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to rename folder'),
  })

  const remove = useMutation({
    mutationFn: () => deleteFolder(site, folder.id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: key })
      // FK on delete set null: элементы папки стали Unsorted — обновить контент.
      queryClient.invalidateQueries({ queryKey: contentKey })
      setConfirmingDelete(false)
      onDeleted()
      toast.success('Folder deleted')
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to delete folder'),
  })

  function submitRename() {
    const name = editName.trim()
    if (!name || name === folder.name) {
      setEditing(false)
      return
    }
    rename.mutate(name)
  }

  if (editing) {
    return (
      <div className="flex items-center gap-1">
        <Input
          autoFocus
          value={editName}
          onChange={(e) => setEditName(e.target.value)}
          onKeyDown={(e) => {
            if (e.key === 'Enter') submitRename()
            if (e.key === 'Escape') setEditing(false)
          }}
          disabled={rename.isPending}
          className="h-9 w-44 text-sm"
          aria-label="Folder name"
        />
        <Button
          variant="ghost"
          size="icon-sm"
          aria-label="Save folder name"
          onClick={submitRename}
          disabled={rename.isPending}
        >
          {rename.isPending ? <Loader2 className="animate-spin" /> : <Check />}
        </Button>
        <Button
          variant="ghost"
          size="icon-sm"
          aria-label="Cancel renaming"
          onClick={() => setEditing(false)}
          disabled={rename.isPending}
        >
          <X />
        </Button>
      </div>
    )
  }

  return (
    <div className="flex items-center gap-1">
      <span className="max-w-40 truncate text-sm font-medium" title={folder.name}>
        {folder.name}
      </span>
      <Button
        variant="ghost"
        size="sm"
        className="h-9"
        onClick={() => {
          setEditName(folder.name)
          setEditing(true)
        }}
      >
        <Pencil className="size-3.5" />
        Rename
      </Button>
      <Button
        variant="ghost"
        size="sm"
        className="h-9 text-destructive hover:text-destructive"
        onClick={() => setConfirmingDelete(true)}
      >
        <Trash2 className="size-3.5" />
        Delete
      </Button>

      <AlertDialog
        open={confirmingDelete}
        onOpenChange={(open) => {
          if (!open) setConfirmingDelete(false)
        }}
      >
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Delete folder "{folder.name}"?</AlertDialogTitle>
            <AlertDialogDescription>
              {itemCount > 0
                ? `${itemCount} ${itemNoun}${itemCount === 1 ? '' : 's'} will be moved to Unsorted. `
                : 'The folder is empty. '}
              The {itemNoun}s themselves are not deleted.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>Cancel</AlertDialogCancel>
            <AlertDialogAction onClick={() => remove.mutate()}>Delete</AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  )
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npm --prefix web.admin test -- src/features/folders/FolderActionsBar.test.tsx`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
cd web.admin && git add src/features/folders/FolderActionsBar.tsx src/features/folders/FolderActionsBar.test.tsx && git commit -m "feat(folders): add FolderActionsBar (rename/delete toolbar for active folder)"
```

---

## Task 2: Strip FoldersPanel and wire the toolbar into the section pages

Done as one task so the TypeScript build stays green: removing `itemNoun`/`contentKey` from `FoldersPanel`'s props requires updating all three call sites in the same change.

**Files:**
- Modify: `web.admin/src/features/folders/FoldersPanel.tsx`
- Test: `web.admin/src/features/folders/FoldersPanel.test.tsx`
- Modify: `web.admin/src/features/media/MediaPage.tsx`
- Modify: `web.admin/src/features/products/ProductsPage.tsx`
- Modify: `web.admin/src/features/posts/PostsPage.tsx`

- [ ] **Step 1: Write the failing test**

Create `web.admin/src/features/folders/FoldersPanel.test.tsx`:

```tsx
import { describe, expect, it, vi } from 'vitest'
import { fireEvent, render, screen } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import type { SiteConfig } from '@/config/sites'
import type { FolderRow } from '@/lib/folders'
import { FoldersPanel } from './FoldersPanel'

vi.mock('sonner', () => ({ toast: { success: vi.fn(), error: vi.fn() } }))

const site: SiteConfig = {
  slug: 'cozycorner',
  label: 'CozyCorner',
  projectUrl: 'https://demo.supabase.co',
  anonKey: 'sb_publishable_test',
  schema: 'cozycorner',
  bucket: 'cozycorner-photos',
}

const folders: FolderRow[] = [
  { id: 'f1', created_at: '2026-01-01T00:00:00Z', section: 'media', name: 'Vases' },
]

function renderPanel(overrides: Partial<React.ComponentProps<typeof FoldersPanel>> = {}) {
  const onSelect = overrides.onSelect ?? vi.fn()
  const qc = new QueryClient()
  render(
    <QueryClientProvider client={qc}>
      <FoldersPanel
        site={site}
        section="media"
        folders={folders}
        active="all"
        onSelect={onSelect}
        counts={{ all: 16, unsorted: 7, byFolder: new Map([['f1', 4]]) }}
        {...overrides}
      />
    </QueryClientProvider>,
  )
  return { onSelect }
}

describe('<FoldersPanel />', () => {
  it('renders All / Unsorted and folder rows with their counts', () => {
    renderPanel()
    expect(screen.getByText('All')).toBeInTheDocument()
    expect(screen.getByText('Unsorted')).toBeInTheDocument()
    expect(screen.getByText('Vases')).toBeInTheDocument()
    expect(screen.getByText('16')).toBeInTheDocument()
    expect(screen.getByText('4')).toBeInTheDocument()
  })

  it('no longer renders a per-row actions (ellipsis) menu — actions moved to the toolbar', () => {
    renderPanel()
    expect(
      screen.queryByRole('button', { name: /Actions for folder/ }),
    ).not.toBeInTheDocument()
  })

  it('selecting a folder row calls onSelect with its id', () => {
    const { onSelect } = renderPanel()
    fireEvent.click(screen.getByText('Vases'))
    expect(onSelect).toHaveBeenCalledWith('f1')
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npm --prefix web.admin test -- src/features/folders/FoldersPanel.test.tsx`
Expected: FAIL — the "no longer renders … ellipsis menu" test fails because the current
`FoldersPanel` still renders the `Actions for folder …` dropdown trigger.

- [ ] **Step 3: Replace `FoldersPanel.tsx` with the stripped version**

Overwrite `web.admin/src/features/folders/FoldersPanel.tsx` with:

```tsx
import { useState } from 'react'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { Check, FolderPlus, Loader2, X } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import { createFolder, foldersKey, type FolderRow, type FolderSection } from '@/lib/folders'
import { cn } from '@/lib/utils'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'

// Активный фильтр грида: служебные вкладки All/Unsorted или id папки
// (uuid, со служебными значениями не пересекается).
export type FolderFilter = 'all' | 'unsorted' | string

type Counts = { all: number; unsorted: number; byFolder: Map<string, number> }

type Props = {
  site: SiteConfig
  section: FolderSection
  folders: FolderRow[] | undefined // undefined — ещё грузятся
  active: FolderFilter
  onSelect: (filter: FolderFilter) => void
  counts: Counts
}

// Сайдбар папок раздела: служебные All/Unsorted + список папок + «New folder».
// Rename/Delete конкретной папки живут в FolderActionsBar (тулбар над контентом),
// поэтому строки папок — простые кнопки, и их счётчики выровнены с All/Unsorted.
export function FoldersPanel({ site, section, folders, active, onSelect, counts }: Props) {
  const queryClient = useQueryClient()
  const key = foldersKey(site, section)

  const [creating, setCreating] = useState(false)
  const [draftName, setDraftName] = useState('')

  const create = useMutation({
    mutationFn: (name: string) => createFolder(site, section, name),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: key })
      setCreating(false)
      setDraftName('')
      toast.success('Folder created')
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to create folder'),
  })

  function submitCreate() {
    const name = draftName.trim()
    if (!name) {
      setCreating(false)
      setDraftName('')
      return
    }
    create.mutate(name)
  }

  return (
    <nav className="flex flex-col gap-0.5" aria-label="Folders">
      <FolderRowButton
        label="All"
        count={counts.all}
        isActive={active === 'all'}
        onClick={() => onSelect('all')}
      />
      <FolderRowButton
        label="Unsorted"
        count={counts.unsorted}
        isActive={active === 'unsorted'}
        onClick={() => onSelect('unsorted')}
      />

      {!folders &&
        Array.from({ length: 3 }, (_, i) => (
          <Skeleton key={i} className="h-8 w-full rounded-md" />
        ))}

      {folders?.map((folder) => (
        <FolderRowButton
          key={folder.id}
          label={folder.name}
          count={counts.byFolder.get(folder.id) ?? 0}
          isActive={active === folder.id}
          onClick={() => onSelect(folder.id)}
        />
      ))}

      {creating ? (
        <div className="flex items-center gap-1">
          <Input
            autoFocus
            placeholder="Folder name"
            value={draftName}
            onChange={(e) => setDraftName(e.target.value)}
            onKeyDown={(e) => {
              if (e.key === 'Enter') submitCreate()
              if (e.key === 'Escape') {
                setCreating(false)
                setDraftName('')
              }
            }}
            disabled={create.isPending}
            className="h-8 text-sm"
          />
          <Button
            variant="ghost"
            size="icon-sm"
            aria-label="Create folder"
            onClick={submitCreate}
            disabled={create.isPending}
          >
            {create.isPending ? <Loader2 className="animate-spin" /> : <Check />}
          </Button>
          <Button
            variant="ghost"
            size="icon-sm"
            aria-label="Cancel creating folder"
            onClick={() => {
              setCreating(false)
              setDraftName('')
            }}
            disabled={create.isPending}
          >
            <X />
          </Button>
        </div>
      ) : (
        <Button
          variant="ghost"
          size="sm"
          className="justify-start px-2 text-muted-foreground"
          onClick={() => setCreating(true)}
        >
          <FolderPlus className="size-4" />
          New folder
        </Button>
      )}
    </nav>
  )
}

function FolderRowButton({
  label,
  count,
  isActive,
  onClick,
  className,
}: {
  label: string
  count: number
  isActive: boolean
  onClick: () => void
  className?: string
}) {
  return (
    <button
      type="button"
      onClick={onClick}
      aria-current={isActive || undefined}
      className={cn(
        'flex items-center justify-between gap-2 rounded-md px-2 py-1.5 text-sm transition-colors hover:bg-accent/50',
        isActive && 'bg-accent font-medium',
        className,
      )}
    >
      <span className="truncate">{label}</span>
      <span className="shrink-0 text-xs text-muted-foreground">{count}</span>
    </button>
  )
}
```

- [ ] **Step 4: Update `MediaPage.tsx`**

Add the import next to the other folders imports (after the `SelectionBar` import, ~line 12):

```tsx
import { FolderActionsBar } from '@/features/folders/FolderActionsBar'
```

After the `useFolders({...})` destructure block (~line 51), add the active-folder lookup:

```tsx
  const activeFolder = activeFolderId ? folders?.find((f) => f.id === activeFolderId) : undefined
```

In the non-selection branch, insert the bar as the first child of the `<>` fragment
(before the search `<div className="relative ml-auto w-full max-w-56">`):

```tsx
          <>
            {activeFolder && (
              <FolderActionsBar
                key={activeFolder.id}
                site={site}
                section="media"
                folder={activeFolder}
                itemCount={counts.byFolder.get(activeFolder.id) ?? 0}
                itemNoun="image"
                contentKey={mediaKey}
                onDeleted={() => selectFolder('all')}
              />
            )}
            <div className="relative ml-auto w-full max-w-56">
```

In the `<FoldersPanel .../>` call, remove the `itemNoun="image"` and `contentKey={mediaKey}`
lines so it reads:

```tsx
          <FoldersPanel
            site={site}
            section="media"
            folders={foldersError ? [] : folders}
            active={folder}
            onSelect={selectFolder}
            counts={counts}
          />
```

- [ ] **Step 5: Update `ProductsPage.tsx`**

Add the import next to the other folders imports (after the `SelectionBar` import, ~line 24):

```tsx
import { FolderActionsBar } from '@/features/folders/FolderActionsBar'
```

Add `activeFolderId` to the `useFolders` destructure (it is not currently pulled out).
Change the destructure list (~lines 56-70) to include it, e.g. after `folder,`:

```tsx
    folders,
    foldersError,
    folder,
    activeFolderId,
    selectFolder,
```

After the `useBulkDelete({...})` block (~line 84), add:

```tsx
  const activeFolder = activeFolderId ? folders?.find((f) => f.id === activeFolderId) : undefined
```

In the non-selection branch, insert the bar as the first child of the `<>` fragment
(before the search `<div className="relative w-full max-w-56">`, ~line 143):

```tsx
          <>
            {activeFolder && (
              <FolderActionsBar
                key={activeFolder.id}
                site={site}
                section="products"
                folder={activeFolder}
                itemCount={counts.byFolder.get(activeFolder.id) ?? 0}
                itemNoun="product"
                contentKey={productsKey}
                onDeleted={() => selectFolder('all')}
              />
            )}
            <div className="relative w-full max-w-56">
```

In the `<FoldersPanel .../>` call (~lines 218-227), remove `itemNoun="product"` and
`contentKey={productsKey}`:

```tsx
          <FoldersPanel
            site={site}
            section="products"
            folders={foldersError ? [] : folders}
            active={folder}
            onSelect={selectFolder}
            counts={counts}
          />
```

- [ ] **Step 6: Update `PostsPage.tsx`**

Add the import next to the other folders imports (after the `SelectionBar` import, ~line 24):

```tsx
import { FolderActionsBar } from '@/features/folders/FolderActionsBar'
```

Add `activeFolderId` to the `useFolders` destructure (~lines 47-61), after `folder,`:

```tsx
    folders,
    foldersError,
    folder,
    activeFolderId,
    selectFolder,
```

After the `useBulkDelete({...})` block (~line 75), add:

```tsx
  const activeFolder = activeFolderId ? folders?.find((f) => f.id === activeFolderId) : undefined
```

In the non-selection branch, insert the bar as the first child of the `<>` fragment
(before the search `<div className="relative w-full max-w-56">`, ~line 128):

```tsx
          <>
            {activeFolder && (
              <FolderActionsBar
                key={activeFolder.id}
                site={site}
                section={section.folderSection}
                folder={activeFolder}
                itemCount={counts.byFolder.get(activeFolder.id) ?? 0}
                itemNoun="post"
                contentKey={postsKey}
                onDeleted={() => selectFolder('all')}
              />
            )}
            <div className="relative w-full max-w-56">
```

In the `<FoldersPanel .../>` call (~lines 186-195), remove `itemNoun="post"` and
`contentKey={postsKey}`:

```tsx
          <FoldersPanel
            site={site}
            section={section.folderSection}
            folders={foldersError ? [] : folders}
            active={folder}
            onSelect={selectFolder}
            counts={counts}
          />
```

- [ ] **Step 7: Run the panel test to verify it passes**

Run: `npm --prefix web.admin test -- src/features/folders/FoldersPanel.test.tsx`
Expected: PASS (3 tests).

- [ ] **Step 8: Type-check the whole project (catches any stale `FoldersPanel` prop usage)**

Run: `npm --prefix web.admin run build`
Expected: build succeeds. If tsc errors that `itemNoun`/`contentKey` do not exist on
`FoldersPanel` props, a call site still passes them — fix that page's `FoldersPanel` call.

- [ ] **Step 9: Commit**

```bash
cd web.admin && git add src/features/folders/FoldersPanel.tsx src/features/folders/FoldersPanel.test.tsx src/features/media/MediaPage.tsx src/features/products/ProductsPage.tsx src/features/posts/PostsPage.tsx && git commit -m "feat(folders): move rename/delete into FolderActionsBar, align row counters"
```

---

## Task 3: Full verification

**Files:** none (verification only).

- [ ] **Step 1: Run the full unit suite**

Run: `npm --prefix web.admin test`
Expected: all tests pass (including the two new folder test files).

- [ ] **Step 2: Lint**

Run: `npm --prefix web.admin run lint`
Expected: clean, aside from the 2 known shadcn warnings noted in `web.admin/AGENTS.md`.
If oxlint reports unused imports/vars in `FoldersPanel.tsx` (e.g. a leftover `Ellipsis`,
`DropdownMenu*`, `AlertDialog*`, `renameFolder`, or `deleteFolder` import), delete them.

- [ ] **Step 3: Build**

Run: `npm --prefix web.admin run build`
Expected: succeeds (also run in Task 2 Step 8; re-confirm after lint fixes).

- [ ] **Step 4: Manual check in the running app**

Run: `npm --prefix web.admin run dev`, open Media (and Products, Blog):
- Folder counters on real folders line up on the same right edge as `All` / `Unsorted`.
- Folder rows have no hover `…` menu.
- Opening a real folder shows the Rename/Delete toolbar next to `Select`; `All`/`Unsorted`
  show no toolbar.
- Rename updates the sidebar name; Delete asks to confirm, moves items to Unsorted, and
  resets the active folder to `All`.
- Entering multiselect (`Select`) hides the folder toolbar (SelectionBar takes the row).

- [ ] **Step 5 (optional): Propose e2e coverage**

If the user wants e2e, add a Playwright spec (`web.admin`, port 5173, no live Supabase per
the project registry) covering: open folder → rename via toolbar → delete via toolbar →
items land in Unsorted. Otherwise note this in the completion report as a suggestion.

---

## Self-Review

**Spec coverage:**
- Counter alignment → Task 2 Step 3 (folder rows become plain `FolderRowButton`, identical to `All`/`Unsorted`) + Task 3 Step 4 manual check. ✓
- Remove per-folder ellipsis menu → Task 2 Step 3 (dropdown removed) + `FoldersPanel.test.tsx` assertion. ✓
- Rename/Delete in a top toolbar next to `Select`, only for a real folder → Task 1 (`FolderActionsBar`) + Task 2 Steps 4-6 (rendered when `activeFolder` set, in the non-selection header). ✓
- Toolbar hidden during multiselect → guaranteed by rendering in the non-selection branch only; Task 3 Step 4 verifies. ✓
- Delete moves items to Unsorted + resets active folder → `onDeleted={() => selectFolder('all')}` + `contentKey` invalidation in `FolderActionsBar`. ✓
- Reuse existing mutations (`renameFolder`/`deleteFolder`/`createFolder` in `lib/folders.ts`) → yes, no lib changes. ✓

**Placeholder scan:** none — every code step contains complete code and exact commands.

**Type consistency:** `FolderActionsBar` prop names (`site`, `section`, `folder`, `itemCount`, `itemNoun`, `contentKey`, `onDeleted`) are identical across the component definition (Task 1) and all three call sites (Task 2). `FoldersPanel` `Props` after edit (`site`, `section`, `folders`, `active`, `onSelect`, `counts`) match every updated call site (no `itemNoun`/`contentKey`). `useFolders` already returns `activeFolderId`, `folders`, `counts`, `selectFolder` (verified in `useFolders.ts`).
