# Picker Folders + Bulk Delete + Hover Preview — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Дать выбор папки при добавлении картинки, bulk-удаление сущностей и hover-кнопку смены картинки на превью.

**Architecture:** Три независимые фичи поверх существующих механизмов папок (`src/features/folders/`) и пикера (`ImagePickerDialog`). Два новых переиспользуемых компонента (`BulkDeleteButton`, `ImagePreviewPicker`), остальное — точечные правки страниц. Схему БД не трогаем; lib-функции удаления (`deleteImage`/`deleteProduct`/`deletePost`) и `uploadImage(…, folderId)` уже существуют.

**Tech Stack:** Vite + React 19 + TS, TanStack Query, shadcn/ui (radix), sonner, lucide-react, Tailwind v4.

**Verification model:** В репозитории нет юнит-тестов (см. AGENTS.md). Каждая задача верифицируется через `npm run build` (tsc + vite) и `npm run lint` (oxlint — 2 известных shadcn-warning допустимы) плюс ручную проверку в приложении (`npm run dev`). Рабочая директория команд — `web.admin/`.

**Commits:** Не коммитить и не пушить без явного разрешения пользователя (правило AGENTS.md). Шаги «Commit» ниже выполнять только после разрешения; иначе оставлять изменения в рабочем дереве и отмечать задачу выполненной по прохождении build+lint.

---

## Task 1: Папки в `ImagePickerDialog`

**Files:**
- Modify: `web.admin/src/features/products/ImagePickerDialog.tsx`

Пикер используется и продуктами, и постами (`import { ImagePickerDialog } from '@/features/products/ImagePickerDialog'`). Добавляем фильтр папок (без CRUD) и upload в выбранную папку.

- [ ] **Step 1: Добавить импорты папок и Select**

В блок импортов добавить:

```tsx
import { foldersKey, listFolders, type FolderRow } from '@/lib/folders'
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
```

- [ ] **Step 2: Запрос папок + стейт активной папки**

Внутри компонента, после существующего `useQuery` для media, добавить:

```tsx
const [activeFolder, setActiveFolder] = useState<string>('all') // 'all' | 'unsorted' | folderId

const { data: folders } = useQuery({
  queryKey: foldersKey(site, 'media'),
  queryFn: () => listFolders(site, 'media'),
  enabled: open,
})

// Сброс фильтра при закрытии, чтобы следующий вызов открывался на All.
useEffect(() => {
  if (!open) setActiveFolder('all')
}, [open])
```

Добавить `useEffect` в импорт react: `import { useDeferredValue, useEffect, useRef, useState } from 'react'`.

- [ ] **Step 3: Счётчики папок и фильтрация грида**

Заменить существующий расчёт `visible` (строки с `const query = …` и `const visible = …`) на:

```tsx
const query = deferredSearch.trim().toLowerCase()
const all = items ?? []

// Счётчики для подписей в Select — из уже загруженного списка.
const counts = { all: all.length, unsorted: 0, byFolder: new Map<string, number>() }
for (const item of all) {
  if (item.folder_id) {
    counts.byFolder.set(item.folder_id, (counts.byFolder.get(item.folder_id) ?? 0) + 1)
  } else {
    counts.unsorted++
  }
}

// Поиск идёт сквозь все папки (нашёл — увидел, где бы ни лежал), как в разделе Media.
// Без поиска грид фильтруется активной папкой.
const visible = all.filter((item) => {
  if (query) {
    return (
      item.original_name.toLowerCase().includes(query) ||
      item.path.toLowerCase().includes(query)
    )
  }
  if (activeFolder === 'all') return true
  if (activeFolder === 'unsorted') return item.folder_id === null
  return item.folder_id === activeFolder
})
```

> Требует поля `folder_id` в типе `MediaItem`. Оно уже есть (media имеет `folder_id`, используется `useFolders`). Если tsc ругается — проверить тип `MediaItem` в `src/lib/media.ts`.

- [ ] **Step 4: Upload в активную папку**

В мутации `upload` заменить вызов:

