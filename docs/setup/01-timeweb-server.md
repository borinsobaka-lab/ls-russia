# Runbook: сервер платформы на Timeweb Cloud (Coolify + Supabase)

Статус: черновик к запуску — 2026-10-08.

Итог: `https://office.<домен>` работает на своём сервере в Москве с
базой на том же сервере, файлами в Timeweb S3, бэкапами в другом ЦОД и
мониторингом. Срок — один рабочий день плюс ожидание верификации домена.

## Легенда

- 🤖 — делает агент (в браузере или по SSH) либо владелец по инструкции.
- 🔴 — только человек: оплата, паспорт/ЕСИА, 2FA, пароли, подписание
  документов.

## Правило про секреты

Пароль Postgres, `JWT_SECRET`, `ANON_KEY`, `SERVICE_ROLE_KEY`, ключи S3,
ключ Unisender, VAPID-ключи, пароли панелей живут **только** в менеджере
паролей владельца и переменных Coolify. В чат, репозиторий и документы —
никогда. `SERVICE_ROLE_KEY`, `JWT_SECRET` и пароль базы — это все данные
компании целиком.

## Три правила

1. **Миграции применяет выкат, не человек.** Руками — только разрушающие
   (`drop table/column`, `truncate`), в тихое время, с бэкапом, с
   отметкой в `app.schema_migrations`.
2. **Никаких адресов в коде.** Домен, адрес базы, отправитель писем, S3
   — только переменные Coolify.
3. **Образ сервера перед каждым изменением инфраструктуры.**

---

## Этап 0. Аккаунт, деньги, домен

1. 🔴 Аккаунт Timeweb Cloud на нужное юрлицо/ИП, 2FA, баланс **5 000 ₽**,
   проект `ls-platform`.
2. 🔴 Соглашение об обработке персональных данных с Timeweb (в панели).
3. 🤖 Домен в Timeweb (сразу на их NS). 🔴 Для `.ru` — верификация
   владельца (паспорт или ЕСИА), до суток.
4. Записи DNS создаются после Этапа 3, все с **TTL 300**.

| Поддомен | Что |
|---|---|
| `office.<домен>` | портал (имя — решение владельца) |
| `api.<домен>` | Supabase (Kong) |
| `coolify.<домен>` | панель Coolify |
| `status.<домен>` | Uptime Kuma |
| `errors.<домен>` | GlitchTip |
| `app.<домен>` | клиентское приложение (релиз 3) |

## Этап 1. Сеть

1. 🤖 **VPC** в Москве (там включена DDoS-защита L3–L4).
2. 🤖 Группа файрвола `ls-core`: входящие 22/tcp — с IP владельца и из
   VPC; 80/tcp, 443/tcp — отовсюду; остальное закрыто. **Исходящие:
   разрешить TCP, UDP, ICMP по IPv4 и IPv6 явно** — группа работает как
   allow-list и наружу. Порт 8000 открывать на 15 минут при первом входе
   в Coolify и закрыть.

## Этап 2. S3

S3 Timeweb — в Санкт-Петербурге (сервер — в Москве): для файлов
нормально, для бэкапов плюс.

1. 🤖 Бакет **`ls-files`**: Standard, приватный, версионирование включено.
2. 🤖 Бакет **`ls-backups`**: Cold, приватный, версионирование включено.
3. 🤖 **Два отдельных ключа**: один для Storage (только `ls-files`),
   другой для бэкапов (только `ls-backups`).
4. Эндпоинт `https://s3.twcstorage.ru`, регион `ru-1`. 🔴 Ключи — в
   менеджер паролей.

## Этап 3. Сервер и Coolify

1. 🤖 Облачный сервер: Москва, в VPC; **Ubuntu 24.04**; **4 vCPU /
   8 ГБ / 80 ГБ NVMe**; публичный IPv4; группа файрвола `ls-core`;
   SSH-ключ владельца; cloud-init:

   ```yaml
   #cloud-config
   swap:
     filename: /swapfile
     size: 4G
     maxsize: 4G
   package_update: true
   packages: [fail2ban, unattended-upgrades]
   runcmd:
     - curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
   ```

2. 🤖 Сразу — **образ сервера** («чистый сервер»).
3. 🤖 По SSH после установки — зеркало Docker Hub (стоит за Cloudflare,
   из РФ тянется с обрывами), ограничение логов и IPv6 для Telegram
   (Bot API из российских ЦОД доступен только по IPv6):

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

   Установщик и обновления Coolify могут править этот файл — после
   обновления Coolify проверять `docker info`.
