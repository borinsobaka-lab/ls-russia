# Runbook: монорепо с нуля

Статус: 2026-10-08. Выполняется одной сессией Claude Code на этапе 0.
Итог — портал с входом по паролю и TOTP, `/api/health`, первой
миграцией ядра, тестом RLS и CI, выкатывающийся в Coolify по push.

## 1. Структура

```
ls-russia/
  apps/
    portal/            # портал (Next.js 15); он же воркер при PORTAL_ROLE=worker
    client/            # клиентское приложение — релиз 3
  packages/
    shared/            # @ls/shared: типы, зеркалящие схему; коды прав; схема редактора; разбор IP
  supabase/
    migrations/        # <YYYYMMDDHHMMSS>_<имя>.sql — только добавление
    tests/             # run.sh, 00-supabase-shim.sql, 01-core-rls.sql, …, concurrency.sh
    apply.sh           # применение миграций с журналом app.schema_migrations
  scripts/             # ci-affected.sh, check-db-contract.mjs, check-csp.mjs, check-service-worker.mjs,
                       # check-service-role.mjs, check-doc-size.mjs, bootstrap-admin.mjs, mfa-reset.mjs
  docs/architecture/   docs/setup/   docs/patterns/
  .github/workflows/ci.yml
  CLAUDE.md  README.md  package.json  pnpm-workspace.yaml  pnpm-lock.yaml  .npmrc  .dockerignore  .gitignore
```

Пакеты: `@ls/shared`, `portal`, `client` (`pnpm --filter portal build`).

## 2. Инициализация

```bash
pnpm init                                   # "private": true, "packageManager": "pnpm@10.x"
printf 'packages:\n  - "apps/*"\n  - "packages/*"\n' > pnpm-workspace.yaml
pnpm dlx create-next-app@latest apps/portal --ts --app --tailwind --src-dir --no-eslint --import-alias "@/*"
mkdir -p packages/shared/src && cat > packages/shared/package.json <<'JSON'
{ "name": "@ls/shared", "version": "0.0.0", "private": true,
  "main": "./src/index.ts", "types": "./src/index.ts" }
JSON
```

`apps/portal/next.config.ts`:

```ts
import path from "node:path";
export default {
  output: "standalone",
  outputFileTracingRoot: path.join(__dirname, "../../"),
  transpilePackages: ["@ls/shared"],
  serverExternalPackages: ["nodemailer", "sharp"],
  experimental: { serverActions: { bodySizeLimit: "4mb" } },  // файлы идут в Storage напрямую
};
```

Заголовки безопасности — в `middleware.ts` из окружения, не в
`next.config`. `tsconfig`: `strict`, `moduleResolution: "bundler"`,
`paths: {"@/*": ["./src/*"]}`.

Зависимости портала на старте: `next`, `react`, `react-dom`,
`@supabase/supabase-js`, `@supabase/ssr`, `jose`, `zod`, `web-push`,
`sharp`, `sanitize-html`, `@sentry/nextjs`, `@hugeicons/react` +
`@hugeicons/core-free-icons`; к этапу 2 — `@tiptap/react`,
`@tiptap/starter-kit`; dev — `tailwindcss`, `@tailwindcss/postcss`,
`typescript`, `@playwright/test`. Шрифты — файлами в `src/fonts` через
`next/font/local`.

## 3. Слой `lib/` портала (что написать первым)

