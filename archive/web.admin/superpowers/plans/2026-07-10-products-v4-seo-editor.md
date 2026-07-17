# Products v4 (SEO fallback, dialog Save, sort, filters, markdown editor) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Спека `docs/superpowers/specs/2026-07-10-products-v4-seo-editor-design.md` — пять доработок products.

**Architecture:** Сортировка/фильтры — точечные правки. SEO — локальный collapsible-стейт в `ProductForm` + нормализация в `toInput`. Save в модалке — `form.handleSubmit` + `save.mutateAsync` + `blocker.proceed/reset`, `leavingRef` подавляет внутренний redirect. Редактор — новый generic `src/components/RichTextEditor.tsx` (TipTap v3 + официальный `@tiptap/markdown`), в форме через Controller.

**Tech Stack:** как v1–v3 + `@tiptap/react@^3`, `@tiptap/starter-kit@^3`, `@tiptap/markdown@^3`.

**Отступления:** без тест-раннера (build+lint+ручная e2e); Commit — только с явного разрешения.

---

### Task 1: Сортировка (tiebreaker) и порядок фильтров

**Files:**
- Modify: `src/lib/products.ts` (listProducts)
- Modify: `src/features/products/ProductsPage.tsx` (панель фильтров)

- [ ] **Step 1: tiebreaker в listProducts**

```ts
    .order("created_at", { ascending: false })
    .order("title", { ascending: true })
```

(вторая строка добавляется после существующей `.order("created_at", …)`.)

- [ ] **Step 2: Swap фильтров** — в `ProductsPage` перенести весь `<Select value={brand} …>…</Select>` блок ПЕРЕД `<Select value={category} …>` (JSX блоки целиком, без изменений внутри).

- [ ] **Step 3: Проверка** — `npm run build && npm run lint` → успех.

---

### Task 2: SEO-блок со свёрнутым состоянием

**Files:**
- Modify: `src/features/products/ProductEditPage.tsx`

- [ ] **Step 1: Нормализация в toInput**

Внутри `toInput` добавить хелпер и заменить два seo-поля:

```ts
  // Пустое или совпадающее с фолбэком SEO-значение не храним — фолбэк «живой».
  const seoOrNull = (v: string, fallback: string) => {
    const t = v.trim()
    return t && t !== fallback.trim() ? t : null
  }
  // ...
    seo_title: seoOrNull(values.seo_title, values.title),
    seo_description: seoOrNull(values.seo_description, values.description),
```

- [ ] **Step 2: Collapsible-стейт и хендлеры в ProductForm**

В импорт lucide добавить `Pencil, RotateCcw`. После `pickerOpen`:

```tsx
  const [seoOpen, setSeoOpen] = useState(
    Boolean(product?.seo_title || product?.seo_description),
  )

  // Префилл текущими title/description (их и использует SEO); dirty не трогаем —
  // toInput всё равно свернёт совпадающие значения в null.
  function openSeo() {
    if (!form.getValues('seo_title')) {
      form.setValue('seo_title', form.getValues('title'))
    }
    if (!form.getValues('seo_description')) {
      form.setValue('seo_description', form.getValues('description'))
    }
    setSeoOpen(true)
  }

  // shouldDirty сравнивает с defaultValues: очистка изначально пустых полей
  // форму dirty не сделает.
  function resetSeo() {
    form.setValue('seo_title', '', { shouldDirty: true })
    form.setValue('seo_description', '', { shouldDirty: true })
    setSeoOpen(false)
  }
```

- [ ] **Step 3: JSX** — заменить два Field (SEO title / SEO description) на:

