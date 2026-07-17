# Blog v2 Layout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Sticky-панель действий (Back / статус / Save / Delete) в редакторах поста и товара, грид-шапка «картинка + ключевые поля» вместо пустого столба, фильтр статуса в списке постов.

**Architecture:** Спека — `docs/superpowers/specs/2026-07-10-blog-v2-layout-design.md` (утверждена). Только раскладка и один клиентский фильтр: слой данных, мутации, гард, схемы — НЕ трогаем. Save уезжает из `<form>` → сабмит через `id` формы + атрибут `form` на кнопке. Шапка AppShell не sticky (проверено) — `sticky top-0` панели работает от вьюпорта.

**Tech Stack:** без новых зависимостей; существующие shadcn-компоненты.

**Правила проекта:** НЕ коммитить без разрешения; UI-текст английский; стиль single quotes/без «;» в features. Автотестов нет — проверка `npm run lint && npm run build` + ручная (dev).

---

## File Structure

| Файл | Изменение |
|---|---|
| Modify `src/features/posts/PostsPage.tsx` | + Select-фильтр статуса |
| Modify `src/features/posts/PostEditPage.tsx` | sticky-панель, грид-шапка, Status из тела → панель |
| Modify `src/features/products/ProductEditPage.tsx` | sticky-панель, грид-шапка |
| Modify `docs/blog.md`, `docs/products.md`, `CLAUDE.md` | зафиксировать изменения |

---

### Task 1: Фильтр статуса в списке постов

**Files:**
- Modify: `src/features/posts/PostsPage.tsx`

- [ ] **Step 1.1: Добавить Select-фильтр**

Импорты — добавить:

```tsx
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select'
```

Над компонентом (как в ProductsPage):

```tsx
const ALL = '__all__' // radix SelectItem не принимает пустое value
```

Состояние и фильтрация — заменить блок от `const [search, setSearch]` до `const visible`:

```tsx
  const [search, setSearch] = useState('')
  const [status, setStatus] = useState(ALL)
  const deferredSearch = useDeferredValue(search)
  const isFiltering = deferredSearch !== search

  const { data: posts, isPending, error } = useQuery({
    queryKey: ['posts', site.slug],
    queryFn: () => listPosts(site),
  })

  const query = deferredSearch.trim().toLowerCase()
  const hasFilter = Boolean(query) || status !== ALL
  const visible = (posts ?? []).filter(
    (p) =>
      (!query || p.title.toLowerCase().includes(query)) &&
      (status === ALL || p.is_published === (status === 'published')),
  )
```

- [ ] **Step 1.2: Разметка фильтра**

После `</div>` поискового инпута (внутри строки фильтров) вставить Select, а кнопку Clear заменить на сбрасывающую оба состояния:

```tsx
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
```

Текст пустого результата `No posts match your search.` → `No posts match your filters.`

- [ ] **Step 1.3: Проверка**

`npm run lint && npm run build` → exit 0. Dev: фильтр Draft/Published сужает список, счётчик `видимые / все`, Clear сбрасывает поиск и статус.

---

### Task 2: PostEditPage — sticky-панель + грид-шапка

**Files:**
- Modify: `src/features/posts/PostEditPage.tsx`

Слой данных, схема, конвертеры, мутации, гард, `saveAndLeave`, `beforeunload`, SEO-логика (`openSeo`/`resetSeo`), диалоги — НЕ меняются. Меняется только JSX-раскладка `PostEditPage` (родителя) и `PostForm`.

- [ ] **Step 2.1: Родитель `PostEditPage` — ссылка назад только вне формы**

Ссылка «Back to blog» переезжает в sticky-панель `PostForm`; родитель показывает её только в состояниях loading/error/not-found:

