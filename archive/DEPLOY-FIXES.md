# Правки для репозитория deploy (по итогам установки на lct.velikoss.ru)

Офлайн-бандл v0.2.0 ставился на сервер с публичным доменом. `install.sh --host lct.velikoss.ru`
даёт стенд, который на домене не работает. Всё ниже пришлось делать руками; здесь то, что нужно
зашить в deploy.

## Что сломалось и почему

| # | Симптом | Причина | Где чинить |
|---|---|---|---|
| 1 | Сайт только на `:8080` (http) и `:8443` (самоподписанный сертификат) | Caddyfile слушает `:8080`/`{host}:8443` с `tls internal`; 80/443 не публикуются, ACME невозможен | Caddyfile, compose |
| 2 | С `--host` все URL получаются `http://host:8080`, https нет | `gen_env.sh` жёстко пишет `http://${HOST}:${HTTP_PORT}` | gen_env.sh |
| 3 | Presigned-ссылки S3 идут на `http://host:8333`, страница на https → mixed content | То же, S3 без TLS | gen_env.sh, Caddyfile |
| 4 | Keycloak: `invalid_redirect_uri` на новом домене | `sed` в gen_env.sh подменяет только `//localhost:8080` и `//localhost:8443` на `//host:порт`; для https на 443 получается неверный URI. Правка `runtime/realm-crm.json` после первого старта ничего не меняет: realm уже импортирован в БД | gen_env.sh, отдельный скрипт смены хоста |
| 5 | Нет шрифтов Rostelecom Basis, в консоли «rejected by sanitizer» | Файлов `/fonts/*.woff` нет ни в образе `web`, ни в бандле. Сервер отдаёт `index.html` (SPA-fallback) с кодом 200 | build бандла |
| 6 | Порты 8080/8443/8333 открыты в интернет | Docker публикует порты в обход ufw | compose |
| 7 | Смена домена после установки не применяется | `install.sh` при существующем `compose/.env` пишет «не трогаю» | install.sh |
| 8 | Пароль `admin` Keycloak из `.env` не подходит | Bootstrap-админ создаётся при первом старте; `.env` потом менялся, БД Keycloak хранит прежний пароль | RUNBOOK, скрипт сброса |
| 9 | CSP блокирует inline-скрипт (importmap) на странице входа Keycloak | Caddy навешивает `script-src 'self'` на `/auth/*`, а тема Keycloak использует inline-скрипты | Caddyfile |

## 1. install.sh: явно спрашивать, как ставим

Сейчас `--host` необязателен, молча получается `localhost`. Предложение: вопросы задаются, если
не переданы флаги и есть TTY. Без TTY и без `--host` установка останавливается с ошибкой.

```bash
# новые флаги
#   --host <домен|IP>     обязателен (или интерактивно)
#   --tls  acme|internal  acme = Let's Encrypt (нужен публичный домен, порты 80/443)
#                         internal = самоподписанный (закрытый контур, IP, localhost)
#   --email <адрес>       для ACME (уведомления об истечении)
#   --yes                 не задавать вопросов
ask() { # ask VAR "вопрос" "по умолчанию"
  local __v="$1" __q="$2" __d="${3:-}" __a
  [[ -t 0 && "${ASSUME_YES}" != 1 ]] || return 0
  read -r -p "${__q}${__d:+ [${__d}]}: " __a; printf -v "$__v" '%s' "${__a:-$__d}"
}

ask HOST    "Публичный домен или IP сервера" "${HOST}"
[[ -n "${HOST}" ]] || { echo "ОШИБКА: не задан --host" >&2; exit 2; }
ask PROFILE "Профиль (demo|prod)" "${PROFILE}"

# режим TLS по умолчанию выводим из хоста
if [[ -z "${TLS_MODE}" ]]; then
  if [[ "${HOST}" =~ ^[0-9.]+$ || "${HOST}" == localhost || "${HOST}" != *.* ]]; then TLS_MODE=internal
  else TLS_MODE=acme; fi
fi
ask TLS_MODE "TLS: acme (Let's Encrypt) или internal (самоподписанный)" "${TLS_MODE}"
```

