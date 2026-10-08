# Модель данных и система доступов

Статус: предложено — 2026-10-08. Реализуется первой миграцией
`supabase/migrations/<дата>_core.sql`; решения — ADR
`platform-001-tenancy-roles` и `platform-002-staff-auth`.

## 1. Иерархия

```
orgs (сеть; одна строка на старте — задел под другие сети)
  ├── org_members        сотрудники управляющей компании: admin / staff (+ коды прав)
  ├── franchisees        юрлица-франчайзи: реквизиты, договор — атрибут для группировки
  ├── clubs              студии: город, адрес, часовой пояс, статус, franchisee_id, curator_id
  │     └── club_members люди клуба: owner / manager / administrator / trainer
  ├── clients            клиенты сети — релиз 3; принадлежат сети, не клубу
  │     └── client_club_links
  └── продукты: kb_* , tasks_* , chat_* , позже crm_* , schedule_* , membership_*

profiles — все пользователи портала (id = auth.users.id)
```

Принципы:

- **Доступ вычисляется по строкам членства, а не по полю «мой клуб».**
  Собственник пяти студий — пять строк `club_members`. Сотрудники УК
  видят все клубы через `org_members`; «кураторство» — атрибут клуба
  (`clubs.curator_id`) для маршрутизации задач и чатов, не механизм
  доступа.
- **Франчайзи — атрибут, не уровень доступа.** `clubs.franchisee_id`
  нужен отчётам; права считаются по клубам.
- **Клиенты принадлежат сети.** Один человек ходит в две студии и
  покупает абонемент в третьей; личность клиента одна — телефон. Это
  фиксируется сейчас, чтобы CRM и переписка релиза 4 не получили дубли
  по числу клубов.
- **Продукты — модули над общим ядром**: свои таблицы со ссылками на
  `org_id` и `club_id`, а не jsonb-свалки.

## 2. Роли — три уровня

**Платформенная** (`profiles.platform_role`): `super_admin` (владелец
платформы и ядро команды — всё, включая настройку сети и аварийный сброс
второго фактора) и `member` (все остальные — только то, что дают
членства).

**В сети** (`org_members.role`): `admin` (заводит клубы, людей и роли,
создаёт доски и каналы, видит всё) и `staff` (сотрудник УК: видит все
клубы, работает в задачах и чатах, читает базу знаний). Тонкие права
сотрудников — **текстовые коды в `org_members.permissions text[]`**
(`kb.editor`, `tasks.assign_network`, `people.manage`, `clubs.manage`,
`finance.read`, `audit.read`): отделов больше, чем разумно заводить
ролей, а код расширяется без миграции. У `admin` все коды неявно.

**В клубе** (`club_members.role`):

| Роль | Кто | Права в клубе |
|---|---|---|
| `owner` | собственник | всё по клубу: люди, задачи, чаты, позже клиенты и финансы |
| `manager` | управляющий | как owner, кроме финансов и состава владельцев |
| `administrator` | ресепшен | задачи и чаты клуба; позже клиенты, расписание, запись |
| `trainer` | тренер | свои задачи и чаты; позже своё расписание и группы |

**Клиенты** (релиз 3) — отдельный мир со своими клеймами
(`cl_client_id`, `cl_org_id`). Пространство имён клеймов своё: токен
клиента не должен проходить ни одну политику сотрудников.

## 3. Функции прав (схема `app`)

Определяются один раз в первой миграции: `language sql stable security
definer set search_path = ''`, `revoke execute from public, anon`,
`grant execute to authenticated, service_role`, `comment on function`.
Ключевое свойство — **второй фактор проверяется в базе**:

```sql
-- Личность сотрудника: только после второго фактора и только активная учётка.
-- Все политики сотрудников используют app.staff_id(), а не auth.uid().
create or replace function app.staff_id() returns uuid
language sql stable security definer set search_path = '' as $$
  select case
    when (select auth.jwt()) ->> 'aal' = 'aal2'
     and exists (select 1 from public.profiles p
                 where p.id = (select auth.uid()) and p.status = 'active')
    then (select auth.uid()) end;
$$;

create or replace function app.is_super_admin() returns boolean ... as $$
  select exists (select 1 from public.profiles p
    where p.id = (select app.staff_id()) and p.platform_role = 'super_admin');
$$;

create or replace function app.org_role(p_org uuid) returns public.org_role ... as $$
  select m.role from public.org_members m
  where m.org_id = p_org and m.user_id = (select app.staff_id());
$$;

create or replace function app.is_org_staff(p_org uuid) returns boolean ... as $$
  select app.is_super_admin() or app.org_role(p_org) is not null;
$$;

create or replace function app.club_role(p_club uuid) returns public.club_role ... as $$
  select m.role from public.club_members m
  where m.club_id = p_club and m.user_id = (select app.staff_id());
$$;

-- Чтение по клубу: сотрудник УК видит все клубы сети, человек клуба — свой.
create or replace function app.has_club_access(p_club uuid) returns boolean ... as $$
  select app.is_org_staff((select c.org_id from public.clubs c where c.id = p_club))
      or app.club_role(p_club) is not null;
$$;

-- Запись по клубу: НИКОГДА не возвращает null — иначе `if not can_manage`
-- в security-definer-функции молча пропустит.
create or replace function app.can_manage_club(p_club uuid) returns boolean ... as $$
  select coalesce(
    app.is_super_admin()
    or app.org_role((select c.org_id from public.clubs c where c.id = p_club)) = 'admin'
    or app.club_role(p_club) in ('owner','manager'), false);
$$;

create or replace function app.has_permission(p_org uuid, p_code text) returns boolean ... as $$
  select coalesce(app.is_super_admin()
    or app.org_role(p_org) = 'admin'
    or exists (select 1 from public.org_members m
               where m.org_id = p_org and m.user_id = (select app.staff_id())
                 and p_code = any (m.permissions)), false);
$$;

-- Клеймы клиентского мира (релиз 3). Клеймы читаются ТОЛЬКО через auth.jwt():
-- внутри него nullif(current_setting('request.jwt.claims', true), ''), а прямое
-- приведение ::jsonb падает на пуле соединений, где настройка читается пустой строкой.
create or replace function app.client_id() returns uuid ... as $$
  select nullif(coalesce((select auth.jwt()) ->> 'cl_client_id', ''), '')::uuid;
$$;
```

Роль сессии внутри security-definer-функции спрашивается у
`current_setting('role', true)`, а не у `current_user` (там уже
владелец функции).

## 4. Шаблон RLS для таблицы продукта

Четыре политики по операциям. `for all` запрещён: у DELETE нет
`with check`, и шаблон `using(читать) with check(писать)` разрешает
удалять тому, кто может читать.

```sql
alter table public.tasks enable row level security;

create policy tasks_select on public.tasks for select to authenticated
  using (app.task_visible(id));                      -- security definer
create policy tasks_insert on public.tasks for insert to authenticated
  with check (app.task_can_create(board_id));
create policy tasks_update on public.tasks for update to authenticated
  using (app.task_can_edit(id)) with check (app.task_can_edit(id));
create policy tasks_delete on public.tasks for delete to authenticated
  using (app.task_can_delete(id));
```

Правила на каждую таблицу (проверяет тест `01-core-rls.sql`):

1. **Условие с подзапросом к таблице, невидимой вызывающему, — в
   security-definer-функцию.** Подзапрос в политике молча даст ложь, и
   законная запись будет отвергнута при исправной с виду схеме.
2. **Stable-функции в политике — в скобках `(select app.f())`**: это
   InitPlan, один вызов на запрос, а не на строку; на тысячах строк
   разница в разы.
3. **Update от имени рядового пользователя стережёт триггер-белый список
   колонок** `app.only_columns('col1','col2')`: `with check` видит только
   новую строку и не знает, что изменилось. Пример — `chat_members`:
   право «отметить прочитанным» без сторожа позволило бы переставить
   `chat_id` и читать чужую переписку.
