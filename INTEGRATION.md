# Интеграция скелетона «HR-аналитика»

Документ для разработчика: как перенести решение из демо в продуктовый проект.

**Макет Figma:** [HR-аналитика](https://www.figma.com/design/3bEptYKwLvZ3IN26h5l4RY)  
**Базовая ширина:** 394px (адаптив через `clamp` / `vw`)

---

## 1. Что передать

### Обязательные файлы

| Файл | Назначение |
|------|------------|
| `page-skeleton.css` | Скелетон: анимация, layout, overlay-цвета |
| `upload/*.svg` | Иконки для скелетона и UI (4 файла) |

### Стили loaded-состояния

| Файл | Назначение |
|------|------------|
| `demo.css` | Стили реального контента страницы |

В продукте `demo.css` нужно **переименовать/разбить** (например `hr-analytics.page.css`) и **удалить** блок `.demo-controls` — это только для локального демо.

### Демо (не переносить в prod)

| Файл / блок | Причина |
|-------------|---------|
| `index.html` целиком | Эталон разметки, не готовый модуль |
| `.demo-controls` + JS кнопок | Только для QA |
| `#content-template` с мок-данными | Заменить на реальный компонент/роут |

---

## 2. Архитектура

Скелетон и контент — **два независимых слоя**:

```
┌─────────────────────────────────────────┐
│  .sk-chrome (fixed, z-index 10002)      │  ← скелетон, pointer-events: none
│    .sk-heading                          │
│    .sk-region-block                     │
│    .sk-chips                            │
│    .sk-filters-row                      │
│    .sk-cards-grid → .sk-metric-card × N  │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│  body::before (fixed, z-index 9998)     │  ← белая подложка
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│  #app (z-index 9999)                    │  ← реальный контент
│    .cf  ← маркер «данные загружены»     │
└─────────────────────────────────────────┘
```

### Переключение скелетон → контент

Чисто на CSS, без JS для hide/show:

```css
/* page-skeleton.css */
body:has(#app .cf)::before,
body:has(#app .cf) .sk-chrome {
  opacity: 0;
  visibility: hidden;
}

html:has(#app .cf) {
  animation: none; /* останавливает пульсацию overlay */
}
```

**Контракт:** внутри `#app` появляется элемент с классом `.cf` — скелетон скрывается.

---

## 3. HTML: скелетон-слой

Скопировать в layout страницы (рядом с `#app`, не внутри):

```html
<div id="app"></div>

<div class="sk-chrome" aria-hidden="true">
  <header class="sk-heading">
    <p class="sk-page-title">HR-аналитика</p>
    <div class="sk-search-field">
      <img src="…/search-icon.svg" alt="" width="20" height="20" aria-hidden="true" />
      <span class="sk-search-placeholder">поиск показателя</span>
    </div>
  </header>

  <div class="sk-region-block">…</div>
  <div class="sk-chips"><span></span><span></span><span></span></div>

  <div class="sk-filters-row">…</div>

  <div class="sk-cards-grid"></div>
  <div class="sk-chips-fade"></div>
</div>

<template id="sk-card-template">
  <div class="sk-metric-card">
    <span class="sk-skel-bar sk-card-bar--title"></span>
    <span class="sk-skel-bar sk-card-bar--subtitle"></span>
    <div class="sk-card-value-row">
      <span class="sk-skel-bar sk-card-bar--value"></span>
      <span class="sk-skel-bar sk-card-bar--pill"></span>
    </div>
    <span class="sk-skel-bar sk-card-bar--baseline"></span>
  </div>
</template>
```

Полная разметка — в `index.html` (блок `.sk-chrome`).

### Инициализация карточек (JS)

Сетка карточек пустая; карточки клонируются из `<template>`:

```js
const SKELETON_CARD_COUNT = 6; // или из API / конфига

function initSkeletonCards() {
  const grid = document.querySelector(".sk-cards-grid");
  const tpl = document.getElementById("sk-card-template");
  const fragment = document.createDocumentFragment();
  for (let i = 0; i < SKELETON_CARD_COUNT; i += 1) {
    fragment.appendChild(tpl.content.cloneNode(true));
  }
  grid.appendChild(fragment);
}

initSkeletonCards();
```

В React/Vue/Angular: рендерить 6 (или N) `.sk-metric-card` с той же внутренней структурой вместо template + JS.

---

## 4. HTML: loaded-состояние

Корневой контейнер **обязан** иметь класс `cf`:

```html
<div id="app">
  <div class="cf">
    <header class="page-header">…</header>
    <section class="region-block">…</section>
    <!-- остальной контент -->
  </div>
</div>
```

Структура и BEM-классы loaded UI — см. `#content-template` в `index.html` и стили в `demo.css`.

### Соответствие скелетон ↔ loaded

| Скелетон | Loaded |
|----------|--------|
| `.sk-heading` | `.page-header` |
| `.sk-region-block` | `.region-block` |
| `.sk-chips` | `.scenarios` |
| `.sk-filters-row` | `.filters-row` |
| `.sk-metric-card` | `.metric-card` |

Отступы loaded-состояния используют те же CSS-переменные `--sk-*` через алиасы в `:root` (`demo.css`).

---

## 5. Подключение стилей

```html
<link rel="stylesheet" href="page-skeleton.css" />
<link rel="stylesheet" href="hr-analytics.page.css" /> <!-- бывший demo.css -->
```

**Порядок важен:** `page-skeleton.css` первым.

### Дизайн-токены продукта

Скелетон читает переменные с fallback. Подключите на `html` или `:root`:

| Переменная продукта | Использование |
|---------------------|---------------|
| `--cMain_Background` | Фон подложки `body::before` |
| `--cMain_TextMain` | Цвет заголовка скелетона |
| `--cMain_Stroke` | Обводка поиска, карточек loaded |
| `--cMain_FontFamily` | Шрифт (через `--sk-font`) |

Если токенов нет — работают fallback из CSS.

---

## 6. Overlay и анимация

Спека компонента overlay:

```css
/* default */
background-color: rgba(0, 0, 0, 0.10);
/* loading */
background-color: rgba(0, 0, 0, 0.04);
/* transition при смене класса */
transition: background-color 600ms ease-in-out 300ms;
```

В скелетоне реализовано через пульсацию:

```css
--sk-overlay: rgba(0, 0, 0, var(--sk-alpha));
/* --sk-alpha: 0.1 ↔ 0.04, цикл 1.8s */
```

Используется для: `.sk-skel-bar`, border `.sk-metric-card`, чипов.

`prefers-reduced-motion: reduce` — анимация отключается, alpha фиксируется на `0.1`.

---

## 7. Интеграция в SPA (React пример)

```tsx
// App.tsx
export function HrAnalyticsPage() {
  const { data, isLoading } = useHrMetrics();

  return (
    <>
      {/* Скелетон — пока нет .cf внутри #app */}
      <SkeletonChrome cardCount={6} />

      <div id="app">
        {!isLoading && data && (
          <div className="cf">
            <HrAnalyticsContent data={data} />
          </div>
        )}
      </div>
    </>
  );
}
```

Альтернатива без `:has()` — вешать класс на `body`:

```css
body.is-content-ready .sk-chrome { opacity: 0; visibility: hidden; }
```

Тогда в `page-skeleton.css` заменить селектор `#app .cf` на `.is-content-ready`.

---

## 8. Кастомизация

| Задача | Где менять |
|--------|------------|
| Число карточек скелетона | `SKELETON_CARD_COUNT` / цикл рендера |
| Тексты фильтров в скелетоне | `.sk-filter-item` в HTML |
| Прозрачность поиска (50%) | `.sk-search-field { opacity: 0.5 }` |
| Высота карточки (128px @ 394) | `--sk-card-h` в `page-skeleton.css` |
| Отступ baseline от value-row (12px) | `--sk-card-section-gap` |
| Ширины полосок в карточке | `--sk-bar-w1` … `--sk-bar-w5` |
| Селектор контейнера приложения | Заменить `#app` в `page-skeleton.css` и `demo.css` |

### Карточка метрики (Figma `1801:68020`)

- Размер: 128×186px (высота × ширина колонки на 394px)
- Padding: 12px верт. / 16px гор.
- Секции: title → gap 12 → value → gap 12 → baseline
- Полоски: 79 / 135 / 68×20 + pill 30 / 81 px (max, с `min()` для адаптива)

---

## 9. Чеклист приёмки

- [ ] `page-skeleton.css` подключён до стилей страницы
- [ ] `.sk-chrome` в DOM до загрузки данных
- [ ] Карточки скелетона отрисованы (6 или нужное N)
- [ ] После загрузки в `#app` есть `.cf` — скелетон исчезает
- [ ] Пульсация overlay 0.10 ↔ 0.04 на полосках и border карточек
- [ ] На 394px: заголовок 34px, поиск 48px, карточка 128px
- [ ] `prefers-reduced-motion` — без анимации
- [ ] Один `<h1>` только в loaded-состоянии (скелетон использует `<p>`)
- [ ] Иконки из `upload/` доступны по путям в prod-сборке

---

## 10. Локальный запуск демо

```bash
cd Аналитика
python3 -m http.server 8877
# http://localhost:8877/index.html        — скелетон, кнопки «Загрузить» / «Сбросить»
# http://localhost:8877/index.html?autoload — автозагрузка через 3 с
```

---

## 11. Структура репозитория (финал)

```
Аналитика/
├── INTEGRATION.md          ← этот файл
├── page-skeleton.css       ← переносить в проект
├── demo.css                ← стили loaded (без .demo-controls)
├── index.html              ← референс разметки
├── upload/
│   ├── search-icon.svg
│   ├── chevron-down.svg
│   ├── chevron-down-16.svg
│   └── trash-icon.svg
└── .gitignore
```

---

## 12. Вопросы к продуктовой команде

1. **Селектор `#app`** — совпадает с корневым mount-point в приложении?
2. **Класс `.cf`** — можно ли использовать или нужен другой маркер (например data-attribute)?
3. **Количество карточек** в скелетоне — всегда 6 или зависит от viewport/данных?
4. **Status bar iOS** (54px в Figma) — в вебе используется `safe-area-inset-top` через `--sk-safe-top`.
