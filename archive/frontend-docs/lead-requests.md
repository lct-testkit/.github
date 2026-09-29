# Запросы к лиду / общему слою

Формат: **кто — что — зачем.** Одна строка на запрос; лид отмечает выполненные.

## A — CRM

- A — договориться о справочнике пользователей для HEAD/KAM (`GET /api/admin/users` со скоупом команды и усечёнными полями `id, full_name, display_name, role, team_id`, либо `GET /api/users/directory?ids=&q=`) — без него имена авторов/ответственных/исполнителей показываются как короткие id, а HEAD не может выбрать преемника при переназначении (см. `backend-issues.md` A-1).
- A — стор заголовка/хлебных крошек в `$lib/stores` (например `pageTitle.set('D-2026-000123')`) — динамические маршруты `/deals/[id]`, `/organizations/[id]`, `/contacts/[id]` должны показывать номер/название в крошках.
- A — `uploadFile`: учитывать `upload_headers` из `upload-intent`, отдавать `{ fileId }` после `commit`, `onProgress`, отмена через `AbortSignal`, ошибки `CRM-1401/1402/1403` как `ApiError` — трёхшаговая загрузка вложений.
- A — `format.ts`: деньги из строки Decimal (`"1250000.00"` → `1 250 000 ₽`), относительное время («2 ч назад»), склонения `plural(n, ['день','дня','дней'])`; `Money`/`DateText` принимают строки — суммы и даты в API приходят строками.
- A — `errorMessage(e)`: для `CRM-1002` стандартный текст «изменено другим пользователем», для `CRM-9503` — «повторите попытку»; в `ApiError.extra` нужны `current_version`, `unmet`, `organization_id` без сужения типа — обработка конфликтов и переходов.
- A — главная `/`: встроить `src/lib/features/crm/recent/RecentList.svelte` («Недавно открытые», `GET /api/me/recent`) — ручка иначе не имеет своего экрана.
- A — `nav.ts`: подтвердить маршрут агента C для документов подписи (`/signing/...`) — deep-link из уведомлений `entity_type=signature_document`.
- A — согласовать с B сид SLA-правил в воронках (`sla_rules` через конструктор, например 3 рабочих дня на промежуточный статус) — иначе индикаторы и фильтры SLA в демо пустые.
- A — `gen:api`: `GET /api/me/recent` без `response_model` → тип `unknown`; типизирую локально, править схему не нужно.

## C — Доступы, подпись, публичные страницы

- C — корневой `+layout.svelte` не требует сессии и не зовёт `/api/me` на `/sign/**`, `/verify/**`, `/invite/**`; на них есть `ThemeProvider` и `ToastNotificationsProvider` — публичные страницы работают без входа.
- C — `api`: поддержка не-JSON ответов (`parseAs: 'blob' | 'text'`) и `FormData`-тела без `Content-Type: application/json` — экспорт NDJSON аудита и `POST /api/signatures/verify` (multipart).
- C — `ApiError`: поля `retryAfter` (из заголовка `Retry-After`, секунды) и `extra` (`approval_id`, `status`, `current_version`, `user_id`) — таймеры 429 и поток «четырёх глаз».
- C — `X-CSRF-Token` читать из cookie `crm_csrf` на каждый запрос, а не кэшировать при входе — `POST /api/me/password` переиздаёт cookie сессии и CSRF.
- C — запросы на `/public/*` без `Authorization`/CSRF и без глобального редиректа на `/login` при 401/403 — публичная страница подписи.
- C — при `session.me.password_change_required` вести на `/profile?tab=security` (моя страница) — принудительная смена пароля.
- C — `vite.config.ts`: proxy путей-бакетов `/signatures`, `/files`, `/reports`, `/imports` → `http://localhost:8333` с `changeOrigin: true`, без переписывания пути — presigned-ссылки на PDF становятся same-origin (CORS/CSP), pdf.js может их загрузить (см. `backend-issues.md` C-1).
- C — `config.mode` (`demo`/`prod`) доступен из общего слоя — предупреждения о смене пароля демо-учётки и подсказка `debug_code`.
- C — `nav.ts`: подтвердить пути `/profile`, `/signing`, `/signing/requests/[id]`, `/signing/documents/[id]`, `/admin/users`, `/admin/users/[id]`, `/admin/users/[id]/offboard`, `/admin/teams`, `/admin/approvals`, `/admin/audit`, `/admin/erasure`, `/admin/erasure/[id]`, `/admin/settings`, `/admin/edm`; при наличии слота бейджа в меню — дам счётчик ожидающих согласований.
- C — `src/lib/ui`: `StatusChip` с картой цветов, `DateText` с относительным временем, `CopyButton` (если нет — сделаю у себя в `features/`).
- C — (выполнено у себя, при желании — поднять в общий слой) универсальные `features/identity/ReasonModal.svelte` («действие + причина», ошибка сервера внутри диалога) и `features/signing/FileDrop.svelte` (зона выбора файла) пригодны всем разделам — вынести в `$lib/ui`.
- C — `ResponsiveTable`: строки таблицы на десктопе очень плотные (≈25–31 px) и длинный текст ячейки налезает на соседнюю колонку; сейчас обхожу обёрткой ячейки `block truncate py-2` — лучше задать минимальную высоту строки и `overflow:hidden` на ячейке в самом компоненте.
- C — `AppShell`/`nav.ts`: счётчик `badge: 'signatures'` у пункта «Подписание» объявлен, но не отрисован; данные — `GET /api/me/signature-requests` (статусы `sent`/`viewed`).

