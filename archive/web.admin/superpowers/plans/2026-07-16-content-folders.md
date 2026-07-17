# Content Folders (Products/Blog) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Папки в разделах Products и Blog (та же механика, что в Media) + рефакторинг generic-частей папок в `src/features/folders/` с переводом Media на общий код.

**Architecture:** Спека — `docs/superpowers/specs/2026-07-16-content-folders-design.md` (утверждена). Миграция добавляет `folder_id` в `products`/`posts` (таблица `admin_folders` уже готова). `lib/folders.ts` получает generic `moveToFolder` (маппинг секция→таблица), `moveImages` из `lib/media.ts` удаляется. `FoldersPanel`/`MoveToFolderMenu` переезжают в `src/features/folders/` (панель обобщается пропсами `section`/`itemNoun`/`contentKey`), туда же — новый хук `useFolders` со всей обвязкой (query папок, effectiveFolder, счётчики, выборка, move-мутация). MediaPage переводится на хук БЕЗ изменения поведения (регресс-критерий). ProductsPage/PostsPage получают панель + чекбоксы в строках + bulk-бар. Правила базовой фичи — `docs/media-manager.md` §3 (прочитать перед началом).

**Tech Stack:** Vite + React 19 + TS, TanStack Query, shadcn/ui, Supabase через `getDb(site)` / MCP для миграции.

**Правила проекта (переопределяют шаблон плана):**
- Автотестов нет — каждая задача завершается `npm run lint && npm run build`; ручной e2e — Task 6.
- **НЕ коммитить без явного разрешения пользователя** — шагов commit нет; спросить в конце.
- Текст UI — английский.
- Миграцию продублировать в репозитории сайта cozycorner (напомнить пользователю).

---

## File Structure

| Файл | Ответственность |
|---|---|
| Миграция `add_content_folders` (MCP) | `products.folder_id`, `posts.folder_id` + индексы |
| Modify `src/lib/folders.ts` | `FolderSection` += products/posts; generic `moveToFolder` |
| Modify `src/lib/products.ts` | `folder_id` в `ProductListItem` + select |
| Modify `src/lib/posts.ts` | `folder_id` в `PostListItem` + select |
| Create `src/features/folders/MoveToFolderMenu.tsx` | Переезд из features/media без изменений |
| Create `src/features/folders/FoldersPanel.tsx` | Переезд + generic-пропсы section/itemNoun/contentKey |
| Create `src/features/folders/useFolders.ts` | Хук-обвязка (query папок, фильтр, счётчики, выборка, move) + `intersectSelected` |
| Delete `src/features/media/{FoldersPanel,MoveToFolderMenu}.tsx` | Заменены общими |
| Modify `src/features/media/MediaPage.tsx` | Перевод на useFolders, поведение без изменений |
| Modify `src/features/media/{MediaCard,MediaDetailsDialog}.tsx` | Только пути импортов |
| Modify `src/lib/media.ts` | Удаление `moveImages` |
| Modify `src/features/products/ProductsPage.tsx` | Панель + чекбоксы строк + bulk-бар |
| Modify `src/features/posts/PostsPage.tsx` | То же для блога |
| Modify `docs/{media-manager,products,blog}.md`, `CLAUDE.md` | Правила и статус |

`MediaGrid.tsx` импортирует только `FolderRow` из `@/lib/folders` — не трогаем. Редакторы товара/поста, пикеры (`ImagePickerDialog`, `ProductPickerDialog`) — не трогаем.

---

### Task 1: Миграция БД — `add_content_folders`

**Files:** нет файлов в этом репо (MCP; файл продублировать в репозитории сайта — напомнить в конце).

- [x] **Step 1.1: Применить миграцию через MCP `mcp__supabase__apply_migration`**

Имя: `add_content_folders`. SQL:

```sql
-- Папки для products/posts: та же admin_folders (section 'products' | 'posts'),
-- в контентные таблицы добавляется только folder_id. Сайт колонку не читает.
-- ВНИМАНИЕ: у products/posts есть публичное чтение (сайт) — folder_id виден
-- anon-выборкам; это просто uuid без чувствительных данных, принято осознанно.
-- Спека: web.admin/docs/superpowers/specs/2026-07-16-content-folders-design.md
alter table cozycorner.products
  add column folder_id uuid references cozycorner.admin_folders(id) on delete set null;
create index products_folder_id_idx on cozycorner.products (folder_id);

alter table cozycorner.posts
  add column folder_id uuid references cozycorner.admin_folders(id) on delete set null;
create index posts_folder_id_idx on cozycorner.posts (folder_id);
```

- [x] **Step 1.2: Проверка через MCP `mcp__supabase__execute_sql`**

```sql
select
  (select count(*) from information_schema.columns
    where table_schema = 'cozycorner' and table_name in ('products','posts')
      and column_name = 'folder_id') as folder_cols,
  (select count(*) from pg_indexes
    where schemaname = 'cozycorner'
      and indexname in ('products_folder_id_idx','posts_folder_id_idx')) as idx;
```

Expected: `folder_cols = 2`, `idx = 2`.

---

### Task 2: Слой данных — `folders.ts` / `products.ts` / `posts.ts` (только аддитивно)

**Files:**
- Modify: `src/lib/folders.ts`
- Modify: `src/lib/products.ts`
- Modify: `src/lib/posts.ts`

`moveImages` в `media.ts` НЕ трогать в этой задаче (её удаление — Task 3, вместе с переводом MediaPage на generic; иначе build красный между задачами).

- [x] **Step 2.1: `src/lib/folders.ts` — расширить секции и добавить `moveToFolder`**

