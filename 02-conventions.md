---
name: conventions
description: |
  Канонические соглашения и повторяющиеся шаблоны инфраструктуры homelab: базлайн LXC, шаблон nftables для сервисного контейнера, набор systemd-sandbox, hardening SSH, паттерн restic + rest-server, соглашения об именах. Сервисные документы ссылаются сюда вместо дублирования этих блоков.
---

# Соглашения и шаблоны

Этот документ — единый источник для повторяющихся конструкций инфраструктуры. Сервисные файлы (Vaultwarden, Authelia, Gotify, Monitoring и др.) не повторяют эти блоки, а ссылаются на соответствующий раздел здесь и описывают только специфику: имя, порт, конфиг, зависимости.

Плейсхолдеры в шаблонах записаны в угловых скобках (`<SERVICE_PORT>`, `<service>`) и подставляются под конкретный сервис.

---

## 1. Базлайн LXC

Все сервисные LXC разворачиваются по единому базлайну.

Контейнер **unprivileged** (`unprivileged: 1`) — root внутри маппится в nobody на хосте через user namespaces, эскалация до хоста невозможна. Из features включён только **`nesting=1`**. Он нужен не для Docker (его в сервисных LXC нет), а для корректной работы systemd 257 в Debian Trixie: современный systemd активно использует user namespaces для собственного sandbox'а юнитов (`PrivateUsers=`, `PrivateTmp=` и т.п.), и без `nesting=1` падают journald, tmpfiles-setup и часть служебных юнитов. Эскалации до хоста `nesting=1` в unprivileged-контейнере не даёт.

Опасные features не включаются: `keyctl`, `mount=`, `mknod` отсутствуют. Hostpath-mounts не используются. Диск лежит на ZFS-пуле гипервизора.

Сеть контейнера — тегированный VLAN-интерфейс на VLAN-aware мосту гипервизора (тег назначается по роли сервиса согласно схеме VLAN в `03-network.md`). Адрес статический либо резервируется в Kea DHCP по MAC; шлюз и DNS — адрес OPNsense в соответствующем VLAN (`192.168.<vlan>.1`).

Каждый сервис слушает **только на своём конкретном адресе**, не на `0.0.0.0` — loopback и прочие интерфейсы для сервиса недоступны. Прямой доступ к порту дополнительно ограничивается на уровне nftables (см. раздел 2).

---

## 2. Шаблон nftables для сервисного LXC

Сетевая фильтрация на всех LXC построена по единому whitelist-подходу: `policy drop` на цепочках `input` и `forward`, разрешено только явно перечисленное, всё остальное молча отбрасывается. Исходящий трафик не ограничивается (`output` policy accept). Конфиг лежит в `/etc/nftables.conf` и загружается через `nftables.service` при старте контейнера.

### Стандартный сервисный контейнер

Типовой сервис (Vaultwarden, Authelia, Gotify и подобные) принимает входящие **только от Traefik** — прямой доступ к порту из сети, минуя reverse-proxy с его middleware-цепочкой (CrowdSec, rate-limit, security headers), закрыт. SSH разрешён только из **MGMT** и через **VPN**.

Разница между сервисами в этом шаблоне — ровно одна строка: порт сервиса (`<SERVICE_PORT>`). Всё остальное идентично.

