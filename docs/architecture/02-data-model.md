# Модель данных и система доступов

Статус: предложено — 2026-10-08. Реализуется первой миграцией
`supabase/migrations/<дата>_core.sql` по образцу `20260719120000_core.sql`
Eventbase; формальные решения — ADR `platform-001-tenancy-roles` и
`platform-002-staff-auth`.

## 1. Иерархия

```
orgs (сеть; одна строка — Lady Stretch; задел под другие сети)
  ├── org_members      (сотрудники управляющей компании: admin / staff)
  ├── franchisees      (юрлица-франчайзи: реквизиты, договор; атрибут группировки)
  ├── clubs            (студии: город, адрес, часовой пояс, статус, franchisee_id)
  │     └── club_members   (люди клуба: owner / manager / administrator / trainer)
  ├── clients          (клиенты сети — РЕЛИЗ 3; принадлежат сети, а не клубу)
  │     └── client_club_links (в какие клубы ходит; абонементы — отдельно)
  └── продукты: kb_* (база знаний), tasks_* (задачи), chat_* (чаты), позже crm_*, schedule_*, membership_*

profiles (все пользователи портала: сотрудники УК, собственники, персонал клубов)
```

Ключевые принципы (переносятся из Eventbase):

- **Доступ вычисляется по строкам членства, а не по полю «мой клуб» у
  человека.** Собственник пяти студий — пять строк `club_members`.
  Куратор УК, прикреплённый к региону, строк `club_members` не получает:
  сотрудники УК видят **все** клубы через `org_members`, а «кураторство»
  — атрибут клуба (`clubs.curator_id`) для маршрутизации задач и чатов,
  не механизм доступа.
- **Франчайзи — атрибут, а не уровень доступа.** `clubs.franchisee_id`
  нужен отчётам и фильтрам; права человека считаются по клубам.
- **Клиенты принадлежат сети, а не клубу** (аналог «участник принадлежит
  проекту» в Eventbase): один человек ходит в две студии, покупает
  абонемент в одной, занимается в другой. Личность клиента одна —
  телефон. Это решение фиксируется сейчас, хотя таблицы клиентов
  появятся в релизе 3: иначе мессенджер с клиентами и CRM получат
  дубли одного человека по числу клубов.
- **Продукты — модули над общим ядром**: своих таблиц много, но ссылки
  всегда на `org_id` и, где уместно, на `club_id`; никаких jsonb-свалок
  вместо таблиц.

## 2. Роли — три уровня

**Платформенная роль** (`profiles.platform_role`):

| Роль | Кто | Права |
|---|---|---|
| `super_admin` | владелец платформы и ядро команды | всё, включая настройку сети, расширений, журнал, аварийный сброс второго фактора |
| `member` | все остальные | только то, что дают членства ниже |

**Роль в сети** (`org_members.role`) — управляющая компания:

| Роль | Права |
|---|---|
| `admin` | заводит клубы, людей и роли, пишет базу знаний, создаёт доски задач и каналы, видит все клубы и всю аналитику |
| `staff` | сотрудник УК: видит все клубы, работает в задачах и чатах, читает базу знаний; пишет в неё, если выдано право `kb.editor` |

Тонкие права сотрудников УК (кто редактирует базу знаний, кто ведёт
массовые поручения, кто видит финансы) — **набор строковых кодов в
`org_members.permissions text[]`**, не отдельные роли: отделов в УК
больше, чем разумно заводить enum'ов, а текстовый код копируется и
расширяется без миграции (урок ролей участников Eventbase, ADR
event-app-013).

**Роль в клубе** (`club_members.role`):

| Роль | Кто | Права в клубе |
|---|---|---|
| `owner` | собственник (франчайзи или его представитель) | всё по клубу: люди, задачи, чаты, позже клиенты и финансы |
| `manager` | управляющий | как owner, кроме финансов и смены состава владельцев |
| `administrator` | администратор на ресепшене | задачи и чаты клуба; позже — клиенты, расписание, запись |
| `trainer` | тренер | свои задачи и чаты; позже — своё расписание и свои группы |

Что видно кому в первом релизе — таблица в `03-products.md`, § «Матрица
видимости».

**Роли клиентов** (релиз 3) — отдельный мир со своими клеймами, как у
участника Eventbase: `cl_client_id`, `cl_org_id`. Пространство имён
клеймов **своё** — токен клиента не должен проходить ни одну политику
сотрудников (урок ADR event-app-005: назови клейм `org_id` — и клиент
прошёл бы политики УК).

## 3. Функции прав (схема `app`)

Определяются один раз, в первой миграции, `language sql stable security
definer set search_path = ''`, с `revoke execute from public, anon` и
`grant execute to authenticated, service_role`. Образец — `core.sql`
Eventbase; здесь добавлено одно принципиальное отличие — **второй фактор
проверяется в базе**:

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

-- Запись по клубу: НИКОГДА не возвращает null (урок can_edit_not_null Eventbase).
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

