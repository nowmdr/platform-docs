# Media Folders Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Виртуальные папки в разделе Media: таблица `admin_folders` (+ `media.folder_id`), панель папок слева, перемещение через меню «Move to…» на карточке/в модалке и bulk move через multi-select.

**Architecture:** Спека — `docs/superpowers/specs/2026-07-16-media-folders-design.md` (утверждена). Папки — организационная сущность админки; сайт о них не знает, `media.path`/Storage/URL не меняются. Generic-таблица `admin_folders` с полем `section` (сейчас только `'media'`), в `media` — nullable FK `folder_id` c `on delete set null`. Слой данных: новый `src/lib/folders.ts` + `moveImages`/`folder_id` в `src/lib/media.ts`. UI: `FoldersPanel` (CRUD папок + фильтр), общий `MoveToFolderMenu`, чекбоксы multi-select на карточках, bulk-бар. Паттерны — из `docs/media-manager.md` (прочитать перед началом).

**Tech Stack:** Vite + React 19 + TS, TanStack Query, shadcn/ui (radix, добавляется `checkbox`), Supabase через `getDb(site)` / MCP для миграции.

**Правила проекта (переопределяют шаблон плана):**
- Автотестов в репозитории нет — вместо TDD каждая задача завершается `npm run lint && npm run build`; финальный ручной e2e-чек-лист — Task 6.
- **НЕ коммитить без явного разрешения пользователя** (CLAUDE.md) — в задачах нет шагов commit; в конце спросить разрешение.
- Текст UI — английский.
- Миграция применяется через MCP `apply_migration` к проекту base-one; **файл миграции обязательно продублировать в репозитории сайта cozycorner** (напомнить пользователю, как с `add_pages_body`).

---

## File Structure

| Файл | Ответственность |
|---|---|
| Миграция `add_admin_folders` (MCP) | Таблица `admin_folders` + RLS + grant, колонка `media.folder_id` + индекс |
| Create `src/lib/folders.ts` | Типы FolderRow/FolderSection, listFolders/createFolder/renameFolder/deleteFolder, маппинг 23505 |
| Modify `src/lib/media.ts` | `MediaRow.folder_id`, `uploadImage(..., folderId)`, `moveImages` |
| Create `src/components/ui/checkbox.tsx` | shadcn checkbox (генерируется CLI) |
| Create `src/features/media/MoveToFolderMenu.tsx` | Общее dropdown-меню «Move to…» (карточка, модалка, bulk-бар) |
| Create `src/features/media/FoldersPanel.tsx` | Панель папок: All/Unsorted/список + счётчики, create/rename/delete, тип FolderFilter |
| Modify `src/features/media/MediaPage.tsx` | Состояние папки/выборки, query папок, счётчики, фильтрация, bulk-бар, upload в папку, мутация move |
| Modify `src/features/media/MediaGrid.tsx` | Прокидка новых пропсов, empty state «This folder is empty.» |
| Modify `src/features/media/MediaCard.tsx` | Чекбокс выборки на превью, кнопка «Move to» в футере |
| Modify `src/features/media/MediaDetailsDialog.tsx` | Строка «Folder» + перемещение из модалки |
| Modify `docs/media-manager.md` | Правила раздела: папки |
| Modify `CLAUDE.md` | Статус |

`SiteLayout`, маршруты, `ImagePickerDialog` не трогаем (`uploadImage` получает параметр с дефолтом — вызов `uploadImage(site, file)` в пикере остаётся валидным; пикер папки не показывает — сознательно).

---

### Task 1: Миграция БД — `add_admin_folders`

**Files:** нет файлов в этом репо (миграция через MCP; файл дублируется в репозитории сайта — напомнить пользователю в конце).

- [x] **Step 1.1: Применить миграцию через MCP `mcp__supabase__apply_migration`**

Имя миграции: `add_admin_folders`. SQL:

