# Решения лида и обновления (читай после AGENT-BRIEF.md)

## Общее (важно, перекрывает бриф)

* **Dev-сервер: http://localhost:5273** (НЕ 5173: там чужой проект пользователя — не трогай). Запущен, HMR. Свои серверы не поднимай.
* **Бэкенд живой** (docker): http://localhost:8080, схема сгенерирована: `src/lib/api/schema.d.ts`, `docs/openapi.json`, `docs/api-endpoints.md`
  (172 операции). `pnpm gen:api` не запускай. `pnpm install/add` не запускай (зависимости — просьбой в `docs/dep-requests.md`).
* **Стили — Tailwind v4** (см. AGENT-BRIEF §6). Заказчик ненавидит «чистый CSS»: утилиты в разметке, `<style>` только в крайнем случае.
* Проверка: `node tools/shot.mjs --as kam --url /deals --out .shots/<агент> --sizes all` (PowerShell/Git Bash ок), сценарии — `tools/lib.mjs`
  (`newBrowser`, `newPage(browser,{who:'kam'|'head'|'admin'|'auditor'|null,width,height})`, `accessToken('admin')`). Системный Chrome.
  Согласие ПДн у демо-учёток принимается автоматически. **Экономь токены:** во время разработки — 1–2 размера (desktop + phone), полный
  `--sizes all` — один раз, когда экран готов; не читай один и тот же скриншот/файл повторно.
* Демо-учётки: `kam.ivanov / Kam123456789!`, `head.petrov / Head12345678!`, `admin.crm / Admin12345678!`, `auditor.smirnov / Audit12345678!`.
* Общий слой: `$lib/api` (api, unwrap, idem, ifMatch, ApiError (+retryAfter), errorMessage, createPager, uploadFile, `people` — имена коллег),
  `$lib/auth/session.svelte`, `$lib/config` (`getConfig().mode`), `$lib/ui` (Btn, IconBtn, Page, PageHeader, AppModal, AppDrawer, ResponsiveTable, EmptyState,
  ErrorState, Skeleton, StatusChip, Money, DateText, Avatar, SearchField, CopyButton, UserName, UserPicker, toast, confirm), `$lib/utils/*`
  (format.ts, query-state.svelte.ts, markdown.ts, debounce.ts).
* Справочник сотрудников (добавлен в бэкенд): `GET /api/users/directory?q=&ids=<uuid,uuid>&roles=KAM,HEAD&limit=50` →
  `{ items: [{ id, full_name, display_name, role, team_id, status }] }` (любой авторизованный; по `ids` — любые статусы).
  **Не пиши свой сервис имён**: `people.name(id)` / `<UserName id/>` / `<UserPicker/>` уже есть в общем слое.
* Локальные пользователи появляются при первом входе (JIT): сид обязан залогинить всех четверых (`GET /api/me` с Bearer) перед созданием данных.
* Presigned-ссылки S3 (`http://localhost:8333`) — SeaweedFS отдаёт CORS `*`, грузить/качать напрямую можно (pdf.js тоже). Прокси не нужен.
* Не-JSON ответы: `api.GET(path, { parseAs: 'blob' | 'text' | 'arrayBuffer' })`; FormData: `api.POST(path, { body: fd as never, bodySerializer: (b) => b as never })`.
* `/health/ready` при 503 отдаёт JSON — читай обычным `fetch`, не через `api`.
* Публичные страницы `/sign/**`, `/verify/**`, `/invite/**` — вне группы `(app)`: сессию не грузят; Bearer добавляется только на `/api/*`.
* `password_change_required` → баннер в шелле ведёт на `/profile?tab=security`.
* Хлебных крошек нет: у detail-страниц `PageHeader` с `back="/список"` и заголовком-номером.

## Агент A (CRM)
* Заглушки для замены (те же путь и props): `features/crm/notifications/NotificationBell.svelte`, `features/crm/files/Attachments.svelte`,
  `features/crm/lookup/OrgLookup.svelte`. Плюс `features/crm/recent/RecentList.svelte` (без пропсов; лид встроит на главную).
