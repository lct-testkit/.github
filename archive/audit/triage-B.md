# Triage B: checklist.md lines 45-55, 57-66, 68-74, 86-95

Read-only triage of the working tree (read 2026-09-26 00:10-00:35). While I read it, another engineer edited `catalog/service.py`, `catalog/models.py`, `imports/models.py` and added migration `0017_people_import.py`, `core/normalize.py`, `catalog/learner.py`; contact rows reflect the tree at ~00:33. `[BEING-REWRITTEN]` = import pipeline, CMS webhook (`cms.py`, `public_router.py`), ContactService dedup. Effort: S <30 min, M 1-3 h, L >3 h. Evidence is `file:line` in `D:\testkit-lct\backend\app\...` (paths shortened). A few read-only probes were run (stdlib, lxml on inline XML, repo `validators.py` loaded with bytecode writing off: `²`, `1e999`, empty/garbage XML, key length); nothing in the repo was changed.

## 1. Каталог и реестр (lines 45-55)

| # | claim | STATUS | evidence | effort | fix idea |
|---|---|---|---|---|---|
| 1 | ИНН: `str.isdigit()` пропускает «²» | OPEN | catalog/validators.py:36,59,65,74 use `isdigit()`; probe: КПП `77070100²` returns ok=True, ИНН/ОГРН raise ValueError | S | `isascii() and isdigit()` (or regex `[0-9]`) in all four validators |
| 2 | ...из-за этого 500 в создании организации | OPEN | catalog/service.py:299 `validate_inn` -> `int('²')` ValueError (probe `770708389²`), no handler -> 500 | S | Fixed by row 1; add regression test |
| 3 | ...500 в импорте [BEING-REWRITTEN] | OPEN | imports/fields.py:189 same validator; `_extract_row` has no try/except, one such row 500s the whole dry-run | S | Row 1 fix + per-row exception guard |
| 4 | ...500 в автоподстановке | OPEN | registry/service.py:97 (`GET /org-lookup/inn/{inn}`) and registry/router.py:97 (`/validate`) call the same validator | S | Row 1 fix |
| 5 | Лицензии читают все KAM/HEAD без скоупа | OPEN | catalog/router.py:784-802 + service.py:1665-1671: only `catalog:read`, no organization scope filter | M | Filter by `organization_scope_clause`; 404 when out of scope |
| 6 | ...вместе с ФИО менеджеров | OPEN | catalog/schemas.py:523-524 `manager_full_name`, `responsible_contacts` returned in list and card | S | Drop or mask for non-ADMIN / out-of-scope org |
| 7 | Контакт без орг./сделки: создатель видит 201, затем 404 [BEING-REWRITTEN] | FIXED | service.py:136 scope now has `Contact.created_by == user`; models.py:258 + migration 0017:71; uncommitted work | S | Keep; verify 0017 applied, add regression test |
| 8 | Поиск контактов по email в обход маскирования | OPEN | service.py:623 `Contact.email.ilike('%q%')` while output masks email: substring oracle for full addresses | S | Drop email from `q` (or exact match only after reveal/ADMIN) |
| 9 | POST /contacts не требует согласия | OPEN | schemas.py:211 `consent_id` optional; service.py:842 stored as-is; `find_or_create` passes None (:884) | M | Require consent_id/legal basis for personal contacts; validate FK |
| 10 | ...не нормализует телефон и email [BEING-REWRITTEN] | FIXED | service.py:637-649 `_identity` normalizes (E.164/lowercase), 422 on garbage; used in create :814, update :1047; uncommitted | S | Also normalize `Organization.main_phone/main_email` |
| 11 | Повтор code после мягкого удаления направления/продукта: 500 | PARTIAL | service.py:1181,1315 check live rows only; models.py:291,305 global UNIQUE; core/problem.py:218-246 now maps to 409 instead of 500 | M | Partial unique index `WHERE deleted_at IS NULL` (migration) or explicit "code held by deleted item" |
| 12 | ...и после отката импорта | PARTIAL | imports/service.py:897 rollback soft-deletes; same UNIQUE gives 409 now; re-import treats deleted row as existing (row 39) | M | Same as row 11 |
| 13 | Повтор ИНН удалённой организации: 409 без восстановления | OPEN | service.py:249 `_find_by_inn` ignores deleted_at, :312 raises 409 (`deleted:true`); DB index is partial; no restore endpoint | S | Filter `deleted_at IS NULL` in lookup, or add restore endpoint |
| 14 | Определения пользовательских полей нигде не применяются | OPEN | `CustomFieldDefService` (service.py:1565) is CRUD only; crm/service.py:1405,1485 and org/product custom_fields unvalidated; dsl `known_custom_fields` never passed | L | Validate values on write against active defs (type/required/options) |
| 15 | Кэша справочников по спеке нет | OPEN | Only principal cache exists (core/cache.py); regions, directions, products, loss reasons, holidays hit DB every call | M | Redis TTL cache, invalidate via `run_after_commit` on writes |
| 16 | Обезличенный контакт можно снова заполнить ПДн | OPEN | service.py:1026-1097 `update` never checks `is_anonymized`; PATCH accepts names/email/phone | S | 409 on PATCH of anonymized contact |
| 17 | Псевдоним из старших битов UUIDv7, коллизии ~минуту | OPEN | service.py:990,570 and identity/erasure_service.py:492 use `str(id)[:8]`; core/ids.py:52 timestamp-first: same prefix for 65.5 s | S | Random suffix or `sha256(id+secret)[:12]` |
| 18 | Удаление орг. при обезличивании падает на FK контактов/лицензий, повтор каждые 15 мин | OPEN | service.py:580 eligibility counts deals only; erasure_service.py:615 delete hits RESTRICT FKs; identity/tasks.py:74 swallows, request stays due; act PDF re-uploaded each try (:601) | M | Add contacts/licenses to blockers, mark BLOCKED; issue act only after success |
| 19 | Импорт ЕГРЮЛ: лимит 32 767 параметров asyncpg (5000x21) | OPEN | registry/tasks.py:44 batch 5000, :51-76 21 columns, :77 one multi-VALUES insert = 105k params; breaks past ~1 560 rows | S | Batch <=1 500 rows or COPY/executemany |
| 20 | ...идёт одной транзакцией | OPEN | tasks.py:147 single `session_scope`; RUNNING only flushed (:155); any failure rolls back everything, no checkpoints | M | Commit per batch; separate status transaction; resumable |
| 21 | ...следующий тик может запустить повторно | OPEN | tasks.py:148-157 selects PENDING without lock, RUNNING uncommitted; cron every minute (worker/main.py:137) | S | `FOR UPDATE SKIP LOCKED` and commit RUNNING first |
| 22 | ...пустой или битый файл завершается как completed | OPEN | tasks.py:131-139 no zero-entry check; lxml `recover=True` (egrul_xml.py:247); probe: empty->FAILED, garbage/truncated/wrong schema->COMPLETED | S | Fail on 0 entries or parse errors; drop `recover` |
| 23 | ...выгрузка читается в память целиком | OPEN | tasks.py:105 `download_object_bytes`, :111 sha256(content), :119 BytesIO(content): whole dump in RAM | M | Stream to temp file; incremental sha256 |
| 24 | Сверка держит глобальный лок аудита на весь проход | OPEN | tasks.py:288 one `session_scope` for all orgs; first audit write takes `pg_advisory_xact_lock` (audit/service.py:116) held until commit | M | Commit per organization/batch |
| 25 | ...каждые 30 дней заново поднимает версию и уведомление | OPEN | tasks.py:315-329: unresolved drift re-merged, `version += 1`, audit + notification repeated every interval | S | Skip when merged drift equals existing |
| 26 | ...применяет статус ликвидации автоматически | OPEN | tasks.py:319-320 sets `registry_status` directly; spec line 866: drift must not auto-apply | S | Write drift only; separate liquidation flag, human accepts |