```tsx
mutationFn: (file: File) => uploadImage(site, file, folderFilterToId(activeFolder)),
```

и добавить рядом с компонентом (файловый scope) хелпер:

```tsx
// 'all'/'unsorted' — без папки (null); иначе id папки.
function folderFilterToId(filter: string): string | null {
  return filter === 'all' || filter === 'unsorted' ? null : filter
}
```

- [ ] **Step 5: Отрисовать Select папок в шапке диалога**

В `<div className="flex items-center gap-2">` (строка с поиском), перед блоком поиска, добавить селект:

```tsx
<Select value={activeFolder} onValueChange={setActiveFolder}>
  <SelectTrigger className="h-9 w-40" aria-label="Filter by folder">
    <SelectValue />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="all">All ({counts.all})</SelectItem>
    <SelectItem value="unsorted">Unsorted ({counts.unsorted})</SelectItem>
    {(folders ?? []).map((folder: FolderRow) => (
      <SelectItem key={folder.id} value={folder.id}>
        {folder.name} ({counts.byFolder.get(folder.id) ?? 0})
      </SelectItem>
    ))}
  </SelectContent>
</Select>
```

- [ ] **Step 6: Пустое состояние с учётом папки**

В блоке пустого состояния (`items && visible.length === 0`) уточнить текст, чтобы не сбивать при фильтре папки:

```tsx
{items && visible.length === 0 && (
  <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
    {query
      ? 'No images match your search.'
      : items.length === 0
        ? 'No images yet.'
        : 'This folder is empty.'}
  </p>
)}
```

- [ ] **Step 7: Build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: build проходит; lint — только 2 известных shadcn-warning.

- [ ] **Step 8: Ручная проверка**

`npm run dev` → открыть ProductEditPage → Gallery: виден селект папок, переключение фильтрует грид, поиск игнорирует папку, Upload кладёт файл в выбранную папку (проверить в разделе Media). Повторить в PostEditPage (Gallery cover).

---

## Task 2: Компонент `BulkDeleteButton`

**Files:**
- Create: `web.admin/src/features/folders/BulkDeleteButton.tsx`

- [ ] **Step 1: Создать компонент**

```tsx
import type { ReactNode } from 'react'
import { Trash2 } from 'lucide-react'
import { Button } from '@/components/ui/button'
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
  AlertDialogTrigger,
} from '@/components/ui/alert-dialog'

// Общая кнопка bulk-удаления для media/products/posts: destructive-триггер +
// подтверждение. warning — необязательный текст-предупреждение (media: занятые
// картинки). Мутацию удаления держит страница; сюда приходит только onConfirm.
export function BulkDeleteButton({
  count,
  itemNoun,
  warning,
  onConfirm,
  isDeleting,
}: {
  count: number
  itemNoun: string
  warning?: ReactNode
  onConfirm: () => void
  isDeleting: boolean
}) {
  return (
    <AlertDialog>
      <AlertDialogTrigger asChild>
        <Button
          variant="outline"
          size="sm"
          className="text-destructive hover:text-destructive"
          disabled={isDeleting}
        >
          <Trash2 />
          {isDeleting ? 'Deleting…' : 'Delete'}
        </Button>
      </AlertDialogTrigger>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>
            Delete {count} {itemNoun}
            {count === 1 ? '' : 's'}?
          </AlertDialogTitle>
          <AlertDialogDescription>
            {warning ?? 'This action cannot be undone.'}
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction onClick={onConfirm}>Delete</AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  )
}
```

- [ ] **Step 2: Build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: build проходит (компонент ещё не используется — это нормально). Если tsc ругается на неиспользуемый компонент — игнорировать, он подключится в Task 3-5.

---

## Task 3: Bulk delete — Media

**Files:**
- Modify: `web.admin/src/features/media/MediaPage.tsx`

- [ ] **Step 1: Импорт компонента**

Добавить: `import { BulkDeleteButton } from '@/features/folders/BulkDeleteButton'`

- [ ] **Step 2: Мутация bulk-удаления**