Заменить строку

```ts
export type FolderSection = "media"; // позже: | "posts" | "products"
```

на

```ts
export type FolderSection = "media" | "products" | "posts";
```

В конец файла (после `deleteFolder`) добавить:

```ts
// Таблица контента секции: имена совпадают, колонка folder_id есть во всех
// (миграции add_admin_folders, add_content_folders).
const SECTION_TABLES: Record<FolderSection, string> = {
  media: "media",
  products: "products",
  posts: "posts",
};

// Generic-перемещение в папку: меняет ТОЛЬКО folder_id переданных строк.
// Для media это значит: path/ключ в Storage/публичный URL не трогаются.
export async function moveToFolder(
  site: SiteConfig,
  section: FolderSection,
  ids: string[],
  folderId: string | null,
): Promise<void> {
  if (!ids.length) return;
  const { error } = await getDb(site)
    .from(SECTION_TABLES[section])
    .update({ folder_id: folderId })
    .in("id", ids);
  if (error) throw error;
}
```

- [x] **Step 2.2: `src/lib/products.ts` — `folder_id` в списке**

Тип `ProductListItem` (папка — свойство только списка; в `Product`/`ProductInput` НЕ добавлять, чтобы редактор её не затирал). Было:

```ts
export type ProductListItem = Pick<
  Product,
  "id" | "title" | "created_at" | "brand" | "category" | "image_path"
>;
```

Стало:

```ts
export type ProductListItem = Pick<
  Product,
  "id" | "title" | "created_at" | "brand" | "category" | "image_path"
> & {
  // Папка админки (см. docs/media-manager.md §3); редакторы поле не трогают.
  folder_id: string | null;
};
```

В `listProducts` select. Было:

```ts
    .select("id,title,created_at,brand,category,image_path")
```

Стало:

```ts
    .select("id,title,created_at,brand,category,image_path,folder_id")
```

- [x] **Step 2.3: `src/lib/posts.ts` — `folder_id` в списке**

Было:

```ts
export type PostListItem = Pick<Post, "id" | "title" | "published_at" | "is_published">;
```

Стало:

```ts
export type PostListItem = Pick<Post, "id" | "title" | "published_at" | "is_published"> & {
  // Папка админки (см. docs/media-manager.md §3); редактор поле не трогает.
  folder_id: string | null;
};
```

В `listPosts` select. Было:

```ts
    .select("id,title,published_at,is_published")
```

Стало:

```ts
    .select("id,title,published_at,is_published,folder_id")
```

- [x] **Step 2.4: Проверка**

Run: `npm run lint && npm run build`
Expected: успех (все изменения аддитивные; `moveToFolder` ещё не используется).

---

### Task 3: Рефакторинг — `src/features/folders/` + перевод Media

**Files:**
- Create: `src/features/folders/MoveToFolderMenu.tsx` (перенос без изменений содержимого)
- Create: `src/features/folders/FoldersPanel.tsx` (перенос + generic-пропсы)
- Create: `src/features/folders/useFolders.ts`
- Delete: `src/features/media/MoveToFolderMenu.tsx`, `src/features/media/FoldersPanel.tsx`
- Modify: `src/features/media/MediaPage.tsx` (перевод на хук, поведение НЕ меняется)
- Modify: `src/features/media/MediaCard.tsx`, `src/features/media/MediaDetailsDialog.tsx` (только пути импортов)
- Modify: `src/lib/media.ts` (удалить `moveImages`)

Build собирается только после всех шагов — выполнять одной сессией, проверка в конце.

- [x] **Step 3.1: Перенести `MoveToFolderMenu.tsx`**

Создать `src/features/folders/MoveToFolderMenu.tsx` с СОДЕРЖИМЫМ ТЕКУЩЕГО `src/features/media/MoveToFolderMenu.tsx` без изменений (прочитать и скопировать), затем удалить старый файл.

- [x] **Step 3.2: Создать `src/features/folders/FoldersPanel.tsx` (обобщённый) и удалить старый**

Взять текущий `src/features/media/FoldersPanel.tsx` и внести ровно эти изменения (остальное — без изменений):

1. Импорт из `@/lib/folders` дополнить типом секции:

```tsx
import {
  createFolder,
  deleteFolder,
  foldersKey,
  renameFolder,
  type FolderRow,
  type FolderSection,
} from '@/lib/folders'
```

2. Props и сигнатура. Было:

```tsx
type Props = {
  site: SiteConfig
  folders: FolderRow[] | undefined // undefined — ещё грузятся
  active: FolderFilter
  onSelect: (filter: FolderFilter) => void
  counts: Counts
}

export function FoldersPanel({ site, folders, active, onSelect, counts }: Props) {
  const queryClient = useQueryClient()
  const key = foldersKey(site, 'media')
```

Стало:

```tsx
type Props = {
  site: SiteConfig
  section: FolderSection
  itemNoun: string // 'image' | 'product' | 'post' — тексты delete-диалога
  contentKey: readonly unknown[] // ключ контента раздела: инвалидация при delete
  folders: FolderRow[] | undefined // undefined — ещё грузятся
  active: FolderFilter
  onSelect: (filter: FolderFilter) => void
  counts: Counts
}

export function FoldersPanel({
  site,
  section,
  itemNoun,
  contentKey,
  folders,
  active,
  onSelect,
  counts,
}: Props) {
  const queryClient = useQueryClient()
  const key = foldersKey(site, section)
```

3. В `create`-мутации: `createFolder(site, section, name)` вместо `createFolder(site, 'media', name)`.

