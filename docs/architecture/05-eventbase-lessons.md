# Что берём из Eventbase, что меняем, что не берём

Статус: 2026-10-08. Итог изучения репозитория Eventbase (9 приложений,
229 миграций, 211 таблиц, 58 файлов тестов базы, 27 ADR, 35 runbook'ов,
~1,4 МБ правил `CLAUDE.md`). Это главный документ «переноса опыта»:
каждая строка — конкретный модуль, правило или грабли.

## 1. Переносится как есть (копировать файлы, переименовать пакет)

| Что | Откуда в Eventbase | Зачем |
|---|---|---|
| Монорепо pnpm, `packageManager`, `pnpm-workspace.yaml`, `transpilePackages` для `@ls/shared` | корень, `apps/*/next.config.ts` | один стек, одна сборка |
| `supabase/apply.sh` — журнал `app.schema_migrations`, контрольные суммы, advisory lock, сторож разрушающих команд, `notify pgrst` | `supabase/apply.sh`, `docs/setup/03-apply-migration.md` | автоматическое применение миграций при выкате |
| Тестовый стенд базы: `supabase/tests/run.sh`, `00-supabase-shim.sql`, харнесс `tst`, `concurrency.sh`, `schema-dump.sql` | `supabase/tests/` | RLS проверяется исполняемым тестом в CI |
| `scripts/check-db-contract.mjs` — колонки в `.select()`, аргументы `.rpc()`, подсказки `rel!fk` | `scripts/` | кодовая база и схема не расходятся молча |
| `scripts/ci-affected.sh` и `.github/workflows/ci.yml` — проверки только по затронутому, одна задача «Быстрые проверки» шагами, еженедельный полный прогон | `.github/`, `scripts/` | платные минуты CI |
| Dockerfile: multi-stage `node:22-alpine`, standalone, `USER node`, `HEALTHCHECK` через `node -e fetch` | `apps/voting/Dockerfile` (с healthcheck) | Coolify не откатывает здоровый контейнер |
| `lib/supabase/{server,admin}.ts`, `lib/env.ts` (бросает по-русски на пустой переменной), `lib/auth.ts` с `cache()` | `apps/admin/src/lib/` | клиенты и проверки прав |
| `lib/must-write.ts`, `lib/write-errors.ts`, `lib/action-error.ts`, `lib/form-values.ts`, `lib/request-body.ts` | `apps/admin/src/lib/` | отказы записи называют класс причины; устаревшее действие после выката перезагружает страницу |
| `lib/rate-limit.ts` (ведро в базе `auth_rate_events`), `lib/reset-tokens.ts`, `lib/password.ts`, `lib/audit.ts` | `apps/admin/src/lib/` | вход, приглашения, журнал |
| `lib/security-headers.ts` + `middleware.ts` + `scripts/check-csp.mjs` | `apps/admin`, `apps/event-app` | CSP/HSTS из окружения |
| `packages/shared/src/client-ip.ts` (`CLIENT_IP_HEADER`) | `packages/shared` | лимиты по IP за прокси |
| `lib/email.ts` (Unisender Go по HTTP, `track_links: 0`, проверка `failed_emails`, выбор поставщика по заданному ключу) + шаблоны писем таблицами с инлайновыми стилями | `apps/admin/src/lib/email.ts` | служебная почта |
| Очередь `jobs` + `claim_job`/`claim_jobs`/`jobs_sweep` + `lib/jobs.ts` + `lib/worker.ts` + `lib/heartbeat.ts` + `instrumentation.ts` с `PORTAL_ROLE=worker` | `supabase/migrations/…foundation.sql`, `20261206…claim_job_stale.sql`, `20261213…claim_jobs_batch.sql`, `apps/event-app/src/lib/` | фоновые задачи без Redis |
| Web Push: `lib/push.ts` (белый список хостов FCM/APNs против SSRF, таймаут, классификация 404/410/429), `lib/push-client.ts`, `push_subscriptions`/`push_deliveries`, `pushsubscriptionchange` в SW | `apps/event-app/src/lib/` | уведомления на телефон |
| SSE-канал: `api/live/route.ts` (`retry` с разбросом, `hello{build}`, `fallback` кодом 200), `lib/live.ts` (комната, тик 1 с, `busy`), `lib/poll-stream.ts` (слоты), `components/live-provider.tsx`, `lib/freshness.ts`, `lib/chat-tail.ts`, `components/chat-live.ts` | `apps/event-app` | живые чаты и задачи |
| Service worker маршрутом + `scripts/check-service-worker.mjs` (шаблонная строка вычисляется как JS) | `apps/event-app/src/app/e/[slug]/sw/route.ts` | push не пропадает на трое суток из-за `//` в регулярке |
| Манифест PWA маршрутом, `install-gate.tsx`, `lib/install-platform.ts`, синхронный скрипт `data-standalone` | `apps/event-app` | установка на телефон |
| `lib/cursor.ts` (курсор «время + id»), `lib/paged-rows.ts` (PostgREST режет 1000 строк молча), `lib/parallel.ts` (`mapLimit` 8), `lib/snapshot.ts` (память процесса, обещание в кеше) | `apps/admin`, `apps/event-app` | списки и производительность |
| `lib/uploads.ts`, `components/upload-field.tsx`, `lib/client-image.ts`, `lib/image-pipeline.ts`, `lib/signed-urls.ts` | `apps/admin`, `apps/event-app` | файлы из браузера прямо в Storage, пережатие, подписанные ссылки |
| `lib/sentry.ts` (`scrub`, белый список заголовков), `instrumentation-client.ts` (DSN из атрибутов `<html>`), `/api/health`, `/api/version` | `apps/admin`, `apps/event-app` | GlitchTip без персональных данных |
| `app.only_columns` (бывший `attendee_only_columns`), `app.set_updated_at`, приём `aa_*`/`zz_*` в именах триггеров, `drop function if exists` при смене сигнатуры, новое значение enum отдельным файлом | миграции | ровно те грабли, на которые наступали |
| Модули платежей ЮKassa / CloudPayments / Т-Банк | `apps/registration/src/lib/payments/` | релиз 3 |
| Шрифты файлами через `next/font/local` (Onest + Golos Text у приложения участника, Roboto у админки), `docs/setup/10-fonts.md` | `apps/*/src/fonts` | сборка не ходит в интернет |
| Правила `CLAUDE.md`: параллельные сессии, одна задача — одна папка, общие слои отдельно, миграции только добавлением, решения в файл, тест на изоляцию, CI шагами, «заливай сам, когда готово», отчёт владельцу по-русски | корень | порядок работы с Claude Code |