После существующей мутации `remove` добавить:

```tsx
const removeMany = useMutation({
  mutationFn: async (batch: MediaItem[]) => {
    const failed: string[] = []
    let deleted = 0
    for (const item of batch) {
      try {
        await deleteImage(site, item)
        deleted++
      } catch (e) {
        failed.push(e instanceof Error ? e.message : `"${item.original_name}": delete failed`)
      }
    }
    return { deleted, failed }
  },
  onSuccess: ({ deleted, failed }) => {
    if (deleted > 0) {
      queryClient.invalidateQueries({ queryKey: mediaKey })
      clearSelection()
      toast.success(deleted === 1 ? 'Image deleted' : `${deleted} images deleted`)
    }
    failed.forEach((message) => toast.error(message))
  },
})
```

- [ ] **Step 3: Добавить кнопку в bulk-бар**

В блоке `visibleSelectedIds.size > 0`, между `MoveToFolderMenu` и кнопкой `Clear`, вставить:

```tsx
{(() => {
  const selectedItems = (visibleItems ?? []).filter((i) => visibleSelectedIds.has(i.id))
  const inUse = selectedItems.filter((i) => i.usedBy.length > 0).length
  return (
    <BulkDeleteButton
      count={visibleSelectedIds.size}
      itemNoun="image"
      isDeleting={removeMany.isPending}
      warning={
        inUse > 0
          ? `${inUse} of these ${inUse === 1 ? 'image is' : 'images are'} in use on the site — deleting will break them. This action cannot be undone.`
          : undefined
      }
      onConfirm={() => removeMany.mutate(selectedItems)}
    />
  )
})()}
```

> `visibleItems`, `visibleSelectedIds`, `clearSelection`, `queryClient`, `mediaKey`, `deleteImage`, `toast`, `MediaItem` уже в области видимости `MediaPage`.

- [ ] **Step 4: Build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: проходит.

- [ ] **Step 5: Ручная проверка**

`npm run dev` → Media → выбрать несколько картинок → Delete: диалог с числом; если среди них есть используемые — предупреждение; после удаления выборка сбрасывается, грид обновляется.

---

## Task 4: Bulk delete — Products

**Files:**
- Modify: `web.admin/src/features/products/ProductsPage.tsx`

- [ ] **Step 1: Добавить импорты**

Добавить/дополнить импорты:

```tsx
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { toast } from 'sonner'
import { deleteProduct, listCategoryNames, listProducts } from '@/lib/products'
import { BulkDeleteButton } from '@/features/folders/BulkDeleteButton'
```

(Строку `import { useQuery } from '@tanstack/react-query'` заменить на вариант с `useMutation, useQueryClient`.)

- [ ] **Step 2: queryClient + мутация**

После `const site = useOutletContext<SiteConfig>()` добавить:

```tsx
const queryClient = useQueryClient()
```

После блока `useFolders({...})` добавить:

```tsx
const removeMany = useMutation({
  mutationFn: async (ids: string[]) => {
    const failed: string[] = []
    let deleted = 0
    for (const id of ids) {
      try {
        await deleteProduct(site, id)
        deleted++
      } catch (e) {
        failed.push(e instanceof Error ? e.message : 'Failed to delete product')
      }
    }
    return { deleted, failed }
  },
  onSuccess: ({ deleted, failed }) => {
    if (deleted > 0) {
      queryClient.invalidateQueries({ queryKey: productsKey })
      clearSelection()
      toast.success(deleted === 1 ? 'Product deleted' : `${deleted} products deleted`)
    }
    failed.forEach((message) => toast.error(message))
  },
})
```

- [ ] **Step 3: Кнопка в bulk-бар**

В блоке `visibleSelectedIds.size > 0`, между `MoveToFolderMenu` и `Clear`, вставить:

```tsx
<BulkDeleteButton
  count={visibleSelectedIds.size}
  itemNoun="product"
  isDeleting={removeMany.isPending}
  onConfirm={() => removeMany.mutate([...visibleSelectedIds])}
/>
```

- [ ] **Step 4: Build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: проходит.

