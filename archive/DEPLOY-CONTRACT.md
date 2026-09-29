# Контракт правок deploy: публичный домен, TLS, демо с сидами

Источник задачи: установка офлайн-бандла v0.2.0 на lct.velikoss.ru вручную упёрлась в 9 проблем
(см. `D:\testkit-lct\DEPLOY-FIXES.md`, таблица «Что сломалось и почему», пункты 1–9), плюс выяснилось ещё две:

- **#10 Keycloak-админа не существует.** В `compose/docker-compose.yml` для keycloak заданы `KC_BOOTSTRAP_ADMIN_USERNAME/PASSWORD`,
  а пинится Keycloak **25.0.x** (images.yaml). Эти переменные появились только в Keycloak 26 и в 25.0.6 игнорируются:
  в realm `master` 0 пользователей, войти в консоль администратора нельзя (подтверждено на сервере). Для 25.x нужны
  `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD`. То же самое, вероятно, в `backend/docker-compose.yml`.
- **#11 Сиды.** Контейнер `seed` (образ api) засевает только справочники (воронки, шаблоны отчётов и уведомлений, источники интеграций).
  Организаций, контактов, сделок, задач, ЕГРЮЛ, SLA, праздников, направлений, причин отказа, кастомных полей и сценариев
  «Удаление ПДн» нет. Всё это делают три node-скрипта в репозитории frontend, в поставку они не входят:
  `frontend/tools/seed-demo.mjs` (организации, контакты, сделки, комментарии, задачи, участники, реестр ЕГРЮЛ, SLA-правила),
  `frontend/tools/scenarios/b-seed-catalogs.mjs` (направления, причины отказа, **праздники 2026**, кастомные поля сделки),
  `frontend/tools/scenarios/lead-seed-erasure.mjs` (запросы на удаление ПДн и согласования «четыре глаза»).
  Все три идемпотентны и ходят в API по Bearer-токену демо-учёток (пароль через Keycloak, `client_id=crm-bff` + секрет из
  `/config.json`). Общий хелпер `frontend/tools/lib.mjs` тянет `playwright` (в контейнере его нет): нужен урезанный `lib.mjs`.

Цель: **`install.sh` явно спрашивает, как и на какой домен ставим, ставит рабочий HTTPS-стенд, а для демо-профиля
сразу заливает все демо-данные.** Ручная правка Caddyfile/compose/.env/Keycloak после установки не нужна.

Общие правила для всех, кто правит файлы:

- Правим только рабочее дерево. **Никаких git commit/push/checkout/stash/reset.** Пользователь ревьюит diff сам.
- Стиль как у соседнего кода: комментарии по-русски и «почему», а не «что»; `set -euo pipefail`; bash ≥ 4 (на Debian/Ubuntu),
  но `install.sh`, `gen_env.sh`, `lib_host.sh`, `set_host.sh`, `seed_demo.sh`, `smoke.sh` едут в офлайн-бандл и **не должны требовать
  ничего кроме bash, sed, grep, awk, tr, head, curl, docker** (никакого python/jq/openssl на целевой машине).
  Скрипты, которые не едут в бандл (build_offline_bundle.sh, check_drift.py, sync_seed.sh), могут использовать python.
- Файлы пишем с LF-переводами строк (в репо `.gitattributes`). Локальная среда: Windows + Git Bash, Docker Desktop (linux engine),
  Python 3.11 с PyYAML, node v26, yamllint. shellcheck/actionlint/caddy — только через docker-образы, как в CI:
  `koalaman/shellcheck-alpine:v0.10.0`, `rhysd/actionlint:1.7.7`, `docker.io/library/caddy:2.11-alpine`.
- Обратная совместимость обязательна: **режим `off` (по умолчанию) работает ровно как раньше** (порты 8080/8443/8333/5433,
  `http://localhost:8080`, самоподписанный `:8443`), потому что на нём держатся `e2e_stack.sh`, `deploy.sh` (dev/demo/prod на одной машине через
  `--port-offset`), `backend/docker-compose.yml` (локальная разработка) и CI.