4. В `remove.onSuccess` заменить

```tsx
      // FK on delete set null: файлы папки стали Unsorted — обновить и грид.
      queryClient.invalidateQueries({ queryKey: ['media', site.slug] })
```

на

```tsx
      // FK on delete set null: элементы папки стали Unsorted — обновить контент.
      queryClient.invalidateQueries({ queryKey: contentKey })
```

5. `aria-label` панели. Было `aria-label="Media folders"` → стало `aria-label="Folders"`.

6. Текст delete-диалога. Было:

```tsx
              {deleteCount > 0
                ? `${deleteCount} image${deleteCount === 1 ? '' : 's'} will be moved to Unsorted. `
                : 'The folder is empty. '}
              Files themselves are not deleted.
```

Стало:

```tsx
              {deleteCount > 0
                ? `${deleteCount} ${itemNoun}${deleteCount === 1 ? '' : 's'} will be moved to Unsorted. `
                : 'The folder is empty. '}
              The {itemNoun}s themselves are not deleted.
```

Затем удалить `src/features/media/FoldersPanel.tsx`.

- [x] **Step 3.3: Создать `src/features/folders/useFolders.ts` целиком**

```ts
import { useMemo, useState } from 'react'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import {
  foldersKey,
  listFolders,
  moveToFolder,
  type FolderSection,
} from '@/lib/folders'
import type { FolderFilter } from './FoldersPanel'

// Минимум, который хук требует от элемента раздела.
export type FolderedItem = { id: string; folder_id: string | null }

type Options = {
  site: SiteConfig
  section: FolderSection
  items: readonly FolderedItem[] | undefined
  // query-ключ контента раздела — инвалидируется после перемещения
  contentKey: readonly unknown[]
  // 'image' | 'product' | 'post' — тексты тостов
  itemNoun: string
}

// Вся обвязка папок для страницы раздела: query папок, фильтр (+fallback на All,
// если активную папку удалил второй админ в другой сессии), счётчики из уже
// загруженного списка, выборка и move-мутация. Страница сама считает
// visibleItems (семантика поиска/фильтров у разделов своя) и пересекает
// выборку через intersectSelected.
export function useFolders({ site, section, items, contentKey, itemNoun }: Options) {
  const queryClient = useQueryClient()
  const [folder, setFolder] = useState<FolderFilter>('all')
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set())

  const { data: folders, error: foldersError } = useQuery({
    queryKey: foldersKey(site, section),
    queryFn: () => listFolders(site, section),
  })

  const effectiveFolder: FolderFilter =
    folder === 'all' || folder === 'unsorted' || folders?.some((f) => f.id === folder)
      ? folder
      : 'all'

  const counts = useMemo(() => {
    const byFolder = new Map<string, number>()
    let unsorted = 0
    for (const item of items ?? []) {
      if (item.folder_id) {
        byFolder.set(item.folder_id, (byFolder.get(item.folder_id) ?? 0) + 1)
      } else {
        unsorted++
      }
    }
    return { all: items?.length ?? 0, unsorted, byFolder }
  }, [items])

  const move = useMutation({
    mutationFn: ({ ids, folderId }: { ids: string[]; folderId: string | null }) =>
      moveToFolder(site, section, ids, folderId),
    onSuccess: (_, { ids }) => {
      queryClient.invalidateQueries({ queryKey: contentKey })
      // Точечно: одиночный move не должен сбрасывать остальную выборку.
      setSelectedIds((prev) => {
        if (!prev.size) return prev
        const next = new Set(prev)
        ids.forEach((id) => next.delete(id))
        return next
      })
      const capitalized = itemNoun.charAt(0).toUpperCase() + itemNoun.slice(1)
      toast.success(
        ids.length === 1 ? `${capitalized} moved` : `${ids.length} ${itemNoun}s moved`,
      )
    },
    onError: (e) =>
      toast.error(e instanceof Error ? e.message : `Failed to move ${itemNoun}s`),
  })

  // Смена папки сбрасывает выборку: скрытые чекбоксы не должны незаметно
  // участвовать в bulk move.
  function selectFolder(next: FolderFilter) {
    setFolder(next)
    setSelectedIds(new Set())
  }

  function toggleSelect(id: string) {
    setSelectedIds((prev) => {
      const next = new Set(prev)
      if (next.has(id)) next.delete(id)
      else next.add(id)
      return next
    })
  }

  function clearSelection() {
    setSelectedIds(new Set())
  }

  // Для delete-флоу раздела: удалённый элемент не должен остаться в выборке.
  function removeFromSelection(id: string) {
    setSelectedIds((prev) => {
      if (!prev.has(id)) return prev
      const next = new Set(prev)
      next.delete(id)
      return next
    })
  }

  // true — элемент попадает в активную папку (для visibleItems страницы).
  function matchesFolder(item: FolderedItem): boolean {
    if (effectiveFolder === 'all') return true
    if (effectiveFolder === 'unsorted') return item.folder_id === null
    return item.folder_id === effectiveFolder
  }

  return {
    folders,
    foldersError,
    folder: effectiveFolder,
    // id активной папки (для upload/create в неё); All/Unsorted → null
    activeFolderId:
      effectiveFolder === 'all' || effectiveFolder === 'unsorted' ? null : effectiveFolder,
    selectFolder,
    counts,
    matchesFolder,
    selectedIds,
    toggleSelect,
    clearSelection,
    removeFromSelection,
    moveItems: (ids: string[], folderId: string | null) => move.mutate({ ids, folderId }),
    isMoving: move.isPending,
  }
}

// Пересечение выборки с видимыми элементами: bulk-бар не должен двигать скрытое.
export function intersectSelected(
  selectedIds: Set<string>,
  visibleItems: readonly { id: string }[] | undefined,
): Set<string> {
  if (!selectedIds.size) return selectedIds
  const visible = new Set((visibleItems ?? []).map((i) => i.id))
  return new Set([...selectedIds].filter((id) => visible.has(id)))
}
```

