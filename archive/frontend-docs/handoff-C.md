# Handoff агента C («Доступы, подпись, публичные страницы»)

Агент C завершён по решению пользователя. Этот файл — всё, что нужно, чтобы продолжить без контекста. Область файлов прежняя:
`src/lib/features/{identity,signing}/**`, маршруты `(app)/{profile,signing}/**`, `(app)/admin/{users,teams,approvals,audit,erasure,settings,edm}/**`, публичные `sign|verify|invite`.
Порядок чтения: `AGENT-BRIEF.md` → `LEAD-DECISIONS.md` (внизу — что лид уже починил в бэкенде) → этот файл. `plan-identity-signing.md` остаётся описанием экранов, но его чекбоксы **не обновлялись** — актуальный статус ниже.

## 1. Что готово (проверено вживую на бэкенде, `pnpm check` чисто в моих файлах, `pnpm build` проходит, 83 unit-теста зелёные)

| Маршрут | Файлы | Что умеет / чем проверено |
|---|---|---|
| `/sign/[token]` (публичный) | `routes/sign/[token]/+page.svelte`, `signing/{SignFlow,SignSession(sign-session.svelte.ts),PdfViewer,PdfPage,OtpPanel,OtpInput,SignerList,SignOutcome,PublicShell}` | PDF без скачивания (pdf.js, canvas, зум), подписанты, условия, «Подписать» → код (демо-подсказка `debug_code` + «Подставить»), таймер повторной отправки, счётчик попыток, отклонение с причиной, итоговые экраны. Сквозной сценарий пройден 360×640, 390×844, 844×390 (OTP), 1440×900 |
| `/verify/[[id]]` (публичный) | `routes/verify/[[id]]/+page.svelte`, `signing/{VerifyView,VerifyForm,FileDrop}` | статус подписи, подписант/дата/способ/хэш, сверка файла sha256 в браузере, форма идентификатора, поиск по файлу на сервере (нужен вход) — 1440 и 360 |
| `DealSignatures` (`{dealId}`) | `signing/{DealSignatures,DocumentPanel,SendWizard,send-draft.svelte.ts,SignLinkModal,known.ts}` | список документов сделки (`GET /signature-documents?entity_type=deal`), мастер из 2 шагов (шаблон/свой PDF → подписанты, порядок, срок), отправка, ссылка внешнему подписанту (один раз), аннулирование, протокол, «Отправить повторно» при `blocked_no_agreement`. Проверено через временный маршрут (удалён) на 1440 и 390. **Карточка сделки агента A ещё не встроила блок** |
| `/signing` | `routes/(app)/signing/+page.svelte`, `signing/{InboxTab,DocumentsTab,TemplatesTab}` | вкладки «Мне на подпись» / «Документы» / «Шаблоны» (`?tab=`, `?scope=all`) |
| `/signing/[id]` | `routes/(app)/signing/[id]/+page.svelte` | карточка документа (сюда ведут уведомления `signature_document`) |
| `/signing/requests/[id]` | `routes/(app)/signing/requests/[id]/+page.svelte` | внутренняя подпись сотрудником (тот же `SignFlow`, адаптер `internalAdapter`); пройден под HEAD |
| `/profile` | `routes/(app)/profile/+page.svelte`, `identity/{ProfilePanel,SessionsPanel,PasswordPanel,ThemePicker}` | `?tab=profile|sessions|security`, тема/палитра, сессии (завершение, текущая → выход), смена пароля с проверкой политики и таймером 429 |
| `/admin/users` | `routes/(app)/admin/users/+page.svelte`, `identity/users/{UsersList,UsersFilters,UserCreateModal,InviteLinkModal}` | список (фильтры `q,role,status,team_id` в URL, курсор), создание (ADMIN → «четыре глаза» CRM-1902, дубликат CRM-1301, предзаполнение `?create=1&full_name&email&role&approval_id`) |
| `/admin/users/[id]` | `routes/(app)/admin/users/[id]/+page.svelte`, `identity/users/{UserPage,UserEditDrawer}` | карточка, правка (If-Match, конфликт CRM-1002 без потери ввода), блок/разблок, сброс пароля, повторное приглашение, запрос на удаление ПДн |
| `/admin/users/[id]/offboard` | `routes/(app)/admin/users/[id]/offboard/+page.svelte`, `identity/offboard/OffboardWizard.svelte` | мастер 3 шага (дела → преемник → подтверждение) + итог; preview проверен на данных |
| `/admin/audit` | `routes/(app)/admin/audit/+page.svelte`, `identity/audit/{AuditList,AuditFilters,AuditDetails,ChainCheckModal,audit-query.ts}` | фильтры в URL, подробности записи в Drawer, проверка цепочки (проверено: «цела, 485 записей»), экспорт NDJSON с confirm (кнопка только при `audit:export`; загрузка blob не прогонялась) |