```nft
#!/usr/sbin/nft -f

flush ruleset

define MGMT_NET   = 192.168.10.0/24
define VPN_NET    = 10.8.0.0/24

define TRAEFIK_IP = 192.168.40.11

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        # Loopback
        iifname "lo" accept

        # Conntrack
        ct state established,related accept
        ct state invalid drop

        # ICMPv4 - for ping and path MTU discovery
        ip protocol icmp icmp type {
            destination-unreachable,
            time-exceeded,
            parameter-problem,
            echo-request
        } accept

        # ICMPv6 - for NDP
        ip6 nexthdr icmpv6 icmpv6 type {
            destination-unreachable,
            packet-too-big,
            time-exceeded,
            parameter-problem,
            echo-request,
            echo-reply,
            nd-router-advert,
            nd-router-solicit,
            nd-neighbor-advert,
            nd-neighbor-solicit
        } accept

        # SSH - only from MGMT_NET and VPN_NET
        tcp dport 22 ip saddr { $MGMT_NET, $VPN_NET } accept

        # Service only from Traefik
        tcp dport <SERVICE_PORT> ip saddr $TRAEFIK_IP accept

        # Everything else falls into policy drop
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

Разрешено: loopback, ответные пакеты (conntrack established/related), базовые ICMPv4/ICMPv6 (ping и NDP), SSH (22) из `MGMT_NET` и `VPN_NET`, порт сервиса только с адреса Traefik. Прямой доступ к порту сервиса из других сегментов мимо Traefik закрыт — соединение не устанавливается, отдаётся timeout.

### Отклонения от шаблона

Сервисы, которым нужен доступ к порту не только от Traefik, добавляют дополнительные `accept`-строки. Каждое такое отклонение описывается в файле конкретного сервиса с обоснованием. Типовые случаи: открытие порта для Monitoring LXC (скрейп метрик), открытие UI для всей доверенной сети вместо только Traefik.

---

## 3. Набор systemd-sandbox

Сервисы, работающие нативным бинарником под systemd, запускаются от выделенного системного юзера с единым набором sandbox-директив. Набор ограничивает права процесса на уровне ядра: запрет эскалации, read-only файловая система вне явно разрешённых путей, изоляция `/tmp`, `/home`, устройств и kernel-интерфейсов.

### Базовый набор

```ini
[Service]
User=<service>
Group=<service>

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectKernelLogs=true
ProtectControlGroups=true
RestrictNamespaces=true
LockPersonality=true

ReadWritePaths=/var/lib/<service>

Restart=always
RestartSec=10
```

`ProtectSystem=strict` делает всю файловую систему read-only, кроме путей в `ReadWritePaths` — обычно это единственный `/var/lib/<service>`, куда сервис пишет БД и состояние. Запись в `/var/log` при этом запрещена, поэтому логи идут в journald (`journalctl -u <service>`), файлового логирования нет.

### Усиленный набор

Для сервисов с повышенными требованиями (например, IdP, хранящий все аутентификации) базовый набор дополняется:

```ini
PrivateUsers=true
CapabilityBoundingSet=
SystemCallFilter=@system-service
```

`CapabilityBoundingSet=` (пустой) убирает все Linux capabilities. `SystemCallFilter=@system-service` разрешает только типовой для сервисов набор syscall'ов. Применение усиленного набора отмечается в файле конкретного сервиса.

### Зависимости и автозапуск

Сервисы с зависимостью от локальных компонентов (например, Redis) объявляют её через `Requires=` + `After=`, чтобы зависимость стартовала первой. Все сервисы включены в автозапуск (`systemctl enable`).

---

## 4. Hardening SSH

На всех хостах (LXC и физических серверах) sshd ужесточается одинаковым drop-in `/etc/ssh/sshd_config.d/10-hardening.conf`. Основной `/etc/ssh/sshd_config` не трогается — изменения переживают апгрейды openssh-server.

```
PermitRootLogin prohibit-password
PasswordAuthentication no
KbdInteractiveAuthentication no
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

- `PermitRootLogin prohibit-password` — root только по ключу, не по паролю.
- `PasswordAuthentication no` — парольная аутентификация отключена полностью.
- `KbdInteractiveAuthentication no` — закрыт обходной путь через keyboard-interactive.
- `X11Forwarding no` — на headless-узлах не нужен.
- `AllowAgentForwarding no` — закрыт типичный путь lateral movement через ssh-agent.
- `AllowTcpForwarding no` — узел нельзя использовать как туннельный прокси через `ssh -L`.
- `ClientAliveInterval 300` - если клиент не отвечает на сообщение keepalive в течение заданного интервала, сервер считает, что клиент больше недоступен.
- `ClientAliveCountMax 2` - определяет количество keepalive-сообщений, которые могут быть отправлены клиенту без получения ответа, прежде чем сервер завершит соединение.

Аутентификация — по ed25519-ключам. Доступ к SSH ограничен на уровне nftables (MGMT + VPN, см. раздел 2). Защита от brute-force — коллекция CrowdSec `crowdsecurity/sshd` на Traefik LXC.

---

## 5. Паттерн restic + rest-server

Сервисы с собственным критичным состоянием (БД, конфиги) поверх PBS-снапшота всего LXC делают независимый **restic**-бэкап своих данных. Это даёт гранулярное восстановление отдельных файлов и конкретной версии от N дней назад без подъёма всего хоста. Приёмник — **rest-server** на бэкап-сервере (`192.168.10.15:8000`).

### Разделение репозиториев и режимы rest-server