## 2. Берём с изменениями

| Что | Как было | Как делаем и почему |
|---|---|---|
| **Второй фактор** | пароль → 6-значный код из письма, своя таблица `auth_login_codes`, сессия выдаётся трюком `generateLink + verifyOtp`; код из письма нельзя проверить в RLS | **TOTP через GoTrue MFA** (`aal2` в JWT) → проверяется **в базе** `app.staff_id()`; письмо не стоит на критическом пути входа; при нужде добавляется SMS-фактор тем же механизмом (ADR `platform-002`) |
| **Применение миграций** | pre-deployment command в Coolify = `docker exec` в **старом** контейнере → миграция приезжает на выкат позже кода | `apply.sh` в **entrypoint** веб-контейнера (`MIGRATE_ON_START=1`): новый контейнер накатывает схему и только потом стартует; упал — healthcheck не прошёл, остался старый |
| **Dockerfile** | `COPY . .` → `pnpm install` (кеша слоёв нет, каждый коммит ставит зависимости заново) | сначала манифесты (`package.json`, `pnpm-lock.yaml`, `pnpm-workspace.yaml`, `apps/*/package.json`, `packages/*/package.json`) → `pnpm install --frozen-lockfile` → потом исходники; HEALTHCHECK у всех образов |
| **Чаты** | у общего чата нет членства, нет редактирования/удаления, нет «прочитано собеседником», нет писем-дайджестов | явное членство у всех видов, `edited_at`/`deleted_at` с историей, маркеры прочтения, письмо через 10 минут без прочтения, Telegram-канал (ADR `platform-004`) |
| **Размер правил** | `apps/event-app/CLAUDE.md` — 762 КБ, `apps/admin/CLAUDE.md` — 499 КБ: ни одна сессия не читает их целиком, правила теряются | бюджет: корневой `CLAUDE.md` ≤ 40 КБ, `CLAUDE.md` приложения ≤ 60 КБ; остальное — `docs/patterns/<тема>.md`, на которые `CLAUDE.md` ссылается одной строкой; `scripts/check-doc-size.mjs` в «Быстрых проверках» |
| **База знаний** | статика из markdown в git, без входа и редактирования | статьи в базе, редактор TipTap, версии, аудитория, «ознакомлен», поиск `russian` (ADR `platform-005`); интерфейс чтения — по образцу `apps/docs` |
| **Тенантность** | `organizations → projects → attendees`, проект — короткоживущее мероприятие | `orgs → clubs`, долгоживущие сущности, два уровня членства (`org_members`, `club_members`), тонкие права кодами в `permissions text[]` |
| **Ведро лимитов по IP** | `CROWD_PER_IP` 5 000 под зал на одном Wi-Fi | у портала масштаб 300–400 человек: пороги ниже, но принцип «лимит по IP = лимит на всю студию за NAT» остаётся |
| **Интернационализация** | словарь на ~950 ключей, переводы контента в jsonb `i18n`, CI ловит кириллицу в `.tsx` | портал только по-русски, строки прямо в разметке (как в админке Eventbase), но **словарь терминов** в `CLAUDE.md` обязателен; клиентское приложение — тоже по-русски, i18n не закладываем |
| **Поставщик ИИ** | три поставщика с выбором по проекту | один — Яндекс (другие из РФ не работают или требуют НУЦ); слой `lib/ai/` оставляем с интерфейсом поставщика на случай смены |
| **Наблюдаемость** | два сервера, полный стек сразу | один сервер, GlitchTip + Kuma сразу, Grafana-стек — к клиентскому приложению |