## 2. Файлы и импорт (lines 57-66)

| # | claim | STATUS | evidence | effort | fix idea |
|---|---|---|---|---|---|
| 27 | Антивирус остаётся заглушкой | OPEN | files/service.py:89-99 `NullAntivirusScanner` always clean; nothing registers a real scanner; no clamav in compose | L | clamd sidecar + INSTREAM client, fail-closed in prod |
| 28 | Лимит 500 МБ недостижим | OPEN | files/router.py:64 calls `upload_intent` without `deal_scope=True`; service.py:209-213 always picks 50 MB | S | Pass deal scope for deal attachments (persist purpose) |
| 29 | Размер PUT не ограничен | OPEN | core/storage.py:83-93 presigned PUT has no length constraint; only post-hoc check in commit (service.py:271-288); pending objects never purged | M | Sign ContentLength / POST policy; GC pending uploads |
| 30 | Заниженный size_bytes обходит 50 МБ | PARTIAL | commit (service.py:271-288) now re-measures, deletes >500 MB; ceiling is max(50, 500 MB), scope not stored: 50 MB still bypassable | S | Persist limit/purpose on File; compare real size to it |
| 31 | Дедуп sha256: через 7 дней удаляется объект отчёта, на который ссылался файл пользователя | OPEN | files/service.py:344-357 repoints file to existing key; reporting/tasks.py:144 deletes shared object unconditionally | M | Delete only if no other READY file shares the key; no cross-bucket dedup |
| 32 | Длинные кириллические имена дают 500 | OPEN | service.py:219 `quote(filename)` = 6 chars/letter into `storage_key` String(512) (models.py:58); >=80 Cyrillic chars -> DataError (probe: 517) | S | Key = `{id}/{uuid}.{ext}`; keep real name only in DB |
| 33 | Имя файла не санитизируется | OPEN | service.py:219-224 raw `original_filename`; `quote` keeps `/` so `..` survives; storage.py:112 puts name unescaped in Content-Disposition | S | Strip path/control chars; RFC 5987 encoding |
| 34 | refcount не атомарен | OPEN | service.py:475 `+= 1`, :504 `-= 1`: ORM read-modify-write, no lock or atomic UPDATE | S | `UPDATE files SET refcount = refcount +/- 1` |
| 35 | refcount не ведётся в импорте и подписи | OPEN | imports/service.py:499-526, signing/service.py:700-713,1526 create File+Attachment without refcount, so `soft_delete` (files/service.py:390) allows deleting attached files | S | Increment there, or count live attachments in `soft_delete` |
| 36 | Ретеншена и физического удаления нет | OPEN | Only report files expire (reporting/tasks.py:123); `soft_delete` (files/service.py:386) never deletes the object; no purge cron or bucket lifecycle | M | Purge job (deleted/pending/infected) with shared-key check |
| 37 | Отравленная строка вешает импорт навсегда [BEING-REWRITTEN] | OPEN | imports/tasks.py:31-64 one tx for all jobs, no try; `apply_batch` has no per-row guard; fields.py:152 text kind has no length check | M | Per-row SAVEPOINT -> ERROR status; per-job transaction |
| 38 | PUT mapping без проверки статуса стирает основу отката [BEING-REWRITTEN] | OPEN | imports/service.py:198-221 sets MAPPED from any state; next dry-run deletes `ImportRowResult` (:308) = rollback basis lost | S | Allow only UPLOADED/MAPPED/VALIDATED, else 409 |
| 39 | Мягко удалённые записи считаются существующими [BEING-REWRITTEN] | OPEN | imports/service.py:289-293, 403, 588, 671, 738 lookups lack `deleted_at IS NULL`; upsert edits deleted rows | S | Filter `deleted_at` in all lookups |
| 40 | inf и 1e999 дают 500 [BEING-REWRITTEN] | OPEN | fields.py:156 `int(float())` catches only ValueError: OverflowError uncaught (probe); :161 `Decimal('inf')` accepted, overflows NUMERIC at apply | S | Catch OverflowError; reject non-finite/out-of-range |
| 41 | Upsert лицензии перезаписывает organization_id чужой [BEING-REWRITTEN] | OPEN | imports/service.py:738-746 matches by `contract_number` only; :782-805 update loop includes `organization_id` | S | Key on (org, contract) or refuse org change |
| 42 | Одна пачка 500 строк/мин, прогресса нет [BEING-REWRITTEN] | OPEN | imports/tasks.py:44 one `apply_batch` (500, config.py:146) per job per minute; models.py:101 `processed_rows` added, service never writes it | M | Loop batches within tick budget; update `processed_rows` |
| 43 | Профиль показывает строки любого готового файла [BEING-REWRITTEN] | OPEN | imports/service.py:107-128 `_get_ready_file` has no owner/bucket/purpose check; `profile` (:167-186) returns 100 rows | S | Require own file (or ADMIN) of import category |
| 44 | Откат успешен при заблокированных строках [BEING-REWRITTEN] | OPEN | imports/service.py:923-934 blocked rows leave the pending set, job becomes ROLLED_BACK; :847 `rollback_available` already false | S | Status `rollback_partial`; keep retry available |

