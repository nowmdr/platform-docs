# Products v3 (Clear, public URL, unsaved guard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Кнопка Clear для фильтров списка; публичный URL в поле Image path (с нормализацией в плоский ключ при сохранении); предупреждение о несохранённых изменениях при уходе со страницы товара. Спека: `docs/superpowers/specs/2026-07-10-products-v3-ux-design.md`.

**Architecture:** Clear — локальный сброс трёх состояний в `ProductsPage`. URL — пара `resolveImageUrl`/`toStoragePath` (`lib/images.ts`) на границе форма↔БД. Гард — миграция на data router (`createBrowserRouter`) + `useBlocker` + AlertDialog + `beforeunload`; после мутаций `form.reset` снимает dirty.

**Tech Stack:** как v1/v2; react-router-dom 7 (data mode).

**Отступления:** без тест-раннера (build+lint+ручная e2e); Commit — только с разрешения пользователя.

---

### Task 1: Кнопка Clear (`ProductsPage`)

**Files:**
- Modify: `src/features/products/ProductsPage.tsx`

- [ ] **Step 1: Добавить кнопку**

В импорт lucide добавить `X`:

```tsx
import { Loader2, Plus, Search, X } from 'lucide-react'
```

В конец панели фильтров (после Select Brand, внутри `div.flex.flex-wrap`):

```tsx
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
```

`hasFilter` уже вычислен выше по коду.

- [ ] **Step 2: Проверка** — Run: `npm run build && npm run lint` → успех.

- [ ] **Step 3: Commit (только с разрешения)** — `git add src/features/products/ProductsPage.tsx && git commit -m "feat: clear button for product filters"`

---

### Task 2: `toStoragePath` + публичный URL в Image path

**Files:**
- Modify: `src/lib/images.ts`
- Modify: `src/features/products/ProductEditPage.tsx` (toFormValues, toInput, onSelect, currentPath)

- [ ] **Step 1: Хелпер в lib/images.ts**

Дописать в конец файла:

```ts
// Обратная операция к resolveImageUrl: публичный URL нашего Storage сворачивается
// в плоский ключ (в БД храним только его — спека §7); прочие значения — как есть.
export function toStoragePath(site: SiteConfig, value: string): string {
  const prefix = `${site.projectUrl}/storage/v1/object/public/${site.bucket}/`;
  return value.startsWith(prefix) ? value.slice(prefix.length) : value;
}
```

- [ ] **Step 2: ProductEditPage — границы форма↔БД**

Импорт: `import { resolveImageUrl, toStoragePath } from '@/lib/images'`.

`toInput` и `toFormValues` начинают зависеть от `site` — добавить параметр:

```ts
function toInput(site: SiteConfig, values: ProductFormValues): ProductInput {
  // ...
    image_path: orNull(toStoragePath(site, values.image_path.trim())),
  // остальное без изменений
}

function toFormValues(site: SiteConfig, product: Product | null): ProductFormValues {
  // ...
    image_path: product?.image_path ? resolveImageUrl(site, product.image_path) : '',
  // остальное без изменений
}
```

Вызовы обновить: `toInput(site, values)` в save.mutationFn,
`toFormValues(site, product)` в defaultValues.

Пикеру передавать нормализованный текущий путь:

```tsx
        currentPath={imagePath ? toStoragePath(site, imagePath) : null}
```

`onSelect` кладёт URL:

```tsx
        onSelect={(path) => {
          form.setValue('image_path', resolveImageUrl(site, path), { shouldDirty: true })
          setImageBroken(false)
        }}
```

- [ ] **Step 3: Проверка** — Run: `npm run build && npm run lint` → успех.

- [ ] **Step 4: Commit (только с разрешения)** — `git add src/lib/images.ts src/features/products/ProductEditPage.tsx && git commit -m "feat: show public image URL in form, store flat key"`

---

### Task 3: Data router + гард несохранённых изменений

**Files:**
- Modify: `src/App.tsx` (экспорт router вместо компонента App)
- Modify: `src/main.tsx` (RouterProvider)
- Modify: `src/features/products/ProductEditPage.tsx` (useBlocker, beforeunload, reset, AlertDialog)

- [ ] **Step 1: App.tsx → data router**

