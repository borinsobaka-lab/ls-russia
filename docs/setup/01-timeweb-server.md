# Runbook: сервер платформы на Timeweb Cloud (Coolify + Supabase)

Статус: черновик к запуску — 2026-10-08. Собран из runbook'ов Eventbase
`11-timeweb-second-contour.md` (запуск российской площадки 21.08.2026,
раздел «Грабли»), `35-eu-self-hosted-supabase.md`, `31-observability-agent-runbook.md`,
`14-admin-emergency-access.md`, `29-client-ip.md`, `28-proxy-body-limits.md`.

Итог: `https://<портал>.<домен>` работает на своём сервере в Москве с
базой на том же сервере, файлами в Timeweb S3, бэкапами в другом ЦОД и
мониторингом. Срок — один рабочий день плюс ожидание верификации домена.

## Легенда

- 🤖 — делает агент (в браузере или по SSH), либо владелец по инструкции.
- 🔴 — только человек: оплата, паспорт/ЕСИА, 2FA, пароли, подписание
  документов.

## Правило про секреты

В процессе появятся: пароль root (если включён), пароль Postgres,
`JWT_SECRET`, `ANON_KEY`, `SERVICE_ROLE_KEY`, ключи S3, ключ Unisender,
VAPID-ключи, пароли панелей. Они живут **только** в менеджере паролей
владельца и в переменных Coolify. В чат, репозиторий и документы —
никогда. `SERVICE_ROLE_KEY`, `JWT_SECRET` и пароль базы — это все данные
компании целиком.

## Три правила, которые нельзя нарушать

1. **Миграции применяет выкат, не человек.** Руками — только разрушающие
   (`drop table/column`, `truncate`), в тихое время, с бэкапом, и затем
   отметка в `app.schema_migrations`.
2. **Никаких адресов в коде.** Домен, адрес базы, отправитель писем,
   адрес S3 — только переменные окружения Coolify.
3. **Образ сервера перед каждым изменением инфраструктуры.** Timeweb
   делает снимок за минуту; восстановление из него — единственный быстрый
   откат, если что-то пошло не так на уровне ОС или Docker.

---

## Этап 0. Аккаунт, деньги, домен

1. 🔴 Аккаунт Timeweb Cloud на правильное юрлицо/ИП (счета — на него),
   2FA в аккаунте, баланс **5 000 ₽** (месяц сервера + S3 + домен с
   запасом). Проект в панели — `ls-platform`.
2. 🔴 Соглашение об обработке персональных данных с Timeweb (в панели,
   раздел документов) — нужно оператору ПД для 152-ФЗ.
3. 🤖 Домен покупается в Timeweb (сразу на их NS). 🔴 Для `.ru`
   требуется верификация владельца (паспорт или ЕСИА) — закладывайте до
   суток.
4. 🤖 В DNS домена все записи с **TTL 300**. Пока сервера нет, записи
   не создаём — только после Этапа 3 (Let's Encrypt требует A-запись до
   первого выката).

Выбор поддоменов (имена — решение владельца; ниже заглушки):

| Поддомен | Что |
|---|---|
| `office.<домен>` | портал сотрудников и собственников (приложение `portal`) |
| `api.<домен>` | Supabase (Kong): REST, Auth, Storage |
| `coolify.<домен>` | панель Coolify |
| `status.<домен>` | Uptime Kuma |
| `errors.<домен>` | GlitchTip |
| `app.<домен>` | клиентское приложение (релиз 3) |
| `files.<домен>` | публичный бакет S3 через CNAME (если понадобится) |

## Этап 1. Сеть

1. 🤖 Создать **VPC (приватную сеть)** в Москве — только в Москве и СПб
   включена DDoS-защита L3–L4. Все серверы платформы будут в этой сети.
2. 🤖 Создать **группу файрвола** `ls-core`:
   - входящие: 22/tcp — только с IP владельца (и из подсети VPC);
     80/tcp и 443/tcp — отовсюду; остальное закрыто;
   - **исходящие: разрешить TCP, UDP, ICMP по IPv4 и IPv6** — группа
     файрвола Timeweb работает как allow-list и на исходящий трафик;
     без этого сервер не достучится ни до GitHub, ни до S3, ни до
     Let's Encrypt.
   - порт 8000 (первый вход в Coolify) открывать на 15 минут и закрыть.