4. 🤖 Защита ОС:

   ```bash
   sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
   sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
   systemctl restart ssh
   dpkg-reconfigure -f noninteractive unattended-upgrades
   systemctl enable --now fail2ban
   ```

   `ufw` не включаем: Docker и ufw конфликтуют, а снаружи всё закрывает
   файрвол Timeweb.
5. 🤖 `http://<IP>:8000` → администратор Coolify (🔴 пароль в менеджер)
   → домен панели `coolify.<домен>` → A-запись → сертификат → **закрыть
   8000**. Пока админ не создан, панель открыта всему интернету.
6. 🤖 Уведомления Coolify в Telegram: Deployments — только failures;
   Backups — success и failure; Server unreachable.
7. 🤖 Прокси — **Traefik** (по умолчанию). Caddy не переключать: нужды
   нет, а переключение прокси — это простой панели и сайтов.

## Этап 4. Supabase на своём сервере

Ресурс Docker Compose в Coolify (шаблон «Supabase» из каталога или
официальный compose).

1. 🤖 Проект Coolify `platform`, ресурс Supabase.
2. 🤖 Сервисы: `db`, `kong`, `auth`, `rest`, `storage`, `imgproxy`,
   `meta`, `studio`. **Выключить**: `realtime`, `functions`, `analytics`,
   `vector`.
3. 🤖 Секреты генерирует Coolify (🔴 в менеджер): пароль Postgres,
   `SERVICE_PASSWORD_JWT` (= `JWT_SECRET`, не короче 32 знаков), из него
   `ANON_KEY` и `SERVICE_ROLE_KEY` (JWT-формата).
4. 🤖 GoTrue (переменные сервиса `auth`):

   | Переменная | Значение | Зачем |
   |---|---|---|
   | `GOTRUE_SITE_URL` | `https://office.<домен>` | адреса в ссылках |
   | `GOTRUE_URI_ALLOW_LIST` | `https://office.<домен>/**` | куда можно редиректить |
   | `GOTRUE_DISABLE_SIGNUP` | `true` | вход только по приглашению |
   | `GOTRUE_EXTERNAL_EMAIL_ENABLED` | `true` | почта + пароль |
   | `GOTRUE_EXTERNAL_PHONE_ENABLED` | `false` | телефон — позже и не через GoTrue |
   | `GOTRUE_MAILER_AUTOCONFIRM` | `false` | учётки создаёт админ-API с `email_confirm: true` |
   | `GOTRUE_PASSWORD_MIN_LENGTH` | `12` | политика паролей |
   | `GOTRUE_JWT_EXP` | `3600` | access-токен час |
   | `GOTRUE_SECURITY_REFRESH_TOKEN_ROTATION_ENABLED` | `true` | украденный refresh живёт один раз |
   | `GOTRUE_SECURITY_REFRESH_TOKEN_REUSE_INTERVAL` | `10` | допуск гонки вкладок |
   | `GOTRUE_MFA_TOTP_ENROLL_ENABLED` / `GOTRUE_MFA_TOTP_VERIFY_ENABLED` | `true` | второй фактор (ADR platform-002) |
   | `GOTRUE_SMTP_*` | не задавать | письма шлёт портал через Unisender Go |

   Внешние провайдеры входа выключены.
5. 🤖 Storage на S3 (правится через «Edit Compose file»):
   `STORAGE_BACKEND=s3`, бакет `ls-files`, эндпоинт
   `https://s3.twcstorage.ru`, регион `ru-1`, path-style включён, ключ
   бакета `ls-files`, `FILE_SIZE_LIMIT=52428800`. Имена переменных
   бакета зависят от версии `storage-api` (новые — `STORAGE_S3_BUCKET`,
   `STORAGE_S3_ENDPOINT`, `STORAGE_S3_REGION`, `STORAGE_S3_FORCE_PATH_STYLE`;
   старые — `GLOBAL_S3_BUCKET`, `GLOBAL_S3_ENDPOINT`, `REGION`,
   `GLOBAL_S3_FORCE_PATH_STYLE`) — сверять с compose-файлом шаблона.
6. 🤖 Домен `api.<домен>` — на сервис `kong`. Включить **«Connect to
   Predefined Network»** — иначе контейнер портала не увидит хост базы.