## B — Настройка и данные

- B — `api`/`unwrap`: `GET /health/ready` штатно отвечает 503 с тем же JSON — нужен способ получить тело при не-2xx (например `ApiError.body`/`unwrapAny`) — плашка «Состояние системы» на странице интеграций.
- B — `uploadFile`: вызов без `entity_type/entity_id` (файл импорта и выгрузка ЕГРЮЛ привязываются позже самим заданием), `.xlsx/.xls/.csv/.xml/.zip` в allowlist `upload-intent`, возврат `fileId`, `onProgress` — мастер импорта и загрузка реестра (общий запрос с A).
- B — общий `UserPicker` на базе справочника пользователей из запроса A — поле «эскалация на пользователя» в SLA-правилах воронки; без него оставлю только роль.
- B — `confirm()` с произвольным содержимым (список предупреждений валидации, сводка переноса сделок) — подтверждение публикации воронки и архивации статуса; иначе сделаю свой Modal в `features/config`.
- B — `nav.ts`: подтвердить пути `/workflows`, `/workflows/[id]`, `/catalog/{products,directions,loss-reasons,holidays,custom-fields,regions}`, `/imports`, `/imports/new`, `/imports/[id]`, `/reports`, `/reports/history`, `/reports/dashboards`, `/reports/dashboards/[id]`, `/admin/registry`, `/admin/integrations`, `/admin/notification-templates`; AUDITOR не видит ни одного из них, `/imports` — HEAD/ADMIN, `/admin/*` — ADMIN.
- B — разрешить импорт `@xyflow/svelte/dist/style.css` из компонента холста (глобальный CSS из фичи) либо подключить его в `layout.css` — редактор воронки.
- B — `ResponsiveTable`/`TableCards`: сниппеты ячеек и слот действий строки — единый паттерн шести справочников без дублирования разметки.
- B — бэкенду через лида, приоритетно: `format: "json"` или `GET /reports/{id}/data` (дашборды иначе только PNG/ссылки), `GET /workflows/{id}/mapping-jobs/{job_id}` (прогресс архивации), `POST /admin/integrations/outbox-events/{id}/retry` (dead-letter) — см. `backend-issues.md` B-19/B-1/B-23.
- B — согласовано с A: SLA-правила для сидовых воронок добавлю через конструктор (`PUT /graph` + `publish`) сразу как заработает редактор — чтобы индикаторы SLA в сделках не были пустыми на демо.
- B — `ResponsiveTable`: корень `TableGrid` (`.atmr-tablegrid`) имеет ширину `fit-content`, поэтому `fr`-колонки не растягиваются на всю ширину страницы (таблица справочников занимает ~85 % ширины). Достаточно передать в `<TableGrid style={{ width: '100%' }} …>` (проверено в браузере: колонки заполняют контейнер).
- B — бэкенд/окружение: сиды `reporting`, `notification`, `integration` не входят в `entrypoint.sh seed` (там только воронки) — в свежей БД нет шаблонов отчётов, шаблонов уведомлений и источников интеграций. Я выполнил их вручную: `docker exec rtk-crm-api-1 python -m app.modules.{reporting,notification,integration}.seed`; в `deploy/entrypoint.sh` их стоит добавить.
- B — бэкенд: первый запрос нового пользователя не к `/api/me` (например, к `/api/workflows` без принятого согласия → 403) откатывает JIT-создание пользователя, а Redis-кэш принципала (`cache:kcid:*`, `cache:perm:*`) остаётся → дальше `/api/me` даёт 500 `NoResultFound`. Лечится удалением двух ключей в Redis. Стоит откладывать `set_principal_cache` до commit.
- B — соглашение по пользовательским полям (для A, формы сделки/организации): `custom_field_defs.options = { "choices": ["…"] }` для `select`/`multiselect`; `validation` — `{ "min", "max" }` для `number`, `{ "max_length", "pattern" }` для `string` (бэкенд их не интерпретирует, это договорённость UI); значения — в `custom_fields[code]`.
- B — `Select` / `Multiselect` (в т.ч. `UserPicker`) внутри `AppDrawer`/`AppModal`: меню выпадающего списка имеет `z-index: var(--atmr-z-index-dropdown)` = 1000, а панель/окно — 1500, поэтому список оказывается ПОД панелью (клики по пунктам перехватывает форма). Лечится `dropdownMenuClassName="z-1550"` на компоненте; в `UserPicker` (общий слой) это нужно добавить. Мои обёртки `shared/fields/Pick|MultiPick` уже так сделаны.

