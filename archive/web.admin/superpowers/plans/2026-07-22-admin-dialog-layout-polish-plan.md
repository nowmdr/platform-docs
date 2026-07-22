# Admin Layout Polish + Dialog Unification — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Расширить основной контейнер админки, починить два визуальных бага в пикерах и унифицировать все диалоги вокруг `DialogContent` с `max-h` + скроллящимся `DialogBody`.

**Architecture:** Расширяем примитив `src/components/ui/dialog.tsx` (flex-колонка + `max-h-[85svh]`, новый экспорт `DialogBody`), затем переводим на него три модалки (`ImagePickerDialog`, `ProductPickerDialog`, `MediaDetailsDialog`), попутно закрывая баги со скроллбаром и кольцом выделения.

**Tech Stack:** Vite + React 19 + TypeScript, Tailwind v4, shadcn/ui (radix), TanStack Query.

**Спека:** `../specs/2026-07-22-admin-dialog-layout-polish-design.md`

**Верификация:** тест-раннера в проекте нет. Каждая задача проверяется `npm run build`
(tsc -b + vite build) и `npm run lint` (oxlint; 2 известных shadcn-warning — ок) в
`web.admin/`, плюс ручной прогон в `npm run dev`. **Коммиты — только с разрешения
пользователя (AGENTS.md); шаги commit ниже выполнять по подтверждению.**

---

## Task 1: Расширить основной контейнер до max-w-7xl

**Files:**
- Modify: `web.admin/src/components/AppShell.tsx:19,33`
- Modify (doc): `platform-docs/admin-panel/components.md` (§3.3, §5)

- [ ] **Step 1: Заменить ширину шапки**

В `src/components/AppShell.tsx` строка 19 — заменить класс контейнера шапки:

```tsx
        <div className="mx-auto flex h-14 max-w-7xl items-center gap-3 px-4">
```

- [ ] **Step 2: Заменить ширину main**

Строка 33:

```tsx
      <main className="mx-auto max-w-7xl p-4">
```

- [ ] **Step 3: Обновить доку**

В `platform-docs/admin-panel/components.md`:
- §3.3 «Высота 14, нижняя граница; контент по центру `max-w-5xl`.» → `max-w-7xl`.
- §5 «Контент страницы — `max-w-5xl`, паддинг 4; шапка h-14.» → `max-w-7xl`.

- [ ] **Step 4: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: build без ошибок; lint — не более 2 известных shadcn-warning.
Ручная проверка (`npm run dev`): страницы Media/Products/Blog/Pages стали шире;
двухколоночный SiteLayout (меню `w-44` + контент) не поехал.

- [ ] **Step 5: Commit (по разрешению пользователя)**

```bash
cd web.admin && git add src/components/AppShell.tsx
git commit -m "feat(admin): widen main container to max-w-7xl"
```

---

## Task 2: Расширить примитив Dialog (max-h + DialogBody)

**Files:**
- Modify: `web.admin/src/components/ui/dialog.tsx`

- [ ] **Step 1: DialogContent → flex-колонка с max-h**

В `src/components/ui/dialog.tsx`, в `DialogContent`, заменить строку className
`DialogPrimitive.Content` (сейчас начинается с `"fixed top-1/2 left-1/2 z-50 grid w-full …"`):
заменить `grid` на `flex max-h-[85svh] flex-col`. Итоговое значение:

```tsx
        className={cn(
          "fixed top-1/2 left-1/2 z-50 flex max-h-[85svh] w-full max-w-[calc(100%-2rem)] -translate-x-1/2 -translate-y-1/2 flex-col gap-4 rounded-xl bg-popover p-4 text-sm text-popover-foreground ring-1 ring-foreground/10 duration-100 outline-none sm:max-w-sm data-open:animate-in data-open:fade-in-0 data-open:zoom-in-95 data-closed:animate-out data-closed:fade-out-0 data-closed:zoom-out-95",
          className
        )}
```

- [ ] **Step 2: Добавить компонент DialogBody**

Добавить функцию рядом с `DialogHeader` (после неё):