## 3. Отчёты (lines 68-74)

| # | claim | STATUS | evidence | effort | fix idea |
|---|---|---|---|---|---|
| 45 | Семафор фиктивен: processing никогда не коммитится | OPEN | reporting/service.py:219-220 only flush, commit at end; tasks.py:70-75 counts PROCESSING = always 0; only per-user cap (service.py:173-189) works | M | Claim job (`UPDATE...RETURNING` + commit) before generating |
| 46 | ...задания идут последовательно | OPEN | tasks.py:92-115 sequential loop inside one tick, no concurrency | M | Enqueue one arq job per claimed report |
| 47 | ...429 CRM-1601 нигде не бросается | FIXED | service.py:184-189 raises `REPORTS_LIMIT_EXCEEDED` (429, core/errors.py:158) on per-user unfinished cap | S | none |
| 48 | Перекрытие тиков: один отчёт генерируется дважды | OPEN | tasks.py:78-96 selects QUEUED and `session.get` without lock; cron every minute (worker/main.py:153) | S | `FOR UPDATE SKIP LOCKED` claim + commit |
| 49 | Рендер синхронный, блокирует цикл событий | OPEN | service.py:223 `render_report()` (openpyxl/reportlab/matplotlib) called directly, no `to_thread` | S | `await asyncio.to_thread(render_report, ...)` |
| 50 | Скачивание не аудируется | OPEN | service.py:319-347 `download()` has no `_audit.record`; router.py:161-167 none; no REPORT_DOWNLOADED action | S | Add audit action with job id, actor, ip |
| 51 | Ссылка не одноразовая | OPEN | service.py:338-346 presigned GET valid 15 min, reusable, new URL on every call | M | Proxy via API with one-time token / delete after download |
| 52 | expires_at не проверяется | FIXED | service.py:331-336 `expired` compared to now -> CRM-1603 (410) | S | none |
| 53 | После ретеншена «ещё не готов» вместо «истёк срок» | FIXED | service.py:330-336 file gone (`job.file_id=None`, tasks.py:148) -> `REPORT_EXPIRED` 410 | S | none |
| 54 | REPORT_EXPORTED без параметров и без актора в воркере | PARTIAL | service.py:275-288 logs template/format/row_count/categories, not `job.params`; worker never sets actor or passes `actor_id` -> NULL | S | Pass `actor_id=job.requested_by`; add params |
| 55 | POST /reports всегда отвечает 201 | OPEN | reporting/router.py:81 fixed 201 for completed and queued jobs | S | 200 completed, 202 (+Location) queued |
| 56 | ...не умеет xls и json | OPEN | reporting/schemas.py:13 `FormatLiteral` = xlsx/pdf/png; rendering.py:40-44; JSON only via GET `/data` | M | Add json and xls renderers or restrict per case |
| 57 | Нет обязательного «отчёта по кейсу» | OPEN | builders.py:745-754, seed.py:40-113: no report with вуз/направление/продукт/статус/ответственный; `stuck_deals` lacks direction, product | M | New builder over deals+org+product+direction+status+owner (filters already parsed) |
| 58 | learning_progress пуст | OPEN | builders.py:722-738 returns `rows=[]` although `LearningProgress` (integration/models.py:243) is filled by lms.py:85 | M | Query learning_progress joined to deal/contact/product under deal scope |
| 59 | /data строит данные заново в обход очереди и аудита | OPEN | service.py:302-317 rebuilds per call; router.py:122-148; up to 5 000 rows synchronously | S | Audit reads; apply sync threshold or cache |
| 60 | Цифры MV и построчного пути расходятся | OPEN | migrations 0009:~217-224 MV `hist` counts history of deleted deals; builders.py:246-260,315-339 row path filters `deleted_at`; MV stale <=5 min | S | Join `deals` with `deleted_at IS NULL` in MV (new migration) |
| 61 | Текст исключения с SQL попадает в поле ошибки | OPEN | reporting/tasks.py:114 `f"{type(exc).__name__}: {exc}"` -> `report_jobs.error` (service.py:292) -> `ReportJobOut.error` (schemas.py:71) | S | Generic message + error id; details to log |
| 62 | PATCH дашборда с null даёт 500 | OPEN | schemas.py:115-118 Optional fields; service.py:410-415 `setattr(None)` -> NOT NULL 23502 -> core/problem.py:233-234 falls to 500 (same for widgets :478) | S | Drop/422 on null for non-nullable fields |