## 2. Покрытие ручек: 35 из 56 (плюс 5 машинных/чужих — не считаются)

Готово: `me`-сессии/пароль (3), `/admin/users` (10: list, get, create, patch, block, unblock, reset-password, invite, offboard preview/confirm, erasure-request), аудит (3), `GET /admin/teams` (1, стор), подпись (12: templates, create, get, send, void, protocol, my requests, view, challenge, sign, reject, verify по файлу), публичные (5).

**Не сделано (21 ручка) — сюда переходит новый агент:**

| Экран | Ручки | Подсказки, что уже есть |
|---|---|---|
| `/invite/[token]` | `GET /api/auth/invite/{token}` | `PublicShell width="sm"`; ответ `{valid,email_masked,full_name,expires_at,login_url}`; 404 `CRM-9004` → «ссылка недействительна»; 429 `CRM-8429` (+`ApiError.retryAfter`); кнопка «Перейти ко входу»: demo → `/login`, prod → `login_url`. Ссылка приглашения бэкенда уже `{base_url}/invite/{token}` |
| `/admin/teams` | `POST /admin/teams`, `PATCH /admin/teams/{id}` | стор `identity/teams.svelte.ts` (`teams.load/name/options/put`); тело `{name,parent_id,head_id,region_id}` (у PATCH все поля необязательны, If-Match нет); руководитель — `UserPicker roles=['HEAD']`; регионы — `GET /api/regions`; дерево — `Tree` из rt-ui либо вложенный список; 422 на цикл показывать у поля «Родитель» |
| `/admin/approvals` | `GET /admin/approvals?status=`, `POST …/{id}/approve`, `POST …/{id}/reject` (тело `{reason?}`) | вся логика в `identity/approvals.ts`: `approvalActions(a, meId)`, `describeApproval`, `approvalExecuteHref` (для «Выполнить»), метки — `labels.ts`. Имена инициаторов — `people.name`. Кнопка «Выполнить» ведёт на `/admin/users?create=1&…&approval_id=` (**уже работает**) либо на `/admin/erasure?approval_id=&subject_type=&subject_id=&mode=` (**ждёт реализации** на странице erasure) |
| `/admin/erasure` (+ `/[id]`) | `GET /admin/erasure-requests[/{id}]`, `POST …/recheck|reject|restore`, `GET …/{id}/act`, `POST /admin/{contacts,organizations}/{id}/erasure-request` | `identity/erasure.ts` (`graceCountdown`, `erasureActions`, `ERASURE_STEPS`, `erasureStepIndex`), `labels.ts` (`blockerHint`, `erasureStatusMeta`), готовая форма создания `identity/erasure/ErasureRequestModal.svelte` (умеет субъекты user/contact/organization, «четыре глаза», `presetMode`, `approvalId`; после создания зовёт `onCreated(id)` → перейти на `/admin/erasure/{id}`). Не хватает: список, карточка, шаги процесса (`WizardStepsHorizontal`), выбор субъекта для «Нового запроса» (контакты — `GET /api/contacts?q=`, ИП — `GET /api/organizations?q=&org_type=individual_entrepreneur`), разбор query `approval_id/subject_*`. После создания перечитывать `GET …/{id}` (backend C-15). Акт — открыть presigned `download_url` |
| `/admin/settings` | `GET/PATCH /admin/feature-flags[/{code}]`, `GET/PUT /admin/system-settings[/{key}]` | `identity/settings.ts`: `SECRET_PLACEHOLDER` (**никогда не отправлять `********`**), `parseSettingValue`, `stringifySettingValue`, `settingKind`, `policyPayload(version,text)` (sha256 → `pdn_policy`), `isPolicyVersion`. Две вкладки `?tab=flags|system`, Switch флага — оптимистично с откатом |
| `/admin/edm` | `GET/POST /admin/edm-agreements`, `POST …/{id}/revoke` | метки — `signing/status.ts` (`edmPartyLabel`, `edmMethodLabel`); список без пагинации, имена сторон догружать (`GET /api/contacts/{id}`, `/api/organizations/{id}`, `people.name`); «истекло» считать по `valid_to`; файл — `uploadFile`; **`DocumentPanel` ведёт сюда ссылкой `/admin/edm?party_type=contact&party_id=<uuid>&new=1` — форма создания должна открываться предзаполненной**; AUDITOR только читает (`edm:read`), пишет `edm:admin` |

