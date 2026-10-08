# Runbook: монорепо `ls-russia` и перенос механики из Eventbase

Статус: 2026-10-08. Выполняется одной сессией Claude Code на этапе 0
(после сервера или параллельно с ним). Итог — пустой портал с входом
по паролю и TOTP, `/api/health`, первой миграцией ядра, тестом RLS и
CI, выкатывающийся в Coolify по push в `main`.

## 1. Структура

```
ls-russia/
  apps/
    portal/            # портал сотрудников и собственников (Next.js 15); он же воркер при PORTAL_ROLE=worker
    client/            # клиентское приложение — релиз 3, папка пустая до тех пор
  packages/
    shared/            # @ls/shared: доменные типы, зеркалящие схему; коды прав; схема редактора; client-ip
  supabase/
    migrations/        # <YYYYMMDDHHMMSS>_<имя>.sql — только добавление
    tests/             # run.sh, 00-supabase-shim.sql, 01-core-rls.sql, …, concurrency.sh
    apply.sh           # применение миграций с журналом app.schema_migrations
  scripts/             # ci-affected.sh, check-db-contract.mjs, check-csp.mjs, check-service-worker.mjs,
                       # check-service-role.mjs, check-doc-size.mjs, bootstrap-admin.mjs, mfa-reset.mjs
  docs/
    architecture/      # план, модель данных, продукты, безопасность, ADR
    setup/             # runbook'и
    patterns/          # правила по темам, на которые ссылается CLAUDE.md (чтобы он не рос до 500 КБ)
  .github/workflows/ci.yml
  CLAUDE.md  README.md  package.json  pnpm-workspace.yaml  pnpm-lock.yaml  .npmrc  .dockerignore  .gitignore
```

Именование пакетов: `@ls/shared`, приложения — `portal`, `client`
(фильтры pnpm: `pnpm --filter portal build`).

## 2. Инициализация

```bash
pnpm init                                # корень: "private": true, "packageManager": "pnpm@10.x"
printf 'packages:\n  - "apps/*"\n  - "packages/*"\n' > pnpm-workspace.yaml
pnpm dlx create-next-app@latest apps/portal --ts --app --tailwind --src-dir --no-eslint --import-alias "@/*"
mkdir -p packages/shared/src && cat > packages/shared/package.json <<'JSON'
{ "name": "@ls/shared", "version": "0.0.0", "private": true,
  "main": "./src/index.ts", "types": "./src/index.ts" }
JSON
```

`next.config.ts` портала (по образцу Eventbase):

```ts
import path from "node:path";
const config = {
  output: "standalone",
  outputFileTracingRoot: path.join(__dirname, "../../"),
  transpilePackages: ["@ls/shared"],
  serverExternalPackages: ["nodemailer", "sharp"],
  experimental: { serverActions: { bodySizeLimit: "4mb" } },   // файлы идут в Storage напрямую
};
export default config;
```

Заголовки безопасности — не в `next.config`, а в `middleware.ts` из
окружения (`lib/security-headers.ts` Eventbase). ESLint намеренно не
ставится (его нет и в Eventbase; строгие типы и сборка — проверка),
Prettier — по желанию владельца.

Зависимости портала на старте: `next`, `react`, `react-dom`,
`@supabase/supabase-js`, `@supabase/ssr`, `jose`, `web-push`, `sharp`,
`sanitize-html`, `@sentry/nextjs`, `@tiptap/react` + `@tiptap/starter-kit`
(к этапу 2), `tailwindcss` + `@tailwindcss/postcss`, `typescript`,
`@playwright/test` (dev). Иконки — один модуль `components/icons.tsx`
(Hugeicons, как в Eventbase; эмодзи в интерфейсе запрещены). Шрифты —
файлами в `src/fonts` через `next/font/local`; `next/font/google`
запрещён.

## 3. Что скопировать из Eventbase (с заменой имён)

Копировать `cp`, затем заменить `eventbase` → `ls`, `@eventbase/shared`
→ `@ls/shared`, `admin`/`event-app` → `portal`, `project` → `club`/`org`
по смыслу. Список по порядку зависимости:

