# Дизайн: папки в пикере, bulk-delete, hover на превью

Дата: 2026-07-22
Проект: web.admin

Три независимые доработки админ-панели поверх уже существующих механизмов
(папки — `src/features/folders/`, пикер — `ImagePickerDialog`, bulk-бар —
`useFolders` + `MoveToFolderMenu`). Схему БД не трогаем.

## Фича 1 — Папки в `ImagePickerDialog`

**Проблема.** Папки/сортировка видны только в разделах Media и Products. При
добавлении картинки к продукту или посту пикер (`ImagePickerDialog`) показывает
плоский грид всех медиа — папки недоступны.

**Решение.** Лёгкий фильтр по папкам внутри пикера (без CRUD папок):

- Запрос папок: `useQuery({ queryKey: foldersKey(site, 'media'), queryFn: () =>
  listFolders(site, 'media'), enabled: open })` — тот же общий кэш, что у
  `FoldersPanel`.
- Локальный стейт `activeFolder: FolderFilter` (default `'all'`), живёт только
  внутри диалога; сбрасывается при закрытии.
- UI — компактный **shadcn `Select`** вверху, слева от поиска. Пункты:
  `All (N)` / `Unsorted (N)` / каждая папка `<name> (N)`. Счётчики считаются из
  уже загруженного `items` (как `counts` в `useFolders`).
- Фильтрация грида — семантика как на `MediaPage`: без поиска грид фильтруется
  активной папкой по `folder_id`; при непустом поиске папка игнорируется (ищем по
  всем папкам).
- Upload — грузим в выбранную папку: `uploadImage(site, file, activeFolderId)`
  (`activeFolderId` = `null` для `All`/`Unsorted`). Сигнатура `uploadImage` уже
  принимает `folderId` (используется на `MediaPage`) — проверить и передать.

Управление папками (создание/переименование/удаление) остаётся только в разделе
Media — в пикере не дублируем.

## Фича 2 — Bulk delete (media / products / posts)

**Проблема.** При мультиселекте bulk-бар умеет только `Move to…`. Нужна кнопка
удаления в том же контейнере.

**Решение.** Новый общий компонент `BulkDeleteButton`
(`src/features/folders/BulkDeleteButton.tsx`):

- Пропсы: `count: number`, `itemNoun: string`, `warning?: ReactNode`,
  `onConfirm: () => void`, `isDeleting: boolean`.
- Рендер: destructive icon-кнопка `Trash2` → `AlertDialog` с заголовком
  `Delete {count} {itemNoun}{s}?`, описанием (+ `warning`, если передан) и
  кнопками Cancel / Delete.
- Ставится в bulk-контейнер после `MoveToFolderMenu`, перед `Clear` — на всех
  трёх страницах.

Мутации удаления — своя на каждой странице (lib-функции уже есть):

| Раздел   | Функция          | Warning                                                    |
|----------|------------------|------------------------------------------------------------|
| media    | `deleteImage`    | сколько из выбранных с `usedBy.length > 0` («in use…break») |
| products | `deleteProduct`  | нет                                                        |
| posts    | `deletePost`     | нет                                                        |

- Удаляем пачкой по `visibleSelectedIds` (не по скрытой выборке). Реализация —
  последовательный цикл по id с сбором ошибок (как `upload` на `MediaPage`):
  ошибка одного элемента не прерывает остальные.
- После успеха: `invalidateQueries({ queryKey: contentKey })`, `clearSelection()`,
  тост (`N images/products/posts deleted`), ошибки — отдельными тостами.
- media: warning-текст считается из выбранных `MediaItem` (`usedBy`), поэтому
  bulk-бар media должен иметь доступ к выбранным объектам, не только к id.

## Фича 3 — Hover-оверлей на главных превью

**Проблема.** На `ProductEditPage` (image) и `PostEditPage` (cover) превью —
пассивный квадрат с картинкой или иконкой `ImageOff`. Сменить картинку можно
только через поле `Image path` / кнопку `Gallery`.

**Решение.** Новый общий компонент `ImagePreviewPicker`
(`src/components/ImagePreviewPicker.tsx` — используется двумя разделами):

- Пропсы: `url: string | null`, `alt: string`, `aspect: 'square' | 'video'`,
  `objectFit: 'contain' | 'cover'`, `broken: boolean`, `onErrorAction: () => void`,
  `onPick: () => void`.
- Контейнер `group relative`; при `url` — `<img>` c `onError`, иначе `ImageOff`.
- Поверх — оверлей, скрытый по умолчанию, видимый на `group-hover` и
  `focus-within` (клавиатурная доступность): полупрозрачный фон + центр-кнопка с
  иконкой (`ImagePlus` для пустого / `Pencil` для замены) и текстом
  **Add image** / **Change image**. Клик → `onPick()` (= `setPickerOpen(true)`).
- Поле `Image path` и кнопка `Gallery` сохраняются (внешние URL и ручной ввод).

Применяется в шапках `ProductEditPage` и `PostEditPage` вместо текущего inline
превью-блока. Мелкие миниатюры (строки товаров в модулях постов) не трогаем.

## Затрагиваемые файлы

- `src/features/products/ImagePickerDialog.tsx` — фильтр папок + upload в папку
- `src/lib/media.ts` — проверить сигнатуру `uploadImage(site, file, folderId?)`
- `src/features/folders/BulkDeleteButton.tsx` — новый
- `src/features/media/MediaPage.tsx` — bulk delete
- `src/features/products/ProductsPage.tsx` — bulk delete
- `src/features/posts/PostsPage.tsx` — bulk delete
- `ImagePreviewPicker.tsx` — новый
- `src/features/products/ProductEditPage.tsx` — превью → `ImagePreviewPicker`
- `src/features/posts/PostEditPage.tsx` — cover → `ImagePreviewPicker`

## Вне объёма

- Изменения схемы БД / миграции.
- CRUD папок внутри пикера.
- Hover на мелких миниатюрах.
- Сортировка/группировка внутри пикера (только фильтр).

## Проверка

`npm run build` и `npm run lint` (oxlint; 2 известных shadcn-warning допустимы).
Ручная проверка каждой фичи в реальном приложении.