## 3. Не берём

| Что | Почему |
|---|---|
| Два контура (ЕС + РФ) и всё про Cloudflare, Hetzner, Resend, ECH, DPI | у платформы одна площадка в России; правило «никаких адресов в коде» остаётся как гигиена |
| Копирование проектов/продуктов (`app.copy_plan`, эталон `39-copy-full.sql`) | мероприятия повторяются, студии — нет; шаблоны задач и статей решают задачу «повторить» дешевле |
| Механизм расширений (ADR platform-002 Eventbase: манифесты, хуки, швы) | заказчик один, а не десятки клиентов с «хотелками»; если появятся другие сети — вернёмся к ADR, он готов |
| Сертификаты НУЦ Минцифры в образе (`NODE_EXTRA_CA_CERTS`) | нужны только GigaChat; Яндекс работает на обычных |
| Клиентские домены, on-demand TLS, Caddy | один домен |
| Supabase Realtime, Edge Functions, Logflare | не использовались и там |
| Голосования, геймификация, бейджи, карты площадок, квесты, трансляции | другая предметная область; но их **техника** (замки, счётчики триггерами, пакетная запись) — источник образцов |
| Cloudflare Turnstile, Google-вход | Turnstile → Yandex SmartCaptcha к публичным формам релиза 3; Google-вход вырезан и в Eventbase |

## 4. Грабли Eventbase, записанные в правила новой платформы

Каждый пункт — реальный инцидент или находка аудита; каждому в новом
репозитории соответствует правило в `CLAUDE.md` или тест.

1. `for all` в политике → viewer удаляет (23 таблицы). **Четыре политики
   по операциям.**
2. `with check` не видит, какие колонки изменились → участник переставил
   `chat_id` и читал чужую личку. **Триггер-белый список колонок.**
3. `current_setting('request.jwt.claims')::jsonb` падает на пуле со
   второй транзакции (`''`). **Только `auth.jwt()` / `nullif`.**
4. `can_edit` вернула `null`, `if not can_edit` пропустил. **Функции
   записи никогда не возвращают null (`coalesce(…, false)`).**