| Из Eventbase | В `ls-russia` | Правка |
|---|---|---|
| `supabase/apply.sh` | `supabase/apply.sh` | без правок; `LOCK_KEY` можно оставить |
| `supabase/tests/run.sh`, `00-supabase-shim.sql`, `schema-dump.sql`, `concurrency.sh`, `README.md` | `supabase/tests/` | в шиме добавить клейм `aal` в `auth.jwt()`; харнесс `tst` переписать под роли сети (`as_staff(user)` ставит `aal2`, `as_staff_aal1(user)`, `as_client`, `as_anon`) |
| `scripts/ci-affected.sh`, `.github/workflows/ci.yml` | те же пути | имена приложений; шаги проверок — только существующие |
| `scripts/check-db-contract.mjs`, `check-csp.mjs`, `check-service-worker.mjs` | `scripts/` | список папок приложений и маршрут SW |
| `apps/voting/Dockerfile` (с HEALTHCHECK) | `apps/portal/Dockerfile` | слои: манифесты → install → исходники → build; `CMD` через `docker-entrypoint.sh` с `MIGRATE_ON_START`; без `NODE_EXTRA_CA_CERTS`; `apk add bash postgresql-client` для `apply.sh` |
| `.dockerignore` | корень | `docs` не исключать целиком, если `apps/docs` появится; сейчас можно |
| `apps/admin/src/lib/{env,auth,must-write,write-errors,action-error,form-values,request-body,rate-limit,reset-tokens,password,audit,security-headers,request-meta,sentry,timing,version,paged-rows,parallel,snapshot-cache,signed-urls,client-image,image-pipeline,thumb-url,plural,labels,ui}.ts` | `apps/portal/src/lib/` | `requireProfile` → `requireStaff` (проверяет `aal2` и `status`), `requireProject` → `requireClub`/`requireOrg` |
| `apps/admin/src/lib/supabase/{server,admin,db}.ts` | `apps/portal/src/lib/supabase/` | без правок |
| `apps/admin/src/lib/email.ts` | `apps/portal/src/lib/email.ts` | убрать Resend (один поставщик — Unisender Go), оставить выбор по ключу на будущее |
| `apps/admin/src/middleware.ts` | `apps/portal/src/middleware.ts` | публичные пути: `/login`, `/mfa`, `/reset`, `/invite`, `/api/health`, `/manifest`, `/sw`; `aal1` → только `/mfa` |
| `apps/admin/src/app/login/*`, `reset/*`, `auth/signout/route.ts` | `apps/portal/src/app/(auth)/` | вход: `signInWithPassword` на серверном клиенте **допустим** (сессия `aal1` ничего не видит), затем `/mfa` с `mfa.challengeAndVerify`; регистрация фактора — `mfa.enroll` + QR |
| `apps/admin/src/components/{modal-shell,modal,lazy-modal,confirm-dialog,submit-button,save-scope,instant-button,action-button,keep-values,form-busy,spinner,row-switch,reorder-list,row-actions,server-tabs,tabs,upload-field,upload-gate,admin-shell,sidebar-account,icons}.tsx` | `apps/portal/src/components/` | дизайн-токены свои (`globals.css`, `@theme`), логика без правок |
| `apps/event-app/src/lib/{jobs,worker,heartbeat,push,push-client,live,poll-stream,freshness,chat-tail,cursor,uploads,service-worker,install-platform,safe-path}.ts`, `components/{live-provider,chat-live,chat-list,install-gate,sw-register,push-setup}.tsx`, `app/e/[slug]/{api/live,api/push/subscribe,sw,manifest}/route.ts`, `instrumentation.ts` | `apps/portal/src/lib/`, `components/`, `app/api/…`, `app/{sw,manifest}/route.ts` | «проект» → «сеть», клеймы участника → `app.staff_id()`; слоты и пороги уменьшить (`LIVE_SSE_MAX=2000`, `PER_USER_MAX=4`) |
| `packages/shared/src/client-ip.ts` | `packages/shared/src/client-ip.ts` | без правок |
| Миграции-образцы: `20260719120000_core.sql`, `…foundation.sql` (jobs, push_subscriptions), `20260728100000…160000` (`auth_rate_events`, `auth_reset_tokens`, `audit_log`), `20260819120000_platform_settings.sql`, `20261206120000_claim_job_stale.sql`, `20261213120000_claim_jobs_batch.sql`, `20270109120000_attendee_row_guards.sql` (`only_columns`), `20260830120000_rls_split_delete_policies.sql` (шаблон четырёх политик) | `supabase/migrations/<дата>_core.sql` — **одна** первая миграция, собранная из образцов | таблицы и функции из `02-data-model.md` § 3, 5 |
| `docs/setup/10-fonts.md`, файлы шрифтов | `docs/setup/`, `apps/portal/src/fonts/` | гарнитуру выбирает владелец; до выбора — Golos Text + Onest из Eventbase |
| `apps/registration/src/lib/payments/*` | `packages/shared` или `apps/client` | релиз 3 |