```tsx
export function PostEditPage() {
  const site = useOutletContext<SiteConfig>()
  const { postId } = useParams()
  const isNew = !postId

  const { data, isPending, error } = useQuery({
    queryKey: ['posts', site.slug, postId],
    queryFn: () => getPost(site, postId!),
    enabled: !isNew,
  })

  const loading = !isNew && isPending
  const showForm = !error && !loading && (isNew || data)

  return (
    <div className="flex flex-col gap-4">
      {!showForm && (
        <Link
          to={`/${site.slug}/blog`}
          className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground"
        >
          <ArrowLeft className="size-4" />
          Back to blog
        </Link>
      )}

      {error && (
        <p className="rounded-xl border border-destructive/50 p-4 text-sm text-destructive">
          Failed to load: {error.message}
        </p>
      )}

      {!error && loading && <p className="text-sm text-muted-foreground">Loading…</p>}

      {!error && !loading && !isNew && !data && (
        <p className="rounded-xl border border-dashed p-8 text-center text-sm text-muted-foreground">
          Post not found.
        </p>
      )}

      {showForm && <PostForm site={site} data={data ?? null} />}
    </div>
  )
}
```

- [ ] **Step 2.2: `PostForm` — новый return**

Заменить весь `return (...)` `PostForm` на следующий (тела обработчиков, mutations и т.д. выше по файлу не трогать; содержимое диалогов — прежнее):

```tsx
  return (
    <div className="flex flex-col gap-4">
      {/* Sticky-панель действий: Save вне <form>, сабмит через form="post-form". */}
      <div className="sticky top-0 z-10 flex items-center gap-3 border-b bg-background py-2">
        <Link
          to={`/${site.slug}/blog`}
          className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground"
        >
          <ArrowLeft className="size-4" />
          Back to blog
        </Link>
        <div className="ml-auto flex items-center gap-3">
          <Controller
            control={form.control}
            name="is_published"
            render={({ field }) => (
              <label htmlFor="is_published" className="flex items-center gap-2 text-sm">
                <Switch
                  id="is_published"
                  checked={field.value}
                  onCheckedChange={field.onChange}
                />
                {field.value ? 'Published' : 'Draft'}
              </label>
            )}
          />
          <Button type="submit" form="post-form" disabled={isSubmitting || save.isPending}>
            {save.isPending ? 'Saving…' : 'Save'}
          </Button>
          {!isNew && (
            <AlertDialog>
              <AlertDialogTrigger asChild>
                <Button type="button" variant="destructive" disabled={remove.isPending}>
                  <Trash2 />
                  {remove.isPending ? 'Deleting…' : 'Delete'}
                </Button>
              </AlertDialogTrigger>
              <AlertDialogContent>
                <AlertDialogHeader>
                  <AlertDialogTitle>Delete this post?</AlertDialogTitle>
                  <AlertDialogDescription>
                    “{data.post.title}” and all its content sections will be
                    permanently deleted.
                  </AlertDialogDescription>
                </AlertDialogHeader>
                <AlertDialogFooter>
                  <AlertDialogCancel>Cancel</AlertDialogCancel>
                  <AlertDialogAction variant="destructive" onClick={() => remove.mutate()}>
                    Delete
                  </AlertDialogAction>
                </AlertDialogFooter>
              </AlertDialogContent>
            </AlertDialog>
          )}
        </div>
      </div>

      <form
        id="post-form"
        onSubmit={form.handleSubmit(
          (values) => save.mutate(values),
          () => toast.error('Fix validation errors before saving'),
        )}
        noValidate
        className="flex max-w-3xl flex-col gap-6"
      >
        {/* Грид-шапка: превью cover слева, ключевые поля справа. */}
        <div className="grid gap-6 md:grid-cols-[16rem_1fr]">
          <div>
            <div className="flex aspect-video items-center justify-center overflow-hidden rounded-xl border bg-muted/30 md:aspect-square">
              {coverPath && !coverBroken ? (
                <img
                  src={resolveImageUrl(site, coverPath)}
                  alt={form.watch('title') || 'Cover image'}
                  className="size-full object-cover"
                  onError={() => setCoverBroken(true)}
                />
              ) : (
                <ImageOff className="size-8 text-muted-foreground" />
              )}
            </div>
            {data && (
              <p className="mt-2 truncate text-xs text-muted-foreground" title={data.post.slug}>
                Slug: {data.post.slug}
              </p>
            )}
          </div>

          <FieldGroup>
            <Field data-invalid={!!errors.title}>
              <FieldLabel htmlFor="title">Title</FieldLabel>
              <Input id="title" aria-invalid={!!errors.title} {...form.register('title')} />
              <FieldError errors={[errors.title]} />
            </Field>

            <Field>
              <FieldLabel htmlFor="excerpt">Excerpt</FieldLabel>
              <Textarea id="excerpt" rows={3} {...form.register('excerpt')} />
            </Field>

            <Field data-invalid={!!errors.cover_image_path}>
              <FieldLabel htmlFor="cover_image_path">Cover image</FieldLabel>
              <div className="flex gap-2">
                <Input
                  id="cover_image_path"
                  placeholder="flat key in bucket or external URL"
                  {...form.register('cover_image_path', {
                    onChange: () => setCoverBroken(false),
                  })}
                />
                <Button type="button" variant="outline" onClick={() => setPickerOpen(true)}>
                  <Images />
                  Gallery
                </Button>
              </div>
            </Field>
          </FieldGroup>
        </div>

        <FieldGroup>
          <Field data-invalid={!!errors.published_at} className="max-w-xs">
            <FieldLabel htmlFor="published_at">Published at</FieldLabel>
            <Input
              id="published_at"
              type="datetime-local"
              aria-invalid={!!errors.published_at}
              {...form.register('published_at')}
            />
            <FieldError errors={[errors.published_at]} />
          </Field>

          <PostModulesEditor site={site} form={form} />

          {!seoOpen ? (
            <Field>
              <FieldLabel>SEO</FieldLabel>
              <div className="flex items-center justify-between gap-3 rounded-lg border border-dashed p-3">
                <p className="text-sm text-muted-foreground">
                  Search engines use the post title and excerpt.
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
                <Textarea
                  id="seo_description"
                  rows={3}
                  {...form.register('seo_description')}
                />
              </Field>
            </>
          )}
        </FieldGroup>
      </form>

      <ImagePickerDialog
        site={site}
        open={pickerOpen}
        onOpenChange={setPickerOpen}
        currentPath={coverPath ? toStoragePath(site, coverPath) : null}
        onSelect={(path) => {
          form.setValue('cover_image_path', resolveImageUrl(site, path), {
            shouldDirty: true,
          })
          setCoverBroken(false)
        }}
      />

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
            <AlertDialogCancel disabled={save.isPending}>Stay</AlertDialogCancel>
            <AlertDialogAction variant="destructive" onClick={() => blocker.proceed?.()}>
              Discard
            </AlertDialogAction>
            <Button type="button" onClick={saveAndLeave} disabled={save.isPending}>
              {save.isPending ? 'Saving…' : 'Save'}
            </Button>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  )
```

