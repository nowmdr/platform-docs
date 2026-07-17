# Products v3: Clear-фильтры, публичный URL картинки, гард несохранённых изменений

Дата: 2026-07-10. Статус: утверждён пользователем (чат).
База: спеки products v1/v2 от 2026-07-10.

## 1. Кнопка Clear в панели фильтров (`ProductsPage`)

Ghost-кнопка «Clear» (иконка X) в конце панели поиска/фильтров; видна только когда
активен поиск или любой из фильтров (`hasFilter`). Клик сбрасывает search='',
category=ALL, brand=ALL.

## 2. Image path — публичный URL (форма товара)

Проблема: после выбора из галереи в инпуте плоский ключ (имя файла) — непонятно.
Решение: инпут всегда показывает **публичную ссылку**, в БД по-прежнему плоский
ключ (контракт спеки §7 не меняется).

- `lib/images.ts`: новый хелпер `toStoragePath(site, value)` — если строка
  начинается с `${projectUrl}/storage/v1/object/public/${bucket}/`, вернуть
  остаток (плоский ключ), иначе вернуть как есть.
- `toFormValues`: `image_path` = `resolveImageUrl(site, product.image_path)`
  (пустой — остаётся пустым).
- Пикер `onSelect`: в форму пишется `resolveImageUrl(site, path)`.
- `toInput`: `image_path` = `orNull(toStoragePath(site, values.image_path))` —
  наш storage-URL сворачивается в ключ, внешние URL сохраняются как есть.
- Подсветка текущей в пикере: `currentPath = toStoragePath(site, imagePath)`.
- Превью работает без изменений (`resolveImageUrl` пропускает http-URL).

## 3. Гард несохранённых изменений (форма товара)

- Миграция роутера на data-режим: `App.tsx` экспортирует
  `createBrowserRouter(createRoutesFromElements(<текущее дерево Route>))`;
  `main.tsx` — `<RouterProvider router={router}/>` вместо `<BrowserRouter><App/>`.
  Toaster остаётся рядом с провайдером. Компоненты не меняются.
- В `ProductForm`: `useBlocker` с условием `isDirty && !save.isPending &&
  !remove.isPending`. При `blocker.state === 'blocked'` — AlertDialog
  «Discard unsaved changes?» / «You have unsaved changes. If you leave this page,
  they will be lost.»; Cancel → `blocker.reset()`, Discard (destructive) →
  `blocker.proceed()`.
- `beforeunload` (useEffect, пока isDirty) — нативное подтверждение браузера при
  закрытии вкладки/обновлении.
- Чтобы навигация после мутаций не блокировалась: Save (update) →
  `form.reset(values)` (значения становятся новыми defaults, форма чистая);
  Save (create) → `form.reset(values)` до navigate; Delete → `form.reset()` до
  navigate.

## Проверка

`npm run build` + `npm run lint`. Ручная e2e: Clear сбрасывает всё; выбор из
галереи кладёт в инпут публичный URL, после Save в БД — плоский ключ (проверить
через список/переоткрытие товара), внешний URL сохраняется как есть; изменение
поля → Back to products/браузерная назад → диалог, Discard уходит, Cancel
остаётся; после Save уход свободный; refresh с dirty-формой — браузерный prompt.

## Вне скоупа

Гард на других формах (login и будущих CRUD), автосохранение черновиков.
