---
name: vaultwarden
description: |
  Vaultwarden — self-hosted менеджер паролей. LXC в SERVICES, нативный бинарник из Docker-образа, SQLite, доступ только через Traefik под жёстким rate-limit. Документ описывает установку, конфигурацию, сетевую фильтрацию, публикацию, резервное копирование (PBS + offsite restic в Timeweb S3) и обновление. Используй для вопросов по Vaultwarden, менеджеру паролей, хранению секретов, offsite-бэкапу.
---

# Vaultwarden

Self-hosted менеджер паролей в отдельном непривилегированном LXC (`192.168.50.11`, SERVICES). Работает нативно (без Docker) на бинарнике, извлечённом из официального Docker-образа `vaultwarden/server`. БД — SQLite. Доступ только через Traefik. Версия зафиксирована, обновление вручную через замену бинарника. Базлайн LXC, systemd-sandbox, SSH-hardening, nftables-шаблон и паттерн бэкапа данных — см. `02-conventions.md`.

## 1. Файловая структура

```
/usr/local/bin/vaultwarden                     # binary extracted from the official Docker image
/usr/share/vaultwarden/web-vault/              # web UI static
/etc/vaultwarden/.env                          # configuration (managed-volume on zdata), root:vaultwarden 0640
/var/lib/vaultwarden/data/                     # data (managed-volume on zdata)
├── db.sqlite3                                 # main DB (SQLite, WAL mode)
├── db.sqlite3.bak                             # hourly application-consistent SQLite snapshot, 0600
├── .env.bak                                   # hourly copy of /etc/vaultwarden/.env, 0600
├── attachments/                               # record attachments
├── rsa_key.pem                                # JWT signing key
├── icon_cache/                                # site icon cache
└── tmp/                                       # temporary files
/usr/local/sbin/vaultwarden-backup-prepare.sh  # prepares data dir for backup (.env.bak + db.sqlite3.bak)
/usr/local/sbin/vaultwarden-backup.sh          # offsite restic backup to Timeweb S3
/etc/restic/password                           # restic repository password, root:root 0600
/etc/restic/vaultwarden.env                    # restic repository + S3 credentials, root:root 0600
/var/cache/restic/                             # restic cache (systemd CacheDirectory)
```

И `/etc/vaultwarden` (`.env`), и `/var/lib/vaultwarden/data` вынесены на пул `zdata` отдельными managed-volume (mount points LXC), а не на rootfs: конфигурация и данные — на отказоустойчивом RAIDZ1 и в составе PBS-снапшота (см. `06-backup.md`).

Бинарь и web-vault извлекаются из официального Docker-образа через `skopeo` + `umoci` — те же файлы, что использует Docker, но без Docker-обёртки. Для динамической линковки установлены `libmariadb3` и `libpq5`, хотя используется только SQLite (Vaultwarden слинкован со всеми тремя driver-библиотеками).

## 2. Systemd и конфигурация

Сервис `vaultwarden.service` от системного юзера `vaultwarden` (UID 999), sandbox-набор из `02-conventions.md`, запись только в `/var/lib/vaultwarden`, `Restart=always`, автозапуск.

Файл `/etc/vaultwarden/.env` принадлежит `root:vaultwarden` с правами `0640`: systemd читает его как `EnvironmentFile` от root, а группа `vaultwarden` даёт право чтения сервису подготовки бэкапа.

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

В Traefik публикуется как `vaultwarden.kvasok.xyz` под цепочкой **`chain-external-strict`** (CrowdSec + жёсткий rate-limit + security-headers). Жёсткий лимит выбран сознательно — менеджер паролей публичен, и агрессивное ограничение скорости запросов снижает риск перебора. Детали цепочек — см. `08-traefik.md`.

Исходящий HTTPS к `s3.twcstorage.ru` (Timeweb S3) нужен для offsite-бэкапа и разрешён существующими правилами OPNsense и nftables без отдельных исключений.

## 4. Резервное копирование

Два независимых уровня: локальный PBS-снапшот всего LXC и offsite restic-бэкап данных в Timeweb S3. Оба опираются на один и тот же подготовленный каталог `/var/lib/vaultwarden/data`.

### 4.1. Подготовка данных

Systemd-таймер `vaultwarden-backup-prepare.timer` (`OnCalendar=hourly`, `Persistent=true`) ежечасно запускает `vaultwarden-backup-prepare.service`, который делает каталог `data/` самодостаточным для бэкапа:

- `install -m 0600 /etc/vaultwarden/.env data/.env.bak` — актуальная копия конфигурации внутри каталога данных.
- `sqlite3 db.sqlite3 ".backup 'db.sqlite3.bak'"` — атомарный консистентный снапшот работающей БД, затем `chmod 0600` на результат.

Сервис работает от `vaultwarden:vaultwarden`, `Type=oneshot`, sandbox `NoNewPrivileges=true`, `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`, запись только в `/var/lib/vaultwarden/data`.

Оба `.bak` содержат секреты (токены, хэши, метаданные хранилищ), поэтому права `0600` выставляются явно при каждом запуске: `sqlite3 .backup` сохраняет права уже существующего файла.

### 4.2. PBS

Данные (`/var/lib/vaultwarden/data`) и конфигурация (`/etc/vaultwarden/.env`) лежат на managed-volume пула `zdata` и попадают в ежедневный PBS-снапшот всего LXC вместе с rootfs (см. `06-backup.md`). PBS-снапшот захватывает свежие `.bak` в консистентном виде; `attachments/` и `rsa_key.pem` попадают в снапшот как есть. Восстановление LXC из PBS возвращает Vaultwarden вместе с БД, вложениями, ключом и конфигом.