```tsx
// Скроллящаяся середина модалки: header/footer фиксированы, тело — flex-1 со скроллом.
// -mx-4 px-4 тянет скролл-область на всю ширину контента, но сохраняет внутренний
// отступ, чтобы кольца/чек-индикаторы у краёв не срезались (см. ImagePickerDialog).
function DialogBody({ className, ...props }: React.ComponentProps<"div">) {
  return (
    <div
      data-slot="dialog-body"
      className={cn("-mx-4 min-h-0 flex-1 overflow-y-auto px-4", className)}
      {...props}
    />
  )
}
```

- [ ] **Step 3: Экспортировать DialogBody**

В блоке `export { … }` добавить `DialogBody,` (по алфавиту — после `DialogClose`... фактически рядом с остальными):

```tsx
export {
  Dialog,
  DialogBody,
  DialogClose,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogOverlay,
  DialogPortal,
  DialogTitle,
  DialogTrigger,
}
```

- [ ] **Step 4: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: build/lint чисто (кроме известных warning). Визуально ничего не поменялось —
потребители ещё не используют `DialogBody`; проверка что примитив компилируется.

- [ ] **Step 5: Commit (по разрешению)**

```bash
cd web.admin && git add src/components/ui/dialog.tsx
git commit -m "feat(admin): dialog max-h + DialogBody scroll region"
```

---

## Task 3: ImagePickerDialog — DialogBody + кольцо выделения + паддинг

**Files:**
- Modify: `web.admin/src/features/products/ImagePickerDialog.tsx`

- [ ] **Step 1: Импортировать DialogBody**

В импорте из `@/components/ui/dialog` (строки 13–18) добавить `DialogBody`:

```tsx
import {
  Dialog,
  DialogBody,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'
```

- [ ] **Step 2: Заменить скролл-контейнер на DialogBody с паддингом**

Найти `<div className="max-h-[55svh] overflow-y-auto">` (строка ~180) и заменить
открывающий и соответствующий закрывающий тег. Открывающий → `<DialogBody className="py-2">`
(ВАЖНО: `py-2`, не `p-1` — базовый `DialogBody` уже имеет `px-4`, а `p-1`/`pr-1` через
twMerge перебивает его и урезает горизонтальный отступ; нужен только вертикальный запас
под кольцо). Закрывающий `</div>` этого блока (строка ~230) → `</DialogBody>`.

> Реализовано 2026-07-22: `<DialogBody className="py-2">`.

- [ ] **Step 3: Заметный, но мягкий цвет кольца**

Найти className кнопки плитки (строка ~209) — заменить строку `item.path === currentPath && 'ring-2 ring-ring'` на:

```tsx
                    item.path === currentPath &&
                      'ring-2 ring-primary ring-offset-2 ring-offset-background',
```

- [ ] **Step 4: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: чисто. Ручная проверка (`npm run dev` → товар → Gallery / Change image):
модалка со скроллом при многих картинках; выбранная картинка помечена кольцом цвета
primary; кольцо у крайних (в т.ч. угловых) плиток не обрезано; при переполнении
скроллится только грид, header фиксирован.