* Маршруты C: документы подписи `/signing`, `/signing/[id]` (deep-link из уведомлений `entity_type=signature_document` → `/signing/<id>`).
* `tools/seed-demo.mjs` (первым делом, на живом бэкенде): залогинить 4 демо-учётки; админом создать команду «Отдел вузов»
  (`POST /api/admin/teams`, head_id = Петров) и назначить Иванова (`PATCH /api/admin/users/{id}` + If-Match: team_id, manager_id = Петров;
  учти «четыре глаза» CRM-1902); 10–15 организаций-вузов (с ИНН/регионами), контакты, 15–25 сделок B2B/B2C в разных статусах у Иванова и
  Петрова, комментарии, задачи, несколько won/lost с причинами; идемпотентно.
* SLA-правила в сид-воронках: через API (`PUT /api/workflows/{id}/graph` + publish) админом — короткие сроки на промежуточных статусах.

## Агент B (настройка и данные)
* Пути подтверждены: `/workflows`, `/workflows/[id]`, `/catalog/{products,directions,loss-reasons,holidays,custom-fields,regions}`, `/imports`, `/imports/new`,
  `/imports/[id]`, `/reports`, `/reports/history`, `/reports/dashboards[/id]`, `/admin/registry`, `/admin/integrations`, `/admin/notification-templates`.
  В меню верхнего уровня: Воронки (workflow:write), Справочники `/catalog` (catalog:write), Импорт (import:run → HEAD/ADMIN), Отчёты (report:read), Интеграции,
  Реестр ЕГРЮЛ, Шаблоны уведомлений. Внутренняя навигация по `/catalog/*` и `/reports/*` (табы/сегменты) — на тебе.
* `uploadFile(file, { purpose: 'import', onProgress, signal })` — без привязки к сущности; allowlist типов на бэкенде: если отвергает нужный тип — запиши в `backend-issues.md`.
* Xyflow: `import '@xyflow/svelte/dist/style.css'` из компонента холста — можно (единственный разрешённый глобальный CSS из фичи).
* `confirm()` — только title/message/labels/danger; для богатого содержимого — `AppModal`.
* Бэкенд-просьбы (JSON-данные отчётов, прогресс mapping-job, retry outbox): не жди, дашборды — на PNG/ссылках/счётчиках.
* SLA-правила сидовых воронок добавляет A скриптом; ты проверь, что редактор их видит.

## Агент C (доступы, подпись, публичные страницы)
* Заглушка для замены: `features/signing/DealSignatures.svelte` (`{ dealId: string }`).
* Профиль: вкладки `tab=profile|sessions|security` через query. Смена темы (`$lib/stores/theme.svelte`: `theme.set`, `theme.toggleMode`, палитры default/purple).
* Пути: `/profile`, `/signing`, `/signing/[id]`, `/admin/users[/id]`, `/admin/teams`, `/admin/approvals`, `/admin/audit`, `/admin/erasure`, `/admin/settings`, `/admin/edm`,
  публичные `/sign/[token]`, `/verify/[[id]]`, `/invite/[token]`. Вложенные маршруты — свободно в своих каталогах; подсветка меню по префиксу.
* В demo-профиле код OTP приходит в ответе (`debug_code`): показывай как подсказку «Демо-режим» с кнопкой «Подставить»; в консоль не пиши.
* `CopyButton` есть в `$lib/ui`.

## Изменения бэкенда, сделанные лидом (не репортить повторно; схема в `schema.d.ts` обновлена, 173 операции)

* `GET /api/users/directory` — справочник сотрудников (см. выше).
* `PUT /api/workflows/{id}/graph` — больше не падает 500 на воронках со сделками в истории: переходы обновляются по паре статусов (id сохраняются),
  удаление использованного статуса/перехода отдаёт понятный 422 вместо 500 (backend-issues B/A №27 закрыт).
* `GET /api/signature-documents?entity_type=deal&entity_id=<uuid>&limit=` — список документов сущности с подписантами (история: аннулированные, просроченные) (C №5 закрыт).
* `GET /api/me/signature-requests` — в элементах теперь `document_title`, `deadline_at`, `entity_type`, `entity_id` (C №8 закрыт).
* `sign_url` (ответ `/send`) теперь `{base_url}/sign/{token}` — страница SPA; QR и ссылка проверки в штампе/протоколе — `{base_url}/verify/{id}` — страница SPA (C №2, №3 закрыты).
* Caddy: SPA отдаётся с того же origin (`/api`, `/public`, `/health` → API; `/auth` → Keycloak; остальное → `web`); CSP допускает presigned S3 (connect/img/frame).
* Аудит: имена акторов резолвятся через `people` (`/api/users/directory` доступен и AUDITOR) — `actor_display` в бэкенде не нужен.