Предполётные проверки перед шагом 2 (до `docker load`, чтобы не терять 10 минут на распаковку):

```bash
if [[ "${TLS_MODE}" == acme ]]; then
  pub_ip="$(curl -fsS -m5 https://ifconfig.me || true)"
  dns_ip="$(getent hosts "${HOST}" | awk '{print $1; exit}')"
  [[ -n "${dns_ip}" ]] || { echo "ОШИБКА: ${HOST} не резолвится" >&2; exit 1; }
  [[ -z "${pub_ip}" || "${dns_ip}" == "${pub_ip}" ]] || echo "ПРЕДУПРЕЖДЕНИЕ: ${HOST} -> ${dns_ip}, а внешний IP сервера ${pub_ip}"
fi
for p in 80 443 8333; do
  ss -tln "sport = :$p" | grep -q LISTEN && { echo "ОШИБКА: порт $p занят" >&2; exit 1; }
done
if command -v ufw >/dev/null && ufw status | grep -q 'Status: active'; then
  echo "ufw активен: нужно открыть 80, 443, 8333/tcp"; ask OPEN_UFW "Открыть сейчас? (y/n)" y
  [[ "${OPEN_UFW}" == y ]] && ufw allow 80/tcp && ufw allow 443 && ufw allow 8333/tcp
fi
```

Итоговая сводка перед стартом (`Домен / профиль / TLS / порты / URL входа`) и подтверждение,
если нет `--yes`. В конце вместо `http://host:8080` печатать реальный `https://host`.

Smoke в шаге 5 гонять по публичному URL, а не по `http://localhost:8080`, иначе поломки 1–4
остаются незамеченными: `bash scripts/smoke.sh "https://${HOST}"` (для `internal` — с `-k`).

## 2. gen_env.sh: URL от схемы, а не жёстко http

```bash
if [[ "${TLS_MODE}" == internal && ( "${HOST}" == localhost || "${HOST}" =~ ^[0-9.]+$ ) ]]; then
  # локальный стенд
  BASE="http://${PUBLIC_HOST}:${HTTP_PORT}"; KC="${BASE}/auth"; S3="http://${PUBLIC_HOST}:${S3_PORT}"
else
  BASE="https://${PUBLIC_HOST}"; KC="${BASE}/auth"; S3="https://${PUBLIC_HOST}:${S3_PORT}"
fi
set_var BASE_URL "${BASE}"
set_var KEYCLOAK_URL "${KC}"              # сейчас остаётся localhost
set_var KEYCLOAK_PUBLIC_URL "${KC}"
set_var S3_PUBLIC_ENDPOINT_URL "${S3}"
set_var CRM_TLS_HOST "${PUBLIC_HOST}"
set_var CRM_TLS_MODE "${TLS_MODE}"        # см. п.3
set_var ACME_EMAIL "${ACME_EMAIL:-}"
```

Realm: заменять целиком, а не по подстроке `//localhost:порт`:

```bash
sed -e "s|http://localhost:8080|${BASE}|g" \
    -e "s|https://localhost:8443|${BASE}|g" ... realm-crm.json
```

`localhost:5173` (Vite) в prod-профиле из realm выкидывать. Смещения портов (`--port-offset`) для
https-варианта нужны только для S3; 80/443 фиксированы, поэтому на одной машине по домену может
жить одно окружение.

## 3. Caddyfile и compose

Итоговое состояние, которое работало на lct:

```caddyfile
{
	# auto_https disable_redirects убрать: иначе нет редиректа 80 -> 443
	email {$ACME_EMAIL}                       # только если задан
	servers { trusted_proxies static private_ranges }
}

:8080 { import crm_routes }                   # только localhost: healthcheck и локальный smoke

{$CRM_TLS_HOST:localhost} {
	{$CRM_TLS_DIRECTIVE}                      # пусто для acme, "tls internal" для internal
	import crm_routes
}

{$CRM_TLS_HOST:localhost}:8333 {
	{$CRM_TLS_DIRECTIVE}
	reverse_proxy seaweedfs:8333
}
```

