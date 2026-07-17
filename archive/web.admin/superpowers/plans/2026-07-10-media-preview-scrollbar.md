# Media preview wrapper + global scrollbar Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Спека `docs/superpowers/specs/2026-07-10-media-preview-scrollbar-design.md` — два точечных UI-фикса.

**Architecture:** Только классы в `MediaDetailsDialog` и CSS в `@layer base` `src/index.css`.

**Tech Stack:** без изменений. Отступления: без тестов (build+lint+глазами); Commit — с разрешения.

### Task 1: Врапер превью

**Files:** Modify `src/features/media/MediaDetailsDialog.tsx`

- [ ] **Step 1:** Заменить контейнер и img:

```tsx
          <div className="flex aspect-square w-full items-center justify-center self-start overflow-hidden rounded-lg border bg-muted">
            <img
              src={publicUrl}
              alt={item.original_name}
              className="max-h-full max-w-full object-contain"
              onLoad={(e) =>
                setDimensions(`${e.currentTarget.naturalWidth} × ${e.currentTarget.naturalHeight} px`)
              }
            />
          </div>
```

- [ ] **Step 2:** `npm run build && npm run lint` → успех.

### Task 2: Скроллбар

**Files:** Modify `src/index.css` (в конец блока `@layer base`, после правила cursor)

- [ ] **Step 1:** Добавить:

```css
  /* Стабильный gutter: появление скролла не меняет ширину контента. */
  html {
    scrollbar-gutter: stable;
  }

  /* Минималистичный скроллбар (Chrome 121+, Firefox). */
  * {
    scrollbar-width: thin;
    scrollbar-color: var(--border) transparent;
  }

  /* Safari: стандартные свойства не поддержаны — webkit-фолбэк. */
  @supports not (scrollbar-color: auto) {
    *::-webkit-scrollbar {
      width: 8px;
      height: 8px;
    }
    *::-webkit-scrollbar-track {
      background: transparent;
    }
    *::-webkit-scrollbar-thumb {
      background: var(--border);
      border-radius: 9999px;
    }
    *::-webkit-scrollbar-thumb:hover {
      background: var(--muted-foreground);
    }
  }
```

- [ ] **Step 2:** `npm run build && npm run lint` → успех.

### Task 3: Доки

- [ ] ux-ui.md §3.6 (модалка: квадратный врапер превью) и §1 (глобальные правила:
  скроллбар); CLAUDE.md — строка в статус. Финальный build+lint.
