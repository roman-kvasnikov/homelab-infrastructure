---
name: identity-authelia
description: |
  Authelia — IdP и forward-auth провайдер для Traefik. LXC в SERVICES, хранит хеши паролей, TOTP-секреты, WebAuthn-credentials, OIDC-ключи; 2FA через TOTP и WebAuthn, OIDC для приложений. Документ описывает файловую структуру, конфигурацию через env-секреты, локальный Redis, модель forward-auth в Traefik и нюанс с rate-limit. Используй для вопросов по аутентификации, SSO, forward-auth, OIDC, TOTP/WebAuthn.
---

# Authelia

Authelia — Identity Provider и forward-auth прокси для Traefik: единый вход, вторая фаза аутентификации (2FA) и OIDC для приложений. Развёрнута отдельным LXC в сегменте SERVICES, `192.168.50.12`.

Вынесена в отдельный LXC осознанно ради изоляции: Authelia хранит хеши паролей пользователей, TOTP-секреты, WebAuthn-credentials, OIDC client secrets и signing keys. Компрометация Authelia означает компрометацию всех аутентификаций ко всем сервисам, поэтому она не делит контейнер ни с чем.

Работает нативно (без Docker) на бинарнике из официальных GitHub releases. База данных — SQLite. Сессии — локальный Redis в том же LXC. Версия зафиксирована, обновление вручную через замену бинарника. Базлайн LXC (unprivileged + `nesting=1`), systemd-sandbox, SSH-hardening и nftables-шаблон — см. `conventions.md`.

## 1. Файловая структура

```
/usr/local/bin/authelia                # binary from github.com/authelia/authelia/releases
/etc/authelia/                         # configs and secrets
├── configuration.yaml                 # server, log, notifier, ntp
├── storage.yaml                       # SQLite path
├── session.yaml                       # cookies + Redis connection
├── authentication-backend.yaml        # file backend + argon2 params
├── access-control.yaml                # default policy + rules
├── identity-providers.yaml            # OIDC clients and signing keys
├── regulation.yaml                    # rate limiting on failed logins
├── users.yaml                         # users with argon2 password hashes
├── .env                               # env vars for systemd
└── secrets/
    ├── JWT_SECRET
    ├── SESSION_SECRET
    ├── REDIS_PASSWORD
    ├── STORAGE_ENCRYPTION_KEY
    ├── OIDC_HMAC_SECRET
    └── oidc/jwks/rsa.2048.key          # RSA key for signing OIDC tokens

/var/lib/authelia/                     # home of the authelia system user
├── db.sqlite3                         # main DB (TOTP, WebAuthn, OIDC consents)
├── notifier.log                       # fallback notifier
├── .restic-password                   # restic repo encryption password
├── .restic-http-password              # rest-server HTTP basic-auth password
└── .cache/restic/                     # restic cache (excluded from backup)
```

Бинарь Authelia — обычный Go-бинарь со статически вкомпилированными зависимостями, скачивается напрямую с GitHub releases (в отличие от Vaultwarden, где приходится извлекать из Docker-образа).

## 2. Systemd

Сервис `authelia.service` запускает бинарник от системного юзера `authelia`. Sandbox-набор усиленный (`NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateUsers`, `ProtectKernel*`, `RestrictNamespaces`, `LockPersonality`, пустой `CapabilityBoundingSet`, `SystemCallFilter=@system-service`) — см. `conventions.md`. Запись разрешена только в `/var/lib/authelia`. `Restart=always`. Зависимость от Redis: `Requires=redis-server.service` + `After=redis-server.service`.

ExecStart передаёт **семь** конфигурационных файлов через повторяющийся `--config` — Authelia мерджит их в один документ при старте. Это разделяет конфиг по областям и упрощает ревью. Автозапуск при старте контейнера.

## 3. Секреты через env-переменные

Все секреты лежат отдельными файлами в `/etc/authelia/secrets/` и подгружаются через переменные вида `AUTHELIA_*_FILE` — Authelia сама читает файл по указанному пути. Переменные собраны в `/etc/authelia/.env`, подключаемом через `EnvironmentFile=`:

```
AUTHELIA_SERVER_DISABLE_HEALTHCHECK=true
X_AUTHELIA_CONFIG_FILTERS=template
AUTHELIA_IDENTITY_VALIDATION_RESET_PASSWORD_JWT_SECRET_FILE=/etc/authelia/secrets/JWT_SECRET
AUTHELIA_SESSION_SECRET_FILE=/etc/authelia/secrets/SESSION_SECRET
AUTHELIA_SESSION_REDIS_PASSWORD_FILE=/etc/authelia/secrets/REDIS_PASSWORD
AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE=/etc/authelia/secrets/STORAGE_ENCRYPTION_KEY
```

`X_AUTHELIA_CONFIG_FILTERS=template` включает Go-template обработку YAML — нужно для `identity-providers.yaml`, где RSA-ключ OIDC подгружается через `{{ secret "..." | mindent 10 "|" | msquote }}`. Без фильтра шаблон-конструкции ломают YAML.

## 4. Ключевые параметры конфига