4. **В `with check` сверяется сеть/клуб ЦЕЛИ ссылки**, а не только своя:
   реакция на чужое сообщение, вложение к чужой задаче.
5. **RLS фильтрует строки, не колонки.** Секрет в строке, которую читает
   широкий круг, — колоночный `revoke` или отдельная таблица без
   политик, читаемая только сервером.
6. **Служебные таблицы** (`jobs`, `audit_log`, `auth_*`, сессии клиентов,
   подписки push) — RLS включён, политик нет: только `service_role`.
7. **Хвост сортировки — уникальная колонка** (`id`): порядок равных строк
   Postgres меняет после каждого `update`.
8. **Таблица с файлами** знает свой бакет (`app.file_bucket(table,
   column)`); удаление строки возвращает пути файлов, их удаляет
   приложение — Postgres до Storage не дотягивается.

Приёмы миграций: `create table if not exists`, `drop policy if exists`
+ `create policy`, новое значение enum отдельным файлом (каждый файл —
своя транзакция), `drop function if exists` при смене сигнатуры (иначе
PostgREST — «function is not unique»), имена триггеров `aa_*` для
сторожей и `zz_*` для хвостовых (BEFORE-триггеры одного события идут по
алфавиту), `app.set_updated_at()` на каждую таблицу с `updated_at`.

## 5. Таблицы ядра (первая миграция)

| Таблица | Ключевые колонки | Политики |
|---|---|---|
| `profiles` | `id → auth.users`, `email unique`, `full_name`, `phone`, `avatar_path`, `platform_role`, `status (active/blocked)`, `mfa_enrolled_at`, `last_seen_at` | свою строку читает и правит (`full_name`, `phone`, `avatar_path` — колоночный grant); чужие карточки — RPC `people_cards(ids)` без телефона, кроме коллег по клубу |
| `orgs` | `id`, `name`, `slug`, `settings jsonb` | читают члены; пишет super_admin |
| `org_members` | PK `(org_id, user_id)`, `role`, `permissions text[]`, `title`, `department`, `invited_by` | читают члены сети; пишет org admin |
| `franchisees` | `id`, `org_id`, `legal_name`, `inn`, `contract_no`, `contract_until`, `contact_*` | читают org staff и собственники своих клубов; пишет org admin |
| `clubs` | `id`, `org_id`, `franchisee_id`, `name`, `city`, `address`, `timezone`, `status (opening/open/closed)`, `opened_at`, `curator_id`, `settings jsonb` | `has_club_access` / `can_manage_club`; создание — org admin |
| `club_members` | PK `(club_id, user_id)`, `role`, `invited_by` | читают члены клуба и org staff; пишет `can_manage_club`; owner'а назначает только org admin |
| `invitations` | `id`, `org_id`, `email`, `token_hash`, `kind`, `payload jsonb`, `expires_at`, `used_at`, `invited_by` | только service_role |
| `audit_log` | `id bigint`, `created_at`, `action`, `actor_id`, `actor_email`, `target_type`, `target_id`, `club_id`, `ip`, `meta` | только service_role; только добавление; IP обезличивается через 365 дней |
| `platform_settings` | `key`, `value jsonb`, `updated_by` | читают вошедшие, пишет super_admin |
| `user_prefs` | PK `(user_id, key)`, `value jsonb` | своя строка |
| `files` | `id`, `org_id`, `bucket`, `path unique`, `owner_id`, `size`, `mime`, `sha256`, `created_at` | реестр всего в Storage; только service_role |
| `jobs` | `id`, `kind`, `payload jsonb`, `status`, `run_at`, `attempts`, `max_attempts`, `locked_by`, `locked_at`, `last_error`; RPC `claim_jobs(kinds, locked_by, limit)` (`for update skip locked`, забирает и зависшие `running` старше 20 минут), `jobs_sweep` | только service_role |
| `notifications` | `id bigint`, `user_id`, `kind`, `vars jsonb`, `href`, `subject_key`, `read_at`, `created_at` | своя строка: select; update только `read_at` через `only_columns` |
| `notification_prefs` | PK `(user_id, kind)`, `channels text[]`, `quiet_from`, `quiet_to` | своя строка |
| `push_subscriptions` | `user_id`, `endpoint unique`, `p256dh`, `auth`, `user_agent`, `last_seen_at`, `revoked_at`, `fail_count` | только service_role |
| `auth_rate_events` | `bucket`, `created_at` | только service_role — лимиты входа в базе, не в памяти |
| `telegram_links` | `user_id`, `chat_id`, `linked_at` | только service_role |

