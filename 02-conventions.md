---
name: conventions
description: |
  Канонические соглашения и повторяющиеся шаблоны инфраструктуры homelab: базлайн LXC (нативный и Docker-in-LXC), шаблон nftables для сервисного контейнера, набор systemd-sandbox, hardening SSH, паттерн бэкапа данных сервиса через managed-volume + vzdump с SQLite-хуком, соглашения об именах. Сервисные документы ссылаются сюда вместо дублирования этих блоков.
---

# Соглашения и шаблоны

Этот документ — единый источник для повторяющихся конструкций инфраструктуры. Сервисные файлы (Vaultwarden, Authelia, Gotify, Monitoring и др.) не повторяют эти блоки, а ссылаются на соответствующий раздел здесь и описывают только специфику: имя, порт, конфиг, зависимости.

Плейсхолдеры в шаблонах записаны в угловых скобках (`<SERVICE_PORT>`, `<service>`) и подставляются под конкретный сервис.

---

## 1. Базлайн LXC

Все сервисные LXC — **unprivileged** (`unprivileged: 1`): root внутри маппится в nobody на хосте через user namespaces, эскалация до хоста невозможна. По способу запуска сервиса контейнеры делятся на два типа.

### Нативный LXC

Сервис работает нативным бинарником под systemd. Из features включён только **`nesting=1`** — не для Docker, а для корректной работы systemd в Debian: современный systemd активно использует user namespaces для собственного sandbox'а юнитов (`PrivateUsers=`, `PrivateTmp=` и т.п.), и без `nesting=1` падают journald, tmpfiles-setup и часть служебных юнитов. Эскалации до хоста `nesting=1` в unprivileged-контейнере не даёт. `keyctl`, `mount=`, `mknod` не включаются.

### Docker-in-LXC

Сервисы, у которых нет вменяемой нативной установки (Immich, Frigate и подобные), работают в Docker внутри unprivileged LXC. Features — **`nesting=1,keyctl=1`** (`keyctl` нужен Docker для работы с ключами). В `/etc/docker/daemon.json` задаётся `default-address-pools` из приватного диапазона `10.200.0.0/16`, чтобы автоматически создаваемые Docker-сети не пересеклись с физическими подсетями homelab. Данные приложений и медиатеки монтируются в контейнер через mount points (см. раздел про хранилище в `05-proxmox.md`), внутри Docker пробрасываются в сервис как volume.

### Общее для обоих типов

Диск лежит на ZFS-пуле гипервизора. Данные приложения, которые нужно бэкапить, размещаются на **Proxmox-managed volume** (`storage:SIZE`, попадает в vzdump); крупные медиа и записи — на **bind-mount** обычного датасета (не в vzdump). Hostpath-mounts под конфиги не используются.

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

## 5. Паттерн бэкапа данных сервиса

Бэкап всех LXC — снапшот целиком через **vzdump** в PBS (`06-backup.md`). Данные сервиса, которые нужно резервировать, кладутся на **Proxmox-managed volume** отдельным mount point — такой том попадает в vzdump вместе с rootfs. Крупные датасеты, которые в PBS-снапшот не гонятся (медиатеки, фото, записи камер), выносятся на bind-mount с флагом `backup=0`.

### Managed-volume под данные

Данные сервиса (БД, конфиг, состояние) размещаются на managed-volume ZFS-пула, а не на rootfs. Том добавляется как mount point (`storage:SIZE,mp=/var/lib/<service>/data`) и по умолчанию входит в vzdump. Это даёт две вещи: данные лежат на пуле с CoW-снапшотами, и восстановление LXC из PBS возвращает сервис вместе с его состоянием.

### Консистентность SQLite

Vzdump снимает файловую систему в один момент, но живой SQLite может быть в середине транзакции с неслитым WAL. Для сервисов на SQLite рядом с боевой базой поддерживается application-consistent копия: systemd-таймер `<service>-db-backup.timer` ежечасно запускает `<service>-db-backup.service`, который делает `sqlite3 db.sqlite3 ".backup db.sqlite3.bak"` — атомарный снапшот, безопасный на работающем сервисе. Vzdump захватывает свежий `.bak` в консистентном виде; при восстановлении из него берётся рабочая база.

Скрипт `/usr/local/sbin/<service>-db-backup.sh` работает под юзером сервиса, oneshot-юнит несёт тот же sandbox-набор, что и сам сервис (`ProtectSystem=strict`, `ReadWritePaths=` на директорию данных, см. раздел 3). WAL/`-shm` в `.bak` не переносятся — SQLite сливает их в момент `.backup`.

### Восстановление

LXC целиком поднимается из PBS-снапшота (`06-backup.md`). Для сервиса на SQLite после восстановления актуальная консистентная база — это `db.sqlite3.bak`: перед стартом сервиса её переименовывают в `db.sqlite3`, живой `db.sqlite3` из снапшота и его WAL-файлы отбрасывают.

---

## 6. Соглашения об именах

- **kebab-case** — для hostname, имён файлов, systemd-юнитов, storage ID, идентификаторов сервисов (`rest-server`, `vaultwarden-backup.timer`).
- **snake_case** — для переменных в bash-скриптах.
- **UPPER_SNAKE_CASE** — для ENV-констант и `define`-блоков nftables (`TRAEFIK_IP`, `MGMT_NET`).
- Без type-префиксов в именах.

**PBS:** storage `pbs`, datastore `Homelab`, namespace `pve`.

**Язык:** документация и обсуждение — на русском; комментарии в конфигах и заголовки алертов — на английском.

---

## 7. Общие зависимости сервисов

Каждый сервисный LXC типово зависит от трёх внешних узлов. В файле сервиса перечисляются только отклонения и специфика.

- **Traefik (`192.168.40.11`, DMZ)** — единственный разрешённый источник запросов к порту сервиса (nftables). Без Traefik сервис недоступен снаружи LXC.
- **Unbound на OPNsense** (шлюз VLAN сервиса) — DNS, включая split-horizon `*.kvasok.xyz → 192.168.40.11`.
- **Бэкап-сервер / PBS (`192.168.10.15`, MGMT)** — приёмник vzdump-снапшотов LXC.