- [ ] **Step 5: Ручная проверка**

`npm run dev` → Products → выбрать несколько → Delete → подтвердить: строки исчезают, выборка сбрасывается.

---

## Task 5: Bulk delete — Posts

**Files:**
- Modify: `web.admin/src/features/posts/PostsPage.tsx`

- [ ] **Step 1: Добавить импорты**

```tsx
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { toast } from 'sonner'
import { deletePost, listPosts } from '@/lib/posts'
import { BulkDeleteButton } from '@/features/folders/BulkDeleteButton'
```

(Заменить строку `import { useQuery } from '@tanstack/react-query'`.)

- [ ] **Step 2: queryClient + мутация**

После `const site = useOutletContext<SiteConfig>()` добавить `const queryClient = useQueryClient()`.

После блока `useFolders({...})` добавить:

```tsx
const removeMany = useMutation({
  mutationFn: async (ids: string[]) => {
    const failed: string[] = []
    let deleted = 0
    for (const id of ids) {
      try {
        await deletePost(site, id)
        deleted++
      } catch (e) {
        failed.push(e instanceof Error ? e.message : 'Failed to delete post')
      }
    }
    return { deleted, failed }
  },
  onSuccess: ({ deleted, failed }) => {
    if (deleted > 0) {
      queryClient.invalidateQueries({ queryKey: postsKey })
      clearSelection()
      toast.success(deleted === 1 ? 'Post deleted' : `${deleted} posts deleted`)
    }
    failed.forEach((message) => toast.error(message))
  },
})
```

- [ ] **Step 3: Кнопка в bulk-бар**

В блоке `visibleSelectedIds.size > 0`, между `MoveToFolderMenu` и `Clear`, вставить:

```tsx
<BulkDeleteButton
  count={visibleSelectedIds.size}
  itemNoun="post"
  isDeleting={removeMany.isPending}
  onConfirm={() => removeMany.mutate([...visibleSelectedIds])}
/>
```

- [ ] **Step 4: Build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: проходит.

- [ ] **Step 5: Ручная проверка**

`npm run dev` → Blog / SEO Posts → выбрать несколько → Delete → подтвердить.

---

## Task 6: Компонент `ImagePreviewPicker`

**Files:**
- Create: `web.admin/src/components/ImagePreviewPicker.tsx`

- [ ] **Step 1: Создать компонент**

```tsx
import { ImageOff, ImagePlus, Pencil } from 'lucide-react'

// Превью картинки с hover/focus-оверлеем «Add/Change image», открывающим пикер.
// url — уже разрешённый публичный URL (или null/'' если картинки нет).
export function ImagePreviewPicker({
  url,
  alt,
  aspect,
  objectFit,
  broken,
  onErrorAction,
  onPick,
}: {
  url: string | null
  alt: string
  aspect: 'square' | 'video'
  objectFit: 'contain' | 'cover'
  broken: boolean
  onErrorAction: () => void
  onPick: () => void
}) {
  const hasImage = Boolean(url) && !broken
  return (
    <button
      type="button"
      onClick={onPick}
      aria-label={hasImage ? 'Change image' : 'Add image'}
      className={`group relative flex w-full items-center justify-center overflow-hidden rounded-xl border bg-muted/30 ${
        aspect === 'square' ? 'aspect-square' : 'aspect-video md:aspect-square'
      }`}
    >
      {hasImage ? (
        <img
          src={url ?? undefined}
          alt={alt}
          className={`size-full ${objectFit === 'contain' ? 'object-contain' : 'object-cover'}`}
          onError={onErrorAction}
        />
      ) : (
        <ImageOff className="size-8 text-muted-foreground" />
      )}
      <span className="absolute inset-0 flex items-center justify-center gap-2 bg-black/50 text-sm font-medium text-white opacity-0 transition-opacity group-hover:opacity-100 group-focus-visible:opacity-100">
        {hasImage ? <Pencil className="size-4" /> : <ImagePlus className="size-4" />}
        {hasImage ? 'Change image' : 'Add image'}
      </span>
    </button>
  )
}
```

