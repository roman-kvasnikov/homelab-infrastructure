---
name: vaultwarden
description: |
  Vaultwarden — self-hosted менеджер паролей. LXC в SERVICES, нативный бинарник из Docker-образа, SQLite, доступ только через Traefik под жёстким rate-limit. Документ описывает установку, конфигурацию, сетевую фильтрацию, публикацию, резервное копирование и обновление. Используй для вопросов по Vaultwarden, менеджеру паролей, хранению секретов.
---

# Vaultwarden

Self-hosted менеджер паролей в отдельном непривилегированном LXC (`192.168.50.11`, SERVICES). Работает нативно (без Docker) на бинарнике, извлечённом из официального Docker-образа `vaultwarden/server`. БД — SQLite. Доступ только через Traefik. Версия зафиксирована, обновление вручную через замену бинарника. Базлайн LXC, systemd-sandbox, SSH-hardening, nftables-шаблон и restic-паттерн — см. `02-conventions.md`.

## 1. Файловая структура

```
/usr/local/bin/vaultwarden            # binary extracted from the official Docker image
/usr/share/vaultwarden/web-vault/     # web UI static
/etc/vaultwarden/.env                 # configuration (env vars)
/var/lib/vaultwarden/                 # home of the vaultwarden system user
├── data/
│   ├── db.sqlite3                    # main DB (SQLite, WAL mode)
│   ├── attachments/                  # record attachments
│   ├── rsa_key.pem                   # JWT signing key
│   └── icon_cache/                   # site icon cache (excluded from backup)
├── .restic-password                  # restic repo encryption password
└── .restic-http-password             # rest-server HTTP basic-auth password
```

Бинарь и web-vault извлекаются из официального Docker-образа через `skopeo` + `umoci` — те же файлы, что использует Docker, но без Docker-обёртки. Для динамической линковки установлены `libmariadb3` и `libpq5`, хотя используется только SQLite (Vaultwarden слинкован со всеми тремя driver-библиотеками).

## 2. Systemd и конфигурация

Сервис `vaultwarden.service` от системного юзера `vaultwarden` (UID 999), sandbox-набор из `02-conventions.md`, запись только в `/var/lib/vaultwarden`, `Restart=always`, автозапуск.

Ключевые параметры `/etc/vaultwarden/.env`:

- `DATA_FOLDER=/var/lib/vaultwarden/data`, `DATABASE_URL=/var/lib/vaultwarden/data/db.sqlite3` — SQLite, Postgres не используется.
- `ROCKET_ADDRESS=192.168.50.11`, `ROCKET_PORT=8000` — слушает только на конкретном LAN-IP, не на `0.0.0.0`.
- `DOMAIN=https://vaultwarden.kvasok.xyz` — публичный URL (CORS, WebAuthn RPID, email-ссылки, push).
- `SIGNUPS_ALLOWED=false`, `INVITATIONS_ALLOWED=false` — регистрация и приглашения закрыты, включаются временно для разовых нужд.
- `IP_HEADER=X-Forwarded-For` — реальный IP клиента из заголовка от Traefik. Безопасно, потому что nftables пускает 8000 только с Traefik.
- `ENABLE_WEBSOCKET=true` — real-time sync между клиентами.
- `ADMIN_TOKEN` не задан — админка `/admin` отключена.
- `LOG_LEVEL=warn`, логи в journald.

## 3. Сетевая фильтрация и публикация

nftables по шаблону сервисного LXC (`02-conventions.md`): `policy drop`, порт `8000` разрешён только с Traefik (`192.168.40.11`), SSH из MGMT и VPN. Прямой доступ к 8000 мимо Traefik закрыт — соединение не устанавливается.

В Traefik публикуется как `vaultwarden.kvasok.xyz` под цепочкой **`chain-external-strict`** (CrowdSec + жёсткий rate-limit + security-headers). Жёсткий лимит выбран сознательно — менеджер паролей публичен, и агрессивное ограничение скорости запросов снижает риск перебора. Детали цепочек — см. `07-traefik.md`.

## 4. Резервное копирование

Два механизма (см. `05-backup.md`): **PBS-снапшот всего LXC** ежедневно, и **restic-снапшот данных** через `/usr/local/sbin/vaultwarden-backup.sh` (таймер). Скрипт делает online-снапшот SQLite (`sqlite3 .backup` — атомарная копия работающей БД), затем `restic backup` всей `/var/lib/vaultwarden/data/`, исключая живой `db.sqlite3`, WAL-файлы, `icon_cache/`, `tmp/`. В бэкап попадают `db.sqlite3.backup`, `attachments/`, `rsa_key.pem`. Тег/host `vaultwarden`. Транспорт — rest-server на PBS (`rest:http://vaultwarden:...@192.168.10.15:8000/vaultwarden/`) в append-only, retention централизованно на PBS.

**Восстановление БД**: после `restic restore` файл лежит как `db.sqlite3.backup` — переименовать в `db.sqlite3` перед запуском. WAL-файлы пересоздадутся при старте.

## 5. Обновление

Замена бинаря и web-vault без перекомпиляции. Процедура: `skopeo copy docker://vaultwarden/server:X.Y.Z oci:...` → `umoci unpack` → остановить сервис → забэкапить текущий бинарь и web-vault (`.bak` для отката) → установить новые → запустить, проверить. Skopeo и umoci оставлены установленными для повторного использования.

## 6. Зависимости

- **Traefik (`192.168.40.11`)** — единственный разрешённый источник запросов к 8000 (nftables). Без Traefik сервис недоступен снаружи LXC.
- **Unbound на OPNsense** — DNS, split-horizon `vaultwarden.kvasok.xyz → 192.168.40.11`.
- **PBS (`192.168.10.15`)** — restic-бэкапы и PBS-снапшоты.