```tsx
import {
  Navigate,
  Route,
  createBrowserRouter,
  createRoutesFromElements,
} from 'react-router-dom'
// остальные импорты страниц/лейаутов без изменений

// ComingSoonPage без изменений

// Data router (а не <Routes>): нужен для useBlocker (гард несохранённых форм).
export const router = createBrowserRouter(
  createRoutesFromElements(
    <>
      <Route path="/login" element={<LoginPage />} />
      <Route path="/dashboard" element={<RequireAuth />}>
        <Route element={<AppShell />}>
          <Route index element={<DashboardPage />} />
        </Route>
      </Route>
      <Route path="/:siteSlug" element={<RequireAuth />}>
        <Route element={<AppShell />}>
          <Route element={<SiteLayout />}>
            <Route index element={<Navigate to="media" replace />} />
            <Route path="media" element={<MediaPage />} />
            <Route path="pages" element={<ComingSoonPage section="Pages" />} />
            <Route path="products" element={<ProductsPage />} />
            <Route path="products/new" element={<ProductEditPage />} />
            <Route path="products/:productId" element={<ProductEditPage />} />
          </Route>
        </Route>
      </Route>
      <Route path="*" element={<Navigate to="/dashboard" replace />} />
    </>,
  ),
)
```

(Функция `App` и `export default App` удаляются.)

- [ ] **Step 2: main.tsx → RouterProvider**

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { RouterProvider } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { Toaster } from '@/components/ui/sonner'
import './index.css'
import { router } from './App.tsx'

const queryClient = new QueryClient()

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
      <Toaster richColors position="top-center" />
    </QueryClientProvider>
  </StrictMode>,
)
```

- [ ] **Step 3: Гард в ProductForm**

Импорты: `useEffect` из react; `useBlocker` из react-router-dom.

В `ProductForm` после объявления мутаций:

```tsx
  // Гард несохранённых изменений: внутренняя навигация — через blocker,
  // закрытие/обновление вкладки — через beforeunload.
  const { isDirty } = form.formState
  const blocker = useBlocker(isDirty && !save.isPending && !remove.isPending)

  useEffect(() => {
    if (!isDirty) return
    const warn = (e: BeforeUnloadEvent) => e.preventDefault()
    window.addEventListener('beforeunload', warn)
    return () => window.removeEventListener('beforeunload', warn)
  }, [isDirty])
```

(`errors, isSubmitting` уже деструктурированы из `form.formState` — isDirty
добавить туда же: `const { errors, isSubmitting, isDirty } = form.formState`.)

Снятие dirty после мутаций:
- в `save.mutationFn` перед `createProduct`/`updateProduct` ничего не менять;
- в `save` колбэк `onSuccess` получает `(created, values)` — вторым аргументом
  RHF-значения мутации; вызвать `form.reset(values)` первой строкой;
- в `remove.onSuccess` — `form.reset()` первой строкой.

```tsx
    onSuccess: (created, values) => {
      form.reset(values) // форма чистая — уход не блокируется
      queryClient.invalidateQueries({ queryKey: ['products', site.slug] })
      toast.success(isNew ? 'Product created' : 'Product saved')
      if (created) navigate(`/${site.slug}/products/${created.id}`, { replace: true })
    },
```

В JSX рядом с `<ImagePickerDialog …/>` добавить диалог гарда:

```tsx
      <AlertDialog
        open={blocker.state === 'blocked'}
        onOpenChange={(open) => {
          if (!open) blocker.reset?.()
        }}
      >
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Discard unsaved changes?</AlertDialogTitle>
            <AlertDialogDescription>
              You have unsaved changes. If you leave this page, they will be lost.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>Stay</AlertDialogCancel>
            <AlertDialogAction variant="destructive" onClick={() => blocker.proceed?.()}>
              Discard
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
```

- [ ] **Step 4: Проверка** — Run: `npm run build && npm run lint` → успех.

- [ ] **Step 5: Commit (только с разрешения)** — `git add src/App.tsx src/main.tsx src/features/products/ProductEditPage.tsx && git commit -m "feat: unsaved changes guard on product form (data router + useBlocker)"`

---

### Task 4: Документация и финальная проверка

**Files:**
- Modify: `docs/ux-ui.md` (§2 — data router; §3.7 — Clear, URL, гард)
- Modify: `CLAUDE.md` (статус: v3)

- [ ] **Step 1: docs/ux-ui.md** — в §2 отметить, что роутер data-mode
  (`createBrowserRouter` в `App.tsx`, `RouterProvider` в `main.tsx`). В §3.7:
  панель фильтров — ghost-кнопка Clear при активных фильтрах; Image path
  показывает публичный URL (в БД — плоский ключ через `toStoragePath`); гард
  несохранённых изменений (blocker-диалог Stay/Discard + beforeunload).

- [ ] **Step 2: CLAUDE.md** — в «Сделано» дополнить пункт products v2 → v2+v3
  (Clear, публичный URL c нормализацией, unsaved-guard, data router).

- [ ] **Step 3: Финальная проверка** — `npm run build && npm run lint` → успех.
  Ручная e2e — чек-лист в спеке v3.

- [ ] **Step 4: Commit (только с разрешения)** — `git add CLAUDE.md docs/ && git commit -m "docs: products v3 spec, plan and status"`