- [ ] **Step 5: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/products/ImagePickerDialog.tsx
git commit -m "fix(admin): image picker ring color + clipping, use DialogBody"
```

---

## Task 4: ProductPickerDialog — DialogBody + паддинг под скроллбар

**Files:**
- Modify: `web.admin/src/features/posts/ProductPickerDialog.tsx`

- [ ] **Step 1: Импортировать DialogBody**

В импорте из `@/components/ui/dialog` (строки 11–17) добавить `DialogBody`:

```tsx
import {
  Dialog,
  DialogBody,
  DialogContent,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'
```

- [ ] **Step 2: Заменить скролл-контейнер на DialogBody с правым паддингом**

Найти `<div className="max-h-[50svh] overflow-y-auto">` (строка ~88) → заменить на
`<DialogBody>` (БЕЗ `pr-1`: базовый `px-4` даёт 16px правого отступа, чего достаточно,
чтобы тонкий скроллбар не перекрывал круглый чек-индикатор; `pr-1` наоборот урезал бы до
4px). Соответствующий закрывающий `</div>` (строка ~153) → `</DialogBody>`.

> Реализовано 2026-07-22: чистый `<DialogBody>`.

- [ ] **Step 3: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: чисто. Ручная проверка (`npm run dev` → пост в блоге → модуль Products →
Add products): при 10+ товарах и активном скролле круглый чек-индикатор полностью
виден, между ним и скроллбаром есть зазор; header (поиск) и footer (Add N products)
фиксированы, скроллится только список.

- [ ] **Step 4: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/posts/ProductPickerDialog.tsx
git commit -m "fix(admin): product picker scrollbar no longer overlaps radios"
```

---

## Task 5: MediaDetailsDialog — обернуть контент в DialogBody

**Files:**
- Modify: `web.admin/src/features/media/MediaDetailsDialog.tsx`

- [ ] **Step 1: Импортировать DialogBody**

В импорте из `@/components/ui/dialog` (строки 14–20) добавить `DialogBody`:

```tsx
import {
  Dialog,
  DialogBody,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from '@/components/ui/dialog'
```

- [ ] **Step 2: Обернуть сетку контента в DialogBody**

Найти `<div className="grid gap-4 md:grid-cols-[3fr_2fr]">` (строка ~92) — обернуть весь
этот блок (до его закрывающего `</div>` на строке ~253) в `DialogBody`. Практично:
открывающий тег превратить в `<DialogBody>` + перенести классы сетки на вложенный div,
ЛИБО просто добавить обёртку. Минимальный вариант — заменить открывающий
`<div className="grid gap-4 md:grid-cols-[3fr_2fr]">` на:

```tsx
        <DialogBody>
          <div className="grid gap-4 md:grid-cols-[3fr_2fr]">
```

и парный закрывающий `</div>` (строка ~253, перед `</DialogContent>`) на:

```tsx
          </div>
        </DialogBody>
```

- [ ] **Step 3: Верификация**

Run: `cd web.admin && npm run build && npm run lint`
Expected: чисто. Ручная проверка (`npm run dev` → Media → клик по карточке): модалка
деталей открывается; на узком/низком окне контент скроллится внутри тела, header
(имя файла) остаётся сверху; кнопки Copy/Delete доступны (в т.ч. через скролл).

- [ ] **Step 4: Commit (по разрешению)**

```bash
cd web.admin && git add src/features/media/MediaDetailsDialog.tsx
git commit -m "refactor(admin): media details dialog uses DialogBody"
```

---

## Task 6: Финальная верификация + доки паттерна

**Files:**
- Modify (doc): `platform-docs/admin-panel/components.md` (§1 или §4 — зафиксировать паттерн DialogBody)

- [ ] **Step 1: Зафиксировать паттерн в доке**

В `platform-docs/admin-panel/components.md`, в §4 «Паттерны взаимодействия», добавить пункт:

```markdown
- **Модалки** — единый примитив `ui/dialog.tsx`: `DialogContent` ограничен `max-h-[85svh]`
  (flex-колонка), середина — `DialogBody` (скроллится, `-mx-4 px-4` сохраняет отступ для
  колец/индикаторов), header/footer фиксированы. Ручные `max-h-[NNsvh]` на внутренних div
  не заводить — использовать `DialogBody`.
```

- [ ] **Step 2: Полная верификация проекта**

Run: `cd web.admin && npm run build && npm run lint`
Expected: build без ошибок; lint ≤ 2 известных shadcn-warning.

- [ ] **Step 3: Ручной прогон всех модалок**

`npm run dev`, проверить по очереди: пикер картинок (кольцо+скролл), пикер товаров
(скроллбар не режет индикаторы), детали медиа (скролл тела); ресайз окна — ни одна
модалка не вылезает за `85svh`, скроллится только `DialogBody`.

- [ ] **Step 4: Commit доки (по разрешению)**

```bash
cd platform-docs && git add admin-panel/components.md
git commit -m "docs(admin): widen container, DialogBody modal pattern"
```

---

## Self-Review (spec coverage)

- Спека §1 (ширина) → Task 1. ✓
- Спека §2 (скроллбар ProductPicker) → Task 4. ✓
- Спека §3 (кольцо ImagePicker + паддинг) → Task 3. ✓
- Спека §4 (DialogContent max-h + DialogBody + рефактор 3 модалок) → Task 2 (примитив) +
  Task 3/4/5 (потребители). ✓
- Доки (components.md §3.3/§5 ширина, §4 паттерн) → Task 1 Step 3 + Task 6 Step 1. ✓
- `AlertDialog` намеренно не трогаем (вне scope) — ни одной задачи, верно.