- Не выводить секреты в лог/summary. Демо-пароли в realm публичны, их можно упоминать.

## 1. Режимы TLS

`TLS_MODE` ∈ `off | internal | acme`:

| режим | назначение | URL | Caddy TLS | публикуемые порты хоста |
|---|---|---|---|---|
| `off` (default) | локально, e2e, тест, dev/demo на одной VM | `http://HOST:HTTP_PORT` | сайт `HOST:8443` c `tls internal` (как сейчас) | `HTTP_PORT`→8080 (0.0.0.0), `HTTPS_PORT`→8443, `S3_PROXY_PORT`→8333 (http) |
| `internal` | закрытый контур, IP или внутреннее имя: HTTPS на 443, самоподписанный | `https://HOST` | `tls internal` на 443 | 80 (редирект), 443 (+udp), 8333 (https), 8080 только 127.0.0.1 |
| `acme` | публичный домен, Let's Encrypt | `https://HOST` | автоматический ACME на 443 (HTTP-01 через 80) | то же, что `internal` |

Порт-офсет (`--port-offset`) допустим только в `off`; в `internal`/`acme` порты 80/443/8333 фиксированы, сочетание с офсетом — ошибка.

Дефолт TLS-режима: `gen_env.sh` без `--tls` → **`off`** (ничего не меняется для deploy.sh/e2e). **Установщик** `install.sh` без `--tls` при заданном
`--host` выводит режим функцией `rtk_tls_default_for_host` (ниже) и всегда печатает выбранное в сводке.