## A — CRM (Фаза 2)

- A — `src/app.css`: добавить `body.rt-base { --atmr-z-index-dropdown: 1650; }` (сейчас 1000 < Drawer/Modal 1500): выпадающие списки `Select`/`UserPicker`/`Multiselect` внутри `AppDrawer`/`AppModal` спрятаны за окном и не кликаются — у всех агентов. Пока у меня стоит обход `:global(body)` в `features/crm/shared/fields/Pick.svelte` — после правки в app.css удалить.
- A — `ApiError`/`errorMessage`: для `CRM-9503` текст «Сервис временно недоступен» не годится для блокировки перехода сделки («сделку обрабатывает другой переход — повторите»); сейчас диалог перехода подменяет текст сам.
- A — `uploadFile` (`src/lib/api/upload.ts`, `put()`): ссылка из `upload-intent` подписана вместе с заголовком `content-type` (`X-Amz-SignedHeaders=content-type;host`, `upload_headers` пуст). XHR должен явно слать `Content-Type` = тот же `mime_type`, что ушёл в intent (`file.type || 'application/octet-stream'`); иначе для файлов, у которых браузер не определил тип (`.rar`, `.gz` и др.), хранилище отвечает 400 `AccessDenied: headers … not signed`.
- B — `$lib/ui/index.ts`: не экспортирует `UserName` (импортирую напрямую из `$lib/ui/UserName.svelte`) и `UserPicker`; `Page fill` в редакторе воронки использую с `class="max-lg:h-auto"`. Итог работ B: `pnpm check` 0 ошибок, `pnpm test` 229 тестов, `pnpm build` проходит; сценарии `tools/scenarios/b-*.mjs` (каталоги, воронки, импорт, отчёты, админ-страницы) зелёные на живом бэкенде.

## D — перевод CRM-экранов на rt-ui

- D — `src/lib/ui/Notice.svelte`: добавить по умолчанию `shrink-0` в класс корня. У `InlineNotification` `overflow:hidden`, и в колонке `flex` внутри `AppDrawer`/`AppModal` (тело с `overflow-y-auto` и высотой по окну) плашка сжимается до полоски в 4 px, как только форма выше окна. Пока у каждого `<Notice>` в моих экранах стоит `class="shrink-0"`.
- D — `Notice`: пробросить в `actions[]` необязательные `variant` (`primary`/`secondary`/`tertiary` у FunctionButton), `disabled` и произвольные атрибуты (`data-testid`); сейчас кнопки всегда `secondary` — чёрный текст без обводки, главное действие («Принять все» в баннере расхождений ЕГРЮЛ) не выделено, а на время запроса кнопку нельзя заблокировать (защищаюсь в обработчике). Ограничение rt-ui: показывается не больше двух кнопок.
- D — rt-ui (для сведения): `Tooltip` кладёт триггер в `aria-hidden` (нельзя оборачивать кнопки и переключатели — пропадут для скринридера), кнопки внутри `File` (`onAction`, `onFunctionButton`) без `aria-label`, `FileUpload` заменяет список при каждом выборе и держит его сам — поэтому вложения (`features/crm/files/Attachments.svelte`) остались на своей зоне перетаскивания + `Btn`/`IconBtn`/`Progress`.

## E — перевод на rt-ui (config / identity / signing / публичные страницы)

- E — `Notice`: по умолчанию добавить `shrink-0` (то же, что записал D): в `AppModal`/`AppDrawer`/скролле-колонке плашка сжимается в полоску; у всех `<Notice>` в моих экранах стоит `class="shrink-0"`.
- E — rt-ui: у `Tabs` нет вертикального варианта — меню разделов справки (`routes/(app)/help`) остаётся списком `Btn` (ghost/secondary) в колонке; нужен вертикальный `TabsGroup` или `NavList`.
- E — rt-ui `FileUpload`: `subtitle` обрезается многоточием (не переносится) и `multiple` включён всегда, хотя зоне нужен один файл (`features/signing/FileDrop.svelte` берёт первый); нужен проп `multiple={false}` и перенос подписи.
- E — rt-ui: нет ввода одноразового кода (OTP): `features/signing/OtpInput.svelte` — шесть ячеек поверх одного настоящего `<input>` — остаётся своим.
- E — rt-ui: `Accordion` собран на `Box` без `<button>` в заголовке (нет клавиатуры/aria-expanded) — свёртки (`config/shared/Collapse`, «Ещё» в фильтрах аудита, «Дополнительно» в создании пользователя, «Условия подписания») оставлены на нативном `<details>`.
- E — rt-ui `File`/`FileUpload`: размер файла подписан по-английски («45 Bytes», «KB»), а в интерфейсе всё по-русски; нужен проп локализации единиц («Б», «КБ», «МБ»).
