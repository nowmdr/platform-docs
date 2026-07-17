# Products v4: SEO-фолбэк, Save в гард-модалке, сортировка, фильтры, markdown-редактор

Дата: 2026-07-10. Статус: утверждён пользователем (чат). База: спеки products v1–v3.

## 1. SEO-поля со свёрнутым состоянием (форма товара)

- Если `seo_title` и `seo_description` пусты (в БД null) — вместо полей блок:
  пунктирная рамка, текст «Search engines use the product title and description.»
  и ghost-кнопка **Customize** (иконка Pencil).
- Customize разворачивает секцию: заголовок «SEO» + кнопка **Use defaults**
  (RotateCcw) и два поля, предзаполненные текущими title/description формы
  (то, что SEO использует сейчас; предзаполнение НЕ делает форму dirty).
- Use defaults очищает оба поля (dirty считается против defaults) и сворачивает.
- Если у товара SEO-значения есть — секция сразу развёрнута.
- Сохранение: значение пустое ИЛИ равное (trim) соответствующему title/description
  → в БД `null` (фолбэк остаётся «живым» — SEO меняется вместе с товаром).
  Контракт: сайт при пустых seo-полях использует title/description.

## 2. Save в модалке «Discard unsaved changes?»

Три кнопки: **Stay** (cancel) / **Discard** (destructive, `blocker.proceed()`) /
**Save** (primary, чёрная): сабмит формы →
- валидна: мутация save → `blocker.proceed()` — уход туда, куда шёл пользователь;
  для нового товара внутренний redirect на созданный товар подавляется
  (`leavingRef`), навигация продолжается по прерванному пути;
- невалидна: `blocker.reset()` — диалог закрывается, остаёмся на странице,
  ошибки подсвечены;
- мутация упала: toast ошибки + `blocker.reset()` (остаёмся).
На время сохранения кнопки блокируются («Saving…»).

## 3. Сортировка списка

`created_at desc` уже стоит, но у импортированных товаров совпадают timestamps
(39 строк / 7 уникальных `created_at`) — внутри пачки порядок нестабилен.
Добавить tiebreaker `title asc`. Новые товары — всегда сверху.

## 4. Порядок фильтров

В панели списка поменять местами: **Brand**, затем **Category** — как в строке
товара (`brand · category`).

## 5. Markdown-редактор description (переиспользуем для блога)

- Новый компонент `src/components/RichTextEditor.tsx` (generic, не products):
  пропсы `{ value: string; onChange: (markdown: string) => void }`.
- TipTap v3 (`@tiptap/react`, `@tiptap/starter-kit`) + официальный
  `@tiptap/markdown` (v3, та же версия). В форме — Controller на `description`.
- Тулбар: Bold, Italic, Bullet list, Ordered list (ghost-кнопки с активным
  состоянием). Отключить в StarterKit: heading, blockquote, code, codeBlock,
  horizontalRule, strike (остаются абзацы, hard break, списки, bold/italic,
  history). Ссылок нет (в StarterKit v3 link отключить, если входит).
- Хранение: в `description` — Markdown (списки `- пункт`, абзацы пустой строкой).
  Читается как плоский текст, пока сайт не переведён на react-markdown.
- Стили контента — arbitrary-варианты Tailwind на контейнере
  (`[&_ul]:list-disc [&_ul]:pl-5 [&_ol]:list-decimal [&_ol]:pl-5` и т.п.),
  рамка/фокус как у Input/Textarea.
- Отдельный деливерабл: промпт для агента репозитория сайта cozycorner
  (рендер markdown в description; сохранить в docs и выдать в чат).

## Проверка

`npm run build` + `npm run lint`; ручная e2e: SEO-блок (свернут у пустых,
Customize предзаполняет, Use defaults сворачивает, сохранение совпадающих
значений оставляет null в БД); Save из гард-модалки уводит по прерванному пути;
порядок списка стабилен, новый товар сверху; фильтры Brand→Category; редактор:
списки/абзацы/жирный, в БД markdown, повторное открытие рендерит обратно.

## Вне скоупа

Рендер markdown на сайте (отдельный репозиторий, промпт прилагается), заголовки
и ссылки в редакторе, редактор для полей блога (возьмём этот же компонент).