| Модуль | Назначение |
|---|---|
| `lib/env.ts` | `env(name)` — бросает понятную ошибку по-русски на пустой переменной |
| `lib/supabase/server.ts`, `admin.ts` | клиент с куками пользователя (`@supabase/ssr`); клиент `service_role` без сессии — только в модулях из ADR platform-001 |
| `lib/auth.ts` | `requireStaff()` в `cache()`: пользователь, `aal2`, профиль активен; `requireOrgAdmin()`, `requireClub(id)` → `{role, canManage}` |
| `lib/must-write.ts`, `lib/write-errors.ts` | запись с разбором `error` PostgREST и редиректом `?error=<класс>` |
| `lib/action-error.ts` | устаревшее серверное действие после выката → перезагрузка |
| `lib/rate-limit.ts` | `rateAllow`/`rateHit` на таблице `auth_rate_events` |
| `lib/auth/invitations.ts`, `lib/auth/reset-tokens.ts`, `lib/auth/mfa.ts` | приглашения, сброс пароля, сброс фактора через админ-API |
| `lib/audit.ts` | запись в `audit_log` (best effort) |
| `lib/security-headers.ts` + `middleware.ts` | CSP, HSTS, Permissions-Policy из окружения; защищённые пути; `aal1` → `/mfa` |
| `lib/request-meta.ts` | `clientIp()` по `CLIENT_IP_HEADER`, внешний хост через свой заголовок |
| `lib/email.ts` | Unisender Go по HTTP; шаблоны писем таблицами с инлайновыми стилями и текстовой частью |
| `lib/jobs.ts`, `lib/worker.ts`, `lib/heartbeat.ts`, `instrumentation.ts` | очередь, воркер по `PORTAL_ROLE`, пульс |
| `lib/notifications.ts` | постановка `notify.*` в очередь, чтение ленты |
| `lib/uploads.ts`, `lib/signed-urls.ts`, `lib/image-pipeline.ts` | подписанные URL на загрузку и чтение, пережатие |
| `lib/live.ts`, `lib/poll-stream.ts`, `lib/freshness.ts`, `app/api/live/route.ts`, `components/live-provider.tsx` | SSE-канал (этап 4, но каркас — в ядре ради ярлыков уведомлений) |
| `lib/sentry.ts`, `instrumentation-client.ts`, `app/api/{health,version}/route.ts` | ошибки и здоровье |
| `lib/cursor.ts`, `lib/paged-rows.ts`, `lib/parallel.ts` | курсоры, страницы PostgREST, `mapLimit` |
| `lib/ui.ts`, `components/*` | классы-константы и примитивы: кнопка, поле, окно, таблица, вкладки, загрузка файла, подтверждение |
| `lib/version.ts` | номер сборки, повышается каждым коммитом в `main` |

## 4. Первая миграция ядра — контрольный список

- схема `app`, `grant usage`; `app.set_updated_at`, `app.only_columns`;
- enum'ы `platform_role`, `org_role`, `club_role`, `club_status`,
  `job_status`;
- таблицы из `03-data-model.md` § 5 с индексами на FK и `(club_id, …)`;
- функции прав из § 3 (`stable security definer set search_path = ''`,
  `revoke from public, anon`, `grant to authenticated, service_role`);
- триггер на `auth.users` → `profiles`;
- четыре политики на каждую таблицу; служебные — RLS без политик;
  колоночный `grant update (full_name, phone, avatar_path)` на `profiles`;
- `jobs` + `claim_jobs` + `jobs_sweep`; `notifications` + сторож
  `only_columns('read_at')`;
- бакеты `avatars`, `kb-files`, `task-files`, `chat-files` — приватные,
  с `allowed_mime_types` и `file_size_limit`;
- `pg_cron`: `jobs_sweep` ночью, обезличивание IP в `audit_log`, чистка
  `auth_rate_events`;
- тесты `01-core-rls.sql` (фикстура, матрица, блок «aal1 не видит
  ничего»), `02-delete-cascade.sql`, `03-claim-job.sql`.

## 5. `supabase/apply.sh`

- Журнал `app.schema_migrations(name, checksum sha256, applied_at)`.
- Режимы: без аргументов — применить непринятое; `--status`; `--dry-run`;
  `--baseline` — пометить всё применённым (разово, для базы, накатанной
  руками).
- Один сеанс `psql -v ON_ERROR_STOP=1`: `pg_advisory_lock`, на каждый
  файл `begin; <файл>; insert into app.schema_migrations; commit;`, в
  конце `notify pgrst, 'reload schema'`.
