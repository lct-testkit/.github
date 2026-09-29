# Аудит Markdown-документации фронтенда (frontend main, 2026-09-27)

Источник истины: `D:\testkit-lct\_wt-frontend-main` (main, коммит 41dde50), бэкенд `_wt-backend-main` (main, 3b0d5cd, PR #15). Рабочее дерево `frontend/` не трогал. Ничего не менял, кроме этого отчёта.

Как проверял: `docs/openapi.json` и `docs/api-endpoints.md` сверены скриптом (203 операции, множества совпадают), `openapi.json` фронтенда и бэкенда совпадают по смыслу (различается только порядок ключей); `node tools/coverage.mjs` запущен на main-копии (175 из 203); файлы и тесты посчитаны через find/grep (тесты: 285 строк `it(` в 36 файлах, сам `vitest` не запускал, чтобы не трогать зависимости); чтение кода и `grep -rn` по репозиториям `_wt-frontend-main`, `_wt-backend-main`, `deploy`, `.github`, корневой `README.md`, `architecture`, `DEPLOY-*.md`.

## 1. Сводная таблица

Вердикты: KEEP 2, TRIM 6, ARCHIVE 8 (всего 16 файлов, без `src/lib/content/help/*.md`).

| Файл | Вердикт | Причина | Кто ссылается |
|---|---|---|---|
| `README.md` | TRIM | Живая витрина клиента, в целом точная. Устарели числа и несколько фраз (список в разделе 2). Заголовки нельзя переименовывать: на якоря ссылаются другие репозитории. | Внешние: `.github/README.md` (якоря `#как-устроено`, `#блоки-интерфейса`, `#проверка-качества`, `#быстрый-старт`, `#один-origin-с-api-dev-клиент-за-caddy`), `.github/profile/README.md` (`#интеграция-с-битрикс24`, `#редактор-воронки-графовый-конструктор`, `#экраны-и-права`, `#как-устроено`, `#проверка-качества`), корневой `README.md`:13,241,334,378,505,639,673, `_wt-backend-main/README.md`:227 (`#интеграция-с-битрикс24`),517, справка `help/13-about.md`:11. Внутри: `STATUS.md` ссылается на README в разделе «Соответствие требованиям кейса» (корневой). |
| `SECURITY.md` | KEEP | Короткая политика, соответствует CI (Trivy, `pnpm audit`, Dependabot, cosign/SBOM в общем конвейере `lct-testkit/deploy`). Нашёл лишь то, что GitHub подхватывает файл сам. | Явных ссылок нет (`deploy/docs/REPO-SETTINGS.md`:59 упоминает README, а не SECURITY). |
| `.github/PULL_REQUEST_TEMPLATE.md` | TRIM (косметика) | Шаблон скопирован с бэкенда: п. «RUNBOOK» (строка 9) и «миграция и `alembic check`» (строка 14) к фронтенду не относятся. | GitHub подхватывает сам, ссылок нет. |
| `docs/AGENT-BRIEF.md` | ARCHIVE | Одноразовый бриф параллельным агентам A/B/C и «лиду» (20-21.09): владение каталогами, «не делай git commit», «не запускай pnpm install», путь `docs/dep-requests.md` (файла нет), сломанные пути `D:\testkit-lct\rt-ui` (строки 125-126 испорчены экранированием). Правила стилей (Tailwind, радиусы) уже живут в README «Соглашения» (строки 324-331). | `docs/LEAD-DECISIONS.md`:1,8; `docs/handoff-C.md`:5; `README.md`:503. |
| `docs/DS-MIGRATION.md` | ARCHIVE | Инструкция агентам по переводу самописных элементов на rt-ui (миграция выполнена: в `src` остался один `<style>` (`ui/fields/Pick.svelte`), остальное на компонентах). Строка 11 указывает на несуществующий `FilterChips` (удалён, REBUILD-PLAN:118). Строки 22-24 — правила для агентов («коммитов не делать»). Суть правила уже в README:331. Альтернатива: свернуть в один абзац README. | `README.md`:331,499; `docs/STATUS.md`:36; корневой `README.md`:677; `.github/README.md`:685. |
| `docs/LEAD-DECISIONS.md` | ARCHIVE | Решения лида для агентов на 21.09: «172/173 операции», роли A/B/C, «`pnpm gen:api` не запускай», список правок бэкенда лидом (все давно в main). Полезное перенесено: справочник `people` (README:337), не-JSON ответы и `/health/ready` (в коде `features/config/integrations/health.ts`). | `README.md`:501; `docs/handoff-C.md`:5. |
| `docs/REBUILD-PLAN.md` | TRIM | Не архив: §3 (блоки и критерии приёмки), §6 (ловушки rt-ui), §9 (правила раскладки) живые, на них ссылаются README, код и корневые README. Но верх документа — «план на завтра» с бэклогом и открытыми вопросами. | `README.md`:180,486,497; `docs/STATUS.md`:3; `docs/USERFLOWS.md`:3,8; `tools/tops.mjs`:1 (`§9.2`); корневой `README.md`:675; `.github/README.md`:683; `.github/profile/README.md`:171. |
| `docs/STATUS.md` | TRIM | Единственный документ с таблицей покрытия ручек (нужна) и командами проверки. Но шапка датирована 22.09, есть отчёт «Ночь 22.09», устаревшие числа и упоминания удалённых компонентов. | `README.md`:445 (якорь `#покрытие-ручек-175-из-203-86`),483,496; `docs/AGENT-BRIEF.md`:175; корневой `README.md`:658,674; `.github/README.md`:666,682; `.github/profile/README.md`:157,171. |
| `docs/USERFLOWS.md` | TRIM | Полезная карта потоков и матрица «экран × блок». Часть потоков описывает то, чего нет (запросить доступ, восстановить организацию, задать пароль по приглашению), и раздел открытых вопросов уже закрыт. | `README.md`:498; `src/routes/(app)/+layout.svelte`:29 (комментарий «USERFLOWS §0.6»); корневой `README.md`:317,676; `.github/README.md`:325,684; `.github/profile/README.md`:41,171. |
| `docs/api-endpoints.md` | KEEP | Генерируется `tools/gen-api.mjs`; сверено: 203 операции, множество равно `docs/openapi.json` и `backend/openapi.json`. CI проверяет дрейф (`ci.yml`:97). | `README.md`:340,502; `.github/workflows/ci.yml`:97; `tools/gen-api.mjs`:1,47; `tools/check-contract.mjs`:48; `docs/LEAD-DECISIONS.md`:6; `_wt-backend-main/README.md`:518. |
| `docs/backend-issues.md` | TRIM (только классификация; построчную проверку делает другой аудитор) | Живой трекер: 101 пункт (B 34, A 39, C 28). Помечены «ИСПРАВЛЕНО»: 57, «ЧАСТИЧНО»: 4, «закрыто лидом»: 1, то есть около 39 без пометки. Нумерация зашита в код и тесты бэкенда, поэтому архивировать нельзя. Рекомендация: закрытые пункты вынести в конец («Закрыто»), сверху оставить открытые, добавить пункты за 26.09 (приём вендоров, оплат, LMS, отчёт внешнего тестирования). Номера не менять. | `README.md`:320,485,500; `docs/STATUS.md`:10,86; `docs/handoff-C.md`:55; `docs/LEAD-DECISIONS.md`:44,62; `docs/lead-requests.md`:7,25,42; `docs/plan-config.md`:144; `docs/plan-crm.md`:6,282; `docs/plan-identity-signing.md`:241,252; код фронтенда (22 комментария: `src/lib/api/schema.d.ts`:6627, `features/config/{catalog/directions.ts:14,51; integrations/health.ts:1; integrations/limits.ts:1; reports/api.ts:37; reports/params.ts:118; reports/xlsx.ts:2; workflows/editor/ArchiveWizard.svelte:3}`, `features/crm/{contacts/ContactForm.svelte:3; deals/card/dealCard.svelte.ts:15; deals/card/DealOverview.svelte:2; deals/list/DealsBoard.svelte:3; deals/list/DealsTable.svelte:35; files/Attachments.svelte:4,109; notifications/notifications.svelte.ts:2; organizations/DriftBanner.svelte:3; recent/localRecent.ts:1; shared/entityCache.svelte.ts:1}`, `features/signing/{known.ts:1; status.ts:204; urls.ts:2}`); бэкенд: `README.md`:498,509,519, `app/modules/reporting/{schemas.py:92, seed.py:164, service.py:347}`, `deploy/entrypoint.sh`:66, тесты `tests/{test_deals.py:309, test_integrity_errors.py:2, test_principal_cache.py:2, test_reporting.py:355,497, test_workflow.py:457}`; корневой `README.md`:678; `.github/README.md`:686; `.github/profile/README.md`:160. |
| `docs/handoff-C.md` | ARCHIVE | Передача дел завершённого агента C. §2 «не сделано 21 ручка» — всё сделано (по `coverage.mjs` из области C без экрана только `GET /api/admin/teams/{id}`); §4 «DealSignatures не встроен в карточку» неверно (`DealCardPage.svelte`). Ценное — §3 «Приёмы» (ловушки Svelte) стоит перенести в README (см. раздел 2). | `README.md`:503. |
| `docs/lead-requests.md` | ARCHIVE | Журнал запросов агентов к лиду (все выполнены или сняты; замечания к rt-ui продублированы в REBUILD-PLAN §6). | `README.md`:501; `docs/handoff-C.md`:55; `docs/AGENT-BRIEF.md`:77,175. |
| `docs/plan-config.md` | ARCHIVE | План агента B (фаза 1). Чек-лист эндпоинтов заменён `tools/coverage.mjs` и `STATUS.md`; риски (#17, #19, #1) закрыты или описаны в backend-issues. | `README.md`:503; `docs/AGENT-BRIEF.md`:161 (`docs/plan-<область>.md`). |
| `docs/plan-crm.md` | ARCHIVE | План агента A (фаза 1); §0 «факты контракта» устарели (например, «нет total, сортировки, счётчика непрочитанных», всё исправлено 25.09). | `README.md`:503; `docs/AGENT-BRIEF.md`:161. |
| `docs/plan-identity-signing.md` | ARCHIVE | План агента C; все чекбоксы остались `[ ]` (handoff-C прямо говорит, что их не обновляли), то есть документ вводит в заблуждение. | `README.md`:503; `docs/handoff-C.md`:5; `docs/AGENT-BRIEF.md`:161. |

Итого по ссылкам при архивации восьми документов нужно поправить: `README.md` строки 331 (ссылка на DS-MIGRATION), 499, 501, 503 (строки таблицы «Документы»); корневой `README.md`:677 и `.github/README.md`:685 (строка про `frontend/docs/DS-MIGRATION.md`); `docs/STATUS.md`:36 и 87 (упоминание DS-MIGRATION и lead-requests). Остальные архивируемые файлы больше ни на кого не ссылаются. Вне `docs/*.md` других ссылок на них нет (в том числе в `src`, `tools`, CI, `deploy`, `architecture`).

## 2. Устаревшие места в TRIM-файлах

### 2.1 `README.md`

| Место | Что не так | Как сейчас |
|---|---|---|
| Строка 25 («65 компонентов в `src/lib/ui` (53 общих и 12 полей)») | Противоречит строке 10 (66) и строке 156 (54 общих). | 54 общих + 12 полей = 66 (подсчёт `src/lib/ui/*.svelte` и `ui/fields/*.svelte`). Исправить на «66 (54 + 12)». |
| Строка 84 (стек, `pdfjs-dist`) | `^5.4.0` | В `package.json`: `^6.3.289`. Остальные версии таблицы (строки 78-88) совпадают с `package.json`. |
| Строки 60 и 402-403 (что проксирует dev) | Перечислены `/api`, `/public`, `/health`, `/auth`; в таблице переменных нет `API_URL`. | `vite.config.ts`:14-17 проксирует ещё `/static` (ассеты Swagger) и понимает `API_URL` (API отдельно от Keycloak). |
| Строка 139 (mermaid: «блоки B1–B14») | B15 давно принят. | «B1–B15». |
| Строка 167 («285 файлов `.svelte` и 141 `.ts` … на 22.09.2026») | Числа старые. | main: 293 `.svelte`, 148 `.ts` (36 тестовых файлов), включая `routes/dev`. Лучше убрать «по состоянию на…» и числа. |
| Строка 220 («Удалить черновик можно (`DELETE /api/workflows/{id}`, 22.09.2026)») | Читается как возможность редактора, но UI не вызывает эту ручку (`coverage.mjs`), кнопки нет; ниже, строка 483, она сама названа «без экрана». | Переформулировать: «ручка есть на бэкенде, кнопки в UI пока нет». |
| Строка 274 vs 315 (`sourceId`) | Строка 274: настраивается `BITRIX_SOURCE_ID` (по умолчанию `OTHER`); строка 315: «зашит как `OTHER`». Противоречие. | Бэкенд: `app/core/config.py`:212 `bitrix_source_id: str = "OTHER"`, настраиваемый. Правка строки 315: «по умолчанию `OTHER`, код источника берётся из настроек портала». |
| Строка 424 («Прогон от 25.09.2026: 32 файла, 232 теста … 2577 файлов») | Противоречит шапке (строка 10, 285 тестов) и `STATUS.md`:76 (36 файлов, 285). | 36 файлов, 285 тестов (по коду; прогон повторить перед публикацией и обновить `svelte-check`). |
| Строка 447 («`_x2.mjs` — черновой скрипт») и 449 («`buttons-report.mjs` (файл появится позже)») | `tools/_x2.mjs` не существует; `tools/buttons-report.mjs` есть (STATUS:79 его описывает). | Убрать `_x2.mjs`, убрать «файл появится позже». В списке инструментов (432-445) нет `buttons-page.mjs`, `buttons-report.mjs`, `check-bundle-size.mjs`, `check-contract.mjs`, `bundle-budget.json`. |
| Строка 477 (`.dockerignore`: «не попадают `docs` и `tools/scenarios`») | Неполно. | `.dockerignore` также исключает `.github`, `coverage`, `.env*`, `.shots`. Мелочь. |
| Строки 494-504 (таблица «Документ») | После архивации строки для DS-MIGRATION, LEAD-DECISIONS/lead-requests и AGENT-BRIEF/plan-*/handoff-C ведут в пустоту. | Заменить одной строкой «`docs/archive/` — история первой сборки» либо убрать. Ссылка на `#покрытие-ручек-175-из-203-86` (строка 445) держится на заголовке `STATUS.md`: не переименовывать. |
| Строка 486 («Ловушки rt-ui») | Первая ловушка про `Popover` с `showCloseButton` перенесена из REBUILD-PLAN §6; в исходниках `rt-ui/src/lib/components/Popover/Popover.svelte` (строки 106, 151, 280) заголовок с кнопкой закрытия уже рисуется. | Проверить на версии `0.1.1` из GitHub Packages и, если исправлено, убрать пункт из README и REBUILD-PLAN §6. Остальные ловушки не перепроверял. |
| Строка 485 («в демо остались тестовые записи прогонов сценариев») | Не проверяется по репозиторию (это состояние стенда). | Оставить как оговорку или удалить. |

Не менять (внешние ссылки): заголовки «Как устроено», «Блоки интерфейса», «Редактор воронки: графовый конструктор», «Интеграция с Битрикс24», «Экраны и права», «Проверка качества», «Один origin с API (dev-клиент за Caddy)», «Быстрый старт».

Сверено и верно: цифры 47 экранов (49 `+page.svelte` минус 2 в `dev`), 175/203, 285 тестов (по коду), 66 компонентов, 40 сценариев, пороги покрытия (45/78/82), Dockerfile (`node:22-alpine`, `caddy:2.11-alpine`, UID 10001, `HEALTHCHECK`), `.trivyignore`, схема доставки в Битрикс (`_BACKOFF_SECONDS = [1, 5, 30, 300, 1800, 7200]`, `_MAX_ATTEMPTS = 8`, флаг `bitrix_connector` третьим условием), учётки в `static/config.json`, `HELP_REV`, каталог `dev/blocks` (демо B01-B10, B12-B14), «12 ручек 25.09 и 8 ручек 22.09 без экрана» (совпало с `coverage.mjs`).

Стоит перенести в README (из handoff-C §3) перед архивацией: имя сниппета не должно совпадать с именем `$derived` в компоненте; сброс формы при открытии делать внутри `untrack`; кнопка отправки в футере модалки — `<Btn type="submit" form="<id>">`.

### 2.2 `docs/STATUS.md`

| Место | Что не так | Как сейчас |
|---|---|---|
| Строка 3 («Обновлено: 2026-09-22 (ночь)») | Документ частично обновлялся 25-26.09 (строки 39-76), шапка не менялась. | Обновить дату и переписать вводный абзац. |
| Строки 5-12 («Ночь 22.09: обход кнопок и аудит») | Исторический отчёт с отсылкой «см. `git log`/переписку» (ссылки в никуда); строка 12 «232/232 теста» не совпадает с актуальными 285. | Свернуть до 2-3 строк (что есть `buttons.mjs`) или вынести в архив. |
| Строка 19 («сейчас Caddy отдаёт Vite с хоста (`WEB_UPSTREAM=…`)») | Утверждение о состоянии чьей-то машины, а не документа. | Оставить только способ (README «Один origin с API»). |
| Строка 31 («типизированный клиент по OpenAPI (181 операция)») | Устарело. | 203. |
| Строка 36 («Общие обёртки — `Notice`, `StatusChip`, `FilterChips`, `ResponsiveTable`…», «Правила — `docs/DS-MIGRATION.md`») | `FilterChips`, `ResponsiveTable` удалены (REBUILD-PLAN:118; в `src/lib/ui` их нет); DS-MIGRATION уходит в архив. | Актуальный список: `DataTable`, `Notice`, `StatusChip`, `Card`, `FormDrawer`…; правила — README «Соглашения». |
| Строка 57 («Появилось на экранах 26.09 (11)») | Перечислено 10 ручек (entity-types, rows, contacts/{id}/products, learner-profile GET/PUT, reveal, products/{id}/contacts GET/PUT/DELETE, POST feature-flags). | Пересчитать: рост покрытия 165 → 175 = 10 новых операций. |
| Строка 86 («кэш принципала после отката JIT, скачивание скана ЭДО только у ADMIN») | Оба исправлены 25.09 (backend-issues A-28/B-30, C-28). | Убрать; оставить реально открытые (см. раздел 3). |
| Строка 87 (ссылка на `lead-requests.md`) | Файл уходит в архив. | Убрать. |
| Строка 89 («тестовые записи … не почищены») | Проверить по стенду нельзя. | Оставить как оговорку либо убрать. |

### 2.3 `docs/REBUILD-PLAN.md`

| Место | Что не так | Что делать |
|---|---|---|
| Строки 1-3 («План на завтра… по нему идём завтра») | Работа закончена 22.09 (все B1-B15 приняты, строка 137). | Переименовать в «Блоки интерфейса: правила и приёмка» (ссылки на файл при этом не менять). |
| Строки 5-12 (§0 «Как работаем»), 10 («Агентов не привлекаем… Sonnet отвечает 403») | Процессные договорённости на один день. | Убрать или сократить до правил приёмки («слово „готово" говорит заказчик»). |
| Строка 17 («тесты (230)») и 96 («`pnpm test` (230)») | Устарело. | 285. |
| Строки 20-22 («Перед началом: сохранить копию… решает заказчик»); 99-105, п. 5 («`frontend-old` или `git init`») | Сделано: в проекте есть git, каталог `frontend-old` существует. | Убрать. |
| Строки 24-38 (§2 «Замечания заказчика от 21.09») | Все 8 замечаний исправлены блоками B1-B9. | Убрать таблицу (сохранить одну строку-итог). |
| Строки 62, 64-66 (§4 «Как показывать блоки… перед выдачей удалить») | Каталог `src/routes/dev` существует; решение «удалить перед выдачей» осталось открытым (см. раздел 3). | Оставить одной строкой в разделе «Что не сделано». |
| Строки 92-105 (§7 «Окружение», §8 «Открытые вопросы») | §7 частично дублирует README; §8: п. 1 решён (строка 132), п. 5 решён; п. 2 (бренд), 3 (плотность таблиц), 4 (кнопки темы и справки) в документах нигде не закрыты. | Закрыть или явно пометить «оставлено как есть». |
| Строки 123-127 («Что не сделано (на 22.09)») | Верно: каталоги «Направления», «Причины отказа», «Календарь» и редактор воронки не на `DataTable`; `src/routes/dev` не удалён. | Оставить, обновить дату. |
| Строки 72-90 (§6 ловушки rt-ui) | Возможно частично исправлены в rt-ui (см. README:486). | Перепроверить на 0.1.1. |

Живое ядро (оставить как есть): §3 (строки 40-62, критерии блоков), §6, §9 (107-121), приёмка B9-B15 (135-184), правило «не менять принятые блоки без слова заказчика».

### 2.4 `docs/USERFLOWS.md`

| Место | Что не так | Как сейчас |
|---|---|---|
| Строка 3 («Составлено 2026-09-21… по которой собираются экраны… Блоки — в каталоге `/dev/blocks`») | Написано в будущем времени; каталог доступен только в dev-режиме. | Переписать как «карта потоков и блоков». |
| Строка 12 (§0.6: «недоступное действие — в меню „Ещё" с подписью „Недоступно вашей роли"») | Реализовано только в шапке сделки (`features/crm/deals/card/DealHeader.svelte`:59); STATUS:11 (S2-5) признаёт, что не везде. | Написать «где применимо» либо отметить как частично выполненное. |
| Строки 34-36 (F0, п. 4: «приглашение: пароль по политике → согласие → вход») и строка 25 | Страница `/invite/[token]` только проверяет ссылку и ведёт ко входу (`routes/invite/[token]/+page.svelte`, комментарий в 2-й строке; пароль задаёт Keycloak). | «Ссылка → ФИО, срок → перейти ко входу; пароль задаёт Keycloak». |
| Строки 45-48 (F2: «[Запросить доступ]», «есть в удалённых → Восстановить») | Таких ручек нет (backend-issues A-18); UI пишет «Восстановить её может администратор» (`crm/organizations/OrgCreateDrawer.svelte`:173). | Убрать эти действия или пометить «ограничение бэкенда». |
| Строка 85 (F9: «Открытый вопрос к заказчику: структура раздела (см. §4)») | Решено 21.09 (REBUILD-PLAN:132: «Отчёты» оставить как есть). | Убрать. |
| Строки 131-136 (§4 «Открытые вопросы к заказчику») | Вопрос 1 решён; 2-4 повторяют REBUILD-PLAN §8 и нигде не закрыты. | Удалить раздел, оставшееся перенести в единый список решений. |

### 2.5 `.github/PULL_REQUEST_TEMPLATE.md`

* Строка 9: «README / RUNBOOK / docs» — в этом репозитории RUNBOOK нет (он в `lct-testkit/deploy`).
* Строка 14: «для БД: есть миграция и `alembic check`» — у фронтенда нет БД.

### 2.6 `docs/backend-issues.md` (только классификация)

* Формат: три раздела B/A/C, у каждого пункта либо «ИСПРАВЛЕНО <дата>», либо открытая формулировка. Нет раздела за 26.09 (приём вендоров/оплат/LMS, отчёт внешнего тестирования); `backend/README.md`:519 называет «101 несоответствие».
* Заголовок раздела C и ссылки на «агента B/A/C» — артефакт первой сборки.
* Уходит в TRIM, не в архив: номера использует код обоих репозиториев (список в таблице 1).
* Комментарии в коде фронтенда, которые ссылаются на уже исправленные пункты и вводят в заблуждение: `deals/list/DealsTable.svelte`:35 и `DealsBoard.svelte`:3 (A-11), `notifications/notifications.svelte.ts`:2 (A-24), `crm/shared/entityCache.svelte.ts`:1 (A-29), `signing/known.ts`:1 (C-5), `signing/urls.ts`:2 (C-2/C-3), `config/reports/xlsx.ts`:2 (#19), `workflows/editor/ArchiveWizard.svelte`:3 (#1), `crm/files/Attachments.svelte`:109 (A-23), `signing/status.ts`:204 (C-7). Они же фиксируют то, что клиент не адаптирован (см. раздел 3).

### 2.7 Справка `src/lib/content/help/*.md`: очевидно неверное сегодня

| Файл:строка | Утверждение | Как есть |
|---|---|---|
| `08-security.md`:6 | «второй фактор (TOTP)… включается при создании или в карточке пользователя» | В UI только при создании (`identity/users/UserCreateModal.svelte`, поле `totp`); `UserPatchRequest` в OpenAPI поля TOTP не имеет, в карточке пользователя включить нельзя. |
| `09-tasks.md`:7 | «почта и Telegram идут через шлюзы, которые в этой сборке не подключены… доставки наружу нет» | После PR #15 бэкенда (`app/modules/notification/email_transport.py`) есть SMTP-транспорт: при заданном `SMTP_HOST` письма реально уходят (в том числе ссылки подписантам). Telegram по-прежнему заглушка (`notification/service.py`:20-21, 467). |
| `10-integrations.md`:19 | «Доставка почтой и в Telegram не подключена» | То же: почта подключается настройкой `SMTP_HOST`. |
| `13-about.md`:17 | «почта и Telegram не подключены» | То же. |

Остальное (11-design-system: 111 компонентов, 1349 иконок, 374/374 историй — совпадает с `rt-ui/README.md`:8; Ctrl+K по `e.code === 'KeyK'` — `ui/GlobalSearch.svelte`:120; отсутствие кнопки «Повторить доставку» — верно) не противоречит коду.

## 3. Открытые пункты (проверены по коду)

Тип: BUG / GAP (сделано на бэкенде, клиент не адаптирован) / LIMITATION / DECISION. Важность: высокая / средняя / низкая.

| # | Суть | Тип | Источник (файл:строка) | Доказательство в коде | Важность |
|---|---|---|---|---|---|
| 1 | Нет кнопки «Повторить» для событий Битрикс/интеграций в статусе «Не доставлено» (ручка есть с 25.09) | GAP | `README.md`:483; `STATUS.md`:63; `help/10-integrations.md`:19 | `tools/coverage.mjs`: `POST /api/admin/integrations/outbox-events/{event_id}/retry` не вызывается; в `features/config/integrations/OutboxTab.svelte` нет `retry` (только колонка «повтор» в строке 44) | средняя |
| 2 | Битрикс24 работает только в одну сторону (CRM → Битрикс), реальные события портала не принимаются; стадии, ответственные, удаление не синхронизируются | LIMITATION | `README.md`:311-316,484; `help/10-integrations.md`:19 | `_wt-backend-main/app/modules/integration/bitrix.py` (докстринг модуля: нет `event.bind`); в OpenAPI есть только упрощённый `POST /api/v1/integrations/bitrix/webhook` | средняя |
| 3 | Предпросмотр шаблона уведомления не подключён | GAP | `STATUS.md`:64 | `coverage.mjs`: `POST /api/admin/notification-templates/preview` не вызывается; в `features/config/notifications` слова `preview` нет | низкая |
| 4 | Профиль только для чтения (нельзя менять имя, часовой пояс, телефон); внутренний подписант без телефона получает код только на почту | GAP | `STATUS.md`:67 | `coverage.mjs`: `PATCH /api/me`; в `features/identity/ProfilePanel.svelte` нет вызова `PATCH` | средняя |
| 5 | Команду нельзя открыть карточкой и править с `If-Match`: правка идёт без версии | GAP | `STATUS.md`:65 | `routes/(app)/admin/teams/+page.svelte`:110 `api.PATCH('/api/admin/teams/{team_id}', …, body)` без `headers: ifMatch(…)`; `GET /api/admin/teams/{team_id}` не вызывается | низкая |
| 6 | Продукты сделки после создания не правятся (ручка `PUT /api/deals/{id}/products` есть) | GAP | `STATUS.md`:66; `plan-crm.md` риск №4 | `features/crm/deals/card/DealOverview.svelte`:2 («продукты (только чтение — backend-issues A-7)»); `coverage.mjs`: `PUT /api/deals/{deal_id}/products` не вызывается | средняя |
| 7 | Счётчик непрочитанных и список кодов событий берутся обходом (до 100 записей, «99+»; коды зашиты в клиенте) | GAP | `STATUS.md`:68 | `features/crm/notifications/notifications.svelte.ts`:1-2,7 (`LIMIT = 100`); `notifications/eventCodes.ts`:24 (`EVENT_CODES`); `coverage.mjs`: `unread-count`, `event-codes` не вызываются | низкая |
| 8 | Публичная страница подписи грузит PDF по presigned-ссылке S3 с подменой origin; готовый same-origin PDF (`file_url`, `/public/sign/{token}/file`, `/api/signature-requests/{id}/file`) не используется | GAP | `STATUS.md`:69 | `features/signing/SignFlow.svelte`:79 (`page.document.preview_url`); `PdfViewer.svelte`:42 (`previewUrlForFetch`); `coverage.mjs`: обе `…/file` не вызываются; `SigningDocumentPreview.file_url` есть в OpenAPI | средняя (зависит от Caddy/CORS стенда) |
| 9 | «Переиздать ссылку» подписанту нет в UI, мастер по-прежнему предупреждает, что ссылку внешнему не первому подписанту сервер не выдаст | GAP | `STATUS.md`:70 | `features/signing/SendWizard.svelte`:112; `coverage.mjs`: `POST /api/signature-requests/{id}/reissue-link` не вызывается | средняя |
| 10 | Воронку нельзя переименовать/назначить по умолчанию и удалить черновик из UI | GAP | `STATUS.md`:71,55; `README.md`:220 | `coverage.mjs`: `PATCH` и `DELETE /api/workflows/{workflow_id}` не вызываются | низкая |
| 11 | Прогресс переноса сделок при архивации статуса — опрос воронки вместо задачи `mapping-jobs` | GAP | `STATUS.md`:72 | `features/config/workflows/editor/ArchiveWizard.svelte`:3 («Бэкенд не отдаёт ход переноса»); `coverage.mjs`: `GET …/mapping-jobs/{job_id}` не вызывается | низкая |
| 12 | Нет экрана лицензий/договоров вуз-вендор-ПО | GAP | `STATUS.md`:53; `README.md`:483; `STATUS.md`:11 | `coverage.mjs`: `GET /api/organization-licenses` и `/{license_id}` не вызываются; в `src` нет маршрута лицензий | средняя (требование кейса, п. 4) |
| 13 | Дашборды читают xlsx в браузере, хотя есть JSON `GET /api/reports/{id}/data` | GAP | `STATUS.md`:54; `plan-config.md` §D | `features/config/reports/xlsx.ts`:2; `coverage.mjs`: `…/reports/{report_id}/data` не вызывается | низкая |
| 14 | Нет кнопок «Удалить» у направлений, причин отказа, шаблонов уведомлений, версий реестра | GAP | `STATUS.md`:55 | `coverage.mjs`: четыре `DELETE` не вызываются (`rg "api.DELETE"` по `src` показывает только продукты-контакты, дашборды, комментарии, участников, вложения, сессии, файлы) | низкая |
| 15 | Отчёты «Воронка», «Динамика», «Застрявшие» без периода, фильтров по вузу/направлению/продукту и выбора колонок; период есть только у `lms_users_upload` | GAP | `STATUS.md`:11 («большой разрыв с исходным кейсом») | `features/config/reports/params.ts`:31-40 (`REPORT_PARAMS`) | средняя |
| 16 | Список сделок не использует `total`, `sort`, `is_closed`: сортировка по загруженным строкам, «Закрытые» через хак `closed_from=1970-01-01`, счётчиков вкладок нет | GAP | `plan-crm.md` риск №3; `backend-issues.md` A-11 (исправлено 25.09) | `features/crm/deals/list/dealFilters.ts`:79; `DealsTable.svelte`:35; `DealsBoard.svelte`:3; в `src/lib/api/schema.d.ts` ответ содержит `total`, в `deals/list` он не читается | низкая-средняя |
| 17 | Названия организаций и номера сделок в списках догружаются пачками запросов (N+1), хотя `DealOut.organization_name`, `contact_name`, `TaskOut.deal_number`, `deal_title` уже отдаются | GAP | `backend-issues.md` A-13, A-29 (исправлено 25.09) | `features/crm/shared/entityCache.svelte.ts`:1; поиск `organization_name\|deal_number` по `features/crm` пуст | низкая |
| 18 | Каталоги «Направления», «Причины отказа», «Календарь» и редактор воронки не на `DataTable` (намеренно: дерево, ручной порядок, группировка) | DECISION | `README.md`:481; `REBUILD-PLAN.md`:126 | Страницы `DirectionsPage`, `LossReasonsPage`, `HolidaysPage` не используют `CatalogList`/`DataTable` (использует только `ProductsPage`, `CustomFieldsPage`, `RegionsPage`) | низкая |
| 19 | Каталог блоков `src/routes/dev` «удаляется перед выдачей» (пока остаётся, в production-сборку не попадает, отвечает 404) | DECISION | `REBUILD-PLAN.md`:66,127; `README.md`:481 | `src/routes/dev/blocks/**`, `src/routes/dev/+layout.ts` | низкая |
| 20 | Редактор воронки на телефоне без перетаскивания, «Сохранить» и «Опубликовать» отдельным рядом | LIMITATION | `README.md`:482; `STATUS.md`:88; `help/12-shortcuts.md` | `features/config/workflows/editor/MobileLists.svelte`, `CanvasFlow.svelte` | низкая |
| 21 | У праздников производственного календаря нет `DELETE` (только `is_active` у справочников остальных) | LIMITATION (бэкенд) | `README.md`:485; `STATUS.md`:86 | `_wt-backend-main/openapi.json`: у `/api/holidays/{holiday_id}` только `patch` | низкая |
| 22 | Доставка по Telegram не подключена (заглушка), почта только при заданном `SMTP_HOST` | LIMITATION | `help/09-tasks.md`:7; `help/13-about.md`:17 | `_wt-backend-main/app/modules/notification/service.py`:20-21,467 | средняя |
| 23 | Разделы «Справочники» и «Воронки» скрыты в меню для КАМ/HEAD (нужен `*:write`), хотя прямая ссылка даёт чтение (решение заказчика/лида S2-3) | DECISION | `STATUS.md`:11 | `src/lib/nav.ts`:66-67; комментарий `routes/(app)/+layout.svelte`:28-32 | низкая |
| 24 | Диалог перехода теряет черновик при переходе по подсказке-ссылке (S2-12); правило «недоступно вашей роли» реализовано не везде (S2-5) | LIMITATION | `STATUS.md`:11; `USERFLOWS.md`:12 | S2-5: подпись есть только в `crm/deals/card/DealHeader.svelte`:59. S2-12 в коде не перепроверял | низкая |
| 25 | Для `sign-session.svelte.ts`, `send-draft.svelte.ts`, `identity/audit/audit-query.ts` нет unit-тестов | GAP | `handoff-C.md`:52 | `grep -rn "sign-session\|send-draft\|audit-query" --include=*.test.ts src` пуст | низкая |
| 26 | Приглашение в закрытом контуре: страница `/invite` не даёт задать пароль (только «Перейти ко входу», пароль ставит Keycloak) | LIMITATION | `backend-issues.md` C-18; `USERFLOWS.md`:34-36 | `routes/invite/[token]/+page.svelte`:2,30-33 | средняя (статус на бэкенде проверяет другой аудитор) |
| 27 | Смена пароля и сброс на демо-стенде меняют реальную демо-учётку (ломают вход в один клик) | LIMITATION | `backend-issues.md` C-21 | `identity/PasswordPanel.svelte` (предупреждение в UI); бэкенд `router_me.py` | низкая |
| 28 | В `ci.yml`:107 предупреждение ссылается на `docs/REPO-SETTINGS.md`, которого в этом репозитории нет (файл лежит в `lct-testkit/deploy`: `deploy/docs/REPO-SETTINGS.md`) | BUG (ссылка) | `.github/workflows/ci.yml`:107 | В `_wt-frontend-main/docs/` файла нет | низкая |

Пункты, не вошедшие сюда: остальные открытые пункты `backend-issues.md` (A-8, A-14, A-15, A-16, A-18, A-19, A-21, A-22, B-7, B-14, B-15, B-21, B-22, B-25, B-26, B-32, B-33, C-11, C-22 и др.) — их состояние в бэкенде проверяет другой аудитор.

## 4. Прочие наблюдения (вне списка файлов)

* Корневой `D:\testkit-lct\README.md`:658,674 («16 из 181», «165 из 181») и `.github/README.md`:666,682 («28 из 193», «165 из 193») отстают от `STATUS.md` (175 из 203, без экрана 28). В `.github/README.md` число «28» совпало случайно, знаменатель нужно обновить. Оба README будут ссылаться на `docs/DS-MIGRATION.md` (корневой:677, `.github`:685), что сломается после архивации.
* `deploy/docs/REPO-SETTINGS.md`:59 утверждает, что «`frontend/README.md` прямо говорит, что лицензии нет», но `LICENSE` во фронтенде добавлен (коммит 1e37c8b). Репозиторий deploy — чужой, только для сведения.
* Прогон `vitest`/`svelte-check`/`eslint` не выполнялся: числа 285 тестов и 36 файлов получены статически. Утверждения о состоянии стенда (тестовые записи, что именно включено на сервере) из репозитория не проверить.