`server.address: tcp://192.168.50.12:9091/` — bind на конкретный LAN-IP, не на `0.0.0.0`. `storage.local.path: /var/lib/authelia/db.sqlite3` — SQLite, без PostgreSQL. `session.redis: { host: 127.0.0.1, port: 6379, database_index: 0 }` — локальный Redis, пароль из env. `authentication_backend.file.password.argon2`: `iterations: 3, parallelism: 4, memory: 65536` (64 MiB) — OWASP-recommended. `notifier.filesystem.filename: /var/lib/authelia/notifier.log` — fallback-нотификатор (SMTP не настроен). `log.keep_stdout: true` — логи в journald (`ProtectSystem=strict` запрещает запись в `/var/log`).

## 5. Локальный Redis

Redis установлен в этом же LXC, не разделяется с другими сервисами. `/etc/redis/redis.conf`: `bind 127.0.0.1 -::1` (только loopback), `requirepass` из `/etc/authelia/secrets/REDIS_PASSWORD`, `maxmemory 64mb` + `maxmemory-policy allkeys-lru`. Нужен только для активных сессий Authelia. При потере данных Redis (рестарт) пользователи перелогинятся, но БД с TOTP/WebAuthn/OIDC (SQLite) остаётся нетронутой.

## 6. Сетевая фильтрация

nftables по шаблону сервисного LXC из `conventions.md`: `policy drop`, разрешён порт Authelia `9091` только с адреса Traefik (`192.168.40.11`), SSH из MGMT и VPN. Прямой доступ к 9091 из других сетей мимо Traefik закрыт (соединение не устанавливается). Redis отдельным правилом не открывается — он слушает только loopback, что покрывается правилом для `lo`.

## 7. Модель аутентификации и forward-auth

Authelia — основной IdP инфраструктуры. 2FA через TOTP и WebAuthn, поддержка PKCE для мобильных приложений, OIDC для приложений-клиентов. Forward-auth реализован middleware `authelia` в Traefik, указывающим на `http://192.168.50.12:9091/api/authz/forward-auth`.

Защищённый сервис получает цепочку `internal-chain + authelia` или `external-chain + authelia` (см. `ingress-traefik.md`). Запрос проходит через Traefik → forward-auth к Authelia → если сессия не авторизована, редирект на страницу логина Authelia; после успешной аутентификации (и 2FA, если требуется по access-control) запрос пропускается к бэкенду.

## 8. Нюанс с rate-limit

Authelia **несовместима** с middleware `rate-limit` в Traefik. Её веб-интерфейс — React SPA, который при загрузке подтягивает десятки JS-чанков **параллельно**. Обычный rate-limit (`external-chain`) считает это всплеском запросов и банит клиента на этапе загрузки страницы логина. Поэтому публичный доступ к Authelia идёт через цепочку **без** rate-limit — `external-chain-no-rate-limit` (CrowdSec + security-headers, но без ограничения скорости). Защита от перебора паролей обеспечивается собственным механизмом Authelia (`regulation.yaml` — лимит неудачных логинов), а не Traefik-rate-limit.

## 9. Резервное копирование

Два независимых механизма (по общей схеме, см. `backup.md`):

**PBS-снапшот всего LXC** — ежедневно в составе общего pve-задания. Сценарий «снёс LXC, восстановить за минуту».

**Restic-снапшот данных и конфигов** — ежедневно через `/usr/local/sbin/authelia-backup.sh` (таймер `authelia-backup.timer`). Скрипт делает online-снапшот SQLite (`sqlite3 .backup` — консистентная копия работающей БД), затем `restic backup` директорий `/var/lib/authelia/` (БД, `notifier.log`) и `/etc/authelia/` (все YAML, секреты). Исключаются живой `db.sqlite3`, WAL-файлы, `.restic-*`-пароли, `.cache/`. Тег/host `authelia`. Транспорт — rest-server на PBS (`rest:http://authelia:...@192.168.10.15:8000/authelia/`) в append-only. Retention централизованно на PBS (см. `backup.md`).

**Восстановление БД**: после `restic restore` файл лежит как `db.sqlite3.backup` — переименовать в `db.sqlite3` перед запуском. WAL-файлы не нужны, пересоздадутся при старте.

Restic-репозиторий, два пароля (encryption + HTTP basic-auth), режим append-only + private-repos — паттерн общий, см. `conventions.md` и `backup.md`.

## 10. Обновление

Бинарь скачивается с GitHub releases напрямую. Процедура: остановить сервис, забэкапить текущий бинарь (`.bak` для отката), установить новый, запустить, проверить лог. При мажорных обновлениях Authelia автоматически мигрирует схему SQLite при старте (в логе `Storage schema migration from N to M is complete`).

## 11. Зависимости

- **Traefik (`192.168.40.11`)** — единственный разрешённый источник запросов к 9091 (nftables); forward-auth завязан на Authelia. Без Traefik Authelia недоступна снаружи LXC.
- **Unbound на OPNsense** — DNS, split-horizon `authelia.kvasok.xyz → 192.168.40.11`.
- **PBS (`192.168.10.15`)** — restic-бэкапы и PBS-снапшоты.