### `rtk_tls_default_for_host <host>` (scripts/lib_host.sh; печатает `off|internal|acme`)
- пусто, `localhost`, `127.*`, `::1` → `off`
- IPv4/IPv6-литерал → `internal` (для IP Let's Encrypt не выдаёт)
- имя из одной метки либо суффикс `.local .lan .internal .intranet .corp .home .test .example .invalid .localhost .localdomain` → `internal`
- иначе (обычное FQDN) → `acme`

## 2. Контракт переменных `.env` (пишет gen_env.sh / set_host.sh, читают compose и Caddy)

Все переменные обязаны иметь дефолты в compose (`${VAR:-...}`), чтобы `docker compose --env-file compose/.env.example config` (CI) проходил.

| переменная | off | internal / acme | смысл |
|---|---|---|---|
| `TLS_MODE` | `off` | `internal` / `acme` | для скриптов (smoke, set_host, installer); compose её не использует |
| `CRM_TLS_HOST` | `HOST` или `localhost` | `HOST` | имя TLS-сайта Caddy (SNI/ACME) |
| `CRM_TLS_PORT` | `8443` | `443` | порт TLS-сайта Caddy **внутри** контейнера |
| `CRM_TLS_SNIPPET` | `tls_internal` | `tls_internal` / `tls_acme` | имя snippet'а Caddyfile, импортируемого в TLS-сайт |
| `S3_SITE_ADDRESS` | `:8333` | `HOST:8333` | адрес сайта S3-прокси в Caddyfile |
| `S3_TLS_SNIPPET` | `tls_none` | `tls_internal` / `tls_acme` | TLS S3-сайта (в `off` S3 остаётся plain http) |
| `HTTP_BIND` | `0.0.0.0` | `127.0.0.1` | на каком адресе хоста публикуется plain-http `:8080` (Docker обходит ufw, наружу в публичных режимах он не нужен) |
| `HTTP_PORT` | `8080`+офсет | `8080` | порт хоста для plain-http |
| `HTTPS_PORT` | `8443`+офсет | `443` | порт хоста для TLS (→ `CRM_TLS_PORT` контейнера, tcp и udp) |
| `CRM_PORT80_PUBLISH` | `127.0.0.1::80` | `80:80` | **целиком** спецификация публикации контейнерного :80 (ACME HTTP-01 + редирект на https); `127.0.0.1::80` = эфемерный порт на loopback, т.е. «наружу не публикуем» |
| `S3_PROXY_PORT` | `8333`+офсет | `8333` | порт хоста S3-прокси |
| `BASE_URL` | `http://HOST:HTTP_PORT` | `https://HOST` | публичный адрес приложения |
| `KEYCLOAK_URL`, `KEYCLOAK_PUBLIC_URL` | `BASE_URL/auth` | `https://HOST/auth` | issuer токенов; **обе** переменные (сейчас `KEYCLOAK_URL` остаётся localhost) |
| `S3_PUBLIC_ENDPOINT_URL` | `http://HOST:S3_PROXY_PORT` | `https://HOST:8333` | presigned-ссылки; https обязателен (страница на https, иначе mixed content) |
| `FONTS_DIR` | не задан | не задан | абсолютный путь к каталогу шрифтов; задаёт `gen_env.sh --fonts DIR` (копирует `*.woff`/`*.woff2` в `runtime/fonts`); по умолчанию compose берёт `./fonts` |

Дополнительно gen_env.sh пишет `KEYCLOAK_ADMIN=admin` (уже есть в example) и случайный `KEYCLOAK_ADMIN_PASSWORD` (уже делает).

### Realm (`runtime/keycloak/realm-crm.json`)
Сейчас `sed` подменяет только подстроки `//localhost:8080` и `//localhost:8443`, для https на 443 получаются неверные URI, а после первого импорта
realm правка файла ничего не меняет. Правило нового рендера:

- `off`: как сейчас (`//localhost:8080` → `//HOST:HTTP_PORT`, `//localhost:8443` → `//HOST:HTTPS_PORT`).
- `internal`/`acme`: `http://localhost:8080/*`, `https://localhost:8443/*` **остаются** (доступ с самого сервера), и к `redirectUris`, `webOrigins`, `post.logout.redirect.uris`
  клиента `crm-bff` **добавляется** публичный `https://HOST/*` (`https://HOST` для webOrigins). JSON остаётся валидным; дублей нет.
- Проверка валидности JSON: скрипты бандла без python, поэтому проверять структурно (grep наличия URI); тесты агентов могут использовать python.
- В prod-профиле `http://localhost:5173/*` (Vite) из realm убирается.

`scripts/set_host.sh` меняет хост **у уже установленного** стенда (realm уже импортирован): правит `.env`, перерисовывает `runtime/keycloak/realm-crm.json`,
и **прописывает URI напрямую в БД Keycloak** (`docker compose exec -T postgres psql -U "$POSTGRES_USER" -d keycloak`):
`redirect_uris`, `web_origins` (по `client_id`, найденному через `client` по `client_id='crm-bff'` в realm `crm`),
`client_attributes` (`post.logout.redirect.uris`), затем `docker compose up -d` (пересоздать caddy/api/worker/web с новым env) и `restart keycloak`
(кэш Keycloak). Так делалось вручную на lct.velikoss.ru и сработало.

## 3. Caddyfile (`deploy/compose/Caddyfile` и **побайтно идентичная копия** `backend/deploy/Caddyfile`; сверяет `check_drift.py --backend-dir`)

Требования:
- Убрать `auto_https disable_redirects` (иначе нет редиректа 80→443 и странная семантика; проверено на сервере: без неё стек здоров).
- Snippet'ы `(tls_internal) { tls internal }`, `(tls_acme)` (явный ACME-issuer), `(tls_none)` (пустой; проверить, что пустой snippet допустим, иначе придумать нейтральную замену).
- TLS-сайт: `{$CRM_TLS_HOST:localhost}:{$CRM_TLS_PORT:8443} { import {$CRM_TLS_SNIPPET:tls_internal}  import crm_routes }`
- S3-сайт: `{$S3_SITE_ADDRESS::8333} { import {$S3_TLS_SNIPPET:tls_none}  reverse_proxy seaweedfs:8333 }`
- `:8080 { import crm_routes }` без изменений (healthcheck и loopback).
- В `crm_routes` перед финальным `handle { reverse_proxy web }` добавить
  `handle /fonts/* { header Cache-Control "no-cache"  root * /srv  file_server }` — шрифты Rostelecom Basis лицензионные, в git и образ `web` не входят
  (`frontend/static/fonts` в .gitignore, выдаёт заказчик); без этого блока SPA-fallback отдаёт `index.html` с кодом 200 вместо шрифта и браузер пишет
  «downloadable font: rejected by sanitizer». С блоком отсутствующий шрифт даёт честный 404 и запасную гарнитуру.
- **Проверить по факту (`caddy adapt`/`validate` из образа caddy 2.11)**: семантику `{$VAR:default}` когда VAR *не задана* и когда задана *пустой*
  (compose всегда передаёт непустые значения, но убедиться); что при значениях по умолчанию (переменные не заданы) адаптированный конфиг
  эквивалентен старому Caddyfile (кроме удалённой disable_redirects) — это гарантия, что backend-dev не сломается; что для трёх режимов
  (off / internal на 443 / acme на 443) адаптация проходит и listen/tls-политики те, что нужны.
- CSP и остальные заголовки не менять. CSP блокирует inline-скрипты страницы входа Keycloak (`importmap` + inline module): для `/auth/*` заменить на
  ослабленный `script-src 'self' 'unsafe-inline'` в отдельном `handle /auth/*` (заголовок только для этого пути), остальная политика та же.

## 4. compose (`deploy/compose/docker-compose.yml`)

- Сервис `caddy`, ports:
  ```
  - "${HTTP_BIND:-0.0.0.0}:${HTTP_PORT:-8080}:8080"
  - "${CRM_PORT80_PUBLISH:-127.0.0.1::80}"
  - "${HTTPS_PORT:-8443}:${CRM_TLS_PORT:-8443}"
  - "${HTTPS_PORT:-8443}:${CRM_TLS_PORT:-8443}/udp"
  - "${S3_PROXY_PORT:-8333}:8333"
  ```
  environment добавить (все с непустыми дефолтами): `CRM_TLS_PORT`, `CRM_TLS_SNIPPET`, `S3_SITE_ADDRESS`, `S3_TLS_SNIPPET`.
  volumes добавить `${FONTS_DIR:-./fonts}:/srv/fonts:ro`.
  Каталог `deploy/compose/fonts/` создаётся с `.gitkeep`; `.gitignore` deploy: `compose/fonts/*` кроме `.gitkeep`
  (шрифты лицензионные, в репозиторий не коммитим).
  Проверить: синтаксис `127.0.0.1::80` принимает `docker compose config`.
- Сервис `keycloak`: к `KC_BOOTSTRAP_ADMIN_*` **добавить** `KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN:-admin}` и
  `KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD:-admin}` (для 25.x), старые оставить (для апгрейда до 26) с комментарием почему. Проверить на
  сервере нельзя из агента; проверить по документации/исходникам образа: `docker run --rm --entrypoint sh <keycloak:25.0> ...` не запускает создание админа, поэтому
  достаточно сослаться на Keycloak 25 docs (env `KEYCLOAK_ADMIN`, `KEYCLOAK_ADMIN_PASSWORD`, создаётся при первом старте с пустым master realm).
  Проверить и исправить то же в `backend/docker-compose.yml` (минимальная правка, дефолты сохранить).
- Новый сервис `seed-demo` (профиль `demo-data`), образ `${NODE_IMAGE:?выполните scripts/render_env_images.py}`:
  ```yaml
  seed-demo:
    profiles: ["demo-data"]
    image: ${NODE_IMAGE:?...}
    working_dir: /seed
    entrypoint: ["sh", "/seed/run.sh"]
    environment:
      APP_URL: http://caddy:8080
    volumes:
      - ../seed:/seed:ro
    restart: "no"
    read_only: true
    user: node
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    logging: *logging
    networks: [crm]
  ```
  (Без `depends_on`: запускается через `run --rm --no-deps` уже после `up --wait`.) Сиды ходят внутри сети compose на `http://caddy:8080`:
  никакого DNS/TLS снаружи, токены с issuer'ом публичного URL валидны (Keycloak `KC_HOSTNAME_URL` фиксирован).
- `images.yaml`: `external.node`: `ref: docker.io/library/node:24-alpine`, `digest: "sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1"`
  (проверить, что это digest индекса, `docker buildx imagetools inspect`), с комментарием «только профиль demo-data (сиды)»; `profiles.demo-data: [seed-demo]`.
  Образ автоматически попадает в офлайн-бандл (`render_env_images.py --list`); убедиться в этом и в том, что `validate_images.py`, `render_env_images.py`,
  `check_drift.py` (для чарта `node` нужно исключение, как у `local-registry`) проходят.
- `.github/workflows/validate.yml`, job `compose`: добавить `--profile demo-data` к `config --quiet`.
- `compose/.env.example`: все новые переменные из таблицы выше с комментариями (значения режима `off`), плюс описание `FONTS_DIR`.

## 5. Установщик (`scripts/bundle_install.sh` → `install.sh` в бандле)

```
bash install.sh [--host H] [--tls acme|internal|off] [--profile demo|prod] [--seed|--no-seed]
                [--fonts DIR] [--yes] [--reconfigure] [--open-firewall] [--skip-dns-check]
                [--registry] [--no-start]
```
- Интерактив (stdin и stdout — TTY и нет `--yes`): последовательно спрашивает и **всегда** показывает итоговую сводку с подтверждением:
  1. домен/IP (дефолт: `--host` или `localhost`),
  2. схему TLS: меню acme / internal / off с пояснениями (дефолт — `rtk_tls_default_for_host`),
  3. профиль demo/prod (дефолт demo),
  4. для demo: залить демо-данные? (дефолт Y),
  5. каталог со шрифтами Rostelecom Basis `*.woff` (необязательно; дефолт `./fonts` бандла, если там есть файлы).
  Все вопросы пропускаются, если значение передано флагом. Ввод валидируется (хост: `[A-Za-z0-9.-]` или IP; повтор вопроса при ошибке).
- Неинтерактив: берёт флаги и дефолты; без `--host` = `localhost` и режим `off` (поведение CI `bash install.sh --profile demo` не меняется) с явной строкой
  «домен не задан — ставлю на localhost без TLS»; `--seed` по умолчанию включён для `demo`, `--no-seed` отключает; `--seed` с `prod` — ошибка.
- Предполётные проверки — **до** `docker load` (он долгий): docker/compose; свободное место; для `internal`/`acme`: порты 80, 443, 8333 не заняты (`ss -ltn`; если `ss` нет — пропустить с
  предупреждением), для `acme`: имя резолвится (`getent hosts`) и указывает на внешний IP машины (`curl -m5 https://ifconfig.me`; расхождение — предупреждение, неразрешимость — ошибка, обход `--skip-dns-check`),
  доступность `acme-v02.api.letsencrypt.org:443` (предупреждение «нет выхода в интернет — ACME не сработает, используйте --tls internal»); ufw активен и
  не разрешает 80/443/8333 → напечатать точные команды или выполнить их при `--open-firewall` (в интерактиве спросить).
- Если `compose/.env` уже есть: не перегенерировать; сравнить желаемые host/tls с сохранёнными и при расхождении остановиться с понятным сообщением
  («запустите `bash scripts/set_host.sh --host … --tls …` или `install.sh --reconfigure`»); `--reconfigure` вызывает `set_host.sh`.
- Порядок: сводка → SHA256SUMS → docker load → `gen_env.sh --dir compose --profile P --host H --tls M [--fonts DIR]` → (registry) → `up -d --wait --pull never` →
  smoke по публичному URL (`--insecure` для internal) → если seed: `scripts/seed_demo.sh` → итоговая печать: URL, режим TLS, что сидировано, для demo-профиля
  на публичном хосте **заметное предупреждение** (экран входа и `/config.json` публикуют демо-логины и client secret; удалите демо-учётки/переведите на prod до открытия наружу).
  Ошибка сидов не откатывает установку, но код возврата ненулевой и печатается команда для повтора `bash scripts/seed_demo.sh`.
- Шаги нумеровать заново (например `1/7`), сохранив стиль вывода.
- `--help` печатает актуальный заголовок (сейчас `sed -n '2,15p'`: пересчитать диапазон).

`scripts/build_offline_bundle.sh`: копировать в бандл `seed/`, `compose/fonts/.gitkeep`, `scripts/{lib_host.sh,set_host.sh,seed_demo.sh}`;
необязательный `--fonts DIR` (положить шрифты в бандл `compose/fonts/`); обновить проверку обязательных файлов, список «что внутри», README бандла
(примеры с `--host/--tls`, сиды, шрифты). Тег образа node попадает в бандл автоматически из images.yaml.

## 6. Сиды

- `deploy/seed/` (в бандле — `seed/` рядом с `compose/`): `lib.mjs` (урезанный: `BASE` из `APP_URL`, `ACCOUNTS`, `accessToken`, `ensureConsent` — без playwright, поведение как в
  `frontend/tools/lib.mjs`), `seed-demo.mjs`, `catalogs.mjs` (= b-seed-catalogs), `erasure.mjs` (= lead-seed-erasure), `run.sh`, `README.md`.
  В скриптах меняется **только** строка импорта хелпера; остальной код побайтно как в frontend. Файлы хранят шапку-комментарий «копия frontend/tools/…, обновлять `scripts/sync_seed.sh`».
- `seed/run.sh` (POSIX sh; запускается в `node:24-alpine`): порядок **catalogs → demo → erasure**, `set -e`, перед стартом ждёт готовности API (`GET $APP_URL/health/ready`, до ~120 с),
  опциональный `SEED_ONLY=catalogs,demo,erasure`; печатает границы этапов; итоговый код ≠0 при падении любого этапа. Мягкие предупреждения скриптов (`!`-строки, например невыполненный
  переход в LMS без `LMS_BASE_URL`) код возврата не меняют.
- `scripts/seed_demo.sh [--env-file F] [--images-env F] [--project P] [--compose FILE] [--only a,b]` (bash+docker): по умолчанию `compose/.env` и `compose/.env.images`, проект по умолчанию
  `rtk-crm`; запускает `docker compose … --profile demo-data run --rm --no-deps seed-demo`; проверяет, что стек поднят (`ps --status running api`), профиль ≠ prod
  (читает `APP_PROFILE` из env-файла; для prod — отказ), печатает итог. Идемпотентен (безопасно запускать повторно).
- `scripts/sync_seed.sh --frontend-dir ../frontend [--check]`: копирует три скрипта из `frontend/tools` в `seed/` с заменой импорта; с `--check` только сравнивает и падает при расхождении
  (вызывать в CI не обязательно, но скрипт должен работать; проходит shellcheck).
- `scripts/deploy.sh`: `--init` принимает `--tls`, `--fonts`; новый флаг `--seed` (после успешного деплоя запускает `seed_demo.sh` с `STATE_DIR`-файлами; для `RTK_ENV=prod` — отказ).
- `scripts/e2e_stack.sh`: после smoke запускать сиды (`seed_demo.sh --project rtk-e2e ...`) и проверять, что сделки появились (API с Bearer-токеном демо-админа либо прямой запрос к postgres контейнеру: `select count(*) from deals` > 0).
- `scripts/smoke.sh`: `--insecure` (curl `-k`), предупреждающая (не проваливающая) проверка шрифта `/fonts/RostelecomBasis-Regular.woff` (тип `font/*`), проверка редиректа http→https не нужна.
- `.github/workflows/e2e.yml`: в `paths` добавить `seed/**`; шаг офлайн-установки — `bash install.sh --profile demo --yes` (сиды по умолчанию включены, это и есть проверка «сиды работают без сети»).
- Проверка сидов локально: `node --check` для каждого файла (ESM), `sh -n seed/run.sh`; уверенность в паритете с frontend даёт `sync_seed.sh --check`.

## 7. Документация

`deploy/RUNBOOK.md` (разделы: «Домен и TLS (acme / internal / off)», «Демо-развёртка с данными», «Шрифты Rostelecom Basis», «Смена домена после установки (set_host.sh)»,
«Пароль администратора Keycloak (25.x)», «Порты и файрвол: Docker обходит ufw»), `deploy/README.md`, `deploy/compose/README.md` (таблица портов по режимам, новые переменные, seed-demo),
`deploy/docs/` при необходимости (новый файл допустим). Русский язык. Примеры команд — рабочие и совпадающие с реальными флагами скриптов.
Отдельно: в `DEPLOY-FIXES.md` (корень, вне репозиториев) допускается пометить пункты как исправленные.

## 8. Владельцы файлов (пересечений нет)

| владелец | файлы |
|---|---|
| **caddy-compose** | `deploy/compose/Caddyfile`, `backend/deploy/Caddyfile`, `backend/docker-compose.yml` (только admin-env), `deploy/compose/docker-compose.yml`, `deploy/compose/.env.example`, `deploy/compose/fonts/.gitkeep`, `deploy/.gitignore`, `deploy/images.yaml`, `deploy/scripts/check_drift.py`, `deploy/scripts/validate_images.py`/`render_env_images.py` (если нужно), `deploy/.github/workflows/validate.yml` |
| **genenv** | `deploy/scripts/lib_host.sh` (новый), `deploy/scripts/gen_env.sh`, `deploy/scripts/set_host.sh` (новый) |
| **installer** | `deploy/scripts/bundle_install.sh`, `deploy/scripts/build_offline_bundle.sh` |
| **seeds** | `deploy/seed/*` (новые), `deploy/scripts/seed_demo.sh`, `deploy/scripts/sync_seed.sh` (новые), `deploy/scripts/smoke.sh`, `deploy/scripts/deploy.sh`, `deploy/scripts/e2e_stack.sh`, `deploy/.github/workflows/e2e.yml` |
| **docs** | `deploy/RUNBOOK.md`, `deploy/README.md`, `deploy/compose/README.md`, `deploy/docs/*`, `D:\testkit-lct\DEPLOY-FIXES.md` |

Соседний файл чужого владельца менять нельзя: если нужна правка, опишите её в отчёте («нужно от <владелец>»).
Чарт (`deploy/charts/`) не трогаем: он живёт на ingress'е Kubernetes, отдельная схема TLS.

## 9. Критерии готовности (проверяются в конце)

1. CI-эквиваленты локально зелёные: `yamllint --strict -c .yamllint.yml .`; shellcheck `-x -S warning scripts/*.sh` (docker-образ как в CI);
   `python scripts/validate_images.py images.yaml`; `python scripts/check_drift.py --backend-dir ../backend`;
   `docker compose config --quiet` с `--profile integrations --profile registry --profile demo-data` для `.env.example` и для сгенерированных `.env` всех трёх режимов;
   «без .env.images compose обязан падать»; actionlint по workflow.
2. `gen_env.sh` в каждом режиме даёт согласованный `.env` и realm; повторный прогон/`--force` работают; `--port-offset` + публичный режим отвергается.
3. `install.sh --help`, неинтерактивный прогон без docker (mock `docker` в PATH) доходит до предполётных проверок и осмысленно падает/проходит; интерактивный сценарий проверен через `script`/pty или подачей stdin с `--yes` отключённым.
4. Caddyfile в трёх режимах адаптируется; в дефолтном равен старому по смыслу.
5. Все 11 проблем (#1–#11) имеют явное закрытие и ссылку в документации.