### 4.3. Offsite: restic → Timeweb S3

Offsite-копия защищает от потери всего, что находится в квартире (сервер и PBS). Репозиторий — `s3:https://s3.twcstorage.ru/<bucket>/homelab-backups/vaultwarden`, формат v2 со сжатием. Структура `homelab-backups/<service>` общая для всех сервисов с offsite-бэкапом: один репозиторий на сервис, бэкап выполняет LXC самого сервиса.

Секреты в `/etc/restic/` (`root:root`, каталог `0700`, файлы `0600`):

- `password` — пароль репозитория. Пароль единый для всех репозиториев `homelab-backups/*` и хранится оффлайн вне homelab и вне Vaultwarden.
- `vaultwarden.env` — `RESTIC_REPOSITORY`, `RESTIC_PASSWORD_FILE=/etc/restic/password`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `RESTIC_CACHE_DIR=/var/cache/restic`.

Скрипт `/usr/local/sbin/vaultwarden-backup.sh` бэкапит `/var/lib/vaultwarden/data` с тегом `vaultwarden` и затем применяет retention:

- Исключения: `db.sqlite3`, `db.sqlite3-wal`, `db.sqlite3-shm` (живая БД неконсистентна на лету), `icon_cache/`, `tmp/`.
- В снапшот попадают `db.sqlite3.bak`, `.env.bak`, `attachments/`, `rsa_key.pem`.
- Retention: `forget --prune --keep-daily 7 --keep-weekly 4 --keep-monthly 12`.

Юнит `vaultwarden-backup.service`:

- `Requires=` + `After=vaultwarden-backup-prepare.service` — перед бэкапом всегда делается свежая подготовка; если она упала, бэкап не запускается и устаревшие данные не уходят в S3.
- `Wants=` + `After=network-online.target`.
- `EnvironmentFile=/etc/restic/vaultwarden.env`, `CacheDirectory=restic` (`0700`).
- Работает от root с `CapabilityBoundingSet=CAP_DAC_READ_SEARCH` — единственное право, нужное для чтения `data/` (`vaultwarden`, `0700`).
- Sandbox: `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `ProtectKernel*`, `ProtectControlGroups`, `ProtectClock`, `ProtectHostname`, `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX`, `RestrictNamespaces`, `RestrictRealtime`, `RestrictSUIDSGID`, `LockPersonality`, `SystemCallArchitectures=native`, `SystemCallFilter=@system-service`. `PrivateUsers` не используется — в непривилегированном LXC не работает.
- `Nice=10`, `IOSchedulingClass=idle`.

Таймер `vaultwarden-backup.timer`: `OnCalendar=*-*-* 03:30:00`, `RandomizedDelaySec=15min`, `Persistent=true`. PBS-диски не затрагиваются, поэтому время не зависит от ночного графика PBS.

Ручные операции с репозиторием выполняются с загрузкой env в subshell, чтобы переменные не оставались в сессии:

```bash
(set -a; . /etc/restic/vaultwarden.env; set +a
 restic snapshots
 restic check --read-data)
```

### 4.4. Восстановление

**Из PBS**: после восстановления LXC актуальная консистентная база — `db.sqlite3.bak`; перед стартом сервиса её переименовывают в `db.sqlite3`, живой `db.sqlite3` из снапшота и WAL-файлы отбрасывают. Паттерн — общий, см. `02-conventions.md`.

**Из offsite**: для восстановления нужны пароль restic (хранится оффлайн) и S3-ключи (из панели Timeweb, доступ к которой тоже хранится оффлайн). На новом LXC с установленным Vaultwarden (сервис остановлен):

```bash
(set -a; . /etc/restic/vaultwarden.env; set +a
 restic snapshots
 restic restore latest --target /tmp/restore)

r=/tmp/restore/var/lib/vaultwarden/data
d=/var/lib/vaultwarden/data

sqlite3 "$r/db.sqlite3.bak" 'PRAGMA integrity_check;'

rm -f "$d"/db.sqlite3 "$d"/db.sqlite3-wal "$d"/db.sqlite3-shm
cp -a "$r/attachments" "$r/rsa_key.pem" "$d/"
install -m 0600 -o vaultwarden -g vaultwarden "$r/db.sqlite3.bak" "$d/db.sqlite3"
install -m 0640 -o root -g vaultwarden "$r/.env.bak" /etc/vaultwarden/.env
chown -R vaultwarden:vaultwarden "$d"

systemctl start vaultwarden
rm -rf /tmp/restore
```

## 5. Обновление

Замена бинаря и web-vault без перекомпиляции. Процедура: `skopeo copy docker://vaultwarden/server:X.Y.Z oci:...` → `umoci unpack` → остановить сервис → забэкапить текущий бинарь и web-vault (`.bak` для отката) → установить новые → запустить, проверить. Skopeo и umoci оставлены установленными для повторного использования.

restic устанавливается из штатного репозитория Debian (`apt install restic`) и обновляется вместе с системой.

## 6. Зависимости

- **Traefik (`192.168.40.11`)** — единственный разрешённый источник запросов к 8000 (nftables). Без Traefik сервис недоступен снаружи LXC.
- **Unbound на OPNsense** — DNS, split-horizon `vaultwarden.kvasok.xyz → 192.168.40.11`.
- **PBS (`192.168.10.15`)** — PBS-снапшоты LXC (данные и конфиг на zdata попадают в снапшот).
- **Timeweb S3 (`s3.twcstorage.ru`)** — offsite restic-репозиторий `homelab-backups/vaultwarden`.
