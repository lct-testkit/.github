# Аудит Markdown-документов: корень, .github, backend, deploy

Дата аудита: 2026-09-27. Режим: только чтение.

Источники истины при проверке:
- backend: `D:\testkit-lct\_wt-backend-main` (main, коммит 3b0d5cd = PR #15);
- frontend: `D:\testkit-lct\_wt-frontend-main` (main, 41dde50);
- deploy: `origin/main` = 857ae99. Локальная `main` = 903e56f, отстаёт на один коммит (bump образа web, в документах ничего не меняет); рабочее дерево чистое;
- `D:\testkit-lct\backend` — рабочее дерево на ветке `fix/registry-flag-sessions-0927` с чужими правками, проигнорировано. README в нём отличается от main только переводами строк (проверено через `diff` без `\r`).

Что важно знать до чтения таблиц:

1. `D:\testkit-lct\README.md` (корень) и `D:\testkit-lct\.github\README.md` — две расходящиеся копии README продукта (682 и 690 строк). Свежая копия — `.github/README.md`, но её изменения (и изменения `.github/profile/README.md`) **не закоммичены**: `git status` в `.github` показывает `M README.md`, `M profile/README.md`. Опубликованная версия (коммит 1d2c59f) ещё старее.
2. Числа в обеих копиях уже не соответствуют коду: на main бэкенда 203 операции, 20 миграций, 67 таблиц, 2143 теста. В корне написано 181 операция, в `.github` — 193, 65 таблиц и 790 тестов.
3. `dop.md`, `new_spec.md` и `architecture/README.md` в корне побайтно совпадают с копиями в `.github` (md5 равны). `rtk_requiriments.md` в `.github` не копируется, и README это оговаривает.

---

## 1. Сводная таблица

Условные обозначения вердиктов: KEEP — оставить как есть, TRIM — живой документ с устаревшими местами (детали в разделе 2), ARCHIVE — разовый документ, работа сделана.

| Файл | Вердикт | Причина | Кто ссылается |
|---|---|---|---|
| `D:\testkit-lct\README.md` | **TRIM**: свести к одному источнику. Либо генерировать корневую копию из `.github/README.md` с относительными ссылками, либо ARCHIVE корневую копию, если поддерживать две не хотят | Устаревшая копия: 181 операция, 12 сервисов, caddy 2.8, `vendor/…tgz`, пароли демо-учёток открытым текстом, Node 20.19+. Публичная копия эти места уже исправила. Список конкретных мест — раздел 2.1 | Прямых ссылок на файл нет. Frontend и backend ссылаются на `github.com/lct-testkit/.github#…`. Упоминают «корневой README» по имени `frontend/docs/STATUS.md:11,19`. Относительная ссылка `../README.md#как-устроено` в `architecture/README.md:24` |
| `DEPLOY-CONTRACT.md` | **ARCHIVE** | Контракт на переделку установщика (проблемы #1–#11). Всё реализовано в deploy #13/#14 и выпущено в v0.3.1: режимы TLS `off\|internal\|acme`, `lib_host.sh`, `set_host.sh`, `seed/`, `seed_demo.sh`, профиль `demo-data`, снятие `disable_redirects`, `handle /fonts/*`, CSP для `/auth/*`, `KEYCLOAK_ADMIN*`, Caddyfile побайтно совпадает с backend. Нереализованные хвосты (раздел 2.8) перенесены в открытые пункты | Только `DEPLOY-FIXES.md`. В репозиториях, CI-workflow и скриптах ссылок нет (grep по всем репозиториям, включая `.github/workflows` и `scripts`) |
| `DEPLOY-FIXES.md` | **ARCHIVE** | Список из 9 проблем установки v0.2.0 на lct.velikoss.ru и черновые «предложения». В коде уже реализовано иначе и лучше: `CRM_TLS_SNIPPET` вместо предложенного `CRM_TLS_DIRECTIVE`, порты 80/443, `/fonts`, `set_host.sh`, `KEYCLOAK_ADMIN`. Часть предложений так и не реализована (`--email`, `--yes-i-know-demo`, сброс пароля Keycloak) — они учтены в разделе 3 | Только `DEPLOY-CONTRACT.md:4,222,232` |
| `dop.md` | **KEEP** | Документ заказчика и спецификация. Копия в `.github/docs/dop.md` идентична | `.github/README.md:23,273,678`; `.github/profile/README.md:167`; `backend/README.md:515`; ~25 комментариев в коде бэкенда (`config.py`, `permissions.py`, `errors.py`, …); `frontend/docs/AGENT-BRIEF.md:15` (абсолютный путь `D:\testkit-lct\dop.md`), `plan-crm.md:5`, `plan-identity-signing.md:6`, `STATUS.md:3`, `USERFLOWS.md:3`, `backend/_spec/spec.txt`. Перемещать нельзя |
| `new_spec.md` | **KEEP** | То же; копия в `.github/docs/new_spec.md` идентична | Те же места, плюс `frontend/README.md:204,226`, `.github/README.md:60,81,517`, `backend/README.md:514`, `AGENT-BRIEF.md:14` |
| `rtk_requiriments.md` | **KEEP** | Исходное задание. Намеренно не хранится в `.github` | `README.md:23,77,366,668`, `.github/README.md:23,77,374,676`, `architecture/README.md:3`, ~15 комментариев и docstring в коде бэкенда (`catalog/models.py`, `imports/fields.py`, `reporting/builders.py`, миграция 0015, тесты) |
| `architecture/README.md` | **KEEP** | Модель `rtk-school-crm.archimate` проверена: 49 элементов, 69 связей, как в тексте. Идентична `.github/architecture/README.md`. Замечание по полноте — раздел 2.9, необязательное | Корневой README (:390,661,671) и `.github/README.md` (:398,669,679) |
| `.github/README.md` | **TRIM** | Актуальнее корневого, но числа устарели (203 операции вместо 193, 2143 теста вместо 790 и т. д.), и правки не закоммичены. Раздел 2.2 | Целевой файл для `frontend/README.md:19,50,60,444,504`, `backend/README.md:110,516`, `.github/profile/README.md` (якоря `#быстрый-старт`, `#демо-данные`, `#соответствие-требованиям-кейса`, `#используемые-библиотеки`) — эти заголовки при правке менять нельзя |
| `.github/profile/README.md` | **TRIM** | Статистика в шапке: 193 операции, 65 таблиц, 790 тестов; «8 шаблонов отчётов». Раздел 2.3 | Страница организации; ссылки на него в проверенных файлах не найдены |
| `.github/architecture/README.md` | **KEEP** | Идентична корневой | `.github/README.md` |
| `.github/docs/dop.md`, `.github/docs/new_spec.md` | **KEEP** | Идентичны корневым. Нужен способ держать их в синхронизации (сейчас md5 равны) | `.github/README.md:273,677,678`, `backend/README.md:514,515`, `.github/profile/README.md:39,167` |
| `backend/README.md` | **TRIM** (мелко) | Основная часть актуальна: 203 операции, 20 миграций, 2143 теста, 67 таблиц, 30 групп, 40 прав, 17 cron-задач, 11 сервисов — всё подтверждено кодом. Устарели 5 мест, раздел 2.4 | `.github/README.md`, `frontend/README.md`, `deploy` (по имени) |
| `backend/SECURITY.md` | **KEEP** (+ пункт в открытых) | Ссылается на «контакты — в профиле организации», а в профиле контактов нет. Идентичен `deploy/SECURITY.md` | Ссылок нет |
| `backend/loadtest/README.md` | **TRIM** | Результаты от 18.09 помечены «эта сессия» и «не было проверено вживую до этого спринта». После #13–#15 не переигрывались. Раздел 2.5 | `backend/README.md:444,520`, `run_ci.sh:16`, `.github/README.md:392,517,687`, `.github/profile/README.md:139,158`, корневой `README.md:384,509,679` |
| `backend/.github/PULL_REQUEST_TEMPLATE.md` | **KEEP** | Соответствует CI: openapi, alembic check, запрет тихих пропусков | GitHub подхватывает автоматически |
| `deploy/README.md` | **KEEP** | Все перечисленные скрипты и workflow существуют; `--tls`, `--fonts`, `seed/` описаны. Не упомянуты `pull_retry.sh`, `apply_repo_settings.sh`, `seed_lib_preamble.mjs` — несущественно | `.github/profile/README.md:52,169,173`, `.github/README.md` |
| `deploy/RUNBOOK.md` | **TRIM** (2 места) | По существу совпадает с кодом: TLS, seed, `set_host`, консоль Keycloak только из закрытых сетей, `volumePreallocate=false`. Есть две битые ссылки на «корневой README», раздел 2.6 | `deploy/README.md`, `compose/README.md`, `provision_vm.sh` (по имени), `.github/README.md:278,386,690`, `.github/profile/README.md` (многократно), `backend/README.md:472` (REPO-SETTINGS) |
| `deploy/SECURITY.md` | **KEEP** (+ тот же пункт про контакт) | Идентичен backend | — |
| `deploy/compose/README.md` | **KEEP** | Таблица портов по режимам, переменные и профили совпадают с `docker-compose.yml`, `.env.example`, `Caddyfile` | — |
| `deploy/docs/ghcr-setup.md` | **KEEP** | Токены и секреты совпадают с workflow. Мелочь: п.3 говорит о пуше из `build.yml`, а образы `api`/`web` публикует `docker-publish.yml` | `RUNBOOK.md:45,234,236`, `README.md:45`, `REPO-SETTINGS.md:32`, сообщения об ошибках в `docker-publish.yml:164`, `notify.yml:51`, `release.yml:59`, `e2e.yml:10`, `provision_vm.sh:68` |
| `deploy/docs/REPO-SETTINGS.md` | **TRIM** | Устарели строка про `NODE_AUTH_TOKEN`, п.4 «выбрать лицензию», историческая врезка «rt-ui → frontend (выполнено)». Раздел 2.7 | `deploy/README.md:45`, `apply_repo_settings.sh:11`, `frontend/.github/workflows/ci.yml:107`, `backend/README.md:472` |
| `deploy/docs/TASKS.md` | **TRIM** (после чистки станет коротким бэклогом) | Заголовок «после ветки feat/install-choices», а ветка влита в #13. Из 30 пунктов выполнено около 9. Разметка по каждому — раздел 2.8 | Ссылок нет; в `deploy/README.md` в таблице документов не упомянут |
| `deploy/seed/README.md` | **KEEP** | Совпадает с `seed/*.mjs`: 27 записей реестра (13 своих + 14 дополнительных), 14 организаций. В корневом и `.github` README указано «28» — там ошибка, не здесь | `build_offline_bundle.sh:107` (копируется в бандл) |
| `deploy/.github/PULL_REQUEST_TEMPLATE.md` | **KEEP** (косметика) | Копия шаблона backend: пункт про `openapi.json` и миграции для deploy не применим; «см. README, раздел про CI» — такого раздела в `deploy/README.md` нет | GitHub |

Итог по вердиктам: 24 строки таблицы, 25 файлов; по файлам (с учётом дублей): KEEP — 15 файлов (`dop.md`, `new_spec.md`, `rtk_requiriments.md`, `architecture/README.md` ×2, `.github/docs/*` ×2, `backend/SECURITY.md`, backend PR-шаблон, `deploy/README.md`, `deploy/SECURITY.md`, `compose/README.md`, `ghcr-setup.md`, `seed/README.md`, deploy PR-шаблон); TRIM — 8 (корневой README, `.github/README.md`, `.github/profile/README.md`, `backend/README.md`, `loadtest/README.md`, `RUNBOOK.md`, `REPO-SETTINGS.md`, `TASKS.md`); ARCHIVE — 2 (`DEPLOY-CONTRACT.md`, `DEPLOY-FIXES.md`).

---

## 2. TRIM: устаревшие места и правильные факты

Номера строк — по файлам на диске сегодня. Все «правильные» значения проверены по коду или конфигурации.

### 2.1 `D:\testkit-lct\README.md` (корень)

Это устаревшая копия. Правильнее всего перенести правки из `.github/README.md` (в таблице ниже — расхождения, которых в свежей копии уже нет), а затем исправить общие ошибки из 2.2.

| Строка | Что написано | Правильно |
|---|---|---|
| 7, 10, 288 | 181 операция; 165 из 181; «65 блоков» | На main бэкенда 203 операции, 162 пути (`openapi.json`); экран есть у 175 из 203 (`frontend/docs/STATUS.md`, «Покрытие ручек: 175 из 203», перегенерировано 26.09) |
| 184, 193 | файл `frontend/vendor/lct-testkit-rt-ui-0.1.0.tgz` обязателен | Каталога `vendor` во frontend main нет. Пакет `@lct-testkit/rt-ui` 0.1.1 ставится из GitHub Packages (нужен токен `read:packages`); в `.github/README.md:184,194` это уже исправлено |
| 182, 648 | Node 20.19+, 22.13+ или 24+ | `frontend/package.json`: `engines.node >=22`; образ `node:22-alpine` |
| 290, 296–304, 292 | `admin`/`admin`, БД `crm`/`crm`, пароли пяти демо-учёток открытым текстом | В публичной копии убрано намеренно. Пароли остаются только в `realm-crm.json`, `backend/README.md:95-102` и `static/config.json` |
| 386 | «12 сервисов» | 11 (`backend/README.md:71`; `web` — профиль) |
| 389 | «8 глав» справки | 13 файлов в `frontend/src/lib/content/help/` |
| 402 | `file:./vendor/…tgz` | `0.1.1` из GitHub Packages |
| 610–619 | значения секретов приведены в таблице | Публичная копия ссылается на `.env.example`; корень надо привести к ней |
| 631–632 | нет | Нет `AUDIT_HMAC_KEY`, `SETTINGS_ENCRYPTION_KEY`, `SMTP_*` в перечне «перед боевым контуром» (общая ошибка, см. 2.2) |
| 647 | «нет `lct-testkit-rt-ui-0.1.0.tgz`» | Симптом теперь: 401/404 при `pnpm install` без токена GitHub Packages |
| 658 | «16 из 181 без экрана… 8 новых ручек» | 28 из 203: 8 умышленно и 20 без UI (`STATUS.md`) |
| 471, 505 | «Прогон от 22.09: 232 теста, 2574 файла», «бэкенд — командой не перезапускались» | Бэкенд: `pytest --collect-only` даёт 2143 теста (83 файла); в CI — 16 контрактов import-linter, mypy, `alembic check` |

Обязательно синхронизировать остальное со свежей версией: строки 97, 127–131 (Bitrix: четыре условия доставки, `BITRIX_SOURCE_ID`, флаг `bitrix_connector` теперь читается), 175–176 (флаг больше не «ловушка»), 262–278 (карта репозиториев), 366, 384–394.

### 2.2 `D:\testkit-lct\.github\README.md` (правки не закоммичены)

| Строка | Что написано | Правильно и как проверено |
|---|---|---|
| 10, 296, 666, 682 | 193 операции; «165 из 193»; «28 из 193 без экрана» | 203 операции (`openapi.json`: 203 операции, 162 пути); покрытие 175 из 203; без экрана те же 28 (8 умышленно + 20 без UI) — `frontend/docs/STATUS.md` |
| 10 | «66 блоков интерфейса» (в корне 65) | Число нигде не подтверждено. Сверить `node tools/…`/каталог блоков; нужна единая цифра |
| 479 | «232 теста, 32 файла Vitest, svelte-check 2577» | Во frontend main 36 тестовых файлов, около 285 вызовов `it/test` (grep). Перезапустить `vitest` и обновить цифры; менять их до перезапуска нельзя |
| 513 (конец абзаца) | «790 тестов бэкенда, mypy 138 файлов» | `pytest --collect-only`: **2143 теста**; `lint-imports` 16 контрактов (верно); mypy — в CI, список исключений 20 модулей |
| 507 | «Отчёт готовит `frontend/tools/buttons-report.mjs` (появится позже; ссылка будет здесь)» | Файл уже есть (`tools/buttons-report.mjs`). Убрать «появится позже» |
| 379 | «8 шаблонов отчётов» (перечислено 7 названий) | 9 шаблонов: `deal_funnel`, `kam_summary`, `region_summary`, `loss_reasons`, `sla_compliance`, `monthly_dynamics`, `stuck_deals`, `learning_progress`, `lms_users_upload` (`reporting/seed.py`, `backend/README.md:324`) |
| 382 | «антивирусная проверка (карантин)» | Антивирус — заглушка `NullAntivirusScanner` (`files/service.py:215`), статус `quarantined` объявлен, но не присваивается (`backend/README.md:485`). Формулировка вводит в заблуждение |
| 183, 656 | Node 20.19+, 22.13+ или 24+ | `engines.node >=22` |
| 422 | `pdfjs-dist` ^5.4.0 | `^6.3.289` (`frontend/package.json`) |
| 449 | `python-multipart` 0.0.30 | 0.0.31 (`pyproject.toml`) |
| 472 | `caddy` 2.8-alpine | 2.11-alpine (`backend/docker-compose.yml:377`, `deploy/images.yaml`) |
| 253 | «выгрузку реестра ЕГРЮЛ на 28 организаций» | 27 (13 своих + 14 дополнительных, `seed-demo.mjs`, `deploy/seed/README.md`) |
| 220–225 | «выполните три сида», показана одна команда | Показать все три (`reporting`, `notification`, `integration`) или сказать «один сид» |
| 619–627 | «перед боевым контуром»: нет новых секретов | Добавить `AUDIT_HMAC_KEY` (необязателен, пустой — хэш аудита без ключа), `SETTINGS_ENCRYPTION_KEY`, `SMTP_*`; в prod `Settings` отвергает демо-значения (`config.py` `_reject_demo_secrets_in_prod`) |
| 298 | Keycloak-консоль «`KEYCLOAK_ADMIN_*` в `.env.example`» | Верно; но стоит добавить, что `/auth/admin` открыт только из закрытых сетей (`ADMIN_CONSOLE_ALLOW`, `private_ranges`) — в deploy RUNBOOK это есть, в README нет |

Что не менять: якоря `#быстрый-старт`, `#демо-данные`, `#интеграция-с-битрикс24`, `#как-устроено`, `#используемые-библиотеки`, `#соответствие-требованиям-кейса`, `#readme` — на них ссылаются frontend, backend и профиль организации. Все картинки (`raw/main/docs/img/…` во frontend и `profile/img` в `.github`) существуют.

### 2.3 `D:\testkit-lct\.github\profile\README.md` (правки не закоммичены)

| Строка | Что написано | Правильно |
|---|---|---|
| 10 | 193 операции API | 203 |
| 10 | 65 таблиц | 67 (`__tablename__` в `app/`, `backend/README.md:10`) |
| 10 | 790 тестов бэкенда | 2143 |
| 106 | 8 шаблонов | 9 |
| 148 | описание CI бэкенда | Верно; можно добавить `alembic check` и round-trip миграций |

Остальное (13 модулей, 11 сервисов, 47 экранов, 4 роли) соответствует коду. Блок `<!--STATS-->` ведётся вручную: ни одного скрипта, который его обновляет, в `backend`, `frontend`, `deploy` нет (есть только `rt-ui/tools/check-readme.py`). Стоит завести один скрипт «собрать числа» для четырёх README.

### 2.4 `backend/README.md`

| Строка | Что написано | Правильно |
|---|---|---|
| 352 | `x-api-env` — «32 имени» | 42 (`docker-compose.yml`, блок `x-api-env`) |
| 354–378 | в таблице «Настройка» нет `AUDIT_HMAC_KEY`, `SETTINGS_ENCRYPTION_KEY`, `SMTP_HOST/PORT/USER/PASSWORD/FROM/STARTTLS`, `SIGNATURE_EXPOSE_DEBUG_OTP`, `BITRIX_SOURCE_ID` | Добавить (все есть в `x-api-env` и `.env.example`); `SETTINGS_ENCRYPTION_KEY` упомянут только в разделе «Безопасность» (:493) |
| 306 | «39 вызовов `include_router`» | 40 в `app/main.py` |
| 174–184 | пример `verify-chain` показывает `"ok":false` и «разрывы цепочки» как «честный результат» | Противоречит строке 502: причина найдена и исправлена (`clock_timestamp()` под замком, тест `test_audit_chain_concurrency.py`). Заменить примером с `"ok":true` или пометить как исторический |
| 509 | «Открытыми остаются: … построчные результаты и причина сбоя импорта …» | `GET /api/imports/{job_id}/rows` есть в `openapi.json`; убрать. Остаются верными: нет `DELETE` у праздников, аудит на каждый `POST /workflows/{id}/validate`, удаление сделки, архивация воронки целиком |
| 95–102 | пароли демо-учётки открытым текстом | Подходит, пока репозиторий приватный (см. открытый пункт про решение о публикации) |

### 2.5 `backend/loadtest/README.md`

| Строка | Что написано | Правильно |
|---|---|---|
| 47 | заголовок «Результаты (2026-09-18, эта сессия)» | Заменить на «Снимок 2026-09-18, до #13–#15»; слово «сессия» ничего не значит для читателя |
| 55–58 | «просто ещё не было проверено вживую до этого спринта» | Переформулировать в прошедшее время без ссылки на спринт |
| в целом | результаты выдаются за текущие | С тех пор изменились запись аудита (время под замком цепочки, хэш v2/v3), SLA-скан, права; ночной `loadtest.yml` существует, но опубликованных чисел нет. Нужен новый прогон или явная пометка «не переигрывался» |

### 2.6 `deploy/RUNBOOK.md`

| Строка | Что написано | Правильно |
|---|---|---|
| 3 | «см. корневой README» | Такого файла в репозитории нет. Дать ссылку `https://github.com/lct-testkit/.github#readme` |
| 242 | «см. «Устранение неполадок» в корневом README backend» | В `backend/README.md` такого раздела нет. Раздел есть в README продукта (`.github/README.md:641`). Заменить ссылку |
| 60 | «Демо-пользователи realm остаются» | Верно и остаётся (см. открытые пункты). Можно дать ссылку на TASKS |
| 168 | «проверено: 0 пользователей до обновления, 1 после» | Верно; не хватает процедуры сброса пароля, если `.env` менялся после первого старта (проблема #8 из DEPLOY-FIXES) |

### 2.7 `deploy/docs/REPO-SETTINGS.md`

| Строка | Что написано | Правильно |
|---|---|---|
| 12 | «Ruleset `protect-main` — все 4 репозитория» | Скрипт `apply_repo_settings.sh:27` по умолчанию: `backend frontend rt-ui deploy`. Репозиторий `.github` не покрыт |
| 30 | секрет `NODE_AUTH_TOKEN` во `frontend` | Секрета нет: `frontend/.github/workflows/ci.yml:32,75,143` подставляют `secrets.GITHUB_TOKEN`. Строку убрать (в тексте ниже это уже верно) |
| 36–55 | раздел «rt-ui → frontend (выполнено)» | Историческая справка; оставить одну строку и таблицу авторизации |
| 58 | «Выбрать лицензию (`frontend/README.md` прямо говорит, что её нет)» | `LICENSE` есть во всех репозиториях (`deploy/LICENSE`, `_wt-frontend-main/LICENSE`, `.github/LICENSE`, коммит «source available for hackathon judges»). Пункт выполнен |
| 56 | «Удалить устаревшую ветку бота `bot/update-images-35271171079`» | Ветка всё ещё в `origin` (`git branch -r`) — пункт открыт |

### 2.8 `deploy/docs/TASKS.md` — разметка по чекбоксам

Проверено по `origin/main` (857ae99) и известным фактам: v0.3.1 опубликован, развёрнут и проверен на сервере lct 26.09, диск 13%, предвыделение SeaweedFS выключено.

| Раздел | Пункт | Статус | Основание |
|---|---|---|---|
| Шапка | «Что осталось после ветки `feat/install-choices`» | **устарел** | Ветка влита в #13; переименовать в «Открытые задачи deploy». Таблицу «Что сделано в ветке» и абзац «Проверено на сервере» убрать: они отчёт, не задачи |
| 1 | Обновить на VM compose, Caddyfile, `.env.example`, `scripts/`, `seed/` | **выполнено** | v0.3.1 развёрнут на lct |
| 1 | Выполнить `set_host.sh --host lct.velikoss.ru --tls acme` | **вероятно выполнено, подтвердить** | развёртывание v0.3.1 прошло проверку; в репозитории следов нет |
| 1 | Администратор Keycloak в master realm, вход в консоль | **выполнено** | «Проверено на сервере» (0 → 1); `KEYCLOAK_ADMIN` в compose |
| 1 | Освободить место, том SeaweedFS ~22 ГБ | **выполнено** | диск 13%, флаг `-master.volumePreallocate=false` (`compose/docker-compose.yml:133`, чарт `seaweedfs-statefulset.yaml:56`) |
| 1 | Положить шрифты в `compose/fonts/` | **проверить на сервере** | в репозитории шрифтов быть не может |
| 1 | Решить судьбу demo на публичном адресе | **открыто (решение)** | см. п. 6 |
| 2 | `backend/deploy/Caddyfile` = `deploy/compose/Caddyfile` | **выполнено** | `diff` побайтно совпадает |
| 2 | `backend/docker-compose.yml`: `KEYCLOAK_ADMIN*` и `volumePreallocate=false` | **выполнено** | строки 177, 243–245 `_wt-backend-main/docker-compose.yml` |
| 3 | Прогнать `install.sh --tls acme` на свободной машине | **открыто** | нет свидетельств; косвенно ACME на lct работал |
| 3 | `set_host.sh` в публичных режимах | **открыто** | нет свидетельств |
| 4 | `e2e.yml`: добавить прогон `--tls internal` | **открыто** | `e2e.yml:135` — только `install.sh --profile demo` |
| 4 | автотесты `gen_env.sh`/`lib_host.sh` (bats) | **открыто** | тестов в репозитории нет |
| 4 | `sync_seed.sh --check` в `validate.yml` | **открыто** | в `validate.yml` не вызывается |
| 4 | helm lint/kubeconform для `seaweedfs-statefulset.yaml` | **выполнено** | PR #13 влит с зелёным CI (job `helm` в `validate.yml`) |
| 4 | `deploy.sh --seed` | **открыто** | `deploy.sh:51` принимает только `--profile\|--host\|--port-offset\|--tls\|--fonts` |
| 5 | шрифты в релизном бандле | **открыто (решение)** | `release.yml` без `--fonts` |
| 6 | demo-режим на публичном адресе | **открыто (решение)** | установщик только предупреждает (`bundle_install.sh:391`) |
| 6 | `set_host.sh` оставляет старые redirect URI | **открыто** | RUNBOOK:160 прямо так и говорит |
| 6 | S3-прокси https, нужна доверенность корневого сертификата | **описано** | RUNBOOK:147; задачей больше не является |
| 7 | пустые данные: дашборд, ЭДО, история импортов, 5 сделок на переходе в LMS | **открыто** | `seed/README.md` перечисляет только три этапа |
| 8 | алерт на заполнение диска | **открыто** | grep по `scripts`, `.github`, `compose` — ничего |
| 8 | Helm без TLS/шрифтов/сидов | **осознанное ограничение** | RUNBOOK:214; убрать из бэклога |
| 8 | демо-пользователи realm остаются в prod | **открыто** | RUNBOOK:60, 218 |

После чистки (выполненное убрать) остаётся 12–13 пунктов.

### 2.9 `architecture/README.md` — необязательное дополнение (вердикт KEEP)

Модель актуальна на 22.09 и не содержит: внешней системы Битрикс24 и исходящей доставки, заглушек `mock-lms`/`mock-cms`, контейнеров `ntp` и `sms-gateway-mock`, приёма вендоров/оплат/учащихся (импорт с 26.09), конвейера deploy/GHCR. Текст README об этом не врёт (перечислены только пять узлов технологического слоя). Если нужен полный Archi для жюри — дополнить модель.

---

## 3. Открытые пункты (реально нерешённое на 2026-09-27)

Важность: В — высокая, С — средняя, Н — низкая.

| Суть | Тип | Источник | Доказательство | Важность |
|---|---|---|---|---|
| Deploy не пробрасывает в api-контейнеры 11 переменных бэкенда: `AUDIT_HMAC_KEY`, `SETTINGS_ENCRYPTION_KEY`, `SMTP_HOST/PORT/USER/PASSWORD/FROM/STARTTLS`, `SIGNATURE_EXPOSE_DEBUG_OTP`, `ALLOWED_FILE_EXTENSIONS`, `BITRIX_SOURCE_ID`. На развёрнутых стендах HMAC аудита (хэш v3), почта подписантам и шифрование системных настроек выключены | GAP | `deploy/compose/docker-compose.yml:27-61` («скопирован без изменений»); `backend/README.md:352,493` | `x-api-env`: deploy 31 имя, backend main 42; `git grep` по `compose`, `scripts`, `charts` на `AUDIT_HMAC_KEY` пуст; `check_drift.py` этот список не сверяет | В |
| В деплое и в документах бэкенда не описано включение SMTP/HMAC аудита (нет в таблице «Настройка», нет в RUNBOOK и `gen_env.sh`) | GAP | `backend/README.md:354-378`; `deploy/RUNBOOK.md` | grep `SMTP`, `AUDIT_HMAC` в RUNBOOK пуст | С |
| Обязательные результаты хакатона: презентация pptx/pdf (§7) и сопроводительная документация .doc/.pdf (§10.4). В рабочей папке таких файлов нет | GAP / DECISION | `rtk_requiriments.md:151-153,206-211` | `find` по `D:\testkit-lct` (глубина 3, без node_modules) — ни `.pptx`, `.pdf`, `.doc`, `.docx`. Возможно, лежат вне папки — уточнить у владельца | В |
| Файлы `SECURITY.md` (backend, deploy, frontend) отсылают к «контактам в профиле организации», а `profile/README.md` ни контакта, ни ссылки на приватную отправку уязвимости не содержит. В `rt-ui` файла `SECURITY.md` нет | GAP | `backend/SECURITY.md:4`; `deploy/SECURITY.md:4` | grep `mail`, `@`, `контакт`, `advisor` в `.github/profile/README.md` — пусто | С |
| Демо-пароли и демо-учётки открытым текстом в `backend/README.md` и корневом README; демо-режим на публичном домене отдаёт демо-логины и секрет клиента `crm-bff` (`/config.json`). Решение — что публиковать и на lct.velikoss.ru | DECISION | `backend/README.md:95-102`; `deploy/docs/TASKS.md` §6; `RUNBOOK.md:176` | В `.github/README.md` пароли уже убраны; в `backend/README.md` остались. Профиль lct (demo/prod) неизвестен, проверить | С |
| Демо-пользователи realm остаются и в prod (`gen_env.sh` только предупреждает) | GAP | `TASKS.md` §8; `RUNBOOK.md:60,218` | В RUNBOOK «Остаётся вручную: удалить/отключить демо-учётки» | С |
| `set_host.sh` не удаляет старые `redirect_uris`/`web_origins` прежнего хоста | GAP | `TASKS.md` §6; `RUNBOOK.md:160` | Прямо указано в RUNBOOK | Н |
| Шрифты Rostelecom Basis не попадают в релизный бандл (нужна `--fonts` при установке) | DECISION | `TASKS.md` §5; `RUNBOOK.md:164` | `release.yml` не вызывает `--fonts` | Н |
| Нагрузочный тест не переигрывался после #13–#15 (изменена запись аудита, хэш v2/v3); в README цифры от 18.09; целевые 50 RPS на переходе не достигнуты (34,4 RPS) | LIMITATION / OPS | `backend/loadtest/README.md:47-70`; `.github/README.md:392,517` | Результаты датированы 18.09; ночной `loadtest.yml` есть, публикуемых итогов нет | С |
| Праздники производственного календаря без ручки `DELETE` | GAP | `README.md:657`; `backend/README.md:509` | В `openapi.json` у `/api/holidays` только GET/POST/PATCH | Н |
| Сплошной аудит: каждый `POST /workflows/{id}/validate` пишет запись аудита; удаления сделки нет; архивации воронки целиком нет; приглашение в закрытом контуре не даёт войти (по `backend-issues.md` C-18; проверить актуальность) | GAP | `backend/README.md:509`; `frontend/docs/backend-issues.md` | `service.py` workflow вызывает `_audit.record` в `validate`; в `openapi.json` нет `DELETE /api/deals/{id}` | Н |
| Входящий поток Битрикс24 — упрощённый собственный контракт с HMAC, а не события портала (`event.bind`) | LIMITATION | `README.md:167`; `backend/app/modules/integration/bitrix.py:46` | Код прямо описывает ограничение | Н |
| Антивирус — заглушка `NullAntivirusScanner`, статус `quarantined` не присваивается. В README продукта это подано как «антивирусная проверка (карантин)» | LIMITATION / BUG (в документе) | `backend/README.md:485`; `.github/README.md:382` | `files/service.py:215`; grep `quarantin` в `app/` находит только модель | С |
| Автоподстановка по ИНН без реального внешнего провайдера (только локальный реестр, подтверждённые организации, мок; в prod мок отключён). Внешний провайдер `dadata.py` есть только в чужом незакоммиченном дереве | LIMITATION | `backend/README.md:214,506` | `external_org_lookup_enabled=False` (`config.py:153`); на main нет `dadata.py` | Н |
| RPO 15 минут недостижим без WAL-архивирования: суточный дамп даёт RPO сутки | LIMITATION | `RUNBOOK.md:202` | Не реализовано в репозитории, в самом RUNBOOK так и сказано | С |
| Helm-путь экспериментальный: реальная установка в CI не гоняется; порядок хуков `migrate`/`seed`; у workload'ов нет `securityContext` (только у `ntp`) | LIMITATION | `RUNBOOK.md:214` | `git grep securityContext charts` — единственное совпадение `ntp-deployment.yaml:59` | С |
| Нет Prometheus/Grafana и алерта на заполнение диска: инцидент `No space left on device` проявился как 500 в API | OPS | `RUNBOOK.md:227`; `TASKS.md` §8 | Только эндпоинты `/metrics` | С |
| ACME «с нуля» на свободной машине и `set_host.sh` в публичных режимах не проверены; `e2e.yml` не гоняет `--tls internal` | OPS | `TASKS.md` §3, §4 | `e2e.yml:135` — только `--profile demo` | С |
| Нет автотестов `gen_env.sh`/`lib_host.sh`, `sync_seed.sh --check` не в CI, `deploy.sh --seed` не реализован, `e2e_stack.sh` не запускает сиды | GAP | `TASKS.md` §4; `DEPLOY-CONTRACT.md` §6 | grep по `origin/main`: `deploy.sh` без `--seed`; в `e2e_stack.sh` нет `seed_demo` | Н |
| Сиды не заполняют: дашборд «Обзор воронки», соглашения ЭДО, историю импортов; пять сделок стоят на переходе «Передать материалы в LMS» без `LMS_BASE_URL` | GAP | `TASKS.md` §7 | `seed/README.md` описывает только три этапа | Н |
| Не реализованы из DEPLOY-FIXES: `--email` для ACME (нет `ACME_EMAIL`), `--yes-i-know-demo`, процедура сброса пароля админа Keycloak (`kc.sh bootstrap-admin user`) | GAP | `DEPLOY-FIXES.md:31,188,190-191` | `git grep` по `origin/main`: `ACME_EMAIL`, `bootstrap-admin`, `yes-i-know` — пусто | Н |
| Ручные шаги владельца репозиториев: ветка бота `bot/update-images-35271171079` не удалена; ruleset не применён к `.github`; `require_code_owner_review` выключен | OPS | `REPO-SETTINGS.md:56-59,12`; `apply_repo_settings.sh:27` | `git branch -r` показывает ветку; список `REPOS` без `.github` | Н |
| Тег `v0.3.0` (неудачный релиз) остаётся в репозитории рядом с рабочим `v0.3.1` | OPS | (состояние deploy) | `git tag`: v0.1.0, v0.2.0, v0.3.0, v0.3.1 | Н |
| Каталог `/dev/blocks` (`frontend/src/routes/dev`) по плану удалить перед сдачей | LIMITATION | `README.md:660`; `frontend/README.md:32,200` | `src/routes/dev` существует; в prod-сборку не попадает | Н |
| 28 из 203 ручек без экрана: 8 умышленно и 20 без UI (лицензии вуз-вендор-ПО, `DELETE` справочников, повтор доставки, предпросмотр шаблонов, `PATCH /me` и др.) | GAP | `frontend/docs/STATUS.md`; `.github/README.md:666` | Цифра 175/203 из `STATUS.md` (перегенерировано 26.09) | Н |
| Долг типизации: 20 модулей в списке исключений mypy («список не должен расти») | LIMITATION | `backend/README.md:419` | 20 модулей в `[[tool.mypy.overrides]]` | Н |
| Статистика (`<!--STATS-->`) расходится между тремя копиями README и профилем организации; нет скрипта, который собирает числа из кода | GAP (процесс) | `.github/profile/README.md:9-11`, `.github/README.md:9-11`, корневой `README.md:9-11` | Три разных набора чисел; скрипта нет | С |

---

## 4. Проверки, которыми подтверждены числа

- `openapi.json` (backend main): 203 операции, 162 пути, 30 тегов, 241 схема, схемы `BearerAuth`, `CsrfToken`, `SessionCookie`.
- Миграции: `0001_baseline … 0020_signature_unique_request` (20 файлов).
- Тесты: `pytest --collect-only -q` (без записи кэша) — 2143 теста, 83 файла (README не врёт).
- Таблицы: 67 `__tablename__`. Права: 40. Cron-задач `arq`: 17. Контрактов import-linter: 16. Исключений mypy: 20.
- Шаблоны отчётов: 9.
- Compose бэкенда: 11 сервисов; образы postgres 16, redis 7, seaweedfs 3.68, keycloak 25.0, caddy 2.11, ntp.
- Frontend main: 13 глав справки, 40 сценариев в `tools/scenarios`, 36 тестовых файлов.
- Deploy: `Caddyfile` совпадает с `backend/deploy/Caddyfile`; в `compose/Caddyfile` нет `disable_redirects`, есть `handle /fonts/*`, `@app_csp not path /auth/*`, ограничение консоли Keycloak `private_ranges`; `noeviction` в Redis и `127.0.0.1:5433` у Postgres в compose бэкенда и deploy.
- Ссылки: все картинки из `.github/README.md` и `profile/README.md` существуют в frontend main и `.github/profile/img`.
- Контроль изменений: в ходе аудита ничего не изменено; проверка `git status` в `_wt-backend-main` — чисто.