- [x] **Step 3.4: Переписать `src/features/media/MediaPage.tsx` целиком (перевод на хук)**

Поведение идентично текущему — это регресс-критерий.

```tsx
import { useDeferredValue, useMemo, useRef, useState } from 'react'
import { useOutletContext } from 'react-router-dom'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { FolderInput, Loader2, Search, Upload } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import { deleteImage, listImages, uploadImage, type MediaItem } from '@/lib/media'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { FoldersPanel } from '@/features/folders/FoldersPanel'
import { MoveToFolderMenu } from '@/features/folders/MoveToFolderMenu'
import { intersectSelected, useFolders } from '@/features/folders/useFolders'
import { MediaGrid } from './MediaGrid'

const ACCEPT = 'image/jpeg,image/png,image/webp,image/gif'

export function MediaPage() {
  const site = useOutletContext<SiteConfig>()
  const queryClient = useQueryClient()
  const fileInputRef = useRef<HTMLInputElement>(null)
  const mediaKey = ['media', site.slug]

  const [search, setSearch] = useState('')
  const deferredSearch = useDeferredValue(search)
  const isFiltering = deferredSearch !== search

  const { data: items, isPending, error } = useQuery({
    queryKey: mediaKey,
    queryFn: () => listImages(site),
  })

  const {
    folders,
    foldersError,
    folder,
    activeFolderId,
    selectFolder,
    counts,
    matchesFolder,
    selectedIds,
    toggleSelect,
    clearSelection,
    removeFromSelection,
    moveItems,
    isMoving,
  } = useFolders({ site, section: 'media', items, contentKey: mediaKey, itemNoun: 'image' })

  // Поиск идёт по всем папкам (игнорирует активную): нашёл — увидел файл,
  // в какой бы папке он ни лежал. Без поиска грид фильтруется активной папкой.
  const query = deferredSearch.trim().toLowerCase()
  const visibleItems = query
    ? items?.filter(
        (item) =>
          item.original_name.toLowerCase().includes(query) ||
          item.path.toLowerCase().includes(query),
      )
    : items?.filter(matchesFolder)

  const visibleSelectedIds = useMemo(
    () => intersectSelected(selectedIds, visibleItems),
    [selectedIds, visibleItems],
  )

  // Файлы грузим последовательно: ошибка одного не прерывает остальные.
  // Файл кладётся в открытую сейчас папку (All/Unsorted — без папки).
  const upload = useMutation({
    mutationFn: async (files: File[]) => {
      const failed: string[] = []
      let uploaded = 0
      for (const file of files) {
        try {
          await uploadImage(site, file, activeFolderId)
          uploaded++
        } catch (e) {
          failed.push(e instanceof Error ? e.message : `"${file.name}": upload failed`)
        }
      }
      return { uploaded, failed }
    },
    onSuccess: ({ uploaded, failed }) => {
      if (uploaded > 0) {
        queryClient.invalidateQueries({ queryKey: mediaKey })
        toast.success(uploaded === 1 ? 'Image uploaded' : `${uploaded} images uploaded`)
      }
      failed.forEach((message) => toast.error(message))
    },
  })

  const remove = useMutation({
    mutationFn: (item: MediaItem) => deleteImage(site, item),
    onSuccess: (_, item) => {
      queryClient.invalidateQueries({ queryKey: mediaKey })
      // Удалённый файл не должен остаться в выборке (иначе фантом в bulk move).
      removeFromSelection(item.id)
      toast.success('Image deleted')
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to delete image'),
  })

  function handleFilesSelected(fileList: FileList | null) {
    const files = Array.from(fileList ?? [])
    if (files.length) upload.mutate(files)
  }

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center gap-3">
        <h1 className="text-lg font-semibold">Media</h1>
        <div className="relative ml-auto w-full max-w-56">
          {isFiltering ? (
            <Loader2 className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 animate-spin text-muted-foreground" />
          ) : (
            <Search className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 text-muted-foreground" />
          )}
          <Input
            type="search"
            placeholder="Search by name…"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="h-9 pl-8"
          />
        </div>
        <input
          ref={fileInputRef}
          type="file"
          accept={ACCEPT}
          multiple
          className="hidden"
          onChange={(e) => {
            handleFilesSelected(e.target.files)
            e.target.value = '' // позволить выбрать тот же файл повторно
          }}
        />
        <Button onClick={() => fileInputRef.current?.click()} disabled={upload.isPending}>
          <Upload />
          {upload.isPending ? 'Uploading…' : 'Upload'}
        </Button>
      </div>

      {(error || foldersError) && (
        <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
          {error
            ? `Failed to load media: ${error.message}`
            : `Failed to load folders: ${foldersError?.message}`}
        </p>
      )}

      <div className="flex items-start gap-6">
        <aside className="w-44 shrink-0 lg:w-52">
          <FoldersPanel
            site={site}
            section="media"
            itemNoun="image"
            contentKey={mediaKey}
            folders={foldersError ? [] : folders}
            active={folder}
            onSelect={selectFolder}
            counts={counts}
          />
        </aside>

        <div className="flex min-w-0 flex-1 flex-col gap-3">
          {visibleSelectedIds.size > 0 && (
            <div className="flex items-center gap-3 rounded-lg border bg-muted/40 px-3 py-1.5">
              <span className="text-sm">{visibleSelectedIds.size} selected</span>
              <MoveToFolderMenu
                folders={folders ?? []}
                onMove={(folderId) => moveItems([...visibleSelectedIds], folderId)}
              >
                <Button variant="outline" size="sm" disabled={isMoving}>
                  <FolderInput />
                  {isMoving ? 'Moving…' : 'Move to…'}
                </Button>
              </MoveToFolderMenu>
              <Button variant="ghost" size="sm" onClick={clearSelection}>
                Clear
              </Button>
            </div>
          )}

          <div className={isFiltering ? 'opacity-60 transition-opacity' : 'transition-opacity'}>
            <MediaGrid
              site={site}
              items={visibleItems}
              isPending={isPending}
              isSearching={Boolean(query)}
              isFolderFiltered={folder !== 'all'}
              folders={folders ?? []}
              selectedIds={selectedIds}
              onToggleSelect={(item) => toggleSelect(item.id)}
              onMove={(item, folderId) => moveItems([item.id], folderId)}
              onDelete={(item) => remove.mutate(item)}
              deletingId={remove.isPending ? remove.variables.id : null}
            />
          </div>
        </div>
      </div>
    </div>
  )
}
```