- Изменённый применённый файл — остановка с ошибкой.
- Сторож разрушающих команд: `drop table|column|schema|database`,
  `truncate` вне строк-комментариев → остановка без `ALLOW_DESTRUCTIVE=1`.
- Нужен только `psql` (`apk add postgresql-client bash` в runner-образе).

## 6. Тестовый стенд `supabase/tests/run.sh`

Без Docker: берёт локальный PostgreSQL (`initdb`, `pg_ctl` на
unix-сокете во временном каталоге), создаёт базу, накатывает шим
Supabase (роли, схемы `auth`/`storage`, `auth.jwt()` с `nullif`,
`auth.uid()`, default privileges как в Supabase), затем все миграции
**тем же `apply.sh`**, затем слепок схемы → `check-db-contract.mjs`,
затем файлы `[0-9][0-9]-*.sql` по порядку в одной базе, затем
`concurrency.sh`. Роли в тестах — `set_config('role', …, true)` и
`set_config('request.jwt.claims', …, true)`. В CI —
`apt-get install postgresql` и запуск скрипта.

## 7. Dockerfile портала

```dockerfile
FROM node:22-alpine AS builder
ENV COREPACK_ENABLE_DOWNLOAD_PROMPT=0
RUN corepack enable
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY apps/portal/package.json apps/portal/
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm --filter portal build

FROM node:22-alpine AS runner
WORKDIR /app
RUN apk add --no-cache bash postgresql-client
ENV NODE_ENV=production PORT=3000 HOSTNAME=0.0.0.0
COPY --from=builder --chown=node:node /app/apps/portal/.next/standalone ./
COPY --from=builder --chown=node:node /app/apps/portal/.next/static ./apps/portal/.next/static
COPY --from=builder --chown=node:node /app/apps/portal/public ./apps/portal/public
COPY --from=builder --chown=node:node /app/supabase ./supabase
COPY --from=builder --chown=node:node /app/scripts ./scripts
COPY --chown=node:node apps/portal/docker-entrypoint.sh ./
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.PORT||3000)+'/api/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["bash", "docker-entrypoint.sh"]
```

`docker-entrypoint.sh`: если `MIGRATE_ON_START=1` — `bash supabase/apply.sh`
(падение = выход с ошибкой, контейнер не стартует); затем
`exec node apps/portal/server.js`. `.dockerignore`: `**/node_modules`,
`**/.next`, `.git`, `**/.env*` (кроме `.env.example`).

## 8. CI (`.github/workflows/ci.yml`)

1. `checks` «Быстрые проверки» — шагами: `scripts/ci-affected.sh`
   (выходы `apps`, `sql`, `e2e`), `check-csp`, `check-service-worker`,
   `check-service-role`, `check-doc-size`, `node --test
   packages/shared/src/*.test.mjs`, `pnpm audit --prod` (не блокирует).
2. `sql` «Миграции и изоляция (RLS)» — при затронутом `supabase/`.
3. `build` — `pnpm --filter <app> build` по затронутым.
4. `e2e` — Playwright по затронутым, когда появятся стенды (этап 4).

Триггеры: `pull_request`, `workflow_dispatch`, `schedule` в понедельник
утром — полный прогон. На push в `main` CI не идёт — сборку делает
Coolify. `concurrency` с `cancel-in-progress`. Лимит Free — 2 000 минут;
при исчерпании PR вливаются по локальной сборке.

`ci-affected.sh`: приложение затронуто, если изменилась его папка;
`packages/shared` затрагивает всех, кто от него зависит; корневые
манифесты, `ci.yml` и сам скрипт — всё; `supabase/` → `sql=true`.

## 9. После первого выката

```bash
node scripts/bootstrap-admin.mjs --email owner@<домен>   # из веб-терминала контейнера portal
```

Вход, TOTP, приглашение второго человека. Затем —
`apps/portal/CLAUDE.md` с разделами «Паттерны», «Словарь интерфейса»,
«Маршруты», «Проверка» (бюджет ≤ 60 КБ, темы — в `docs/patterns/`).