- [ ] **Step 2: Build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: проходит (компонент ещё не подключён — ок).

---

## Task 7: Подключить `ImagePreviewPicker` в ProductEditPage

**Files:**
- Modify: `web.admin/src/features/products/ProductEditPage.tsx`

- [ ] **Step 1: Импорт**

Добавить: `import { ImagePreviewPicker } from '@/components/ImagePreviewPicker'`

- [ ] **Step 2: Заменить inline-превью**

Заменить блок превью (обёртка `<div className="flex aspect-square items-center justify-center overflow-hidden rounded-xl border bg-muted/30">…</div>` со вложенным `<img>`/`<ImageOff>`) на:

```tsx
<ImagePreviewPicker
  url={imagePath ? resolveImageUrl(site, imagePath) : null}
  alt={form.watch('title') || 'Product image'}
  aspect="square"
  objectFit="contain"
  broken={imageBroken}
  onErrorAction={() => setImageBroken(true)}
  onPick={() => setPickerOpen(true)}
/>
```

> `imagePath`, `imageBroken`, `setImageBroken`, `setPickerOpen`, `resolveImageUrl`, `site`, `form` уже в области видимости. Импорт `ImageOff` в этом файле может стать неиспользуемым — удалить его из lucide-импорта, если lint/tsc укажет.

- [ ] **Step 3: Build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: проходит.

- [ ] **Step 4: Ручная проверка**

`npm run dev` → Products → новый/существующий товар: пустое превью показывает «Add image» на hover; с картинкой — «Change image»; клик открывает пикер; выбор обновляет превью. Проверить клавиатуру (Tab до превью → оверлей виден).

---

## Task 8: Подключить `ImagePreviewPicker` в PostEditPage

**Files:**
- Modify: `web.admin/src/features/posts/PostEditPage.tsx`

- [ ] **Step 1: Импорт**

Добавить: `import { ImagePreviewPicker } from '@/components/ImagePreviewPicker'`

- [ ] **Step 2: Заменить inline-превью cover**

Заменить блок превью cover (обёртка `<div className="flex aspect-video items-center justify-center overflow-hidden rounded-xl border bg-muted/30 md:aspect-square">…</div>` со вложенным `<img>`/`<ImageOff>`) на:

```tsx
<ImagePreviewPicker
  url={coverPath ? resolveImageUrl(site, coverPath) : null}
  alt={form.watch('title') || 'Cover image'}
  aspect="video"
  objectFit="cover"
  broken={coverBroken}
  onErrorAction={() => setCoverBroken(true)}
  onPick={() => setPickerOpen(true)}
/>
```

> `coverPath`, `coverBroken`, `setCoverBroken`, `setPickerOpen`, `resolveImageUrl`, `site`, `form` уже в области видимости. Удалить неиспользуемый импорт `ImageOff`, если lint укажет.

- [ ] **Step 3: Build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: проходит.

- [ ] **Step 4: Ручная проверка**

`npm run dev` → Blog → новый/существующий пост: hover на cover-превью показывает «Add/Change image», клик открывает пикер, выбор обновляет превью.

---

## Task 9: Финальная проверка

- [ ] **Step 1: Полный build + lint**

Run (в `web.admin/`): `npm run build && npm run lint`
Expected: build чистый; lint — только 2 известных shadcn-warning.

- [ ] **Step 2: Сквозная ручная проверка всех трёх фич**

1. Пикер: папки видны и фильтруют, upload в папку — в ProductEditPage и PostEditPage.
2. Bulk delete: media (с предупреждением о занятых), products, posts.
3. Hover-превью: ProductEditPage (square/contain), PostEditPage (video/cover).

- [ ] **Step 3: Commit (только с разрешения пользователя)**

```bash
cd web.admin
git add -A
git commit -m "feat: folder filter in image picker, bulk delete, hover image preview"
```

## Вне объёма

Миграции БД, CRUD папок в пикере, hover на мелких миниатюрах, сортировка/группировка в пикере.