- [x] **Step 3.5: Пути импортов в `MediaCard.tsx` и `MediaDetailsDialog.tsx`**

В обоих файлах заменить

```tsx
import { MoveToFolderMenu } from './MoveToFolderMenu'
```

на

```tsx
import { MoveToFolderMenu } from '@/features/folders/MoveToFolderMenu'
```

(другие изменения не нужны; `MediaGrid.tsx` не трогаем).

- [x] **Step 3.6: Удалить `moveImages` из `src/lib/media.ts`**

Удалить целиком блок (комментарий + функция):

```ts
// Перемещение в папку меняет только media.folder_id: path, ключ в Storage и
// публичный URL не трогаются — ссылки на сайте не ломаются (спека папок).
export async function moveImages(
  site: SiteConfig,
  ids: string[],
  folderId: string | null,
): Promise<void> {
  if (!ids.length) return;
  const { error } = await getDb(site)
    .from("media")
    .update({ folder_id: folderId })
    .in("id", ids);
  if (error) throw error;
}
```

(generic-замена — `moveToFolder(site, 'media', …)` в `lib/folders.ts`; MediaPage уже переведён в Step 3.4).

- [x] **Step 3.7: Проверка**

Run: `npm run lint && npm run build`
Expected: lint — только 2 известных shadcn-warning; build — успех. Также `grep -rn "features/media/FoldersPanel\|features/media/MoveToFolderMenu\|moveImages" src` — пусто.

---

### Task 4: ProductsPage — панель, чекбоксы, bulk-бар

**Files:**
- Modify: `src/features/products/ProductsPage.tsx` (полная замена содержимого)

- [x] **Step 4.1: Переписать `src/features/products/ProductsPage.tsx` целиком**