Таблицы продуктов — `04-products.md`. Таблицы клиентского мира
(`clients`, `client_sessions`, `client_consents`, `client_login_codes`)
— релиз 3; их форма зафиксирована там же.

## 6. Три мира авторизации

| Мир | Кто | Носитель личности | Как ходит в базу |
|---|---|---|---|
| Сотрудники | УК, собственники, персонал | GoTrue: почта + пароль + TOTP, сессия в куках | anon-ключ + JWT пользователя; политики через `app.staff_id()` |
| Клиенты (релиз 3) | клиенты студий | своя таблица `client_sessions` (sha256 случайного токена в куке), вход по телефону | сервер подписывает `SUPABASE_JWT_SECRET` короткий (2 мин) JWT `{role: authenticated, cl_client_id, cl_org_id}`; политики через `app.client_id()` |
| Интеграции | мессенджеры, платёжный шлюз, внешние системы | ключ API (sha256 в таблице) или подпись HMAC | только `service_role` внутри узкого модуля после проверки подписи |

`service_role` разрешён в закрытом списке модулей (ADR `platform-001`):
вход и приглашения, воркер очереди, загрузки в Storage, интеграции,
аналитика по расписанию, клиентские сессии. Любой другой запрос идёт
ключом пользователя под RLS — класс ошибок «забыли фильтр» исключён
структурно, а аудит безопасности сводится к нескольким файлам.

## 7. Что гарантирует исполняемый тест

`bash supabase/tests/run.sh` поднимает одноразовый Postgres, накатывает
шим Supabase (роли `anon`/`authenticated`/`service_role`, схемы `auth` и
`storage`, `auth.jwt()`/`auth.uid()`) и все миграции **тем же
`apply.sh`**, что и площадка, затем гоняет сценарии:

- `01-core-rls.sql` — харнесс `tst` (`as_staff(user)` ставит клеймы с
  `aal2`, `as_staff_aal1(user)`, `as_client(client, org)`, `as_anon()`,
  `reset()`; `expect(name, sql, rows)`, `expect_denied`, `expect_touched`
  — RLS «съедает» строки молча, поэтому считается `returning`,
  `expect_rejected`, `expect_guarded`) и фикстура: сеть, два клуба,
  собственник клуба А, тренер клуба А, сотрудник и админ УК,
  супер-админ. **Матрица видимости**: для каждой таблицы с
  `club_id`/`org_id` — сколько строк видит каждая роль и что ей
  запрещено менять. Новая таблица добавляется тем же коммитом.
- Блок «**сессия `aal1` не видит ни одной строки**» — свойство
  `app.staff_id()`; тест падает, если политика написана через `auth.uid()`.
- `02-delete-cascade.sql` — у каждой таблицы с `org_id`/`club_id`
  внешний ключ каскадный; удаление клуба не оставляет сирот.
- `03-claim-job.sql`, продуктовые файлы — по мере появления;
  `concurrency.sh` — гонки на 20 параллельных соединениях (замки,
  идемпотентность).
- `scripts/check-db-contract.mjs` на слепке схемы: колонки в `.select()`,
  аргументы `.rpc()`, подсказки `rel!fk` при двух внешних ключах между
  таблицами.

CI запускает это на каждом PR, затронувшем `supabase/`.