Примечания:
- Поле Status из тела формы удалено (грид `grid-cols-2 items-end gap-4` со Status/Published at больше не существует; Published at — отдельное поле `max-w-xs`).
- Если shadcn `Field` не принимает `className` — обернуть в `<div className="max-w-xs">`.
- Старый внешний контейнер `flex flex-col gap-6 md:flex-row` и aside `md:w-64` удаляются.

- [ ] **Step 2.3: Проверка**

`npm run lint && npm run build` → exit 0. Dev: панель прилипает при скролле длинного поста; тоггл в панели меняет Published/Draft и делает форму dirty (гард ловит); Save из панели сабмитит форму (в т.ч. с ошибками валидации — тост); Delete работает; на mobile грид схлопывается.

---

### Task 3: ProductEditPage — sticky-панель + грид-шапка

**Files:**
- Modify: `src/features/products/ProductEditPage.tsx`

Логика (schema, toInput, мутации, гард, SEO) — не трогать. Только JSX.

- [ ] **Step 3.1: Родитель `ProductEditPage` — ссылка назад только вне формы**

Аналогично посту: `const showForm = !error && !loading && (isNew || product)`; `{!showForm && <Link ...>Back to products</Link>}`; `{showForm && <ProductForm ... />}`. Остальные состояния без изменений.