```tsx
import { useDeferredValue, useMemo, useState } from 'react'
import { Link, useOutletContext } from 'react-router-dom'
import { useQuery } from '@tanstack/react-query'
import { FolderInput, Loader2, Plus, Search, X } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { listCategoryNames, listProducts } from '@/lib/products'
import { Button } from '@/components/ui/button'
import { Checkbox } from '@/components/ui/checkbox'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
import { FoldersPanel } from '@/features/folders/FoldersPanel'
import { MoveToFolderMenu } from '@/features/folders/MoveToFolderMenu'
import { intersectSelected, useFolders } from '@/features/folders/useFolders'

const ALL = '__all__' // radix SelectItem не принимает пустое value

// Список товаров: одна колонка строк, поиск по названию + фильтры Category/Brand,
// слева — панель папок (правила — docs/media-manager.md §3).
export function ProductsPage() {
  const site = useOutletContext<SiteConfig>()

  const [search, setSearch] = useState('')
  const [category, setCategory] = useState(ALL)
  const [brand, setBrand] = useState(ALL)
  const deferredSearch = useDeferredValue(search)
  const isFiltering = deferredSearch !== search

  const productsKey = ['products', site.slug]
  const { data: products, isPending, error } = useQuery({
    queryKey: productsKey,
    queryFn: () => listProducts(site),
  })

  // Категории — полный справочник сайта (как в форме товара, общий кэш).
  const { data: categories } = useQuery({
    queryKey: ['categories', site.slug],
    queryFn: () => listCategoryNames(site),
  })

  const {
    folders,
    foldersError,
    folder,
    selectFolder,
    counts,
    matchesFolder,
    selectedIds,
    toggleSelect,
    clearSelection,
    moveItems,
    isMoving,
  } = useFolders({
    site,
    section: 'products',
    items: products,
    contentKey: productsKey,
    itemNoun: 'product',
  })

  // Бренды — только реально встречающиеся в товарах.
  const brands = useMemo(() => {
    const unique = new Set<string>()
    for (const p of products ?? []) if (p.brand) unique.add(p.brand)
    return [...unique].sort((a, b) => a.localeCompare(b))
  }, [products])

  const query = deferredSearch.trim().toLowerCase()
  const hasFilter = Boolean(query) || category !== ALL || brand !== ALL
  const folderFiltered = folder !== 'all'
  // Поиск — сквозь все папки (как в media); селекты уточняют ВНУТРИ активной
  // папки (AND). Clear чистит поиск/селекты, папку не трогает.
  const visible = (products ?? []).filter(
    (p) =>
      (query ? p.title.toLowerCase().includes(query) : matchesFolder(p)) &&
      (category === ALL || p.category === category) &&
      (brand === ALL || p.brand === brand),
  )

  const visibleSelectedIds = useMemo(
    () => intersectSelected(selectedIds, visible),
    [selectedIds, visible],
  )

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center gap-3">
        <h1 className="text-lg font-semibold">Products</h1>
        {products && (
          <span className="text-sm text-muted-foreground">
            {hasFilter || folderFiltered
              ? `${visible.length} / ${products.length}`
              : products.length}
          </span>
        )}
        <Button asChild className="ml-auto">
          <Link to="new">
            <Plus />
            New product
          </Link>
        </Button>
      </div>

      <div className="flex flex-wrap items-center gap-2">
        <div className="relative w-full max-w-56">
          {isFiltering ? (
            <Loader2 className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 animate-spin text-muted-foreground" />
          ) : (
            <Search className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 text-muted-foreground" />
          )}
          <Input
            type="search"
            placeholder="Search by title…"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="h-9 pl-8"
          />
        </div>
        <Select value={brand} onValueChange={setBrand}>
          <SelectTrigger className="h-9 w-44" aria-label="Filter by brand">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value={ALL}>All brands</SelectItem>
            {brands.map((name) => (
              <SelectItem key={name} value={name}>
                {name}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>
        <Select value={category} onValueChange={setCategory}>
          <SelectTrigger className="h-9 w-44" aria-label="Filter by category">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value={ALL}>All categories</SelectItem>
            {(categories ?? []).map((name) => (
              <SelectItem key={name} value={name}>
                {name}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>
        {hasFilter && (
          <Button
            variant="ghost"
            className="h-9"
            onClick={() => {
              setSearch('')
              setCategory(ALL)
              setBrand(ALL)
            }}
          >
            <X />
            Clear
          </Button>
        )}
      </div>

      {(error || foldersError) && (
        <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
          {error
            ? `Failed to load products: ${error.message}`
            : `Failed to load folders: ${foldersError?.message}`}
        </p>
      )}

      <div className="flex items-start gap-6">
        <aside className="w-44 shrink-0 lg:w-52">
          <FoldersPanel
            site={site}
            section="products"
            itemNoun="product"
            contentKey={productsKey}
            folders={foldersError ? [] : folders}
            active={folder}
            onSelect={selectFolder}
            counts={counts}
          />
        </aside>

        <div className="flex min-w-0 flex-1 flex-col gap-3">
          {visibleSelectedIds.size > 0 && (
            <div className="flex items-center gap-3 rounded-lg border bg-muted/40 px-3 py-1.5">
              <span className="text-sm">{visibleSelectedIds.size} selected</span>
              <MoveToFolderMenu
                folders={folders ?? []}
                onMove={(folderId) => moveItems([...visibleSelectedIds], folderId)}
              >
                <Button variant="outline" size="sm" disabled={isMoving}>
                  <FolderInput />
                  {isMoving ? 'Moving…' : 'Move to…'}
                </Button>
              </MoveToFolderMenu>
              <Button variant="ghost" size="sm" onClick={clearSelection}>
                Clear
              </Button>
            </div>
          )}

          {isPending && !error && (
            <div className="flex flex-col gap-1">
              {Array.from({ length: 8 }, (_, i) => (
                <Skeleton key={i} className="h-9 w-full rounded-md" />
              ))}
            </div>
          )}

          {products && products.length === 0 && (
            <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
              No products yet.
            </p>
          )}

          {products && products.length > 0 && visible.length === 0 && (
            <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
              {hasFilter ? 'No products match your filters.' : 'This folder is empty.'}
            </p>
          )}

          {visible.length > 0 && (
            <ul
              className={
                isFiltering
                  ? 'flex flex-col opacity-60 transition-opacity'
                  : 'flex flex-col transition-opacity'
              }
            >
              {visible.map((product) => (
                <li key={product.id} className="flex items-center gap-1">
                  <Checkbox
                    checked={selectedIds.has(product.id)}
                    onCheckedChange={() => toggleSelect(product.id)}
                    aria-label={`Select ${product.title}`}
                  />
                  <Link
                    to={product.id}
                    className="flex min-w-0 flex-1 items-baseline justify-between gap-3 rounded-md px-3 py-2 transition-colors hover:bg-accent/50"
                  >
                    <span className="truncate text-sm font-medium">{product.title}</span>
                    {(product.brand || product.category) && (
                      <span className="shrink-0 text-xs text-muted-foreground">
                        {[product.brand, product.category].filter(Boolean).join(' · ')}
                      </span>
                    )}
                  </Link>
                  <MoveToFolderMenu
                    folders={folders ?? []}
                    currentFolderId={product.folder_id}
                    onMove={(folderId) => moveItems([product.id], folderId)}
                  >
                    <Button
                      variant="ghost"
                      size="icon-sm"
                      aria-label={`Move ${product.title} to folder`}
                      className="size-7"
                    >
                      <FolderInput className="size-3.5" />
                    </Button>
                  </MoveToFolderMenu>
                </li>
              ))}
            </ul>
          )}
        </div>
      </div>
    </div>
  )
}
```

- [x] **Step 4.2: Проверка**

Run: `npm run lint && npm run build`
Expected: успех.

---

### Task 5: PostsPage — панель, чекбоксы, bulk-бар

**Files:**
- Modify: `src/features/posts/PostsPage.tsx` (полная замена содержимого)

- [x] **Step 5.1: Переписать `src/features/posts/PostsPage.tsx` целиком**