## Этап 2. S3-хранилище

S3 у Timeweb физически в Санкт-Петербурге (сервер — в Москве): для
файлов это нормально, для бэкапов — плюс («другой город»).

1. 🤖 Бакет **`ls-files`**: класс Standard, **приватный**. Включить
   версионирование объектов (защита от случайного удаления).
2. 🤖 Бакет **`ls-backups`**: класс Cold (Ice), приватный, версионирование
   включено.
3. 🤖 Создать **два отдельных ключа доступа**: один для Supabase Storage
   (только `ls-files`), другой для бэкапов Coolify (только `ls-backups`).
   Один ключ на всё — это один утёкший ключ на всё.
4. Эндпоинт: `https://s3.twcstorage.ru`, регион `ru-1`. 🔴 Ключи — в
   менеджер паролей.
5. Публичный бакет для картинок (`files.<домен>` CNAME на
   `s3.twcstorage.ru`) на первом релизе **не нужен**: все файлы портала
   закрытые и отдаются подписанными ссылками.

## Этап 3. Сервер и Coolify

1. 🤖 Создать облачный сервер:
   - локация **Москва**, в созданной VPC;
   - ОС **Ubuntu 24.04**;
   - конфигурация **4 vCPU / 8 ГБ / 80 ГБ NVMe** (линейка Cloud; в
     Premium-линейке Standard-диск внутри VPC недоступен — брать NVMe);
   - **публичный IPv4 обязателен** (Let's Encrypt, вебхуки GitHub);
   - группа файрвола `ls-core`;
   - SSH-ключ владельца;
   - в поле **cloud-init / user data** вставить:

   ```yaml
   #cloud-config
   swap:
     filename: /swapfile
     size: 4G
     maxsize: 4G
   package_update: true
   packages: [ufw, unattended-upgrades]
   runcmd:
     - curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
   ```

   Swap снимает OOM на шаге «Collecting build traces» сборки Next.js
   (на 8 ГБ он тоже случается, когда рядом собираются два образа).
2. 🤖 Сразу после создания — **образ сервера** («чистый сервер»).
3. 🤖 По SSH после установки Coolify — зеркало Docker Hub (Docker Hub
   стоит за Cloudflare и из РФ тянется с обрывами) и IPv6 для Telegram
   (Bot API из российских ЦОД открывается только по IPv6):

   ```bash
   cat > /etc/docker/daemon.json <<'JSON'
   {
     "registry-mirrors": ["https://dockerhub.timeweb.cloud"],
     "log-driver": "json-file",
     "log-opts": { "max-size": "50m", "max-file": "3" },
     "default-network-opts": { "bridge": { "com.docker.network.enable_ipv6": "true" } }
   }
   JSON
   systemctl restart docker
   docker info | grep -A2 'Registry Mirrors'
   ```

   В Eventbase установщик Coolify один раз перезаписал этот файл, в
   другой раз дописал в него `log-opts` — после любого обновления Coolify
   проверять `docker info`.
4. 🤖 Базовая защита ОС (в Eventbase этого не было записано, здесь — есть):

   ```bash
   # ssh: только ключи
   sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
   sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
   systemctl restart ssh
   # автообновления безопасности
   dpkg-reconfigure -f noninteractive unattended-upgrades
   # fail2ban для ssh
   apt-get install -y fail2ban && systemctl enable --now fail2ban
   ```

   Файрвол Timeweb уже закрывает всё лишнее снаружи; `ufw` на хосте
   не включаем (Docker и ufw конфликтуют, а польза нулевая за облачным
   файрволом).
5. 🤖 Открыть `http://<IP>:8000` → создать администратора Coolify (🔴
   пароль в менеджер) → в настройках панели задать домен
   `coolify.<домен>` → прописать A-запись `coolify` → дождаться
   сертификата → **закрыть порт 8000 в файрволе**. Пока админ не создан,
   панель открыта всему интернету — не затягивать.
6. 🤖 В Coolify: включить уведомления в Telegram (бот + группа владельца):
   Deployments — только failures; Backups — success и failure; Server
   unreachable.
7. 🤖 В Coolify → Servers → этот сервер: убедиться, что прокси —
   **Traefik** (по умолчанию). Caddy не переключать: в Eventbase
   переключение прокси уронило площадку на 40 минут, а нужды в Caddy без
   клиентских доменов нет.

## Этап 4. Supabase на своём сервере

Ресурс **Docker Compose** в Coolify (шаблон «Supabase» из каталога Coolify
или официальный compose Supabase).

1. 🤖 Создать проект Coolify `platform`, в нём ресурс Supabase.
2. 🤖 Сервисы, которые нужны: `db` (`supabase/postgres`), `kong`, `auth`
   (GoTrue), `rest` (PostgREST), `storage`, `imgproxy`, `meta`, `studio`.
   **Выключить**: `realtime`, `functions` (edge runtime), `analytics`
   (Logflare), `vector`. Каждый из них — память и поверхность атаки без
   пользы для платформы.
3. 🤖 Секреты (Coolify сгенерирует сам, 🔴 скопировать в менеджер):
   пароль Postgres, `SERVICE_PASSWORD_JWT` (= `JWT_SECRET`, не короче
   32 знаков), из него — `ANON_KEY` и `SERVICE_ROLE_KEY` (старого
   JWT-формата; `sb_publishable_…`/`sb_secret_…` на self-hosted нет).
4. 🤖 Настройки GoTrue (переменные сервиса `auth`):

   | Переменная | Значение | Зачем |
   |---|---|---|
   | `GOTRUE_SITE_URL` | `https://office.<домен>` | адреса в ссылках |
   | `GOTRUE_URI_ALLOW_LIST` | `https://office.<домен>/**` | куда можно редиректить |
   | `GOTRUE_DISABLE_SIGNUP` | `true` | вход только по приглашению |
   | `GOTRUE_EXTERNAL_EMAIL_ENABLED` | `true` | почта + пароль |
   | `GOTRUE_EXTERNAL_PHONE_ENABLED` | `false` | телефон — позже, для клиентов, и не через GoTrue |
   | `GOTRUE_MAILER_AUTOCONFIRM` | `false` | учётки создаёт админ-API с `email_confirm: true` |
   | `GOTRUE_PASSWORD_MIN_LENGTH` | `12` | политика паролей |
   | `GOTRUE_JWT_EXP` | `3600` | access-токен час, обновление по refresh |
   | `GOTRUE_SECURITY_REFRESH_TOKEN_ROTATION_ENABLED` | `true` | украденный refresh живёт один раз |
   | `GOTRUE_SECURITY_REFRESH_TOKEN_REUSE_INTERVAL` | `10` | допуск гонки вкладок |
   | `GOTRUE_MFA_TOTP_ENROLL_ENABLED` / `GOTRUE_MFA_TOTP_VERIFY_ENABLED` | `true` (по умолчанию) | второй фактор — ADR `platform-002-staff-auth` |
   | `GOTRUE_RATE_LIMIT_EMAIL_SENT` | `30` | лимит писем GoTrue в час (письма шлём сами, запас) |
   | `GOTRUE_SMTP_*` | не задавать на старте | приглашения и сброс пароля шлёт портал через Unisender Go, у GoTrue берётся только админ-API |

   Все внешние провайдеры входа (Google и прочие) выключены: в РФ они
   не нужны и не работают стабильно.
5. 🤖 Storage на S3 (переменные сервиса `storage`; в Coolify они правятся
   через «Edit Compose file», в списке переменных их нет):
   `STORAGE_BACKEND=s3`, бакет `ls-files`, эндпоинт
   `https://s3.twcstorage.ru`, регион `ru-1`, path-style включён,
   `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` — ключ бакета `ls-files`,
   `FILE_SIZE_LIMIT=52428800` (50 МБ потолок, точные лимиты — на бакетах).
   Имена переменных бакета зависят от версии `storage-api`: в новых —
   `STORAGE_S3_BUCKET`, `STORAGE_S3_ENDPOINT`, `STORAGE_S3_REGION`,
   `STORAGE_S3_FORCE_PATH_STYLE`; в старых — `GLOBAL_S3_BUCKET`,
   `GLOBAL_S3_ENDPOINT`, `REGION`, `GLOBAL_S3_FORCE_PATH_STYLE`. Сверять с
   compose-файлом шаблона, а не писать по памяти.
6. 🤖 Домен `api.<домен>` вешается на сервис `kong`. Включить у ресурса
   Supabase **«Connect to Predefined Network»** — иначе контейнер
   портала не увидит хост базы по имени.
7. 🤖 Studio закрыт basic-auth Kong (пароль в менеджер). Studio работает
   под `supabase_admin`, а миграции — под `postgres`: правки в Studio
   делают объекты чужими, и выкат падает на «must be owner». Правило:
   **в Studio только смотрим, схему меняем миграциями.**
8. 🤖 Проверка: `curl -I https://api.<домен>/rest/v1/` отвечает 401
   (Kong требует ключ), `curl https://api.<домен>/auth/v1/health` — 200
   или 401 за basic-auth (тогда монитор Kuma принимает 401).
9. 🤖 Postgres: в compose у `db` поднять `max_connections` до 200 и
   проверить `PGRST_DB_POOL` у `rest` (не меньше 50 на старте).
   Включённые расширения проверить: `pg_trgm`, `pgcrypto`, `pg_cron`,
   `pg_stat_statements`, `uuid-ossp` (в образе `supabase/postgres` они
   есть, `pg_cron` нужно добавить в `shared_preload_libraries`, если его
   там нет).

## Этап 5. Приложение `portal` и воркер

1. 🤖 Coolify → проект `platform` → New Resource → **GitHub App** (или
   deploy key для приватного репозитория `ls-russia`). С deploy key:
   без ключа `git fetch` молчит, а не ругается — проверять клон
   командой из веб-терминала.
2. 🤖 Приложение `portal`:
   - Build Pack **Dockerfile**, Base Directory `/`, Dockerfile Location
     `/apps/portal/Dockerfile` (контекст — корень монорепо, иначе не
     соберётся `@ls/shared`);
   - Ports Exposes `3000`; Domain `https://office.<домен>`;
   - **Watch Paths** (иначе каждый коммит пересобирает всё):
     `apps/portal/**`, `packages/shared/**`, `supabase/**`,
     `package.json`, `pnpm-lock.yaml`, `pnpm-workspace.yaml`;
   - Automatic Deployment on `main` — включить (Coolify создаст вебхук
     GitHub сам).
3. 🤖 Приложение `portal-worker` — **тот же образ**, без домена, с
   переменной `PORTAL_ROLE=worker` и `HEARTBEAT_URL` (push-монитор Kuma,
   Этап 7). Watch Paths те же.
4. 🤖 Переменные окружения (значения — из Этапа 4; оба приложения, если
   не сказано иначе):

   | Переменная | portal | worker | Значение |
   |---|---|---|---|
   | `SUPABASE_URL` | ✓ | ✓ | `https://api.<домен>` |
   | `SUPABASE_ANON_KEY` | ✓ | ✓ | anon |
   | `SUPABASE_SERVICE_ROLE_KEY` | ✓ | ✓ | service_role (только сервер) |
   | `SUPABASE_JWT_SECRET` | ✓ | ✓ | `SERVICE_PASSWORD_JWT` — понадобится клиентскому миру (релиз 3) и подписи билетов |
   | `DATABASE_URL` | ✓ | — | `postgresql://postgres:<пароль>@<имя контейнера db>:5432/postgres` — только для `apply.sh` при старте |
   | `MIGRATE_ON_START` | `1` | `0` | миграции накатывает веб-контейнер при старте (см. ниже) |
   | `PORTAL_PUBLIC_URL` | ✓ | ✓ | `https://office.<домен>` — ссылки в письмах; из `Host` не вычисляется никогда |
   | `CLIENT_IP_HEADER` | ✓ | ✓ | `x-forwarded-for` (за Traefik Coolify) — лимиты по IP |
   | `UNISENDER_GO_API_KEY`, `MAIL_FROM` | ✓ | ✓ | `Lady Stretch <no-reply@<домен>>` |
   | `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` | ✓ | ✓ | `npx web-push generate-vapid-keys`; subject — `mailto:` |
   | `SENTRY_DSN`, `SENTRY_ENVIRONMENT=prod` | ✓ | ✓ | после Этапа 7 |
   | `TELEGRAM_BOT_TOKEN` | — | ✓ | если бот уведомлений нужен |

   `NEXT_PUBLIC_*` не используется вовсе: образ один на любое окружение,
   всё, что нужно браузеру, сервер отдаёт атрибутами `<html>` или ответом.
5. **Миграции при старте, а не pre-deploy.** В Eventbase pre-deployment
   command выполняется `docker exec` в **старом** контейнере, то есть
   миграции едут на один выкат позже кода. Здесь `Dockerfile` портала
   запускает `bash supabase/apply.sh && node apps/portal/server.js`, когда
   `MIGRATE_ON_START=1`: новый контейнер сначала накатывает схему
   (advisory lock серилизует реплики), затем стартует; упавшая миграция
   не проходит healthcheck, и Coolify оставляет старый контейнер. Схема
   всегда не старше кода — механикой, а не дисциплиной.
6. 🤖 Healthcheck в Dockerfile через `node -e "fetch(...)"`: в
   `node:22-alpine` нет curl, а штатная проверка Coolify выполняется
   внутри контейнера и роняет здоровое приложение с «New container is
   not healthy, rolling back».
7. 🤖 DNS: A-записи `office` и `api` на IP сервера → первый выкат →
   сертификаты. Первый вход в портал — учётка супер-админа создаётся
   скриптом из веб-терминала контейнера `portal` (`scripts/bootstrap-admin.mjs`,
   пароль печатается один раз) — почта ещё не настроена, письма не нужны.
8. 🤖 Лимит тела запроса на прокси — метки Traefik на сервисе `portal`:

   ```
   traefik.http.middlewares.portal-limit.buffering.maxRequestBodyBytes=2097152
   traefik.http.routers.<роутер>.middlewares=portal-limit
   ```

   Файлы в хранилище летят из браузера напрямую по подписанному URL, так
   что 2 МБ на тело к приложению хватает с запасом. Проверка: POST на
   5 МБ отвечает 413.

## Этап 6. Почта

1. 🤖 Аккаунт Unisender Go (🔴 договор и оплата), проект, домен
   отправителя `<домен>` (или `mail.<домен>`), API-ключ в менеджер.
2. 🤖 DNS у Timeweb — три известные ловушки:
   - NS-записей в панели нет, поэтому у Unisender выбирается вариант
     подтверждения «CNAME», а не делегирование;
   - Timeweb **молча добавляет SPF к каждому поддомену** — лишнюю запись
     удалить, иначе DKIM не сойдётся; проверять через
     `https://dns.google/resolve?name=…&type=TXT`;
   - SPF — одной записью: `v=spf1 include:_spf.timeweb.ru include:spf.unisender.ru ~all`;
     DMARC пишется **без точки** в конце имени.
3. В коде отправки: `track_links: 0`, `track_read: 0` — почтовые
   антивирусы ходят по ссылкам и сжигают одноразовые токены. Ответ
   «200 + success» у Unisender не значит «ушло» — проверять
   `failed_emails`.
4. 🤖 После выкладки переменных — пробное письмо-приглашение самому себе.

## Этап 7. Бэкапы и мониторинг (сразу)

1. 🤖 **Бэкап базы**: Coolify → ресурс Supabase → `db` → Backups:
   ежедневно в 03:30 МСК, хранение 30 дней, S3 — бакет `ls-backups` с
   его ключом. Нажать «Backup now», убедиться, что файл появился в S3.
2. 🤖 **Проверка восстановления** (сейчас и раз в квартал):

   ```bash
   D=$(ls -t /data/coolify/backups/databases/*/*/*.dmp | head -1)
   docker run -d --rm --name pgtest -e POSTGRES_PASSWORD=test \
     -v "$(dirname "$D")":/bk:ro <образ db из compose Supabase>   # тот же образ и версия, что у сервиса db
   sleep 20
   docker exec pgtest pg_restore -U postgres -d postgres --no-owner --no-privileges "/bk/$(basename "$D")"
   docker exec pgtest psql -U postgres -d postgres -c "select count(*) from public.profiles"
   docker stop pgtest
   ```

   Сотни сообщений про роли Supabase при `pg_restore` в чистый образ —
   норма; важно, что таблицы и строки на месте.
3. 🤖 **Uptime Kuma** (ресурс из каталога Coolify, домен
   `status.<домен>`, 🔴 пароль): мониторы
   `https://office.<домен>/api/health?strict=1`,
   `https://api.<домен>/auth/v1/health` (принимать 401),
   `https://coolify.<домен>`, TCP 5432 изнутри не нужен; **push-монитор**
   воркера (интервал 120 с) — его адрес и есть `HEARTBEAT_URL`. Тревоги —
   в Telegram-группу владельца. Интервал 60 с, повторов 3.
4. 🤖 **GlitchTip** (каталог Coolify, домен `errors.<домен>`):
   `ENABLE_OPEN_USER_REGISTRATION=False`, `GLITCHTIP_MAX_EVENT_LIFE_DAYS=90`,
   `EMAIL_URL` — SMTP Unisender Go. Проект `portal` → DSN → переменные
   `SENTRY_DSN`/`SENTRY_ENVIRONMENT` → смоук
   `https://office.<домен>/api/version?sentry=test`. Если памяти тесно —
   GlitchTip откладывается до второго сервера, Kuma не откладывается.
5. 🤖 **Образ сервера** после завершения всех этапов и «Защита от
   удаления» в панели Timeweb.

## Этап 8. Проверка из России и приёмка

1. 🤖 `check-host.net` и `ping-admin.ru` по адресам `office.<домен>`,
   `api.<домен>` — с мобильных операторов тоже. В консоли браузера на
   странице входа **ни одного запроса к хостам вне `<домен>`**.
2. 🤖 `curl -s -H 'X-Forwarded-For: 1.2.3.4' https://office.<домен>/api/health | jq '{clientIp, clientIpHeader}'`
   — должен вернуться настоящий адрес, а не подделанный.
3. 🤖 Вход супер-админа → настройка TOTP → приглашение второго человека
   письмом → вход приглашённого → задача, чат, статья базы знаний.
4. 🤖 Пробный коммит в `main` → автоматический выкат → версия в подвале
   портала изменилась.

## Паспорт площадки (заполняется по ходу, без секретов)

| Параметр | Значение |
|---|---|
| Домен | |
| IP сервера, VPC | |
| Тариф сервера, дата покупки | |
| Версия Coolify | |
| Версии образов Supabase (`db`, `auth`, `rest`, `storage`, `kong`) | |
| Бакеты S3, регион | |
| Домен отправителя почты | |
| Дата последней проверки восстановления | |
| Telegram-группа тревог | |

## Стоп-краны

- Выкат упал на миграции → старый контейнер работает; смотреть лог
  `apply.sh` в деплое, чинить **новой** миграцией, не правкой старой.
- Панель Coolify недоступна, а сайты работают → не трогать прокси;
  `ssh -L 8000:localhost:8000 root@<IP>` и панель по `localhost:8000`.
- Прокси удалён или сломан → `docker compose -f /data/coolify/proxy/docker-compose.yml up -d`.
- Потерян второй фактор у единственного супер-админа → сброс фактора
  через админ-API GoTrue из веб-терминала контейнера (`scripts/mfa-reset.mjs`),
  запись в `audit_log`; порядок — `docs/setup/…-emergency-access.md`
  (пишется вместе с модулем входа).
- Диск заполнен → `docker system prune -af --volumes=false`, логи
  контейнеров уже ограничены `log-opts`; бэкапы живут в S3, а не на диске.

## Чего здесь нет намеренно

- Клиентских доменов и on-demand TLS (Caddy) — у платформы один домен.
- Балансировщика и реплик — до клиентского приложения.
- Кастомного SMTP у GoTrue — письма шлёт платформа своим шаблоном.
- Supabase Realtime, Edge Functions, Logflare — не используются.