7. 🤖 Studio закрыт basic-auth Kong. Studio работает под
   `supabase_admin`, миграции — под `postgres`: правки в Studio делают
   объекты чужими, и выкат падает на «must be owner». **В Studio только
   смотрим.**
8. 🤖 Проверка: `curl -I https://api.<домен>/rest/v1/` → 401;
   `curl https://api.<домен>/auth/v1/health` → 200 или 401 за basic-auth.
9. 🤖 Postgres: `max_connections` 200, `PGRST_DB_POOL` у `rest` не меньше
   50; расширения `pg_trgm`, `pgcrypto`, `pg_cron` (в
   `shared_preload_libraries`), `pg_stat_statements`.

## Этап 5. Приложение `portal` и воркер

1. 🤖 Coolify → New Resource → GitHub App или deploy key для приватного
   репозитория `ls-russia`. С deploy key: без ключа `git fetch` молчит,
   а не ругается — проверять клон из веб-терминала.
2. 🤖 Приложение `portal`: Build Pack **Dockerfile**, Base Directory `/`,
   Dockerfile Location `/apps/portal/Dockerfile` (контекст — корень
   монорепо), порт `3000`, домен `https://office.<домен>`,
   **Watch Paths**: `apps/portal/**`, `packages/shared/**`,
   `supabase/**`, `package.json`, `pnpm-lock.yaml`, `pnpm-workspace.yaml`;
   Automatic Deployment on `main`.
3. 🤖 Приложение `portal-worker` — **тот же образ**, без домена,
   `PORTAL_ROLE=worker`, `HEARTBEAT_URL` (push-монитор Kuma, Этап 7).
4. 🤖 Переменные:

   | Переменная | portal | worker | Значение |
   |---|---|---|---|
   | `SUPABASE_URL` | ✓ | ✓ | `https://api.<домен>` |
   | `SUPABASE_ANON_KEY` | ✓ | ✓ | anon |
   | `SUPABASE_SERVICE_ROLE_KEY` | ✓ | ✓ | service_role |
   | `SUPABASE_JWT_SECRET` | ✓ | ✓ | `SERVICE_PASSWORD_JWT` — для подписи JWT клиентского мира и билетов |
   | `DATABASE_URL` | ✓ | — | `postgresql://postgres:<пароль>@<контейнер db>:5432/postgres` — только для `apply.sh` при старте |
   | `MIGRATE_ON_START` | `1` | `0` | миграции накатывает веб-контейнер |
   | `PORTAL_PUBLIC_URL` | ✓ | ✓ | `https://office.<домен>` — ссылки в письмах |
   | `CLIENT_IP_HEADER` | ✓ | ✓ | `x-forwarded-for` (за Traefik) |
   | `UNISENDER_GO_API_KEY`, `MAIL_FROM` | ✓ | ✓ | `Lady Stretch <no-reply@<домен>>` |
   | `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` | ✓ | ✓ | `npx web-push generate-vapid-keys` |
   | `SENTRY_DSN`, `SENTRY_ENVIRONMENT=prod` | ✓ | ✓ | после Этапа 7 |
   | `TELEGRAM_BOT_TOKEN` | — | ✓ | релиз 2 |

   `NEXT_PUBLIC_*` не используется: образ один на любое окружение.
5. **Миграции при старте.** Entrypoint портала при `MIGRATE_ON_START=1`
   запускает `bash supabase/apply.sh && node apps/portal/server.js`:
   новый контейнер накатывает схему (advisory lock сериализует реплики),
   затем стартует; упавшая миграция не проходит healthcheck, и Coolify
   оставляет старый контейнер. Схема всегда не старше кода — механикой.
6. 🤖 Healthcheck в Dockerfile — `node -e "fetch(...)"`: в alpine нет
   curl, а штатная проверка Coolify идёт внутри контейнера.
7. 🤖 A-записи `office` и `api` → первый выкат → сертификаты. Первый
   супер-админ — `node scripts/bootstrap-admin.mjs` из веб-терминала
   контейнера (пароль печатается один раз).
8. 🤖 Лимит тела на прокси — метки Traefik у `portal`:

   ```
   traefik.http.middlewares.portal-limit.buffering.maxRequestBodyBytes=2097152
   traefik.http.routers.<роутер>.middlewares=portal-limit
   ```

   Файлы летят в Storage напрямую, 2 МБ к приложению хватает. Проверка:
   POST на 5 МБ → 413.

## Этап 6. Почта