```sql
-- Папки — организационная сущность самой админки: админ раскладывает контент,
-- сайт таблицу не читает. Одна таблица на схему, секции разделяет поле section
-- ('media'; позже 'posts' | 'products'). Спека:
-- web.admin/docs/superpowers/specs/2026-07-16-media-folders-design.md
create table cozycorner.admin_folders (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  section text not null,
  name text not null,
  unique (section, name)
);

alter table cozycorner.admin_folders enable row level security;

-- Как у media: весь доступ — только админам из whitelist, публичного чтения нет.
create policy "Admin all admin_folders" on cozycorner.admin_folders
  for all to authenticated
  using (public.is_admin())
  with check (public.is_admin());

-- Роли API (паттерн остальных таблиц схемы; реальный доступ ограничивает RLS).
grant all on table cozycorner.admin_folders to anon, authenticated, service_role;

-- Виртуальная папка файла: перемещение меняет только folder_id; path, ключ в
-- Storage и публичный URL не трогаются. Удаление папки отправляет файлы в
-- Unsorted (null), файлы не удаляются.
alter table cozycorner.media
  add column folder_id uuid references cozycorner.admin_folders(id) on delete set null;

create index media_folder_id_idx on cozycorner.media (folder_id);
```

- [x] **Step 1.2: Проверить результат через MCP `mcp__supabase__execute_sql`**

```sql
select
  (select count(*) from pg_policies
    where schemaname = 'cozycorner' and tablename = 'admin_folders') as policies,
  (select count(*) from information_schema.columns
    where table_schema = 'cozycorner' and table_name = 'media'
      and column_name = 'folder_id') as media_folder_col,
  (select count(*) from cozycorner.admin_folders) as folders;
```

Expected: `policies = 1`, `media_folder_col = 1`, `folders = 0`.

- [x] **Step 1.3: Anon-проверка RLS (curl, ключ взять из `src/config/sites.ts`)**

```bash
curl -s "https://zwrkphynupdubevzwdzy.supabase.co/rest/v1/admin_folders?select=*" \
  -H "apikey: <anonKey из sites.ts>" -H "Accept-Profile: cozycorner"
```

Expected: `[]` (пустой массив — RLS отрезает анонима; как у `media`).

---

### Task 2: shadcn checkbox

**Files:**
- Create: `src/components/ui/checkbox.tsx` (генерирует CLI)

- [x] **Step 2.1: Добавить компонент**

Run: `npx shadcn@latest add checkbox`
Expected: создан `src/components/ui/checkbox.tsx` с импортом из `"radix-ui"` (unified-пакет уже в зависимостях). Другие файлы не изменены (если CLI спросит про перезапись — отказаться).

- [x] **Step 2.2: Проверка**

Run: `npm run lint && npm run build`
Expected: lint — только 2 известных warning в shadcn-файлах (возможен новый warning в checkbox.tsx — тоже игнорируем, если это стандартный код CLI); build — успех.

---

### Task 3: Слой данных — `src/lib/folders.ts` + правки `src/lib/media.ts`

**Files:**
- Create: `src/lib/folders.ts`
- Modify: `src/lib/media.ts`

- [x] **Step 3.1: Создать `src/lib/folders.ts` целиком**

```ts
import type { SiteConfig } from "@/config/sites";
import { getDb } from "@/lib/supabase";

// Папки — организационная сущность самой админки; сайт о них не знает
// (спека docs/superpowers/specs/2026-07-16-media-folders-design.md).
// Одна таблица admin_folders на схему сайта, секции разделяет поле section.
export type FolderSection = "media"; // позже: | "posts" | "products"

export type FolderRow = {
  id: string;
  created_at: string;
  section: FolderSection;
  name: string;
};

// unique (section, name) в БД; 23505 переводим в понятное сообщение.
function friendlyError(error: { code?: string; message: string }): Error {
  if (error.code === "23505") {
    return new Error("Folder with this name already exists");
  }
  return new Error(error.message);
}

export async function listFolders(
  site: SiteConfig,
  section: FolderSection,
): Promise<FolderRow[]> {
  const { data, error } = await getDb(site)
    .from("admin_folders")
    .select("*")
    .eq("section", section)
    .order("name", { ascending: true });
  if (error) throw error;
  return (data ?? []) as FolderRow[];
}

export async function createFolder(
  site: SiteConfig,
  section: FolderSection,
  name: string,
): Promise<FolderRow> {
  const trimmed = name.trim();
  if (!trimmed) throw new Error("Folder name cannot be empty");
  const { data, error } = await getDb(site)
    .from("admin_folders")
    .insert({ section, name: trimmed })
    .select()
    .single();
  if (error) throw friendlyError(error);
  return data as FolderRow;
}

export async function renameFolder(
  site: SiteConfig,
  id: string,
  name: string,
): Promise<void> {
  const trimmed = name.trim();
  if (!trimmed) throw new Error("Folder name cannot be empty");
  const { error } = await getDb(site)
    .from("admin_folders")
    .update({ name: trimmed })
    .eq("id", id);
  if (error) throw friendlyError(error);
}

// Файлы папки трогать не нужно: FK media.folder_id — on delete set null,
// БД сама отправляет их в Unsorted.
export async function deleteFolder(site: SiteConfig, id: string): Promise<void> {
  const { error } = await getDb(site).from("admin_folders").delete().eq("id", id);
  if (error) throw error;
}
```