## 3. Что можно переиспользовать

* `identity/ReasonModal.svelte` — «действие + причина» (обязательная/необязательная, `note`, `extra`-слот), ошибка сервера остаётся в диалоге. Готов для отклонения заявки, отзыва ЭДО, отказа по erasure.
* `identity/ApprovalPendingModal.svelte` — экран «нужно второе подтверждение» (CRM-1902) с переходом к согласованиям.
* `identity/teams.svelte.ts`, `identity/labels.ts` (роли, статусы, действия аудита `AUDIT_ACTION_LABEL`, типы сущностей), `identity/audit/audit-query.ts` (`entityHref`, даты дня → ISO).
* `signing/FileDrop.svelte` (выбор/перетаскивание файла), `signing/PdfViewer.svelte` (canvas-просмотр, зум), `signing/SignLinkModal.svelte`, `signing/SignerList.svelte`, `signing/hash.ts`, `signing/urls.ts` (`normalizeSignUrl`, `shortHash`).
* Шаблон списка с фильтрами в URL: `users/UsersList.svelte` + `UsersFilters.svelte` (`readQuery`/`setQuery`, `createPager`, `$effect` по ключу фильтров, на телефоне фильтры под кнопкой-иконкой). Шаблон Drawer с подробностями — `audit/AuditDetails.svelte`. Шаблон мастера — `offboard/OffboardWizard.svelte` (на телефоне `textPlacement="bottom"`).
* Приёмы: ячейка таблицы `block truncate py-2` (иначе строки ≈25 px и текст налезает на соседнюю колонку); сброс формы при открытии — внутри `untrack`, иначе перезагрузка справочника (`teams.load()`) стирает ввод; **имя сниппета не должно совпадать с именем `$derived` в том же компоненте** (падение `meta.tone` в `UserPage` именно так и вылезло); кнопка отправки в футере модалки — `<Btn type="submit" form="<id формы>">`.

## 4. Известные проблемы и ограничения

* Блок `DealSignatures` не встроен в карточку сделки (это делает A). Корневой элемент — `data-testid="deal-signatures"`; сценарий `tools/scenarios/c-deal-signing.mjs --path /deals` ждёт его.
* Не проверено вживую: итоговые экраны `locked`/`expired`/`mismatch` (лимит публичных ручек 10 запросов/мин на IP — три неверных кода подряд с прогоном страницы упираются в 429; код обрабатывает оба случая), «Свой PDF» в мастере отправки в браузере (загрузка и создание с `file_id` проверены API-скриптом; `commit` в демо сразу `ready`, для `pending` есть повторы до 12 с), загрузка blob при экспорте NDJSON, поиск по файлу в `/verify` под входом, тёмная тема, планшетные размеры 1024×768/768×1024, ландшафт 844×390 для профиля/пользователей/аудита (проверены 1440 и 360×640; для подписи — ещё 390×844 и 844×390).
* «Документы» на `/signing` — только известные (созданные в этом браузере, где я подписант, и по журналу для `audit:read`); общего списка у бэкенда нет.
* Для новой логики нет unit-тестов: `sign-session.svelte.ts`, `send-draft.svelte.ts`, `audit-query.ts`.
* В demo-режиме смена пароля/сброс пароля меняют реальную демо-учётку (UI предупреждает).
* Внешний подписант не первый при «По очереди» не получает ссылку (backend C-7) — мастер предупреждает и предлагает «Одновременно».
* Новые замечания к бэкенду добавлены в `backend-issues.md` (C-25 кэш принципала после отката JIT — из-за него у демо-админа падал `/api/me`; C-26 лимит публичных ручек). Запросы к лиду — в `lead-requests.md` (раздел C).

## 5. Как проверять

* Демо-учётки и сценарии: `node tools/scenarios/c-sign.mjs --size 360x640 --flow sign|reject|wrong|internal` (создаёт свежий документ и проходит `/sign/<токен>` или внутреннюю подпись), `c-deal-signing.mjs` (мастер отправки), `c-audit.mjs` (подробности + проверка цепочки). Для остальных экранов — `node tools/shot.mjs --as admin --url /admin/users --sizes all --out .shots/c`.
* Публичные ручки подписи ограничены 10 запросами/мин на IP: между прогонами выдерживать паузу.
* Первый запрос нового пользователя стенда — всегда `GET /api/me` (см. C-25).
* API-хелпер для быстрой проверки ручек: скрипты из `scratchpad` не сохранены; получить токен — `accessToken(who)` из `tools/lib.mjs`.