## 4. Интеграции и уведомления (lines 86-95)

| # | claim | STATUS | evidence | effort | fix idea |
|---|---|---|---|---|---|
| 63 | Дедуп вебхука до проверки подписи: неверная подпись «занимает» ключ [BEING-REWRITTEN] | OPEN | integration/cms.py:50-63 and public_router.py:119-121,178-186 look up existing before HMAC; bad-signature row stored FAILED, real delivery gets 200 failed | S | Verify HMAC first; record bad attempts without occupying the key |
| 64 | Нет защиты от повтора [BEING-REWRITTEN] | OPEN | security.py:34-49 HMAC over body only; `Idempotency-Key` unsigned; no timestamp or nonce | M | Sign `timestamp.key.body`, +-5 min window, nonce store |
| 65 | Сырое тело теряется при сбое обработки [BEING-REWRITTEN] | OPEN | cms.py:101-107 marks FAILED then re-raises -> `get_db_session` rollback drops the row; public_router.py:154-162 same; `raw_payload` is parsed JSON (cms.py:68) | M | Persist raw bytes in own committed tx before processing |
| 66 | Bitrix: HTTP внутри одной транзакции без блокировок -> дубли | OPEN | integration/tasks.py:58-76 one tx for <=200 events, no SKIP LOCKED; :170-180 + bitrix.py:239 `crm.item.add` before commit; cron every minute | M | Claim events (SKIP LOCKED, commit), idempotency marker, per-event tx |
| 67 | Конфликт по updatedTime «липкий» | OPEN | bitrix.py:183-212 raises while Bitrix `updatedTime` > `ref.last_synced_at`, never refreshed -> every later push fails -> dead | M | Log discrepancy, apply LWW/refresh base time, alert |
| 68 | Обратное направление правит заголовок в обход сервиса без аудита | OPEN | bitrix.py:302-308 `deal.title = title`; no DealService, version bump, audit or history | S | Route via `DealService.update` as INTEGRATION with audit |
| 69 | LMS: одна плохая строка прогресса откатывает всю пачку | OPEN | lms.py:85-147 skips only missing ids; CHECK/type errors raise at flush; no SAVEPOINT (push handler public_router.py:155-158 [BEING-REWRITTEN], pull tasks.py:203-206) | S | `begin_nested` per row, count skipped |
| 70 | ...pull не двигает курсор и повторяет сбой | OPEN | integration/tasks.py:193-214 row error rolls back incl. cursor; try/except covers only HTTP (:196-200): same batch fails every 30 min | S | Row isolation; advance cursor past poison rows |
| 71 | ...неполная строка затирает поля | OPEN | lms.py:127-145 `setattr` for every field, None for missing keys | S | Update only keys present in the row |
| 72 | Секрет CMS читается только из credentials_ref в БД [BEING-REWRITTEN] | OPEN | core/config.py:178 `cms_webhook_secret_ref` declared, unused; public_router.py:93 `resolve_secret(source.credentials_ref)` | S | Fall back to `resolve_secret(settings.cms_webhook_secret_ref)` |
| 73 | Bitrix: один credentials_ref = URL и ключ подписи [BEING-REWRITTEN] | OPEN | integration/tasks.py:173-175 uses it as webhook URL; public_router.py:185-186 uses same value as HMAC secret | S | Separate refs (url vs signing secret) in source config |
| 74 | PATCH источников не валидирует base_url: SSRF, утечка токена LMS | OPEN | schemas.py:34 plain str; service.py:150-167 stores as-is; tasks.py:157-160 sends LMS bearer to `source.base_url` from worker | S | https-only, host allowlist, deny private IPs; never send token to unlisted host |
| 75 | Контракт CMS расходится со спекой [BEING-REWRITTEN] | OPEN | Spec (line 642): signature + source id + Idempotency-Key headers, raw body stored, may return accepted; code: X-Signature + key only, parsed JSON stored (cms.py:68), plain 200 | M | Align headers, raw body, status with spec |
| 76 | Заявка на другой курс молча считается дублем [BEING-REWRITTEN] | OPEN | cms.py:152-167 any open b2c deal of the contact returns duplicate, product/course ignored | S | Add product/course to the dedup key |
| 77 | Часть событий outbox не публикуется | OPEN | Publish only at crm/service.py:1464 (create), :1767 (transition), :1796 (DSL), signing DOCUMENT_SIGNED; PATCH/reassign/delete publish nothing (spec line 73) | M | Publish DEAL_UPDATED/OWNER_CHANGED etc. in the same tx |
| 78 | ERASURE_REQUESTED уходит в неизвестную цель | OPEN | identity/erasure_service.py:646-654 target = key of `external_ids` (e.g. `cms_lead_id`) -> integration/tasks.py:100-104 dead `unknown_target` | M | Map keys to source codes; dedicated erasure handler per source |
| 79 | Внешние каналы не заработают: нет шаблона и настоящего адреса | OPEN | notification/tasks.py:61 `send(address_masked=..., subject=None, body="")`: no rendering, masked address only | M | Render template; resolve real address at send time |
| 80 | ...telegram без адресов | OPEN | notification/service.py:309 address only for email; no telegram id field; seed.py:16-21 not seeded | M | Store chat_id per user/contact, resolve per channel |
| 81 | Шаблоны уведомлений Jinja2 без песочницы | FIXED | notification/service.py:124 `SandboxedEnvironment` used by `render_template` | S | none |
| 82 | ...шаблоны документов без песочницы | OPEN | signing/rendering.py:75 plain `jinja2.Environment`; latent: no HTTP write path to `SignatureTemplate` today (seed/DB only) | S | Use `SandboxedEnvironment` |
| 83 | ...ошибка в шаблоне даёт 500 всей ленте | PARTIAL | notification/service.py:468 catches only `jinja2.TemplateError`; ZeroDivision/Type/OverflowError from a template still 500 the list (router.py:85) | S | Catch `Exception` per notification |
| 84 | Пагинация шаблонов режет курсор по другому ключу | OPEN | notification/service.py:583 `order_by(code, channel)` + router.py:191-193 appends `created_at desc, id desc`; keyset on (created_at, id) | S | `.order_by(None)` first or keyset on (code, channel) |
| 85 | Настройки получателя не скрывают ленту | OPEN | service.py:232-242 Notification saved before pref check (:278-279); `list_query` (:424-436) ignores prefs | S | Skip creating or filter disabled event codes |
| 86 | Тихие часы отбрасывают уведомления | OPEN | service.py:298-307 quiet hours -> delivery `SKIPPED`, never rescheduled | M | PENDING with `deliver_after` = end of window |
| 87 | mark_read с пустым списком отмечает всё | OPEN | service.py:489 `if filters.ids:` empty list is falsy -> no id filter; schemas.py:63 no min_length | S | `is not None` check: `[]` updates 0 rows |
| 88 | Диспетчер без блокировок и backoff шлёт дубли | OPEN | notification/tasks.py:39-50 no SKIP LOCKED, send inside tx; :63-66,76-78 retry next tick, no backoff; cron every minute | M | SKIP LOCKED claim + `next_attempt_at` exponential backoff |
| 89 | README про Битрикс расходится с кодом | PARTIAL | README.md:245 "4 conditions, all required" but flag row optional; :252 lists 6 pauses, code repeats 2 h (tasks.py:48,144); inactive source -> dead `source_inactive` (:111-115) undocumented | S | Sync README with `integration/tasks.py` |