```tsx
          {!seoOpen ? (
            <Field>
              <FieldLabel>SEO</FieldLabel>
              <div className="flex items-center justify-between gap-3 rounded-lg border border-dashed p-3">
                <p className="text-sm text-muted-foreground">
                  Search engines use the product title and description.
                </p>
                <Button type="button" variant="ghost" size="sm" onClick={openSeo}>
                  <Pencil />
                  Customize
                </Button>
              </div>
            </Field>
          ) : (
            <>
              <div className="flex items-center justify-between">
                <span className="text-sm font-medium">SEO</span>
                <Button type="button" variant="ghost" size="sm" onClick={resetSeo}>
                  <RotateCcw />
                  Use defaults
                </Button>
              </div>
              <Field>
                <FieldLabel htmlFor="seo_title">SEO title</FieldLabel>
                <Input id="seo_title" {...form.register('seo_title')} />
              </Field>
              <Field>
                <FieldLabel htmlFor="seo_description">SEO description</FieldLabel>
                <Textarea id="seo_description" rows={3} {...form.register('seo_description')} />
              </Field>
            </>
          )}
```

- [ ] **Step 4: Проверка** — `npm run build && npm run lint` → успех.

---

### Task 3: Save в гард-модалке

**Files:**
- Modify: `src/features/products/ProductEditPage.tsx`

- [ ] **Step 1: leavingRef и подавление redirect**

Импорт: `useRef` из react. В `ProductForm`:

```tsx
  const leavingRef = useRef(false) // Save из гард-модалки: не редиректить на созданный товар
```

В `save.onSuccess` условие навигации:

```tsx
      if (created && !leavingRef.current) {
        navigate(`/${site.slug}/products/${created.id}`, { replace: true })
      }
```

- [ ] **Step 2: Хендлер Save-and-leave** (после объявления blocker):

```tsx
  // Save из модалки: валидна — сохранить и продолжить прерванный переход;
  // невалидна/ошибка — остаться на странице.
  function saveAndLeave() {
    void form.handleSubmit(
      async (values) => {
        leavingRef.current = true
        try {
          await save.mutateAsync(values)
          blocker.proceed?.()
        } catch {
          blocker.reset?.()
        } finally {
          leavingRef.current = false
        }
      },
      () => blocker.reset?.(),
    )()
  }
```

- [ ] **Step 3: Кнопка в футере модалки** — заменить `AlertDialogFooter` гард-диалога:

```tsx
          <AlertDialogFooter>
            <AlertDialogCancel disabled={save.isPending}>Stay</AlertDialogCancel>
            <AlertDialogAction
              variant="destructive"
              onClick={() => blocker.proceed?.()}
            >
              Discard
            </AlertDialogAction>
            <Button type="button" onClick={saveAndLeave} disabled={save.isPending}>
              {save.isPending ? 'Saving…' : 'Save'}
            </Button>
          </AlertDialogFooter>
```

- [ ] **Step 4: Проверка** — `npm run build && npm run lint` → успех.

---

### Task 4: RichTextEditor (TipTap → Markdown)

**Files:**
- Create: `src/components/RichTextEditor.tsx`
- Modify: `src/features/products/ProductEditPage.tsx` (Controller на description)

- [ ] **Step 1: Установить зависимости**

Run: `npm install @tiptap/react @tiptap/starter-kit @tiptap/markdown`
Expected: версии ^3.x без peer-конфликтов (React 19 поддержан).
После установки прочитать `node_modules/@tiptap/markdown/README.md` и сверить
точный API (имя экспорта расширения и метод получения markdown; ожидается
`Markdown` + `editor.storage.markdown.getMarkdown()` либо `editor.getMarkdown()`).
При расхождении — использовать API из README.

- [ ] **Step 2: Компонент**

