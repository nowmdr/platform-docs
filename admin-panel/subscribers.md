# Раздел Subscribers — правила и концепции

> Last updated: 2026-07-23 | Source project: web.admin — пути `src/…` относятся к репозиторию `web.admin/`

Документ фиксирует, как устроен раздел Subscribers админки: контракт с данными и
решения по UI. Модель данных — [../database/schema.md](../database/schema.md) §5
(таблица `subscribers`); дизайн —
`../archive/web.admin/superpowers/specs/2026-07-23-subscribers-design.md`. Читать
перед правками раздела.

## 1. Главные концепции

- **Источник списка — таблица `<schema>.subscribers`** (для CozyCorner —
  `cozycorner.subscribers`, миграция 0025). Строка: `id`, `email` (unique,
  регистронезависимо), `created_at`. По GDPR таблица минимальна — **ни имени, ни
  IP**; «информация о подписчике» = email + дата подписки.
- **Запись НЕ из админки.** Подписки собирает форма на сайте (`NewsletterForm`) —
  серверным действием под `service_role` после проверки Turnstile. Админка таблицу
  **только читает и удаляет** (RLS `Admin manage subscribers`, `is_admin()`) —
  insert-политики для admin нет и не нужно (импорта нет).
- **Никакого service-role ключа в админке** (как и в остальных разделах): чтение и
  удаление идут под сессией админа.

## 2. Данные (`src/lib/subscribers.ts`)

- `listSubscribers(site)` — `select id,email,created_at`, сортировка
  `created_at desc` (свежие первыми). Через `getDb(site)`.
- `deleteSubscriber(site, id)` — `delete().eq('id', id)`.
- Тип `Subscriber = { id; email; created_at }`.
- Кэш TanStack Query — ключ `['subscribers', site.slug]` (общий, не форкать).

## 3. Экспорт CSV (`src/lib/csv.ts`)

Общий (не привязан к разделу) помощник:
- `toCsv(rows, columns)` — текст CSV с заголовком и экранированием по RFC 4180
  (обёртка в кавычки + удвоение кавычек, если в значении `,`/`"`/перевод строки);
  разделитель строк — CRLF.
- `downloadCsv(filename, csv)` — скачивание через `Blob` + object URL, с UTF-8 BOM
  (чтобы Excel корректно читал кириллицу в доменах). Без новых зависимостей.
- Покрыт юнит-тестом `src/lib/csv.test.ts`.

Кнопка **Export CSV** выгружает **весь** список (игнорирует строку поиска),
колонки `email,subscribed_at` (created_at в ISO), имя файла
`<slug>-subscribers-YYYY-MM-DD.csv`.

## 4. UI (`src/features/subscribers/SubscribersPage.tsx`)

- **Шапка** (одна строка): заголовок «Subscribers» + счётчик (`N` или `N / total`
  при активном поиске) → поле поиска по email (`max-w-56`) → кнопки **Select** и
  **Export CSV** (обе видны только при непустом списке).
- **Поиск** — клиентский фильтр по email по мере ввода (`useDeferredValue`): лупа
  меняется на спиннер, таблица гаснет до 60% — как в Media/Products.
- **Таблица** — shadcn `Table` (первое использование компонента в проекте).
  Колонки: **Email · Subscribed (`formatDate`) · действие удаления**.
- **Удаление**: иконка-корзина в строке (подтверждение `AlertDialog`) и **режим
  выбора** (кнопка Select → колонка чекбоксов + бар «N selected / Cancel /
  BulkDeleteButton»). Переиспользуются `useBulkDelete` и `BulkDeleteButton` из
  `src/features/folders/` (обе не зависят от папок). `SelectionBar` НЕ используется
  (он завязан на папки/перенос) — бар здесь локальный минимальный.
- **Состояния**: загрузка — skeleton-строки; пусто — пунктирная «No subscribers
  yet.»; пустой результат поиска — «No subscribers match your search.»; ошибка
  запроса — красная плашка.

## 5. Мультисайтовость

Пункт меню показывается для всех сайтов (как все разделы), но таблица
`subscribers` сейчас есть только у `cozycorner`. У сайта без неё раздел покажет
ошибку загрузки — принято осознанно: в реестре сейчас только CozyCorner.