- [x] **Step 3.2: `src/lib/media.ts` — добавить `folder_id` в `MediaRow`**

В типе `MediaRow` после `size: number | null;` добавить строку:

```ts
  folder_id: string | null;
```

- [x] **Step 3.3: `src/lib/media.ts` — upload в папку**

Заменить сигнатуру и insert в `uploadImage`. Было:

```ts
export async function uploadImage(site: SiteConfig, file: File): Promise<MediaRow> {
```

Стало (параметр с дефолтом — существующий вызов в `ImagePickerDialog` остаётся валидным):

```ts
export async function uploadImage(
  site: SiteConfig,
  file: File,
  folderId: string | null = null,
): Promise<MediaRow> {
```

В insert добавить `folder_id`. Было:

```ts
    .insert({ original_name: file.name, path: key, mime: file.type, size: file.size })
```

Стало:

```ts
    .insert({
      original_name: file.name,
      path: key,
      mime: file.type,
      size: file.size,
      folder_id: folderId,
    })
```

- [x] **Step 3.4: `src/lib/media.ts` — добавить `moveImages` (после `renameImage`, перед `deleteImage`)**

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

- [x] **Step 3.5: Проверка**

Run: `npm run lint && npm run build`
Expected: успех (новый модуль ещё не используется — это нормально).

---

### Task 4: Компоненты — `MoveToFolderMenu.tsx` и `FoldersPanel.tsx`

**Files:**
- Create: `src/features/media/MoveToFolderMenu.tsx`
- Create: `src/features/media/FoldersPanel.tsx`

- [x] **Step 4.1: Создать `src/features/media/MoveToFolderMenu.tsx` целиком**

```tsx
import type { ReactNode } from 'react'
import type { FolderRow } from '@/lib/folders'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuLabel,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'

type Props = {
  folders: FolderRow[]
  // Текущая папка файла (null — Unsorted) исключается из целей.
  // undefined (bulk-бар: выборка может быть из разных папок) — показать все цели.
  currentFolderId?: string | null
  onMove: (folderId: string | null) => void
  children: ReactNode // триггер (asChild)
}

// Общее меню «Move to…»: карточка, модалка деталей, bulk-бар.
export function MoveToFolderMenu({ folders, currentFolderId, onMove, children }: Props) {
  const targets: { id: string | null; name: string }[] = [
    { id: null, name: 'Unsorted' },
    ...folders.map((f) => ({ id: f.id as string | null, name: f.name })),
  ].filter((t) => currentFolderId === undefined || t.id !== currentFolderId)

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>{children}</DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuLabel>Move to</DropdownMenuLabel>
        {targets.map((target) => (
          <DropdownMenuItem key={target.id ?? 'unsorted'} onClick={() => onMove(target.id)}>
            {target.name}
          </DropdownMenuItem>
        ))}
        {targets.length === 0 && <DropdownMenuItem disabled>No folders yet</DropdownMenuItem>}
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

- [x] **Step 4.2: Создать `src/features/media/FoldersPanel.tsx` целиком**

Инлайн-паттерны create/rename — как rename картинки в `MediaDetailsDialog` (Enter/Esc, Check/X). Мутации CRUD папок живут здесь; move-мутация — НЕ здесь (в `MediaPage`, Task 5).

```tsx
import { useState } from 'react'
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { Check, Ellipsis, FolderPlus, Loader2, X } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import { createFolder, deleteFolder, renameFolder, type FolderRow } from '@/lib/folders'
import { cn } from '@/lib/utils'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { Skeleton } from '@/components/ui/skeleton'
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuTrigger,
} from '@/components/ui/dropdown-menu'
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