- [ ] **Step 3.2: `ProductForm` — новый return**

Заменить весь `return (...)` на:

```tsx
  return (
    <div className="flex flex-col gap-4">
      {/* Sticky-панель действий: Save вне <form>, сабмит через form="product-form". */}
      <div className="sticky top-0 z-10 flex items-center gap-3 border-b bg-background py-2">
        <Link
          to={`/${site.slug}/products`}
          className="flex w-fit items-center gap-1 text-sm text-muted-foreground transition-colors hover:text-foreground"
        >
          <ArrowLeft className="size-4" />
          Back to products
        </Link>
        <div className="ml-auto flex items-center gap-3">
          <Button
            type="submit"
            form="product-form"
            disabled={isSubmitting || save.isPending}
          >
            {save.isPending ? 'Saving…' : 'Save'}
          </Button>
          {!isNew && (
            <AlertDialog>
              <AlertDialogTrigger asChild>
                <Button type="button" variant="destructive" disabled={remove.isPending}>
                  <Trash2 />
                  {remove.isPending ? 'Deleting…' : 'Delete'}
                </Button>
              </AlertDialogTrigger>
              <AlertDialogContent>
                <AlertDialogHeader>
                  <AlertDialogTitle>Delete this product?</AlertDialogTitle>
                  <AlertDialogDescription>
                    “{product.title}” will be permanently deleted. If it is used in
                    blog post product sections, it will disappear from them.
                  </AlertDialogDescription>
                </AlertDialogHeader>
                <AlertDialogFooter>
                  <AlertDialogCancel>Cancel</AlertDialogCancel>
                  <AlertDialogAction
                    variant="destructive"
                    onClick={() => remove.mutate()}
                  >
                    Delete
                  </AlertDialogAction>
                </AlertDialogFooter>
              </AlertDialogContent>
            </AlertDialog>
          )}
        </div>
      </div>

      <form
        id="product-form"
        onSubmit={form.handleSubmit((values) => save.mutate(values))}
        noValidate
        className="flex max-w-3xl flex-col gap-6"
      >
        {/* Грид-шапка: превью слева, ключевые поля справа. */}
        <div className="grid gap-6 md:grid-cols-[16rem_1fr]">
          <div>
            <div className="flex aspect-square items-center justify-center overflow-hidden rounded-xl border bg-muted/30">
              {imagePath && !imageBroken ? (
                <img
                  src={resolveImageUrl(site, imagePath)}
                  alt={form.watch('title') || 'Product image'}
                  className="size-full object-contain"
                  onError={() => setImageBroken(true)}
                />
              ) : (
                <ImageOff className="size-8 text-muted-foreground" />
              )}
            </div>
            {product && (
              <p className="mt-2 truncate text-xs text-muted-foreground" title={product.slug}>
                Slug: {product.slug}
              </p>
            )}
          </div>

          <FieldGroup>
            <Field data-invalid={!!errors.title}>
              <FieldLabel htmlFor="title">Title</FieldLabel>
              <Input id="title" aria-invalid={!!errors.title} {...form.register('title')} />
              <FieldError errors={[errors.title]} />
            </Field>

            <div className="grid grid-cols-2 gap-4">
              <Field data-invalid={!!errors.price}>
                <FieldLabel htmlFor="price">Price</FieldLabel>
                <Input
                  id="price"
                  type="number"
                  step="0.01"
                  min="0"
                  aria-invalid={!!errors.price}
                  {...form.register('price')}
                />
                <FieldError errors={[errors.price]} />
              </Field>
              <Field data-invalid={!!errors.brand}>
                <FieldLabel htmlFor="brand">Brand</FieldLabel>
                <Input id="brand" {...form.register('brand')} />
              </Field>
            </div>

            <Field data-invalid={!!errors.image_path}>
              <FieldLabel htmlFor="image_path">Image path</FieldLabel>
              <div className="flex gap-2">
                <Input
                  id="image_path"
                  placeholder="flat key in bucket or external URL"
                  {...form.register('image_path', {
                    onChange: () => setImageBroken(false),
                  })}
                />
                <Button
                  type="button"
                  variant="outline"
                  onClick={() => setPickerOpen(true)}
                >
                  <Images />
                  Gallery
                </Button>
              </div>
            </Field>
          </FieldGroup>
        </div>

        <FieldGroup>
          <Field data-invalid={!!errors.referral_url}>
            <FieldLabel htmlFor="referral_url">Referral URL</FieldLabel>
            <Input
              id="referral_url"
              type="url"
              placeholder="https://…"
              aria-invalid={!!errors.referral_url}
              {...form.register('referral_url')}
            />
            <FieldError errors={[errors.referral_url]} />
          </Field>

          <div className="grid grid-cols-2 gap-4">
            <Field>
              <FieldLabel htmlFor="category">Category</FieldLabel>
              <Controller
                control={form.control}
                name="category"
                render={({ field }) => (
                  <Select value={field.value} onValueChange={field.onChange}>
                    <SelectTrigger id="category" className="w-full">
                      <SelectValue />
                    </SelectTrigger>
                    <SelectContent>
                      <SelectItem value={NONE}>None</SelectItem>
                      {categories.map((name) => (
                        <SelectItem key={name} value={name}>
                          {name}
                        </SelectItem>
                      ))}
                    </SelectContent>
                  </Select>
                )}
              />
            </Field>
            <Field>
              <FieldLabel htmlFor="image_style">Image style</FieldLabel>
              <Controller
                control={form.control}
                name="image_style"
                render={({ field }) => (
                  <Select value={field.value} onValueChange={field.onChange}>
                    <SelectTrigger id="image_style" className="w-full">
                      <SelectValue />
                    </SelectTrigger>
                    <SelectContent>
                      <SelectItem value="photo">Photo</SelectItem>
                      <SelectItem value="cutout">Cutout</SelectItem>
                    </SelectContent>
                  </Select>
                )}
              />
            </Field>
          </div>

          <Field>
            <FieldLabel htmlFor="description">Description</FieldLabel>
            <Controller
              control={form.control}
              name="description"
              render={({ field }) => (
                <RichTextEditor
                  id="description"
                  value={field.value}
                  onChange={field.onChange}
                />
              )}
            />
          </Field>

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
        </FieldGroup>
      </form>

      <ImagePickerDialog
        site={site}
        open={pickerOpen}
        onOpenChange={setPickerOpen}
        currentPath={imagePath ? toStoragePath(site, imagePath) : null}
        onSelect={(path) => {
          form.setValue('image_path', resolveImageUrl(site, path), { shouldDirty: true })
          setImageBroken(false)
        }}
      />

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
            <AlertDialogCancel disabled={save.isPending}>Stay</AlertDialogCancel>
            <AlertDialogAction variant="destructive" onClick={() => blocker.proceed?.()}>
              Discard
            </AlertDialogAction>
            <Button type="button" onClick={saveAndLeave} disabled={save.isPending}>
              {save.isPending ? 'Saving…' : 'Save'}
            </Button>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </div>
  )
```

- [ ] **Step 3.3: Проверка**

`npm run lint && npm run build` → exit 0. Dev: товар — панель сверху, Save/Delete работают, раскладка без пустого столба, старых кнопок внизу нет.

---

### Task 4: Документация + финальная проверка

**Files:**
- Modify: `docs/blog.md` (§2 список — фильтр статуса; §3 форма — sticky-панель, тоггл в панели, грид-шапка)
- Modify: `docs/products.md` (§3 форма — sticky-панель Save/Delete, грид-шапка)
- Modify: `CLAUDE.md` (строка в «Сделано» про итерацию v2 раскладки)

- [ ] **Step 4.1:** Обновить три документа (компактно, в стиле существующих записей; упомянуть паттерн `form`-атрибута для Save вне `<form>`).
- [ ] **Step 4.2:** `npm run lint && npm run build` → exit 0.
