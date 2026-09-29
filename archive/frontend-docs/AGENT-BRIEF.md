# Бриф для агентов: фронтенд «CRM ИТ Школы Ростелекома»

Читай целиком до начала работы. Это единый источник правил для всех агентов.

## 1. Что делаем

Готовый веб-клиент (SvelteKit 2 + Svelte 5 runes + TypeScript strict) к бэкенду `D:\testkit-lct\backend`
(FastAPI, модульный монолит) на дизайн-системе `D:\testkit-lct\rt-ui` (пакет `@lct-testkit/rt-ui`, Rostelecom Atomaro
на Svelte 5). Проект фронта: `D:\testkit-lct\frontend`.

Цель — **связать все ручки бэкенда** в качественный, интуитивный пользовательский поток. Не «админка над CRUD»,
а продукт, которым приятно пользоваться менеджеру (КАМ), руководителю (HEAD), администратору (ADMIN) и аудитору (AUDITOR).

Источники требований (по приоритету): `D:\testkit-lct\backend\_spec\spec.txt`, `D:\testkit-lct\new_spec.md`,
`D:\testkit-lct\dop.md` (ПЭП, автоподстановка по ИНН, дизайн-система), сам код бэкенда (роутеры/схемы/сервисы — это
истина о контракте; тесты в `backend/tests` показывают реальные payload'ы).

Два режима поставки (одна сборка, режим — `static/config.json` → `mode`):
* **demo** — на экране входа выбор учётной записи (KAM / HEAD / ADMIN / AUDITOR), вход в один клик;
* **prod** — обязательная аутентификация через Keycloak (OIDC, серверная сессия и CSRF на стороне бэкенда).

## 2. Окружение (Windows, Git Bash / PowerShell)

* Менеджер пакетов — **pnpm**. Ты **НЕ** запускаешь `pnpm install/add/remove` (гонки за node_modules/lockfile между агентами).
  Нужна новая зависимость — допиши строку в `docs/dep-requests.md` и продолжай с запасным вариантом; лид поставит.
  Уже стоят: `@lct-testkit/rt-ui`, `@xyflow/svelte`, `dayjs`, `dompurify`, `marked`, `openapi-fetch`, `pdfjs-dist`, `playwright`, `vitest`.
* Dev-сервер уже запущен лидом: **http://localhost:5273** (HMR). Свой dev-сервер на другом порту не поднимай, в чужие порты не лезь.
  Vite проксирует `/api`, `/public`, `/health`, `/auth` на бэкенд `http://localhost:8080` (Caddy: FastAPI + Keycloak).
* Бэкенд поднимает лид в docker (`backend/docker-compose.yml`, профиль `demo`). Пока он не поднялся — работай по коду бэкенда.
  Когда поднимется: Swagger `http://localhost:8080/api/docs`, схема `http://localhost:8080/api/openapi.json`.
  Типы API генерируются в `src/lib/api/schema.d.ts` командой `pnpm gen:api` (её запускает лид; файл появится сам).
* Демо-учётки (Keycloak realm `crm`): `admin.crm / Admin12345678!` (ADMIN), `head.petrov / Head12345678!` (HEAD),
  `kam.ivanov / Kam123456789!` (KAM), `auditor.smirnov / Audit12345678!` (AUDITOR).
* Проверка кода: `pnpm check` (svelte-check, должно быть 0 ошибок в твоих файлах), `pnpm build` (должен собираться).
  Ошибки в чужих каталогах не чини — отметь в отчёте.
* Визуальная проверка: `node tools/shot.mjs` (см. §8) — скриншоты в `.shots/<агент>/`, смотри их инструментом Read.
* **Не делай git commit / push.** Не трогай `backend/` и `rt-ui/` (только читай).

## 3. Архитектура клиента

* SPA (`ssr = false`, `adapter-static`, fallback `index.html`). Бэкенд сам является BFF: `/api/auth/login` → Keycloak →
  `/api/auth/callback` ставит httpOnly-cookie сессии и CSRF-cookie `crm_csrf`. На мутирующих запросах клиент шлёт
  заголовок `X-CSRF-Token` (делает общий слой, тебе руками не надо).
* В demo-режиме токены получает клиент (Keycloak password grant) и шлёт `Authorization: Bearer` — тоже общий слой.
* Все запросы идут на тот же origin: `/api/...`, `/public/...`.

### Общий слой (пишет лид; смотри реальный код в `src/lib`, при расхождении с этим списком прав код)

```ts
import { api, unwrap, ApiError, idem, ifMatch, errorMessage } from '$lib/api';
// api — типизированный клиент openapi-fetch по src/lib/api/schema.d.ts
const page = await unwrap(api.GET('/api/deals', { params: { query: { limit: 50, cursor } } }));
const deal = await unwrap(api.POST('/api/deals', { body, headers: idem() }));            // Idempotency-Key
await unwrap(api.PATCH('/api/deals/{id}', { params: { path: { id } }, body, headers: ifMatch(deal.version) }));
// ApiError: status, code ('CRM-1002'), title, detail, requestId, errors[{field,reason,code}], extra; .fieldErrors()
import { session } from '$lib/auth/session.svelte';       // session.me, session.role, session.can('deal:update')
import { toast } from '$lib/ui/toast.svelte';            // toast.success('...'), toast.error(e), toast.info('...')
import { confirm } from '$lib/ui/confirm.svelte';        // if (await confirm({title, message, confirmLabel, danger:true})) ...
import { createPager } from '$lib/api/pager.svelte';     // курсорная пагинация: items, loading, hasMore, loadMore(), reload()
import { uploadFile } from '$lib/api/upload';            // upload-intent → PUT → commit, onProgress
```

Общие UI-компоненты — `src/lib/ui/*` (PageHeader, Page, EmptyState, ErrorState, ResponsiveTable, StatusChip, Money,
DateText, Avatar, FormSection …). **Читай их исходники перед использованием**, не изобретай второй раз.
Форматтеры — `src/lib/utils/format.ts` (деньги, даты, числа, склонения), `src/lib/utils/*`.

## 4. Владение файлами (не пересекаться!)

| Владелец | Каталоги |
|---|---|
| Лид | `src/lib/api/**`, `src/lib/auth/**`, `src/lib/ui/**`, `src/lib/utils/**`, `src/lib/stores/**`, `src/lib/nav.ts`, `src/routes/+layout*`, `src/routes/login/**`, `src/routes/(app)/+layout*`, `src/routes/(app)/+page.svelte` (главная), `static/**`, `tools/**`, конфиги в корне, `docs/AGENT-BRIEF.md` |
| **A — CRM** | `src/lib/features/crm/**`; маршруты `src/routes/(app)/deals/**`, `organizations/**`, `contacts/**`, `tasks/**`, `notifications/**` |
| **B — Настройка и данные** | `src/lib/features/config/**`; маршруты `src/routes/(app)/workflows/**`, `catalog/**`, `imports/**`, `reports/**`, `admin/registry/**`, `admin/integrations/**`, `admin/notification-templates/**` |
| **C — Доступы, подпись, публичные страницы** | `src/lib/features/identity/**`, `src/lib/features/signing/**`; маршруты `src/routes/(app)/profile/**`, `signing/**`, `admin/users/**`, `admin/teams/**`, `admin/approvals/**`, `admin/audit/**`, `admin/erasure/**`, `admin/settings/**`, `admin/edm/**`; публичные `src/routes/sign/**`, `src/routes/verify/**` |

Внутри своей области структуру выбираешь сам. В чужие каталоги не пиши. Нужно изменение в общем слое или в чужой области —
запиши в `docs/lead-requests.md` (одна строка: кто, что, зачем) и продолжай с обходным путём.

Точки стыковки (лид создал заглушки — владелец ниже ЗАМЕНЯЕТ заглушку своей реализацией с тем же путём и props):
* `src/lib/features/crm/notifications/NotificationBell.svelte` (A) — колокольчик в верхней панели (счётчик + всплывающий список).
* `src/lib/features/signing/DealSignatures.svelte` (C, props `{ dealId: string }`) — блок «Подписание» внутри карточки сделки (A встраивает).
* `src/lib/features/crm/files/Attachments.svelte` (A, props `{ entityType, entityId, canEdit }`) — вложения (используют B/C).
* `src/lib/features/crm/lookup/OrgLookup.svelte` (A) — поле «ИНН или название» с автоподстановкой из реестра (используют B при необходимости).

Навигация (боковое меню, хлебные крошки) — `src/lib/nav.ts`, уже содержит все твои маршруты. Ссылки на свои страницы ставь как в nav.

## 5. Принципы UX (это главный критерий приёмки — заказчик придирчив)

1. **Интуитивно с первого взгляда.** Пользователь без обучения понимает, что делает каждая кнопка. Главное действие страницы — одно,
   визуально выделено (`Button` primary), остальные — вторичные/иконки.
2. **Никаких лишних текстов.** Не пиши подзаголовки-пересказы («Здесь вы можете...»), длинные описания, дублирующие подписи,
   «Нажмите кнопку…». Иконка-кнопка (`IconButton`) с `aria-label` и `Tooltip` там, где смысл очевиден (закрыть, редактировать,
   удалить, обновить, скачать, копировать, ещё). Текст на кнопке — только для главного действия и там, где иконка неоднозначна.
   Заголовки — короткие существительные. Пустые состояния — одна строка + действие.
3. **Поток, а не формы.** Многошаговое → мастер (`Wizard`/степпер) или пошаговые панели; создание сущности — минимум полей
   (остальное — потом в карточке); умные значения по умолчанию; автофокус; Enter отправляет, Esc закрывает; выбор из справочника
   вместо ручного ввода; поиск с дебаунсом; сохранение фильтров в URL (query params), чтобы работала кнопка «Назад» и ссылки.
4. **Состояния всегда:** загрузка (skeleton/`Loader`, без «прыжков» вёрстки), пусто (`EmptyState` с действием), ошибка
   (`ErrorState` с «Повторить», человеческий текст из `errorMessage`), успех (`toast.success` коротко), опасные действия —
   `confirm()` с названием того, что удаляется. Оптимистичная блокировка: на `409 CRM-1002` показать понятное «данные изменены
   другим пользователем» и предложить обновить (не терять ввод пользователя). Идемпотентные создания — `headers: idem()`.
5. **Права.** Кнопки, недоступные роли, **не показываем** (или, если важно объяснить, показываем disabled с Tooltip). Проверка через
   `session.can('permission')` (права приходят в `/api/me → scopes`). Но истина — бэкенд: 403 обрабатываем аккуратно.
6. **Адаптивность — обязательна и проверяется.** Три режима: телефон < 768, планшет 768–1023, десктоп ≥ 1024 (хук `useBreakpoint()`
   из `@lct-testkit/rt-ui/ext`). Нет горизонтальной прокрутки страницы ни на 360 px; таблицы на телефоне → карточки
   (`ResponsiveTable`/`TableCards`); модальные окна и Drawer помещаются в окно на любой высоте (прокрутка внутри, кнопки действий
   закреплены внизу); цели касания ≥ 44 px на телефоне (`size="l"`); учитывай низкие окна (ландшафт телефона 844×390,
   ноутбук 1280×600): фильтры и шапки не должны съедать экран — сворачивай, прокручивай контент, а не всю страницу с фиксированными
   огромными блоками.
7. **Единый стиль.** Компоненты rt-ui + Tailwind-утилиты на токенах дизайн-системы (см. §6). Никаких захардкоженных
   цветов/шрифтов/теней. Тёмная тема работает (проверь хотя бы одну страницу в `--theme rtk_default_dark`).
8. **Язык — русский**, форматы `ru-RU` (даты `дд.мм.гггг`, деньги `1 250 000 ₽`, числа с неразрывным пробелом). Без англицизмов в UI.
9. **Доступность:** `aria-label` у иконок-кнопок, порядок Tab, фокус в модалках, контраст токенов; `prefers-reduced-motion` уважаем.
10. **Скорость:** отклик ≤ 1 с. Списки — курсорная пагинация (`limit` ≤ 100, «Показать ещё»/бесконечная прокрутка), дебаунс поиска 300 мс,
    без N+1 запросов из UI, тяжёлое — лениво (`{#await import(...)}`).

## 6. Технические правила

* Svelte 5 **runes** (`$state`, `$derived`, `$effect`, `$props`, snippets). Никаких `export let`, `on:click` (используй `onclick`).
* TypeScript strict, без `any` (для неизвестных данных — `unknown` + сужение). Типы данных API — из `$lib/api/schema` (`components['schemas']['X']`),
  свои алиасы в `src/lib/features/<область>/types.ts`.
* Компоненты rt-ui — импорт из `@lct-testkit/rt-ui` (баррель), `@lct-testkit/rt-ui/ext` (`useBreakpoint`, `TableCards`, `Progress`,
  `ExtMotionProvider`), `@lct-testkit/rt-ui/charts` (LineChart, BarChart, DonutChart, Sparkline), `@lct-testkit/rt-ui/icons` (иконки: `import { Search } from '@lct-testkit/rt-ui/icons'`;
  список имён — `docs/icons-24.txt`), `@lct-testkit/rt-ui/components/TableGrid`. **API rt-ui — как у React-версии: `value` + `onChange(value)`,
  не `bind:value`.** Контент-пропсы принимают строку или snippet. Смотри исходники в `D:	estkit-lct
t-ui\src\lib\components\<Имя>\*.svelte`
  (JSDoc пропсов там), примеры — `rt-ui/src/routes/examples/crm/*`, справочник таблицы — `rt-ui/src/lib/components/TableGrid/README.md`,
  графики — `rt-ui/src/lib/charts/README.md`, расширения — `rt-ui/src/lib/ext/README.md`.
* Файлы ≤ ~400 строк: дели на компоненты. Логику выноси в `*.svelte.ts` / `*.ts`. Комментарии — только «почему», коротко.
* Никаких моков данных в UI: всё из реального API. Секреты, токены, OTP-коды, пароли **не логируем** и не пишем в `console`.
* Ссылки между экранами — `<a href>`/`goto`; состояние фильтров — в query string. `beforeNavigate` для несохранённых форм там, где это важно.
* Загрузка данных — в компонентах/`*.svelte.ts` через `$lib/api` (SPA, без `+page.ts load`), отмена устаревших запросов (`AbortController`)
  при быстром вводе.

### Стили: TAILWIND CSS v4 (обязательно; заказчик ненавидит «чистый CSS»)

1. **Стилизуем утилитами Tailwind прямо в разметке.** Блок `<style>` в `.svelte` — только когда без него никак (`@keyframes`, переопределение
   внутренностей rt-ui через `:global(...)`); и даже там — `@apply` утилит, а не рукописные списки свойств. Новых `.css`-файлов не создавай.
2. Токены дизайн-системы уже вшиты в тему Tailwind (`src/app.css`), они переключаются вместе с темой (светлая/тёмная/purple) —
   **`dark:` варианты не нужны**. Используй:
   * фон/текст/линии: `bg-page` `bg-surface` `bg-surface-2` `bg-surface-3` `bg-surface-4` `bg-elevated` · `text-fg` `text-muted` `text-soft` `text-disabled` · `border-line` `border-line-strong`;
   * акценты: `bg-accent` `text-accent` `bg-accent-soft` `text-on-accent` · `text-danger` `bg-danger-soft` · `text-success` `bg-success-soft` · `text-warning` `bg-warning-soft` · `text-info` `bg-info-soft` · `bg-neutral-soft`;
   * радиусы `rounded-s` `rounded-m` `rounded-l` (и стандартные `rounded-full`), тени `shadow-s` `shadow-m` `shadow-l`;
   * шрифт-шорткаты дизайн-системы: `t-h1…t-h5`, `t-body-s|m|l` (+ `-strong`), `t-desc-s|m|l` — вместо `text-sm font-medium` и т.п. (по умолчанию для плотных экранов CRM бери `t-body-s`/`t-body-m`);
   * отступы — стандартная шкала Tailwind (`p-4` = 16 px = `--atmr-spacing-4x`), `gap-*`, `space-*`; размеры `size-*`, `w-*`, `min-w-0`, `max-w-*`.
   * Произвольные значения — только если токена нет: `bg-[var(--atmr-…)]`; голых hex-цветов не пиши.
3. **Адаптивность утилитами:** телефон `< 768` = `max-md:`, планшет+ = `md:`, десктоп `≥ 1024` = `lg:` (`max-lg:` — «меньше десктопа»).
   Для высоты — `max-h-[…dvh]`, `h-dvh`, `min-h-0` + `overflow-y-auto`; для коротких экранов `[@media(max-height:500px)]:…`.
   Структурные различия (таблица ↔ карточки, меню ↔ бургер) решает `useBreakpoint()`, а не CSS.
4. Компоненты rt-ui принимают `class` — передавай утилиты (`<Input class="w-full" />`, `<Button class="max-md:w-full" />`);
   слой `rtui` в CSS стоит НИЖЕ `utilities`, поэтому утилита всегда побеждает стиль компонента. Никогда не импортируй CSS rt-ui из JS.
5. Имена классов — **целыми строками** (Tailwind сканирует исходники): никаких `` `bg-${tone}` ``; для вариантов — словарь
   `const TONE = { ok: 'bg-success-soft text-success', … }`. Условные классы — `class={['flex gap-2', active && 'bg-accent-soft']}` (массивы/объекты Svelte 5).
6. Markdown/длинный форматированный текст: `<div class="md">{@html renderMarkdown(text)}</div>`.
7. Общие компоненты (`$lib/ui`) уже на Tailwind — смотри их разметку как образец стиля.

## 7. Что значит «готово»

Для каждой твоей страницы/потока:
1. Все ручки твоей области **покрыты** интерфейсом (чек-лист эндпоинтов в `docs/plan-<область>.md`, каждая строка отмечена: где в UI).
   Ручки, для которых UI действительно бессмыслен (вебхуки/машинные), явно перечисли и объясни.
2. Реальный сквозной сценарий пройден на живом бэкенде (когда он поднят): создание → изменение → ошибки валидации → права.
3. `pnpm check` без ошибок в своих файлах, `pnpm build` проходит.
4. Скриншоты ключевых экранов проверены на 6 размерах: `1440×900`, `1024×768`, `768×1024`, `390×844`, `360×640`, `844×390` —
   нет горизонтальной прокрутки, ничего не обрезано, модалки помещаются, читаемо. Найденное чинишь, а не описываешь.
5. Нет `console.error` в браузере на твоих страницах (скрипт `tools/shot.mjs` печатает ошибки консоли).

## 8. Инструменты проверки

`node tools/shot.mjs --as kam --url /deals --out .shots/a --sizes all` — открывает страницу под ролью (kam|head|admin|auditor), делает
скриншоты на размерах (`all` = 6 размеров выше, либо `1440x900,390x844`), печатает: горизонтальная прокрутка (да/нет), ошибки консоли,
неудачные запросы. Дополнительно: `--wait "<css-селектор>"`, `--click "<css>"` (можно несколько раз), `--theme rtk_default_dark`,
`--full` (полная высота страницы). Для сложных сценариев пиши свои скрипты Playwright в `tools/scenarios/<агент>-*.mjs`
(импортируй `login` и `newPage` из `tools/lib.mjs`). Скриншоты смотри инструментом Read (он показывает PNG).
Пока бэкенд не поднят, `shot.mjs` увидит страницу входа/ошибки сети — ориентируйся на код и дождись бэкенда (лид напишет в `docs/STATUS.md`).

## 9. Формат отчёта (финальное сообщение)

1. Что сделано (по страницам, коротко), 2. Покрытие эндпоинтов (сколько из скольких, что осталось и почему),
3. Найденные проблемы бэкенда/общего слоя (файл:строка, суть), 4. Что не успел, 5. Как проверить (URL + роль).
Без воды, максимум ~40 строк.


## Радиусы скруглений (важно)

Только `rounded-sm` (6 px, метки/код), `rounded-md` (8 px: кнопки, поля, плашки и блоки внутри карточек), `rounded-lg` (12 px: сами карточки, диалоги, панели) и `rounded-full` (пилюли, аватары).
**Никогда `rounded-s` / `rounded-m` / `rounded-l`**: в Tailwind `-s` и `-l` — это стороны (начало/левый край), а не размеры, углы получаются разными слева и справа. Тест `src/lib/ui/class-names.test.ts` ловит это.
