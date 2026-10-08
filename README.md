# Платформа Lady Stretch

Диджитал-продукты сети студий растяжки: портал сотрудников и
собственников (база знаний, задачи, чаты; позже кабинет клуба с
клиентами, расписанием, CRM и единым окном переписки) и клиентское
приложение. Данные — в России (Timeweb Cloud, Supabase на своём
сервере), выкат — Coolify, разработка — Claude Code.

**Состояние (08.10.2026): написан план; кода ещё нет.**

## С чего начать

1. `docs/architecture/00-master-plan.md` — что строим, на чём, в каком
   порядке, какие решения нужны от владельца.
2. `docs/architecture/01-tech-stack.md` — языки, фреймворки, библиотеки.
3. `CLAUDE.md` — правила работы в репозитории.
4. `docs/setup/01-timeweb-server.md` → `02-repo-bootstrap.md` — сервер и
   монорепо с нуля.

## Документы

| Файл | О чём |
|---|---|
| `docs/architecture/02-infrastructure.md` | сервисы, сервер, сеть, бэкапы, стоимость |
| `docs/architecture/03-data-model.md` | иерархия, роли, функции прав, шаблон RLS, таблицы ядра |
| `docs/architecture/04-products.md` | продукты, их таблицы, релизы, матрица видимости |
| `docs/architecture/05-security.md` | модель угроз, меры, 152-ФЗ |
| `docs/architecture/06-engineering-rules.md` | инженерные правила: база, сервер, клиент, PWA, выкат |
| `docs/architecture/adr/` | тенантность и роли, вход со вторым фактором, живые обновления, уведомления, хранение статей |
| `docs/setup/03-claude-code-workflow.md` | как ведём разработку с Claude Code |

## Стек

TypeScript · Next.js 15 · React 19 · Tailwind 4 · pnpm · PostgreSQL +
Supabase (GoTrue, PostgREST, Storage) self-hosted · Coolify · Timeweb
Cloud + S3 · Unisender Go · GlitchTip · Uptime Kuma · GitHub Actions.