Note for rows 8, 9, 16: they live in `ContactService`, which is being edited (dedup). Not dedup itself, but expect merge conflicts.

## TOP RISKS (OPEN items, data integrity / security, most dangerous first)

1. Row 31: dedup + 7-day report retention silently and permanently destroys users' files (shared S3 object deleted, records still READY).
2. Row 38 (with row 44 and extra E1) [BEING-REWRITTEN]: PUT mapping on an applying/completed job resets it and the next dry-run deletes `import_row_results`, so rollback basis is gone; rollback can also soft-delete pre-existing records.
3. Row 18: 152-ФЗ erasure of an organization never completes (RESTRICT FKs), retried every 15 min, leaks an act PDF to S3 per attempt; legal-obligation failure.
4. Row 66: outbox delivery keeps HTTP inside one transaction without locks; overlapping ticks or a crash after `crm.item.add` create duplicate Bitrix deals and LMS enrollments.
5. Rows 63+64 [BEING-REWRITTEN]: unauthenticated request with a guessed key blocks the real delivery (200 failed); no timestamp/nonce so replays are accepted.
6. Rows 35+34: refcount not kept for import/signing files and not atomic, so signed/protocol files can be soft-deleted via `DELETE /files/{id}` by author or HEAD.
7. Row 74: admin PATCH of `base_url` gives SSRF from the worker and leaks the LMS bearer token to an arbitrary host.
8. Rows 5+6+8: broken access control and PII exposure: licenses (with manager names) readable by every KAM/HEAD; contact search by email substring defeats masking.

Next in line: 24 (global audit lock during the 04:30 drift sweep), 16/17 (anonymization), 37 (poison row stalls all imports), 21 (registry import can run twice), 19 (real EGRUL import always fails on batch size).

## Extra findings (not in the checklist lines)

- E1 [BEING-REWRITTEN]: imports/service.py:666,730,808 store `before_snapshot = {}` for no-change updates, but `rollback_batch` (:877) tests truthiness, so such rows fall into the "created" branch and soft-delete the pre-existing record (unless it has deals).
- E2: registry/tasks.py:77-80 duplicate INN inside one batch raises "ON CONFLICT DO UPDATE cannot affect row a second time" (server-side, not executed here); the aborted transaction then breaks `AuditService.record`, so the version stays PENDING and is retried every minute.
- E3: core/deps.py:300 `If-Match` parsing uses `isdigit()` too: header `²` gives 500 (outside my ranges).
- E4: `validate_kpp` accepts `²` as a valid KPP (probe), so garbage can be stored in `organizations.kpp`.