1. 🤖 Аккаунт Unisender Go (🔴 договор, оплата), домен отправителя,
   API-ключ в менеджер.
2. 🤖 DNS у Timeweb — три ловушки: NS-записей в панели нет (у Unisender
   выбирать подтверждение через CNAME); Timeweb **молча добавляет SPF к
   каждому поддомену** — лишнюю запись удалить, иначе DKIM не сойдётся
   (проверка через `dns.google/resolve`); SPF одной записью
   `v=spf1 include:_spf.timeweb.ru include:spf.unisender.ru ~all`; DMARC
   без точки в конце имени.
3. В коде отправки `track_links: 0`, `track_read: 0`; проверять
   `failed_emails` в ответе.
4. 🤖 Пробное приглашение самому себе.

## Этап 7. Бэкапы и мониторинг

1. 🤖 **Бэкап базы**: Coolify → `db` → Backups: ежедневно 03:30 МСК,
   30 дней, S3 `ls-backups` своим ключом. «Backup now» → файл в S3.
2. 🤖 **Проверка восстановления** (сейчас и раз в квартал):

   ```bash
   D=$(ls -t /data/coolify/backups/databases/*/*/*.dmp | head -1)
   docker run -d --rm --name pgtest -e POSTGRES_PASSWORD=test \
     -v "$(dirname "$D")":/bk:ro <образ db из compose Supabase>
   sleep 20
   docker exec pgtest pg_restore -U postgres -d postgres --no-owner --no-privileges "/bk/$(basename "$D")"
   docker exec pgtest psql -U postgres -d postgres -c "select count(*) from public.profiles"
   docker stop pgtest
   ```

   Сообщения про роли Supabase при восстановлении в чистый образ —
   норма; важно, что таблицы и строки на месте.
3. 🤖 **Uptime Kuma** (каталог Coolify, `status.<домен>`, 🔴 пароль):
   `https://office.<домен>/api/health?strict=1`,
   `https://api.<домен>/auth/v1/health` (принимать 401),
   `https://coolify.<домен>`, **push-монитор** воркера (120 с) — его
   адрес и есть `HEARTBEAT_URL`. Тревоги в Telegram. Интервал 60 с,
   повторов 3.
4. 🤖 **GlitchTip** (каталог Coolify, `errors.<домен>`):
   `ENABLE_OPEN_USER_REGISTRATION=False`, `GLITCHTIP_MAX_EVENT_LIFE_DAYS=90`,
   `EMAIL_URL` — SMTP Unisender Go. Проект `portal` → DSN → переменные →
   смоук `https://office.<домен>/api/version?sentry=test`. Если памяти
   тесно — GlitchTip ждёт второго сервера, Kuma не ждёт.
5. 🤖 **Образ сервера** и «Защита от удаления».

## Этап 8. Проверка и приёмка

1. 🤖 `check-host.net` и `ping-admin.ru` по `office.<домен>` и
   `api.<домен>`, включая мобильных операторов. В консоли браузера на
   странице входа — ни одного запроса к хостам вне `<домен>`.
2. 🤖 `curl -s -H 'X-Forwarded-For: 1.2.3.4' https://office.<домен>/api/health | jq '{clientIp, clientIpHeader}'`
   — настоящий адрес, а не подделанный.
3. 🤖 Вход супер-админа → TOTP → приглашение второго человека → его вход
   → задача, чат, статья.
4. 🤖 Пробный коммит в `main` → выкат → версия в подвале изменилась.

## Паспорт площадки (без секретов)

| Параметр | Значение |
|---|---|
| Домен | |
| IP сервера, VPC | |
| Тариф, дата покупки | |
| Версия Coolify | |
| Версии образов Supabase | |
| Бакеты S3, регион | |
| Домен отправителя почты | |
| Дата последней проверки восстановления | |
| Telegram-группа тревог | |

## Стоп-краны

- Выкат упал на миграции → старый контейнер работает; чинить **новой**
  миграцией.
- Панель Coolify недоступна, сайты работают → не трогать прокси;
  `ssh -L 8000:localhost:8000 root@<IP>`.
- Прокси сломан → `docker compose -f /data/coolify/proxy/docker-compose.yml up -d`.
- Потерян второй фактор единственного супер-админа → `node
  scripts/mfa-reset.mjs` из веб-терминала (админ-API GoTrue), запись в
  `audit_log`.
- Диск заполнен → `docker system prune -af`; бэкапы живут в S3.