-- Клеймы клиентского мира (релиз 3), читаются ТОЛЬКО через nullif:
create or replace function app.client_id() returns uuid ... as $$
  select nullif(coalesce((select auth.jwt()) ->> 'cl_client_id', ''), '')::uuid;
$$;
```

`auth.jwt()` Supabase уже читает `request.jwt.claims` через
`nullif(…, '')` — своих приведений `current_setting(...)::jsonb` в коде
не пишем никогда (22P02 на пуле соединений, разбор в миграции
20261230120000 Eventbase).

## 4. Шаблон RLS для таблицы продукта

Четыре политики по операциям (`for all` запрещён: у DELETE нет
`with check`, и шаблон `using(читать) with check(писать)` разрешает
удалять тому, кто может читать — Eventbase чинил это на 23 таблицах):

```sql
alter table public.tasks enable row level security;

create policy tasks_select on public.tasks for select to authenticated
  using (app.task_visible(id));                       -- security definer, см. ниже
create policy tasks_insert on public.tasks for insert to authenticated
  with check (app.task_can_create(board_id));
create policy tasks_update on public.tasks for update to authenticated
  using (app.task_can_edit(id)) with check (app.task_can_edit(id));
create policy tasks_delete on public.tasks for delete to authenticated
  using (app.task_can_delete(id));