Внутри `crm_routes`, перед финальным `handle { reverse_proxy web }`:

```caddyfile
handle /fonts/* { root * /srv; file_server }
```

`CRM_TLS_DIRECTIVE` в compose вычислять из `CRM_TLS_MODE` (`internal` -> `tls internal`, иначе пусто).

compose, сервис `caddy`:

```yaml
ports:
  - "127.0.0.1:${HTTP_PORT:-8080}:8080"   # не наружу: docker обходит ufw
  - "80:80"
  - "443:443"
  - "443:443/udp"
  - "${S3_PROXY_PORT:-8333}:8333"
volumes:
  - ./fonts:/srv/fonts:ro
```

`:8443` убрать. Том `caddy_data` хранит сертификаты: не удалять при `down -v`, иначе упрёмся в лимиты Let's Encrypt.

CSP для Keycloak: в `crm_routes` выделить `handle /auth/*` с ослабленным `script-src 'self' 'unsafe-inline'`
(или убрать наш заголовок и оставить CSP самого Keycloak).

## 4. Шрифты

Проприетарные `RostelecomBasis-{Regular,Medium,Bold}.woff` не попадают ни в образ `web`, ни в
бандл. Варианты:
1. (рекомендуется) класть в репозиторий deploy `compose/fonts/`, копировать в бандл в
   `build_offline_bundle.sh`, монтировать в Caddy (см. п.3). Если лицензия запрещает хранить в
   git, то скачивать на шаге сборки из приватного хранилища.
2. Класть в образ `web` (репозиторий frontend, `static/fonts/`).
3. Минимум: в `smoke.sh` добавить проверку, что `/fonts/RostelecomBasis-Regular.woff` отдаёт
   `Content-Type: font/woff`, а не `text/html`. Сейчас SPA-fallback возвращает 200 и поломку
   не видно.

Файлы на сервере лежат в `/root/rtk-crm-offline-v0.2.0/compose/fonts/`, исходные имена:
`RostelecomBasis-{Bold-OZUYKlU4,Medium-CXMf31X7,Regular-DTtbnrLW}.woff` (хэши из Vite-сборки).

## 5. Новый скрипт scripts/set_host.sh (смена домена без переустановки)

Вручную сделано ровно это, оформить скриптом:

1. Обновить в `.env`: `BASE_URL`, `KEYCLOAK_URL`, `KEYCLOAK_PUBLIC_URL`, `S3_PUBLIC_ENDPOINT_URL`, `CRM_TLS_HOST`.
2. Обновить клиента `crm-bff` в уже импортированном realm. Через kcadm под админом, а при рассинхроне
   пароля напрямую в БД:
   ```sql
   INSERT INTO redirect_uris(client_id,value) SELECT id,'https://HOST/*' FROM client WHERE client_id='crm-bff' ON CONFLICT DO NOTHING;
   INSERT INTO web_origins(client_id,value)  SELECT id,'https://HOST'   FROM client WHERE client_id='crm-bff' ON CONFLICT DO NOTHING;
   UPDATE client_attributes SET value='https://HOST/*' WHERE name='post.logout.redirect.uris' AND client_id=(SELECT id FROM client WHERE client_id='crm-bff');
   ```
   и `docker compose restart keycloak` (кэш).
3. `docker compose up -d --wait caddy api keycloak`, затем smoke по новому URL.

`install.sh` при существующем `.env` должен предлагать `set_host.sh`, а не молча пропускать.

## 6. Прочее

- Демо-профиль отдаёт наружу демо-пользователей realm. В `install.sh --profile demo` при
  публичном домене (`acme`) печатать предупреждение и требовать `--yes-i-know-demo`.
- RUNBOOK: раздел «Домен и TLS» (DNS A-запись, порты 80/443/8333, ufw и обход ufw у docker),
  раздел «Сброс пароля админа Keycloak» (`kc.sh bootstrap-admin user`).
- Не выводить секреты в лог установки; `gen_env.sh` печатает только пути.
- В `.env.example` убрать дубли `S3_PUBLIC_ENDPOINT_URL` (закомментированный и активный) и
  оставить один источник.