```tsx
import { EditorContent, useEditor } from '@tiptap/react'
import { StarterKit } from '@tiptap/starter-kit'
import { Markdown } from '@tiptap/markdown'
import { Bold, Italic, List, ListOrdered } from 'lucide-react'
import { Button } from '@/components/ui/button'
import { cn } from '@/lib/utils'

// Generic markdown-редактор (products description, позже блог): абзацы, списки,
// bold/italic. value/onChange — markdown-строка.
export function RichTextEditor({
  id,
  value,
  onChange,
}: {
  id?: string
  value: string
  onChange: (markdown: string) => void
}) {
  const editor = useEditor({
    extensions: [
      StarterKit.configure({
        heading: false,
        blockquote: false,
        code: false,
        codeBlock: false,
        horizontalRule: false,
        strike: false,
        link: false,
      }),
      Markdown,
    ],
    content: value,
    contentType: 'markdown',
    onUpdate: ({ editor }) => onChange(editor.storage.markdown.getMarkdown()),
  })

  if (!editor) return null

  const marks = [
    { icon: Bold, label: 'Bold', isActive: editor.isActive('bold'), toggle: () => editor.chain().focus().toggleBold().run() },
    { icon: Italic, label: 'Italic', isActive: editor.isActive('italic'), toggle: () => editor.chain().focus().toggleItalic().run() },
    { icon: List, label: 'Bullet list', isActive: editor.isActive('bulletList'), toggle: () => editor.chain().focus().toggleBulletList().run() },
    { icon: ListOrdered, label: 'Numbered list', isActive: editor.isActive('orderedList'), toggle: () => editor.chain().focus().toggleOrderedList().run() },
  ]

  return (
    <div className="rounded-md border shadow-xs focus-within:ring-2 focus-within:ring-ring/50">
      <div className="flex gap-1 border-b p-1">
        {marks.map(({ icon: Icon, label, isActive, toggle }) => (
          <Button
            key={label}
            type="button"
            variant="ghost"
            size="icon"
            aria-label={label}
            aria-pressed={isActive}
            onClick={toggle}
            className={cn('h-7 w-7', isActive && 'bg-accent text-accent-foreground')}
          >
            <Icon className="size-4" />
          </Button>
        ))}
      </div>
      <EditorContent
        id={id}
        editor={editor}
        className="[&_.tiptap]:min-h-24 [&_.tiptap]:px-3 [&_.tiptap]:py-2 [&_.tiptap]:text-sm [&_.tiptap]:outline-none [&_ol]:list-decimal [&_ol]:pl-5 [&_ul]:list-disc [&_ul]:pl-5 [&_p]:my-1"
      />
    </div>
  )
}
```

Примечания: точные имена опций StarterKit v3 сверить по ошибкам tsc (в v3
`link` входит в StarterKit — если опции нет, убрать строку; `code` — mark).
Если `contentType: 'markdown'` не поддерживается опцией useEditor —
использовать API из README (`editor.commands.setContent(value, { contentType:
'markdown' })` в onCreate или парсер из пакета).

- [ ] **Step 3: Подключить в форму** — в `ProductEditPage.tsx` заменить Field description:

```tsx
          <Field>
            <FieldLabel htmlFor="description">Description</FieldLabel>
            <Controller
              control={form.control}
              name="description"
              render={({ field }) => (
                <RichTextEditor id="description" value={field.value} onChange={field.onChange} />
              )}
            />
          </Field>
```

Импорт: `import { RichTextEditor } from '@/components/RichTextEditor'`.
Импорт `Textarea` остаётся (используется для seo_description).

- [ ] **Step 4: Проверка** — `npm run build && npm run lint` → успех.

---

### Task 5: Документация, промпт для сайта, финал

**Files:**
- Create: `docs/site-markdown-prompt.md` (промпт для агента репозитория cozycorner)
- Modify: `docs/ux-ui.md` (§3.7), `CLAUDE.md` (статус)

- [ ] **Step 1: Промпт для агента сайта** — самодостаточный текст: контекст
  (админка пишет `products.description` в Markdown: абзацы пустой строкой,
  списки `-`/`1.`, `**bold**`/`*italic*`, без ссылок/заголовков), задача
  (рендерить markdown на странице товара и карточках, где выводится description;
  рекомендация react-markdown c ограниченным набором элементов; plain-text
  фолбэк для старых записей корректен по построению), критерии приёмки.

- [ ] **Step 2: ux-ui.md §3.7 и CLAUDE.md** — SEO-блок (фолбэк/Customize/Use
  defaults, null-нормализация), Save в гард-модалке, tiebreaker сортировки,
  порядок фильтров Brand→Category, RichTextEditor (TipTap v3 + @tiptap/markdown,
  markdown в БД, компонент generic в `src/components/`).

- [ ] **Step 3: Финал** — `npm run build && npm run lint` → успех; ручная e2e —
  чек-лист из спеки v4.