Каждый сервис пишет в свой репозиторий `/mnt/data/restic/<service>/`. Изоляция обеспечивается режимом **`--private-repos`**: аутентифицированный клиент видит только свой репо `/<service>/`, без доступа к чужим. Режим **`--append-only`** блокирует удаление: клиент может писать новые снапшоты, но DELETE-операции возвращают 403 на уровне API. Скомпрометированный сервис не способен уничтожить существующие бэкапы — только записать новые.

### Два пароля

У каждого репо два независимых пароля, это разные механизмы:

- **HTTP basic-auth пароль** — транспортная аутентификация в rest-server. Хранится на клиенте в `/var/lib/<service>/.restic-http-password` (chmod 600, owner `<service>`).
- **Encryption-пароль** — шифрование содержимого репо. Хранится на клиенте в `/var/lib/<service>/.restic-password` и продублирован на бэкап-сервере в `/etc/restic-retention/<service>.password` для retention-операций.

URL репозитория: `rest:http://<service>:<http-пароль>@192.168.10.15:8000/<service>/`.

Содержимое шифруется restic-клиентом до отправки — encryption-пароль по сети не передаётся, только зашифрованные данные. Транспорт HTTP без TLS приемлем внутри сети: перехват basic-auth ограничен активными атаками с уже скомпрометированного устройства, а append-only не даёт при перехвате уничтожить бэкапы.

### Backup на клиенте

Скрипт `/usr/local/sbin/<service>-backup.sh` запускается systemd-таймером `<service>-backup.timer` ежедневно (типово 01:00 MSK). Для сервисов на SQLite первым шагом делается online-снапшот через `sqlite3 .backup` — атомарная копия, безопасная для работающего сервиса. Затем `restic backup` данных, исключая живой `db.sqlite3`, WAL-файлы (`-shm`, `-wal`), кэши и файлы паролей. Скрипт retention **не** выполняет — append-only это физически блокирует.

### Retention централизованно на бэкап-сервере

Retention для всех репо выполняется единым скриптом `/usr/local/sbin/restic-retention.sh` на бэкап-сервере под юзером **`rest-server`** (тем же, под которым работает сам rest-server). Единый owner на всём `/mnt/data/restic/` критичен: при `restic prune` перепакованные index/pack-файлы наследуют owner процесса, и запуск под root оставил бы файлы `root:root` mode 600, недоступные rest-server — следующий клиент получал бы `500` при открытии репо.

Единая политика для всех репо: `--keep-daily 7 --keep-weekly 4 --keep-monthly 6 --group-by host,tags --prune`. Таймер — ежедневно 03:00 MSK с `RandomizedDelaySec=30min`, после клиентских backup-таймеров, чтобы не пересекаться по локам.

Encryption-пароли репо хранятся на двух хостах (на клиенте — для backup, на бэкап-сервере — для retention). Это ограничение restic: любая операция, изменяющая снапшоты, требует encryption-пароль, включая локальный forget. Изоляция от компрометации клиента обеспечивается не разделением паролей, а append-only режимом rest-server.

### Восстановление БД

После `restic restore` файл SQLite лежит как `db.sqlite3.backup` — перед запуском сервиса его переименовывают в `db.sqlite3`. WAL-файлы при восстановлении не нужны, пересоздаются при первом старте.

---

## 6. Соглашения об именах

- **kebab-case** — для hostname, имён файлов, systemd-юнитов, storage ID, идентификаторов сервисов (`rest-server`, `vaultwarden-backup.timer`).
- **snake_case** — для переменных в bash-скриптах.
- **UPPER_SNAKE_CASE** — для ENV-констант и `define`-блоков nftables (`TRAEFIK_IP`, `MGMT_NET`).
- Без type-префиксов в именах.

**PBS:** storage `pbs`, datastore `Homelab`, namespaces `pve` и `pve-mini`.

**Язык:** документация и обсуждение — на русском; комментарии в конфигах и заголовки алертов — на английском.

---

## 7. Общие зависимости сервисов

Каждый сервисный LXC типово зависит от трёх внешних узлов. В файле сервиса перечисляются только отклонения и специфика.

- **Traefik (`192.168.40.11`, DMZ)** — единственный разрешённый источник запросов к порту сервиса (nftables). Без Traefik сервис недоступен снаружи LXC.
- **Unbound на OPNsense** (шлюз VLAN сервиса) — DNS, включая split-horizon `*.kvasok.xyz → 192.168.40.11`.
- **Бэкап-сервер / PBS (`192.168.10.15`, MGMT)** — PBS-снапшоты LXC и (для сервисов с restic) приём restic-бэкапов.
