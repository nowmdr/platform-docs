# Products v2: поиск/фильтры списка + пикер картинок — дизайн

Дата: 2026-07-10. Статус: утверждён пользователем (чат).
База: `docs/superpowers/specs/2026-07-10-products-crud-design.md` (products v1).

## Цель

1. На странице списка товаров — панель с поиском и двумя фильтрами (Category, Brand).
2. На детальной странице — выбор картинки из медиа-библиотеки: кнопка Gallery рядом
   с инпутом Image path открывает диалог-пикер (выбор из загруженных + Upload).

## 1. Панель поиска и фильтров (`ProductsPage`)

**Данные:** `listProducts` расширяется до `id, title, brand, category, created_at`
(тип `ProductListItem` дополняется `brand`, `category`). Фильтрация — клиентская.

**UI:** под шапкой (Products + счётчик + New product) — панель-строка:
- Поиск: Input с иконкой-лупой, `max-w-56`, фильтр по `title` (substring,
  case-insensitive), `useDeferredValue` + спиннер/приглушение грида — паттерн MediaPage.
- Select **Category**: пункт «All categories» (сентинел `__all__`) + полный список из
  таблицы `categories` (существующий `listCategoryNames`, query
  `['categories', site.slug]`).
- Select **Brand**: пункт «All brands» + уникальные непустые `brand` из загруженных
  товаров, сортировка по алфавиту.
- Условия совмещаются по AND.

**Строка списка:** название слева; справа приглушённый xs-текст `brand · category`
(только заполненные части; ничего — если оба пусты).

**Состояния:** счётчик при активном любом фильтре/поиске — `видимые / все`, иначе
общее число. Пустой результат фильтрации — «No products match your filters.»
(пунктирная плашка); прочие состояния — как в v1.

## 2. Пикер картинок (`ImagePickerDialog` + `ProductEditPage`)

**Точка входа:** в поле Image path инпут и outline-кнопка **Gallery** (иконка Images)
в одной строке. Клик — открывается диалог.

**`src/features/products/ImagePickerDialog.tsx`:**
- Управляемый Dialog (`open`/`onOpenChange`), ширина как у модалки Media (`max-w-3xl`).
  Пропсы: `site`, `open`, `onOpenChange`, `currentPath: string | null`,
  `onSelect(path: string): void`.
- Шапка: заголовок «Choose image», поиск по имени/пути (клиентский), кнопка
  **Upload** (скрытый file input, один файл, accept как в Media). Upload использует
  `uploadImage` из `lib/media` (валидация mime/6МБ, sanitize, суффиксы, откат) →
  инвалидация `['media', site.slug]` → `onSelect(path)` загруженного файла → диалог
  закрывается. Ошибка — toast.
- Тело: грид миниатюр 3/4/5 колонок из `listImages` (query `['media', site.slug]` —
  общий кэш с разделом Media): квадратное превью + имя (truncate). Карточка с
  `path === currentPath` подсвечена (ring). Клик → `onSelect(path)` → закрыть.
- Состояния: skeleton-грид при загрузке; «No images yet.» / «No images match your
  search.»; ошибка списка — красная плашка внутри диалога.

**Интеграция в форму:** `onSelect` → `form.setValue('image_path', path,
{ shouldDirty: true })` + сброс флага битой картинки. Сохранение в БД — по-прежнему
только кнопкой Save.

## Ошибки

Как в v1: мутации — toast, списки — красные плашки. Upload-ошибки пикера — toast
с сообщением из `uploadImage`.

## Проверка

`npm run build` + `npm run lint`; ручная e2e: поиск и фильтры сужают список (и
комбинируются), сброс на All возвращает всё; Gallery открывает пикер, выбор
подставляет путь и обновляет превью, Save сохраняет; Upload из пикера кладёт файл
в бакет+media и сразу подставляет его в товар.

## Вне скоупа

Серверная пагинация/фильтрация, мультивыбор картинок, drag-n-drop в пикер,
сортировки списка, фильтр по цене.