```tsx
import { useDeferredValue, useMemo, useState } from 'react'
import { Link, useOutletContext } from 'react-router-dom'
import { useQuery } from '@tanstack/react-query'
import { FolderInput, Loader2, Plus, Search, X } from 'lucide-react'
import type { SiteConfig } from '@/config/sites'
import { listPosts } from '@/lib/posts'
import { formatDate } from '@/lib/format'
import { Badge } from '@/components/ui/badge'
import { Button } from '@/components/ui/button'
import { Checkbox } from '@/components/ui/checkbox'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
import { FoldersPanel } from '@/features/folders/FoldersPanel'
import { MoveToFolderMenu } from '@/features/folders/MoveToFolderMenu'
import { intersectSelected, useFolders } from '@/features/folders/useFolders'

const ALL = '__all__' // radix SelectItem не принимает пустое value

// Список постов блога: строки-ссылки, поиск по заголовку + фильтр статуса,
// слева — панель папок (правила — docs/media-manager.md §3).
export function PostsPage() {
  const site = useOutletContext<SiteConfig>()

  const [search, setSearch] = useState('')
  const [status, setStatus] = useState(ALL)
  const deferredSearch = useDeferredValue(search)
  const isFiltering = deferredSearch !== search

  const postsKey = ['posts', site.slug]
  const { data: posts, isPending, error } = useQuery({
    queryKey: postsKey,
    queryFn: () => listPosts(site),
  })

  const {
    folders,
    foldersError,
    folder,
    selectFolder,
    counts,
    matchesFolder,
    selectedIds,
    toggleSelect,
    clearSelection,
    moveItems,
    isMoving,
  } = useFolders({
    site,
    section: 'posts',
    items: posts,
    contentKey: postsKey,
    itemNoun: 'post',
  })

  const query = deferredSearch.trim().toLowerCase()
  const hasFilter = Boolean(query) || status !== ALL
  const folderFiltered = folder !== 'all'
  // Поиск — сквозь все папки (как в media); фильтр статуса уточняет ВНУТРИ
  // активной папки (AND). Clear чистит поиск/статус, папку не трогает.
  const visible = (posts ?? []).filter(
    (p) =>
      (query ? p.title.toLowerCase().includes(query) : matchesFolder(p)) &&
      (status === ALL || p.is_published === (status === 'published')),
  )

  const visibleSelectedIds = useMemo(
    () => intersectSelected(selectedIds, visible),
    [selectedIds, visible],
  )

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center gap-3">
        <h1 className="text-lg font-semibold">Blog</h1>
        {posts && (
          <span className="text-sm text-muted-foreground">
            {hasFilter || folderFiltered ? `${visible.length} / ${posts.length}` : posts.length}
          </span>
        )}
        <Button asChild className="ml-auto">
          <Link to="new">
            <Plus />
            New post
          </Link>
        </Button>
      </div>

      <div className="flex flex-wrap items-center gap-2">
        <div className="relative w-full max-w-56">
          {isFiltering ? (
            <Loader2 className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 animate-spin text-muted-foreground" />
          ) : (
            <Search className="absolute top-1/2 left-2.5 size-4 -translate-y-1/2 text-muted-foreground" />
          )}
          <Input
            type="search"
            placeholder="Search by title…"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="h-9 pl-8"
          />
        </div>
        <Select value={status} onValueChange={setStatus}>
          <SelectTrigger className="h-9 w-44" aria-label="Filter by status">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value={ALL}>All statuses</SelectItem>
            <SelectItem value="published">Published</SelectItem>
            <SelectItem value="draft">Draft</SelectItem>
          </SelectContent>
        </Select>
        {hasFilter && (
          <Button
            variant="ghost"
            className="h-9"
            onClick={() => {
              setSearch('')
              setStatus(ALL)
            }}
          >
            <X />
            Clear
          </Button>
        )}
      </div>

      {(error || foldersError) && (
        <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
          {error
            ? `Failed to load posts: ${error.message}`
            : `Failed to load folders: ${foldersError?.message}`}
        </p>
      )}

      <div className="flex items-start gap-6">
        <aside className="w-44 shrink-0 lg:w-52">
          <FoldersPanel
            site={site}
            section="posts"
            itemNoun="post"
            contentKey={postsKey}
            folders={foldersError ? [] : folders}
            active={folder}
            onSelect={selectFolder}
            counts={counts}
          />
        </aside>

        <div className="flex min-w-0 flex-1 flex-col gap-3">
          {visibleSelectedIds.size > 0 && (
            <div className="flex items-center gap-3 rounded-lg border bg-muted/40 px-3 py-1.5">
              <span className="text-sm">{visibleSelectedIds.size} selected</span>
              <MoveToFolderMenu
                folders={folders ?? []}
                onMove={(folderId) => moveItems([...visibleSelectedIds], folderId)}
              >
                <Button variant="outline" size="sm" disabled={isMoving}>
                  <FolderInput />
                  {isMoving ? 'Moving…' : 'Move to…'}
                </Button>
              </MoveToFolderMenu>
              <Button variant="ghost" size="sm" onClick={clearSelection}>
                Clear
              </Button>
            </div>
          )}

          {isPending && !error && (
            <div className="flex flex-col gap-1">
              {Array.from({ length: 8 }, (_, i) => (
                <Skeleton key={i} className="h-9 w-full rounded-md" />
              ))}
            </div>
          )}

          {posts && posts.length === 0 && (
            <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
              No posts yet.
            </p>
          )}

          {posts && posts.length > 0 && visible.length === 0 && (
            <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
              {hasFilter ? 'No posts match your filters.' : 'This folder is empty.'}
            </p>
          )}

          {visible.length > 0 && (
            <ul
              className={
                isFiltering
                  ? 'flex flex-col opacity-60 transition-opacity'
                  : 'flex flex-col transition-opacity'
              }
            >
              {visible.map((post) => (
                <li key={post.id} className="flex items-center gap-1">
                  <Checkbox
                    checked={selectedIds.has(post.id)}
                    onCheckedChange={() => toggleSelect(post.id)}
                    aria-label={`Select ${post.title}`}
                  />
                  <Link
                    to={post.id}
                    className="flex min-w-0 flex-1 items-center justify-between gap-3 rounded-md px-3 py-2 transition-colors hover:bg-accent/50"
                  >
                    <span className="truncate text-sm font-medium">{post.title}</span>
                    <span className="flex shrink-0 items-center gap-2 text-xs text-muted-foreground">
                      {!post.is_published && <Badge variant="outline">Draft</Badge>}
                      {formatDate(post.published_at)}
                    </span>
                  </Link>
                  <MoveToFolderMenu
                    folders={folders ?? []}
                    currentFolderId={post.folder_id}
                    onMove={(folderId) => moveItems([post.id], folderId)}
                  >
                    <Button
                      variant="ghost"
                      size="icon-sm"
                      aria-label={`Move ${post.title} to folder`}
                      className="size-7"
                    >
                      <FolderInput className="size-3.5" />
                    </Button>
                  </MoveToFolderMenu>
                </li>
              ))}
            </ul>
          )}
        </div>
      </div>
    </div>
  )
}
```

