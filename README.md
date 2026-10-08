# Платформа Lady Stretch

Диджитал-продукты для сети студий растяжки: портал сотрудников и
собственников (база знаний, задачи, чаты; позже кабинет клуба с
клиентами, расписанием, CRM и единым окном переписки) и клиентское
приложение. Все данные — в России (Timeweb Cloud, Supabase на своём
сервере), выкат — Coolify, разработка — Claude Code.

**Состояние (08.10.2026): написан план; кода ещё нет.**

## С чего начать

1. `docs/architecture/00-master-plan.md` — что строим, из чего, в каком
   порядке, какие решения нужны от владельца.
2. `CLAUDE.md` — правила работы в репозитории (их читают сессии Claude Code).
3. `docs/setup/01-timeweb-server.md` — покупка и настройка сервера.
4. `docs/setup/02-repo-bootstrap.md` — создание монорепо и перенос
   механики из Eventbase.

## Документы

| Файл | О чём |
|---|---|
| `docs/architecture/01-infrastructure.md` | сервисы, сервер, сеть, бэкапы, стоимость |
| `docs/architecture/02-data-model.md` | иерархия, роли, функции прав, шаблон RLS, таблицы ядра |
| `docs/architecture/03-products.md` | продукты, их таблицы, релизы, матрица видимости |
| `docs/architecture/04-security.md` | модель угроз, меры, 152-ФЗ, регулярные проверки |
| `docs/architecture/05-eventbase-lessons.md` | что берём из Eventbase, что меняем, что не берём |
| `docs/architecture/adr/` | решения: тенантность и роли, вход со вторым фактором, живые обновления, уведомления, хранение статей |
| `docs/setup/03-claude-code-workflow.md` | как ведём разработку с Claude Code |

## Стек

Next.js 15 · React 19 · TypeScript · Tailwind 4 · pnpm · Supabase
(Postgres, GoTrue, PostgREST, Storage) self-hosted · Coolify · Timeweb
Cloud + S3 · Unisender Go · GlitchTip · Uptime Kuma · GitHub Actions.