```

Правила, которые действуют на каждую новую таблицу (и проверяются тестом
`supabase/tests/01-core-rls.sql`, матрица «кто что видит»):

1. **Условие с подзапросом к невидимой вызывающему таблице выносится в
   security-definer-функцию**, не пишется подзапросом в политике:
   подзапрос молча даст ложь, и законная запись будет отвергнута при
   исправной с виду схеме.
2. **Stable-функции в политике — в скобках `(select app.f())`**: так это
   InitPlan, один вызов на запрос, а не на строку. На каталоге из 5 000
   строк это дало 41 → 4,7 мс в Eventbase.
3. **Update от имени рядового участника стережёт триггер-белый список
   колонок** `app.only_columns('col1','col2')` (копия
   `app.attendee_only_columns`), потому что `with check` видит только
   новую строку. Классический случай — `chat_members.last_read_message_id`:
   право «отметить прочитанным» без сторожа позволяло переставить
   `chat_id` и читать чужую переписку.
4. **В `with check` сверяется проект/сеть ЦЕЛИ ссылки**, а не только
   своя: лайк на чужое сообщение, вложение к чужой задаче.
5. **RLS фильтрует строки, не колонки.** Секрет в строке, которую читает
   широкий круг (пароль SMTP-профиля, токен интеграции) — либо колоночный
   `revoke`, либо отдельная таблица без политик, читаемая только
   сервером.
6. **Служебные таблицы** (`jobs`, `audit_log`, `auth_*`, сессии клиентов,
   подписки push) — RLS включён, политик нет: только `service_role`.
7. **Хвост сортировки — уникальная колонка** (`id`): порядок равных строк
   Postgres меняет после каждого `update`.
8. **Таблица с файлами** знает свой бакет через `app.file_bucket(table,
   column)`, и удаление строки возвращает вызывающему список путей:
   Postgres до Storage не дотягивается, файлы удаляет приложение — и
   делает это всегда (урок «удаляя строку с файлом, удаляй и файл»).

## 5. Таблицы ядра (первая миграция)

| Таблица | Ключевые колонки | Политики |
|---|---|---|
| `profiles` | `id → auth.users`, `email unique`, `full_name`, `phone`, `avatar_path`, `platform_role`, `status (active/blocked)`, `mfa_enrolled_at`, `last_seen_at`, `locale` | свою строку читает и правит (только `full_name`, `phone`, `avatar_path` — колоночный grant); чужие — через RPC `people_cards(ids)` без телефона, кроме коллег по клубу |
| `orgs` | `id`, `name`, `slug`, `settings jsonb` | читают члены; пишет super_admin |
| `org_members` | PK `(org_id, user_id)`, `role`, `permissions text[]`, `title` (должность), `department`, `invited_by` | читают члены сети; пишет org admin |
| `franchisees` | `id`, `org_id`, `legal_name`, `inn`, `contract_no`, `contract_until`, `contact_*` | читают org staff и собственники своих клубов; пишет org admin |
| `clubs` | `id`, `org_id`, `franchisee_id`, `name`, `city`, `address`, `timezone`, `status (opening/open/closed)`, `opened_at`, `curator_id → profiles`, `settings jsonb` | `has_club_access` / `can_manage_club`; создание — org admin |
| `club_members` | PK `(club_id, user_id)`, `role`, `invited_by` | читают: сами члены клуба и org staff; пишет `can_manage_club`; owner'а назначает только org admin |
| `invitations` | `id`, `org_id`, `email`, `token_hash`, `kind`, `payload jsonb` (куда и с какой ролью), `expires_at`, `used_at`, `invited_by` | только service_role |
| `audit_log` | `id bigint`, `created_at`, `action`, `actor_id`, `actor_email`, `target_type`, `target_id`, `club_id`, `ip`, `meta` | только service_role; IP обезличивается через 365 дней |
| `platform_settings` | `key`, `value jsonb`, `updated_by` | читают вошедшие, пишет super_admin |
| `user_prefs` | PK `(user_id, key)`, `value jsonb` | только своя строка |
| `files` | `id`, `org_id`, `bucket`, `path unique`, `owner_id`, `size`, `mime`, `sha256`, `scan_status`, `created_at` | реестр всего, что лежит в Storage: удаление строк-носителей находит файлы здесь; только service_role |
| `jobs` | как в Eventbase: `kind`, `payload`, `status`, `run_at`, `attempts`, `max_attempts`, `locked_by`, `locked_at`, `last_error`; RPC `claim_jobs(kinds, locked_by, limit)`, `jobs_sweep` | только service_role |
| `notifications` | `id bigint`, `user_id`, `kind`, `vars jsonb`, `href`, `subject_key`, `read_at`, `created_at` | своя строка: select; update только `read_at` через `only_columns` |
| `notification_prefs` | PK `(user_id, kind)`, `channels text[]` (`inapp`, `push`, `email`, `telegram`), `quiet_from`, `quiet_to` | своя строка |
| `push_subscriptions` | как в Eventbase + `user_id` | только service_role (пишет сервер после проверки сессии) |
| `auth_rate_events` | `bucket`, `created_at` | только service_role — лимиты входа живут в базе, а не в памяти: реплик несколько |
| `telegram_links` | `user_id`, `chat_id`, `linked_at`, `link_code_hash` | только service_role |

Таблицы продуктов (`kb_*`, `tasks_*`, `chat_*`) — в `03-products.md`.
Таблицы клиентского мира (`clients`, `client_sessions`, `client_consents`,
`client_login_codes`) — релиз 3, но их **форма** зафиксирована там же,
чтобы ядро не пришлось перекраивать.

## 6. Три мира авторизации

| Мир | Кто | Носитель личности | Как ходит в базу |
|---|---|---|---|
| Сотрудники | УК, собственники, персонал клубов | Supabase Auth (GoTrue): почта + пароль + TOTP, сессия в куках `@supabase/ssr` | anon-ключ + JWT пользователя; политики через `app.staff_id()` (требует `aal2`) |
| Клиенты (релиз 3) | клиенты студий | своя таблица `client_sessions` (sha256 случайного токена в куке), вход по телефону (код по SMS / через бота) | сервер подписывает `SUPABASE_JWT_SECRET` короткий (2 мин) JWT `{role: authenticated, cl_client_id, cl_org_id}`; политики через `app.client_id()` |
| Интеграции | вебхуки мессенджеров, платёжный шлюз, 1С/внешние системы | ключ API (sha256 в таблице) или подпись HMAC | только `service_role` внутри узкого серверного модуля после проверки подписи |

`service_role` разрешён в перечисленном списке модулей — список живёт в
ADR `platform-001` и расширяется только правкой ADR (как ADR event-app-005
у Eventbase): модуль входа и приглашений, воркер очереди, загрузки в
Storage, интеграции, аналитика по расписанию. Любой другой запрос идёт
ключом пользователя под RLS — так класс ошибок «забыли фильтр» исключён
структурно, а аудит безопасности сводится к четырём-пяти файлам.

## 7. Что гарантирует исполняемый тест

`bash supabase/tests/run.sh` поднимает одноразовый Postgres, накатывает
шим Supabase и все миграции **тем же `apply.sh`**, что и площадка, затем
гоняет сценарии:

- `01-core-rls.sql` — харнесс `tst` (`as_staff(user, aal2)`,
  `as_staff_aal1(user)`, `as_client(client, org)`, `as_anon()`,
  `expect`, `expect_denied`, `expect_touched`, `expect_guarded`) и
  фикстура: сеть, два клуба, собственник клуба А, тренер клуба А,
  сотрудник УК, админ УК, супер-админ. **Матрица видимости**: для
  каждой таблицы с `club_id`/`org_id` — сколько строк видит каждая роль
  и что ей запрещено менять. Новая таблица добавляется в матрицу тем же
  коммитом.
- Отдельный блок: **сессия без второго фактора (`aal1`) не видит ни одной
  строки ни в одной таблице** — это свойство `app.staff_id()`, и тест
  обязан падать, если кто-то напишет политику через `auth.uid()`.
- `02-delete-cascade.sql` — у каждой таблицы с `org_id`/`club_id`
  внешний ключ каскадный; удаление клуба и сети не оставляет сирот.
- `03-claim-job.sql`, `04-notifications.sql`, продуктовые файлы — по
  мере появления.
- `scripts/check-db-contract.mjs` на слепке схемы: колонки в `.select()`,
  аргументы `.rpc()`, однозначность вложенных связей (`PGRST201`).

CI запускает это по каждому PR, затронувшему `supabase/`, задачей
«Миграции и изоляция (RLS)».