// Активный фильтр грида: служебные вкладки All/Unsorted или id папки
// (uuid, со служебными значениями не пересекается).
export type FolderFilter = 'all' | 'unsorted' | string

type Counts = { all: number; unsorted: number; byFolder: Map<string, number> }

type Props = {
  site: SiteConfig
  folders: FolderRow[] | undefined // undefined — ещё грузятся
  active: FolderFilter
  onSelect: (filter: FolderFilter) => void
  counts: Counts
}

export function FoldersPanel({ site, folders, active, onSelect, counts }: Props) {
  const queryClient = useQueryClient()
  const foldersKey = ['folders', site.slug, 'media']

  const [creating, setCreating] = useState(false)
  const [draftName, setDraftName] = useState('')
  const [editingId, setEditingId] = useState<string | null>(null)
  const [editName, setEditName] = useState('')
  const [deleteTarget, setDeleteTarget] = useState<FolderRow | null>(null)

  const create = useMutation({
    mutationFn: (name: string) => createFolder(site, 'media', name),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: foldersKey })
      setCreating(false)
      setDraftName('')
      toast.success('Folder created')
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to create folder'),
  })

  const rename = useMutation({
    mutationFn: ({ id, name }: { id: string; name: string }) => renameFolder(site, id, name),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: foldersKey })
      setEditingId(null)
      toast.success('Folder renamed')
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to rename folder'),
  })

  const remove = useMutation({
    mutationFn: (folder: FolderRow) => deleteFolder(site, folder.id),
    onSuccess: (_, folder) => {
      queryClient.invalidateQueries({ queryKey: foldersKey })
      // FK on delete set null: файлы папки стали Unsorted — обновить и грид.
      queryClient.invalidateQueries({ queryKey: ['media', site.slug] })
      if (active === folder.id) onSelect('all')
      setDeleteTarget(null)
      toast.success('Folder deleted')
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to delete folder'),
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

  function submitRename(folder: FolderRow) {
    const name = editName.trim()
    if (!name || name === folder.name) {
      setEditingId(null)
      return
    }
    rename.mutate({ id: folder.id, name })
  }

  const deleteCount = deleteTarget ? (counts.byFolder.get(deleteTarget.id) ?? 0) : 0

  return (
    <nav className="flex flex-col gap-0.5" aria-label="Media folders">
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

      {folders?.map((folder) =>
        editingId === folder.id ? (
          <div key={folder.id} className="flex items-center gap-1">
            <Input
              autoFocus
              value={editName}
              onChange={(e) => setEditName(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === 'Enter') submitRename(folder)
                if (e.key === 'Escape') setEditingId(null)
              }}
              disabled={rename.isPending}
              className="h-8 text-sm"
            />
            <Button
              variant="ghost"
              size="icon-sm"
              aria-label="Save folder name"
              onClick={() => submitRename(folder)}
              disabled={rename.isPending}
            >
              {rename.isPending ? <Loader2 className="animate-spin" /> : <Check />}
            </Button>
            <Button
              variant="ghost"
              size="icon-sm"
              aria-label="Cancel renaming"
              onClick={() => setEditingId(null)}
              disabled={rename.isPending}
            >
              <X />
            </Button>
          </div>
        ) : (
          <div key={folder.id} className="group flex items-center gap-1">
            <FolderRowButton
              label={folder.name}
              count={counts.byFolder.get(folder.id) ?? 0}
              isActive={active === folder.id}
              onClick={() => onSelect(folder.id)}
              className="min-w-0 flex-1"
            />
            <DropdownMenu>
              <DropdownMenuTrigger asChild>
                <Button
                  variant="ghost"
                  size="icon-sm"
                  aria-label={`Actions for folder ${folder.name}`}
                  className="size-6 opacity-0 group-hover:opacity-100 focus-visible:opacity-100 data-[state=open]:opacity-100"
                >
                  <Ellipsis className="size-3.5" />
                </Button>
              </DropdownMenuTrigger>
              <DropdownMenuContent align="start">
                <DropdownMenuItem
                  onClick={() => {
                    setEditName(folder.name)
                    setEditingId(folder.id)
                  }}
                >
                  Rename
                </DropdownMenuItem>
                <DropdownMenuItem variant="destructive" onClick={() => setDeleteTarget(folder)}>
                  Delete
                </DropdownMenuItem>
              </DropdownMenuContent>
            </DropdownMenu>
          </div>
        ),
      )}

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

      <AlertDialog
        open={deleteTarget !== null}
        onOpenChange={(open) => {
          if (!open) setDeleteTarget(null)
        }}
      >
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Delete folder "{deleteTarget?.name}"?</AlertDialogTitle>
            <AlertDialogDescription>
              {deleteCount > 0
                ? `${deleteCount} image${deleteCount === 1 ? '' : 's'} will be moved to Unsorted. `
                : 'The folder is empty. '}
              Files themselves are not deleted.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel disabled={remove.isPending}>Cancel</AlertDialogCancel>
            <AlertDialogAction
              onClick={() => {
                if (deleteTarget) remove.mutate(deleteTarget)
              }}
            >
              Delete
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
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

- [x] **Step 4.3: Проверка**

Run: `npm run lint && npm run build`
Expected: успех (компоненты ещё не подключены — это нормально; oxlint не ругается на неиспользуемые экспорты).

---

### Task 5: Подключение UI — `MediaPage` / `MediaGrid` / `MediaCard` / `MediaDetailsDialog`

**Files:**
- Modify: `src/features/media/MediaPage.tsx`
- Modify: `src/features/media/MediaGrid.tsx`
- Modify: `src/features/media/MediaCard.tsx`
- Modify: `src/features/media/MediaDetailsDialog.tsx`

Build собирается только после всех четырёх шагов (пропсы сквозные) — выполнять одной сессией, проверка в конце.

- [x] **Step 5.1: Переписать `src/features/media/MediaPage.tsx` целиком**

```tsx
import { useDeferredValue, useMemo, useRef, useState } from 'react'
import { useOutletContext } from 'react-router-dom'
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { FolderInput, Loader2, Search, Upload } from 'lucide-react'
import { toast } from 'sonner'
import type { SiteConfig } from '@/config/sites'
import {
  deleteImage,
  listImages,
  moveImages,
  uploadImage,
  type MediaItem,
} from '@/lib/media'
import { listFolders } from '@/lib/folders'
import { Button } from '@/components/ui/button'
import { Input } from '@/components/ui/input'
import { MediaGrid } from './MediaGrid'
import { FoldersPanel, type FolderFilter } from './FoldersPanel'
import { MoveToFolderMenu } from './MoveToFolderMenu'

const ACCEPT = 'image/jpeg,image/png,image/webp,image/gif'

export function MediaPage() {
  const site = useOutletContext<SiteConfig>()
  const queryClient = useQueryClient()
  const fileInputRef = useRef<HTMLInputElement>(null)
  const mediaKey = ['media', site.slug]

  const [search, setSearch] = useState('')
  const deferredSearch = useDeferredValue(search)
  const isFiltering = deferredSearch !== search

  const [folder, setFolder] = useState<FolderFilter>('all')
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set())

  const { data: items, isPending, error } = useQuery({
    queryKey: mediaKey,
    queryFn: () => listImages(site),
  })

  const { data: folders, error: foldersError } = useQuery({
    queryKey: ['folders', site.slug, 'media'],
    queryFn: () => listFolders(site, 'media'),
  })

  // Счётчики панели — из уже загруженного списка, без лишних запросов.
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

  // Свежие первыми гарантирует listImages (order created_at desc).
  // Поиск идёт по всем папкам (игнорирует активную): нашёл — увидел файл,
  // в какой бы папке он ни лежал. Без поиска грид фильтруется активной папкой.
  const query = deferredSearch.trim().toLowerCase()
  const visibleItems = query
    ? items?.filter(
        (item) =>
          item.original_name.toLowerCase().includes(query) ||
          item.path.toLowerCase().includes(query),
      )
    : folder === 'all'
      ? items
      : items?.filter((item) =>
          folder === 'unsorted' ? item.folder_id === null : item.folder_id === folder,
        )

  // Файлы грузим последовательно: ошибка одного не прерывает остальные.
  // Файл кладётся в открытую сейчас папку (All/Unsorted — без папки).
  const upload = useMutation({
    mutationFn: async (files: File[]) => {
      const folderId = folder === 'all' || folder === 'unsorted' ? null : folder
      const failed: string[] = []
      let uploaded = 0
      for (const file of files) {
        try {
          await uploadImage(site, file, folderId)
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
      setSelectedIds((prev) => {
        if (!prev.has(item.id)) return prev
        const next = new Set(prev)
        next.delete(item.id)
        return next
      })
      toast.success('Image deleted')
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to delete image'),
  })

  const move = useMutation({
    mutationFn: ({ ids, folderId }: { ids: string[]; folderId: string | null }) =>
      moveImages(site, ids, folderId),
    onSuccess: (_, { ids }) => {
      queryClient.invalidateQueries({ queryKey: mediaKey })
      setSelectedIds(new Set())
      toast.success(ids.length === 1 ? 'Image moved' : `${ids.length} images moved`)
    },
    onError: (e) => toast.error(e instanceof Error ? e.message : 'Failed to move images'),
  })

  function handleFilesSelected(fileList: FileList | null) {
    const files = Array.from(fileList ?? [])
    if (files.length) upload.mutate(files)
  }

  // Смена папки сбрасывает выборку: чекбоксы скрытых карточек не должны
  // незаметно участвовать в bulk move.
  function selectFolder(next: FolderFilter) {
    setFolder(next)
    setSelectedIds(new Set())
  }

  function toggleSelect(item: MediaItem) {
    setSelectedIds((prev) => {
      const next = new Set(prev)
      if (next.has(item.id)) next.delete(item.id)
      else next.add(item.id)
      return next
    })
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
          Failed to load media: {(error ?? foldersError)?.message}
        </p>
      )}

      <div className="flex items-start gap-6">
        <aside className="w-44 shrink-0 lg:w-52">
          <FoldersPanel
            site={site}
            folders={folders}
            active={folder}
            onSelect={selectFolder}
            counts={counts}
          />
        </aside>

        <div className="flex min-w-0 flex-1 flex-col gap-3">
          {selectedIds.size > 0 && (
            <div className="flex items-center gap-3 rounded-lg border bg-muted/40 px-3 py-1.5">
              <span className="text-sm">{selectedIds.size} selected</span>
              <MoveToFolderMenu
                folders={folders ?? []}
                onMove={(folderId) => move.mutate({ ids: [...selectedIds], folderId })}
              >
                <Button variant="outline" size="sm" disabled={move.isPending}>
                  <FolderInput />
                  {move.isPending ? 'Moving…' : 'Move to…'}
                </Button>
              </MoveToFolderMenu>
              <Button variant="ghost" size="sm" onClick={() => setSelectedIds(new Set())}>
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
              onToggleSelect={toggleSelect}
              onMove={(item, folderId) => move.mutate({ ids: [item.id], folderId })}
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

- [x] **Step 5.2: Переписать `src/features/media/MediaGrid.tsx` целиком**

```tsx
import type { SiteConfig } from '@/config/sites'
import type { FolderRow } from '@/lib/folders'
import type { MediaItem } from '@/lib/media'
import { Skeleton } from '@/components/ui/skeleton'
import { MediaCard } from './MediaCard'

type Props = {
  site: SiteConfig
  items: MediaItem[] | undefined
  isPending: boolean
  isSearching: boolean
  isFolderFiltered: boolean
  folders: FolderRow[]
  selectedIds: Set<string>
  onToggleSelect: (item: MediaItem) => void
  onMove: (item: MediaItem, folderId: string | null) => void
  onDelete: (item: MediaItem) => void
  deletingId: string | null
}

export function MediaGrid({
  site,
  items,
  isPending,
  isSearching,
  isFolderFiltered,
  folders,
  selectedIds,
  onToggleSelect,
  onMove,
  onDelete,
  deletingId,
}: Props) {
  if (isPending) {
    return (
      <div className="grid grid-cols-2 gap-3 sm:grid-cols-3 lg:grid-cols-4">
        {Array.from({ length: 12 }, (_, i) => (
          <Skeleton key={i} className="aspect-square w-full rounded-xl" />
        ))}
      </div>
    )
  }

  if (!items?.length) {
    return (
      <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
        {isSearching
          ? 'No images match your search.'
          : isFolderFiltered
            ? 'This folder is empty.'
            : 'No images yet. Upload the first one.'}
      </p>
    )
  }

  return (
    <div className="grid grid-cols-2 gap-3 sm:grid-cols-3 lg:grid-cols-4">
      {items.map((item) => (
        <MediaCard
          key={item.id}
          site={site}
          item={item}
          folders={folders}
          selected={selectedIds.has(item.id)}
          onToggleSelect={onToggleSelect}
          onMove={onMove}
          onDelete={onDelete}
          isDeleting={deletingId === item.id}
        />
      ))}
    </div>
  )
}
```

- [x] **Step 5.3: `src/features/media/MediaCard.tsx` — чекбокс выборки + кнопка Move**

Точечные правки (остальное не трогать):

1. Импорты — добавить:

```tsx
import { Checkbox } from '@/components/ui/checkbox'
import type { FolderRow } from '@/lib/folders'
import { MoveToFolderMenu } from './MoveToFolderMenu'
```

и в импорт lucide добавить `FolderInput`:

```tsx
import { Copy, FolderInput, Link, Trash2, TriangleAlert, Unlink } from 'lucide-react'
```

2. Тип Props — добавить поля:

```tsx
type Props = {
  site: SiteConfig
  item: MediaItem
  folders: FolderRow[]
  selected: boolean
  onToggleSelect: (item: MediaItem) => void
  onMove: (item: MediaItem, folderId: string | null) => void
  onDelete: (item: MediaItem) => void
  isDeleting: boolean
}

export function MediaCard({
  site,
  item,
  folders,
  selected,
  onToggleSelect,
  onMove,
  onDelete,
  isDeleting,
}: Props) {
```

3. Корневой Card — сделать `relative` (чекбокс позиционируется от карточки, он сосед кнопки-превью, не вложен в неё — вложенные интерактивные элементы невалидны):

```tsx
    <Card className="relative h-full gap-1.5 overflow-hidden py-0">
```

4. Сразу после открывающего `<Card …>` (перед `<button type="button" …>`) добавить чекбокс. Виден всегда — работает и на таче; фон затемнён для контраста на любой картинке:

```tsx
      <Checkbox
        checked={selected}
        onCheckedChange={() => onToggleSelect(item)}
        aria-label={`Select ${item.original_name}`}
        className="absolute top-1.5 left-1.5 z-10 border-white/80 bg-black/30 shadow-sm data-[state=checked]:border-primary"
      />
```

5. В `CardFooter` между кнопкой `Name` и `AlertDialog` (корзиной) вставить меню перемещения; у кнопки-корзины при этом **убрать** `ml-auto` (правый край теперь начинается с кнопки Move):

```tsx
        <MoveToFolderMenu
          folders={folders}
          currentFolderId={item.folder_id}
          onMove={(folderId) => onMove(item, folderId)}
        >
          <Button
            variant="ghost"
            size="icon-sm"
            aria-label="Move to folder"
            className="ml-auto size-7"
          >
            <FolderInput className="size-3.5" />
          </Button>
        </MoveToFolderMenu>
```

Класс корзины после правки: `className="size-7 text-destructive hover:text-destructive"`.

6. В `<MediaDetailsDialog …>` внизу карточки добавить пропсы:

```tsx
        folders={folders}
        onMove={onMove}
```

- [x] **Step 5.4: `src/features/media/MediaDetailsDialog.tsx` — строка Folder**

Точечные правки:

1. Импорты — добавить:

```tsx
import type { FolderRow } from '@/lib/folders'
import { MoveToFolderMenu } from './MoveToFolderMenu'
```

и в импорт lucide добавить `FolderInput`:

```tsx
import { Check, Copy, FolderInput, Link, Loader2, Pencil, Trash2, X } from 'lucide-react'
```

2. Тип Props — добавить поля (и параметры функции):

```tsx
type Props = {
  site: SiteConfig
  item: MediaItem
  folders: FolderRow[]
  open: boolean
  onOpenChange: (open: boolean) => void
  onMove: (item: MediaItem, folderId: string | null) => void
  onDelete: (item: MediaItem) => void
  isDeleting: boolean
}

export function MediaDetailsDialog({
  site,
  item,
  folders,
  open,
  onOpenChange,
  onMove,
  onDelete,
  isDeleting,
}: Props) {
```

3. После `<DetailRow label="Uploaded" … />` добавить блок:

```tsx
            <div>
              <dt className="text-xs text-muted-foreground">Folder</dt>
              <dd className="flex items-center gap-1">
                <span className="min-w-0 truncate">
                  {folders.find((f) => f.id === item.folder_id)?.name ?? 'Unsorted'}
                </span>
                <MoveToFolderMenu
                  folders={folders}
                  currentFolderId={item.folder_id}
                  onMove={(folderId) => onMove(item, folderId)}
                >
                  <Button variant="ghost" size="icon-sm" aria-label="Move to folder">
                    <FolderInput className="size-3.5" />
                  </Button>
                </MoveToFolderMenu>
              </dd>
            </div>
```

- [x] **Step 5.5: Проверка**

Run: `npm run lint && npm run build`
Expected: lint — только известные shadcn-warnings; build — успех.

---

### Task 6: Документация, статус, ручной e2e

**Files:**
- Modify: `docs/media-manager.md`
- Modify: `CLAUDE.md` (блок «Статус»)

- [x] **Step 6.1: Обновить `docs/media-manager.md`**

Добавить раздел «Папки» (после §2 «Операции» или как §2.5): папки виртуальны (`admin_folders` + `media.folder_id`, `path`/Storage не меняются, сайт о папках не знает); generic-таблица с `section` — переиспользовать для posts/products; операции `lib/folders.ts` (list/create/rename/delete, unique(section,name) → «already exists», delete → файлы в Unsorted по FK); UI (панель All/Unsorted/папки с счётчиками, счётчики с клиента, move через `MoveToFolderMenu` на карточке/в модалке/bulk-баре, multi-select чекбоксами, upload в открытую папку, поиск сквозь папки, смена папки сбрасывает выборку). Дополнить §2 `uploadImage` (параметр folderId) и §1 (колонка folder_id). Ссылка на спеку `docs/superpowers/specs/2026-07-16-media-folders-design.md`.

- [x] **Step 6.2: Обновить `CLAUDE.md`**

В «Сделано» добавить пункт про папки в Media (миграция `add_admin_folders`, `lib/folders.ts`, `moveImages`, `FoldersPanel`/`MoveToFolderMenu`, multi-select; правила — docs/media-manager.md; спека/план — docs/superpowers/{specs,plans}/2026-07-16-media-folders*). В «Следующие шаги» добавить: продублировать файл миграции `add_admin_folders` в репозитории сайта cozycorner; папки для posts/products — после проверки media. В структуру (`src/lib/…`) добавить `folders.ts`.

- [ ] **Step 6.3: Ручной e2e-чек-лист (в `npm run dev` под админом)**

1. Media открывается: слева All/Unsorted со счётчиками (пока равны — папок нет), грид как раньше.
2. New folder → пустое имя/Esc — инпут закрылся без запроса; имя «Banners» → папка в списке, тост.
3. Создать вторую папку «Banners» → тост «Folder with this name already exists».
4. Move одного файла с карточки в Banners → тост, счётчики обновились; открыть Banners — файл там; в модалке деталей строка Folder = Banners.
5. Multi-select: выбрать 2–3 файла в All → бар «N selected», Move to → Banners → файлы переехали, выборка снялась.
6. Смена папки при активной выборке → выборка сброшена.
7. Upload внутри Banners → файл появился в Banners (folder_id проставлен); upload в All → Unsorted.
8. Поиск при открытой папке находит файлы из других папок.
9. Rename папки (Enter/Esc/пустое имя) — работает, как rename картинки.
10. Delete папки с файлами → AlertDialog «N images will be moved to Unsorted» → файлы в Unsorted, активная папка сброшена на All.
11. Перемещение НЕ меняет `path`/публичный URL (проверить Copy URL до/после move); «used by» и предупреждения удаления работают как раньше.
12. Анон (curl из Task 1.3) по-прежнему получает `[]` из `admin_folders`.
13. `ImagePickerDialog` в товаре: пикер работает, загрузка из пикера кладёт файл в Unsorted.

- [x] **Step 6.4: Финальная проверка**

Run: `npm run lint && npm run build`
Expected: успех. Спросить у пользователя: разрешение на commit; напомнить продублировать миграцию `add_admin_folders` в репозиторий сайта.