5. Клейм `project_id` у третьего мира прошёл бы все политики второго.
   **Своё пространство имён клеймов на каждый мир.**
6. Политика с подзапросом к невидимой таблице молча даёт ложь. **Условие
   — в security-definer-функцию, читателям клеймов — `grant execute`.**
7. Поимённый `.select()` с новой колонкой до миграции ронял запрос
   (четыре раза). **Миграция едет тем же PR, что и код; строки форм
   читаются `select("*")`; `check-db-contract` в CI.**
8. Ручное применение миграции → `relation already exists` → все выкаты
   стоят полсуток (дважды). **Миграции применяет только выкат.**
9. Pre-deploy в старом контейнере → схема отстаёт от кода. **Миграции в
   entrypoint нового контейнера.**
10. `cache.addAll` в `install` SW отклонял установку, push пропадал.
    **Офлайн-страница — попыткой.** Регулярка `/\/$/` в шаблонной строке
    → `//` → SW не парсится трое суток. **`check-service-worker` в CI.**
11. `start_url` со слешем → PWA «вышла за scope», полоса браузера.
    **Без слеша.**
12. Кеширование навигации в SW → чужой профиль на общем телефоне.
    **Навигация не кешируется.**
13. `camera=()` в Permissions-Policy запретил камеру своему сканеру.
    **`camera=(self)`.**
14. CSP без `script-src` погасила страницу на обеих площадках. **`check-csp`.**
15. Ссылка сброса из `Host` → отравление. **Только `PORTAL_PUBLIC_URL`.**
16. Чёрный список заголовков в Sentry пропускал IP. **Белый список.**
17. Resend за Cloudflare → код входа приходил через раз. **Unisender Go.**
18. Отслеживание переходов в письмах сжигало одноразовые ссылки.
    **`track_links: 0`.**
19. Timeweb молча добавляет SPF к поддоменам → DKIM ломается. **Проверять
    через dns.google после каждой правки DNS.**
20. Docker Hub за Cloudflare → сборки спотыкаются. **Зеркало
    `dockerhub.timeweb.cloud`.**
21. Telegram API из РФ-ЦОД только по IPv6. **IPv6 в Docker.**
22. Healthcheck Coolify через curl внутри alpine → откат здорового
    контейнера. **`node -e fetch` в HEALTHCHECK.**
23. Пустые Watch Paths → 22 сборки за час. **Watch Paths у каждого
    приложения.**
24. 20 задач CI → минуты кончились за три недели. **Одна задача шагами,
    проверки по затронутому.**
25. PostgREST режет ответ до 1 000 строк без признака. **`loadPaged` и
    курсоры.**
26. `created_at` одинаков у строк одного импорта → порядок плавает.
    **Хвост сортировки — `id`.**
27. `cache()`-клиент Supabase в таймере пережил свой токен. **Комнаты
    и воркер — свой клиент без `cache()`.**
28. Второе серверное действие поверх идущего замораживает роутер
    (Next 15.5 / React 19.2). **Одно действие на вкладку, `form-busy`.**
29. Константа из модуля `"use client"` на сервере — ссылка-объект.
    **Общие значения — в модуле без директив.**
30. `redirect` на тот же путь после действия — пустой экран; действие,
    остающееся на странице, заканчивается `revalidatePath`.
31. Сорвавшийся запрос ≠ пустые данные: функции возвращают `null`, а не
    `[]`, и ошибку не кешируют как «нет такого».
32. Запрос разрешения на push — только из обработчика тапа; смена VAPID
    требует `unsubscribe`; у каждого обещания — лимит времени.
33. Публичный бакет для «закрытых» материалов — «нельзя скачивать» было
    надписью. **Все бакеты приватные с первого дня.**
34. Один буфер пакетной записи на процесс → одна личность останавливала
    приём для всех. **Буферы на тенанта; отвергнутый пакет делится пополам.**
35. Очки начислялись дважды параллельными ответами. **`returning`
    подтверждает вставку; advisory-замки; `concurrency.sh`.**