- [x] **Step 5.2: Проверка**

Run: `npm run lint && npm run build`
Expected: успех.

---

### Task 6: Документация, статус, ручной e2e

**Files:**
- Modify: `docs/media-manager.md`
- Modify: `docs/products.md`
- Modify: `docs/blog.md`
- Modify: `CLAUDE.md`

- [x] **Step 6.1: Обновить `docs/media-manager.md` §3**

Актуализировать: компоненты живут в `src/features/folders/` (`FoldersPanel` — generic с пропсами `section`/`itemNoun`/`contentKey`, `MoveToFolderMenu`, хук `useFolders` + `intersectSelected` — обвязка страниц); папки работают в трёх разделах (media/products/posts — секции в `admin_folders`, `folder_id` в каждой таблице; миграции `add_admin_folders`, `add_content_folders`); `moveImages` заменён generic `moveToFolder(site, section, ids, folderId)`; ссылка на спеку `2026-07-16-content-folders-design.md`. Отметить: у products/posts публичное чтение — `folder_id` виден anon-выборкам (uuid без чувствительных данных, принято).

- [x] **Step 6.2: Дополнить `docs/products.md` и `docs/blog.md`**

В каждый — короткий раздел «Папки»: список раздела имеет панель папок/multi-select/bulk move; канонические правила и контракты — `docs/media-manager.md` §3; `folder_id` есть ТОЛЬКО в списочном типе (`ProductListItem`/`PostListItem`) — редакторы и payload'ы create/update поле не трогают (новые записи попадают в Unsorted); семантика фильтров: поиск сквозь папки, селекты (Brand/Category/Status) — AND с активной папкой, Clear папку не сбрасывает.

- [x] **Step 6.3: Обновить `CLAUDE.md`**

В «Сделано» добавить пункт: папки в Products/Blog + рефакторинг (2026-07-16) — миграция `add_content_folders`, generic `moveToFolder`, `src/features/folders/{FoldersPanel,MoveToFolderMenu,useFolders}` (media переведён на общий код, `moveImages` удалён), панель/multi-select/bulk в ProductsPage/PostsPage; правила — docs/media-manager.md §3. В «Следующие шаги»: продублировать миграцию `add_content_folders` в репо сайта (вместе с `add_admin_folders`); убрать пункт «папки для posts/products» (сделан), добавить ручной e2e по этому плану (Task 6.5). В «Структуре» добавить `src/features/folders/*`.

- [x] **Step 6.4: Проверка**

Run: `npm run lint && npm run build`
Expected: успех.

- [ ] **Step 6.5: Ручной e2e-чек-лист (в `npm run dev` под админом)**

1. **Регресс Media**: панель, счётчики, create/rename/delete папки, move с карточки/из модалки, bulk move, upload в открытую папку, поиск сквозь папки — всё как до рефакторинга.
2. Products: создать папку «Sale» — появляется только в products (в media её нет — секции независимы).
3. Products: move одного товара иконкой на строке → счётчики обновились, в папке «Sale» товар виден; у строки в меню «Move to» папка «Sale» отсутствует (текущая).
4. Products: multi-select 2–3 товаров → бар «N selected» → bulk move → выборка снята.
5. Products: селект Brand при активной папке фильтрует ВНУТРИ папки (AND); поиск находит товары из других папок; Clear сбрасывает поиск/селекты, папка остаётся.
6. Products: пустая папка → «This folder is empty.»; создание нового товара → попадает в Unsorted.
7. Blog: те же проверки (папка, move, bulk, фильтр Status AND, поиск сквозной, новый пост → Unsorted).
8. Blog: `ProductPickerDialog` в модуле товаров работает как раньше.
9. Удаление папки с товарами → «N products will be moved to Unsorted» → товары в Unsorted.
10. Сайт cozycorner (прод): товары/посты рендерятся как раньше (колонка аддитивная).

Спросить у пользователя разрешение на commit; напомнить продублировать миграцию `add_content_folders` в репозитории сайта.