Что **не** копировать: всё с `project_`, `attendee_`, `event_app_`,
`voting_`, `checkin_`, `quest`, `venue_map`, `registration_` в имени;
`app.copy_*`; расширения; `certs/`; i18n-словари.

## 4. Первая миграция ядра — контрольный список

- схема `app`, `grant usage` ролям; `app.set_updated_at`, `app.only_columns`;
- enum'ы `platform_role`, `org_role`, `club_role`, `job_status`, `club_status`;
- таблицы из `02-data-model.md` § 5 с индексами на все внешние ключи и
  `(club_id, …)`;
- функции прав из § 3 (все `stable security definer set search_path = ''`,
  `revoke from public, anon`, `grant to authenticated, service_role`,
  `comment on function`);
- триггер `app.handle_new_user()` на `auth.users` → `profiles`;
- четыре политики по операциям на каждую таблицу; служебные — RLS без
  политик; колоночные `grant update (full_name, phone, avatar_path)` на
  `profiles`;
- `jobs` + `claim_jobs` + `jobs_sweep`; `notifications` + сторож
  `only_columns('read_at')`;
- бакеты Storage: `avatars`, `kb-files`, `task-files`, `chat-files` —
  **все приватные**, `allowed_mime_types` и `file_size_limit` заданы;
- `pg_cron`-задачи: `jobs_sweep` ночью, обезличивание IP в `audit_log`,
  чистка `auth_rate_events`;
- тест `01-core-rls.sql` с фикстурой и матрицей; `02-delete-cascade.sql`;
  `03-claim-job.sql`; блок «aal1 не видит ничего».

## 5. CI (`.github/workflows/ci.yml`)

Копия Eventbase, задачи:

1. `checks` «Быстрые проверки» — шагами: `ci-affected.sh`,
   `check-csp`, `check-service-worker`, `check-service-role`,
   `check-doc-size`, `node --test packages/shared/src/*.test.mjs`,
   `pnpm audit --prod` (не блокирует).
2. `sql` «Миграции и изоляция (RLS)» — `apt-get install postgresql` +
   `bash supabase/tests/run.sh`, если затронут `supabase/`.
3. `build` — `pnpm --filter <app> build` по затронутым.
4. `e2e` — Playwright по затронутым, когда появятся стенды (этап 4:
   чаты и SSE — первые кандидаты, стенд `e2e/harness` из event-app).

Триггеры: `pull_request`, `workflow_dispatch`, `schedule` в понедельник
утром; на push в `main` CI не идёт — сборку делает Coolify. Лимит Free —
2 000 минут; при исчерпании PR вливаются по локальной сборке, как
решил владелец Eventbase 26.09.2026, а не покупаются минуты.

## 6. Coolify

Приложения `portal` и `portal-worker` по `01-timeweb-server.md` Этап 5.
Watch Paths обязательны. После первого выката:

```bash
# из веб-терминала контейнера portal
node scripts/bootstrap-admin.mjs --email owner@<домен>   # создаёт super_admin, печатает пароль один раз
```

Дальше — вход, TOTP, приглашение второго человека.

## 7. Первый коммит `CLAUDE.md`

Корневой `CLAUDE.md` из этого репозитория уже написан (см. корень); при
создании `apps/portal` добавляется `apps/portal/CLAUDE.md` с разделами
«Паттерны», «Словарь интерфейса», «Маршруты», «Проверка» — по образцу
`apps/admin/CLAUDE.md` Eventbase, но с бюджетом размера (≤ 60 КБ) и
вынесением тем в `docs/patterns/`.
