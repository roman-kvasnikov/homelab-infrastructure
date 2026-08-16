---
name: ansible
description: |
  Ansible — декларативное управление конфигурацией homelab с одного control node (нативный LXC `ansible`, запуск плейбуков локально). Единый источник правды для двух областей: per-host nftables и SSH-hardening. Роли рендерят те же канонические конструкции, что описаны в `02-conventions.md`, из `group_vars` и `host_vars`, поэтому firewall-правила и sshd-конфиг каждого хоста задаются кодом, а не правятся руками. Инвентарём покрыт весь фронт сервисов — сеть, гипервизоры, reverse-proxy, аутентификация, мониторинг, медиастек, файловые шары, PostgreSQL. Документ описывает структуру репозитория, `ansible.cfg` и модель доступа, инвентарь homelab с группами, роли `nftables` и `ssh-hardening`, плейбуки обслуживания. Используй для вопросов по Ansible, автоматизации, управлению firewall-правилами и SSH через код, инвентарю, группам и плейбукам.
---

# Ansible

Управление конфигурацией homelab централизовано на одном control node — нативном LXC `ansible` (CTID `199`, `192.168.10.99`, MGMT) на PVE-Main. Плейбуки запускаются на нём же (`ansible_connection=local`), а до остальных хостов Ansible достаёт по SSH из сегмента MGMT. Репозиторий — источник правды: nftables-ruleset и SSH-hardening каждого хоста генерируются из инвентаря шаблонами ролей, а не редактируются на месте. Базовые конструкции (whitelist-шаблон nftables, SSH-hardening baseline) описаны в `02-conventions.md`; этот документ показывает, как они применяются кодом и где хранится специфика каждого хоста.

## 1. Структура репозитория

Репозиторий разделён на инвентарь, плейбуки и роли; общая логика живёт в ролях, а всё, что отличает хосты друг от друга, вынесено в `group_vars` и `host_vars`.

```
ansible.cfg
inventories/
  homelab/
    hosts.ini
    group_vars/
      all.yml              # ansible_user, nft_defines (единый словарь адресов)
      vm.yml               # доступ и SSH-директивы для группы vm
      tcp_forwarding.yml   # Match-исключение AllowTcpForwarding
    host_vars/
      amneziawg.yml  arr.yml         authelia.yml  entrypoint.yml
      frigate.yml    gotify.yml      homepage.yml  immich.yml
      jellyfin.yml   monitoring.yml  omada.yml     organizer.yml
      pbs.yml        postgres.yml    pve.yml       pve-mini.yml
      qbittorrent.yml  shares.yml    traefik.yml   vaultwarden.yml
playbooks/
  update.yml               # apt update/upgrade + отчёт, без перезагрузки
  homelab/
    nftables.yml           # применить роль nftables
    nftables-audit.yml     # показать правила всех хостов без SSH
    ssh-hardening.yml      # применить роль ssh-hardening
roles/
  nftables/                # рендер /etc/nftables.conf
  ssh-hardening/           # рендер drop-in'ов в /etc/ssh/sshd_config.d/
```

## 2. ansible.cfg и модель доступа

`[defaults]` задаёт `roles_path = ./roles` и `playbook_dir = ./playbooks`, чистит вывод (`display_skipped_hosts = false`, `display_ok_hosts = true`) и включает `host_key_checking = true`, чтобы падать явно при неизвестном ключе хоста. `[ssh_connection]` включает `pipelining = true` (меньше SSH-операций на задачу) и передаёт `ssh_args = -o StrictHostKeyChecking=accept-new -o ControlMaster=auto -o ControlPersist=60s`.

`ControlMaster=auto` + `ControlPersist=60s` мультиплексируют SSH: одно соединение переиспользуется для серии задач, что ускоряет прогон и снижает риск словить fail2ban на быстрых последовательных подключениях к внешнему узлу.

Аутентификация — по ed25519-ключам. На большинстве хостов подключение под `root` (`group_vars/all.yml`); группа `vm` управляется под пользователем `romank` с `become` (`group_vars/vm.yml`); VPS-хост `entrypoint` подключается под `super` на порту `12122` с `become` и с `ansible_ssh_pipelining: false` — на нём в sudoers включён `requiretty`, несовместимый с pipelining. Секреты и приватные ключи в репозиторий не коммитятся.

## 3. Инвентарь homelab

### Хосты

| Хост | Адрес | Тип |
| :--- | :--- | :--- |
| `entrypoint` | VPS (порт 12122) | vps |
| `pve-mini` | `192.168.10.11` | proxmox |
| `pve` | `192.168.10.12` | proxmox |
| `pbs` | `192.168.10.15` | proxmox |
| `omada` | `192.168.10.31` | lxc |
| `ansible` | `192.168.10.99` (local) | lxc — control node |
| `amneziawg` | `192.168.20.11` | lxc |
| `homepage` | `192.168.20.20` | lxc |
| `traefik` | `192.168.40.11` | lxc |
| `vaultwarden` | `192.168.50.11` | lxc |
| `authelia` | `192.168.50.12` | lxc |
| `monitoring` | `192.168.50.21` | lxc |
| `gotify` | `192.168.50.22` | lxc |
| `jellyfin` | `192.168.50.31` | lxc |
| `arr` | `192.168.50.32` | lxc |
| `qbittorrent` | `192.168.50.33` | lxc |
| `organizer` | `192.168.50.34` | lxc |
| `immich` | `192.168.50.35` | lxc |
| `shares` | `192.168.50.36` | lxc |
| `frigate` | `192.168.50.37` | lxc |
| `postgres` | `192.168.50.90` | lxc |

Группа `vm` объявлена в инвентаре, но сейчас пуста — хосты Xray, Dev и Be-Free.Online закомментированы, поэтому функциональные группы, включающие `vm`, фактически резолвятся в `proxmox` + `lxc`.

### Группы

Базовые группы — по типу узла (`vps`, `proxmox`, `lxc`, `vm`); функциональные группы поверх них определяют, что и на каких хостах делает автоматизация.

| Группа | Состав | Назначение |
| :--- | :--- | :--- |
| `homelab` | `vps` + `proxmox` + `lxc` + `vm` | все управляемые хосты одним именем |
| `ssh_hardening_managed` | `proxmox` + `lxc` + `vm` | цель роли `ssh-hardening`; `vps` исключён |
| `nftables_managed` | `proxmox` + `lxc` + `vm` | цель роли `nftables`; `vps` исключён (держит свой firewall) |
| `tcp_forwarding` | `traefik`, `homepage`, `ansible` | точечное разрешение TCP-forwarding через SSH-Match |
| `on_pve_mini` / `on_pve` | по хосту размещения | справочные группы физического расположения |

Группы размещения не совпадают с VLAN: например, `homepage` адресован в INFRA (`192.168.20.20`), но физически живёт на PVE-Main (`on_pve`), а `amneziawg` (INFRA, `192.168.20.11`) размещён на PVE-Mini (`on_pve_mini`).

### group_vars

`all.yml` задаёт общие параметры подключения (`ansible_user: root`, `ansible_python_interpreter`) и — главное — словарь `nft_defines`: единый набор имя→адрес, из которого роль `nftables` генерирует блок `define` в ruleset каждого хоста. Это единственный источник адресов для всех firewall-правил, поэтому смена адреса правится в одном месте. Словарь покрывает VLAN-сети (`MGMT_NET`, `INFRA_NET`, `TRUSTED_NET`, `DMZ_NET`, `SERVICES_NET`, `IOT_NET`, `CCTV_NET`, `GUEST_NET`), служебные адреса (`VPS_WG`, `VPN_NET`) и IP отдельных узлов (`PVE_MINI_IP`, `PVE_IP`, `PBS_IP`, `SWITCH_IP`, `AMNEZIAWG_IP`, `XRAY_IP`, `HOMEPAGE_IP`, `TRAEFIK_IP`, `VAULTWARDEN_IP`, `AUTHELIA_IP`, `MONITORING_IP`, `JELLYFIN_IP`, `ARR_IP`, `QBITTORRENT_IP`, `ONLYOFFICE_IP`). В словаре есть и адреса узлов, которых пока нет в активном инвентаре (`XRAY_IP`, `ONLYOFFICE_IP`) — они нужны как разрешённые источники в чужих правилах (`ONLYOFFICE_IP` открывает FileBrowser на `shares`).

`vm.yml` переопределяет доступ для группы `vm`: подключение под `romank` с `become` и `ssh_extra_directives: [AllowUsers romank]`, которая добавляется в SSH-baseline этих хостов.

`tcp_forwarding.yml` задаёт `ssh_match_blocks`, разрешающий `AllowTcpForwarding yes` только для пользователей `root,romank` с адреса админ-ноутбука (`192.168.10.50`) и VPN (`10.8.0.2`). Глобальный baseline forwarding запрещает (см. `02-conventions.md`, раздел 4), а эта группа точечно открывает его на хостах, через которые нужно строить SSH-туннели.

### host_vars

Файлы host_vars хранят специфику узла. Для большинства это `nft_service_rules` — порты сервиса и разрешённые источники сверх SSH; у маршрутизирующего `amneziawg` дополнительно `nft_forward_rules`; гипервизоры (`pve`, `pve-mini`) и хосты с Docker-in-LXC (`frigate`, `immich`, `organizer`) держат `nft_forward_policy: accept`, потому что через их мосты проходит транзитный трафик — bridged-трафик гостей на гипервизорах (в том числе весь трафик OPNsense на PVE-Mini) и трафик внутреннего контейнерного моста в Docker-in-LXC, — который нельзя ронять host-политикой `forward`; у `entrypoint` — только параметры подключения (порт, пользователь, `become`, отключённый pipelining). Хосты без host_vars получают дефолтный ruleset: SSH из MGMT и VPN, всё остальное — drop.

Сквозной мотив whitelist'ов — два разрешённых потребителя почти у каждого сервиса: `TRAEFIK_IP` как reverse-proxy для пользовательского входа и `HOMEPAGE_IP`, потому что дашборд Homepage дёргает бэкенды сервисов напрямую для виджетов статуса.

| Хост | Разрешённые входящие (сверх SSH) | `forward` |
| :--- | :--- | :--- |
| `pve-mini` | `8006` из MGMT, VPN, Homepage, Traefik, Monitoring (pve-exporter); `9100` от Monitoring | `accept` |
| `pve` | `8006` из MGMT, VPN, Homepage, Traefik, Monitoring (pve-exporter); `3493` (NUT) от вторичных клиентов (PVE-Mini, PBS, PeaNUT); `9100` от Monitoring | `accept` |
| `pbs` | `8007` из MGMT, VPN, Homepage, Traefik (публикация UI), Monitoring (pbs-exporter); `9100` от Monitoring | `drop` |
| `omada` | `8043` от Traefik и Homepage; UDP `{29810,19810,27001}` (discovery) и TCP `{29811–29817}` (adoption/management) только от управляемого коммутатора (`SWITCH_IP`) | `drop` |
| `amneziawg` | UDP `51820` из интернета (любой источник) | `drop` + `awg0` in/out accept |
| `homepage` | `3000` только от Traefik | `drop` |
| `traefik` | `443` (eth0) из MGMT, INFRA, TRUSTED, SERVICES, IOT, VPN; `443` (wg0) от VPS (`VPS_WG`, PROXY protocol); `8079` (eth0) от Homepage (Traefik API); `8081` (eth0) от Monitoring (метрики Traefik); `6060` (eth0) от Monitoring (метрики CrowdSec) | `drop` |
| `vaultwarden` | `8000` только от Traefik | `drop` |
| `authelia` | `9091` только от Traefik | `drop` |
| `monitoring` | `3000` (Grafana), `9090` (Prometheus), `8080` (PeaNUT) от Traefik и Homepage | `drop` |
| `gotify` | `8060` от Traefik и Homepage | `drop` |
| `jellyfin` | `8096` от Traefik и Homepage | `drop` |
| `arr` | `5055` (Seer), `9696` (Prowlarr), `7878` (Radarr), `8989` (Sonarr), `8096` (Jellyfin) от Traefik и Homepage | `drop` |
| `qbittorrent` | `8080` от Traefik, Homepage и arr-стека (`ARR_IP`) | `drop` |
| `organizer` | `3000` (Anchor), `5000` (ByteStash), `8080` (Baikal) только от Traefik | `accept` |
| `immich` | `2283` только от Traefik | `accept` |
| `shares` | `445` (Samba) из MGMT, VPN, TRUSTED; `3300` (FileBrowser) от Traefik, OnlyOffice и Homepage | `drop` |
| `frigate` | `5000` только от Traefik | `accept` |
| `postgres` | `5432` (PostgreSQL) и `6379` (Redis) из всего сегмента SERVICES | `drop` |

## 4. Роль nftables

Роль рендерит `/etc/nftables.conf` из шаблона `nftables.conf.j2` с заголовком «Managed by Ansible — do not edit by hand» и ссылкой на источник (`host_vars` + `group_vars/all.yml`). Ruleset собирается из блока `define` (весь `nft_defines`) и таблицы `inet filter`: цепочка `input` с `policy drop` содержит loopback, conntrack established/related (и drop invalid), базовые ICMPv4/ICMPv6, SSH из `nft_ssh_saddr`, затем сгенерированные из `nft_service_rules` строки; цепочка `forward` с политикой `nft_forward_policy` и правилами `nft_forward_rules`; цепочка `output` — `accept`. Опциональная строка `nft_extra_tables` дописывается дословно после таблицы filter — для хостов, которым нужны дополнительные таблицы (например, nat REDIRECT у Xray).

Дефолты роли: `nft_ssh_saddr = { $MGMT_NET, $VPN_NET }`, пустые `nft_service_rules`, `nft_forward_policy: drop`, пустые `nft_forward_rules`. Каждое правило в `nft_service_rules` — словарь: `comment` (обязателен, идёт комментарием над правилом), `port` (одиночный или набор `{ ... }`), `proto` (`tcp` по умолчанию либо `udp`), `iif` (опционально — привязка к интерфейсу вроде `eth0`/`wg0`/`awg0`), `saddr` (опционально — источник; без него разрешено с любого адреса).

Задача деплоит шаблон с `validate: nft -c -f %s` — конфиг проверяется до записи, битый ruleset не применяется, — с `backup: true` и `mode 0644`, и по изменению дёргает handler, перечитывающий `nftables` через systemd (`state: reloaded`); отдельным шагом сервис держится `enabled` и `started`. Это кодовое воплощение whitelist-шаблона из `02-conventions.md` (раздел 2) и двухслойной модели фильтрации из `04-firewall.md`.

## 5. Роль ssh-hardening

Роль управляет двумя drop-in'ами в `/etc/ssh/sshd_config.d/`, не трогая основной `sshd_config`, чтобы изменения переживали апгрейд openssh-server. `10-hardening.conf` — канонический baseline из дефолтов роли: `PermitRootLogin prohibit-password`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `X11Forwarding no`, `AllowAgentForwarding no`, `AllowTcpForwarding no`, `ClientAliveInterval 300`, `ClientAliveCountMax 2`, плюс любые `ssh_extra_directives`. `20-match.conf` — per-host блоки `Match` из `ssh_match_blocks`: каждый ослабляет одну директиву для конкретного пользователя/адреса и закрывается `Match all`, чтобы настройки не протекали дальше.

Задачи всегда деплоят `10-hardening.conf`; `20-match.conf` создаётся только при непустом `ssh_match_blocks`, иначе файл удаляется — состояние остаётся чистым. Отдельная задача выполняет `sshd -t` на полной эффективной конфигурации (`changed_when: false`), а handler по изменению любого drop-in'а делает reload сервиса `ssh`. Цель роли — группа `ssh_hardening_managed` (proxmox, lxc, vm); VPS-хост `entrypoint` под неё не подпадает и держит собственный доступ. `ssh_extra_directives` использует группа `vm` (`AllowUsers romank`), `ssh_match_blocks` — группа `tcp_forwarding`. Baseline и его обоснование — `02-conventions.md`, раздел 4.

## 6. Плейбуки

`update.yml` (hosts `all`, `serial: 1`) обновляет apt-хосты по одному: `update_cache`, `dist-upgrade`, `autoclean`/`autoremove --purge`/`clean`, проверка `/var/run/reboot-required` и отчёт по каждому хосту с флагом необходимости перезагрузки. Сам плейбук не перезагружает — только отчитывается.

`homelab/nftables.yml` применяет роль `nftables` к группе `nftables_managed`. Предпросмотр дрейфа без изменений — `ansible-playbook -i inventories/homelab playbooks/homelab/nftables.yml --check --diff --limit <host>`; раскатка ведётся по одному хосту.

`homelab/nftables-audit.yml` работает `connection: local` и без SSH в гости: читает `host_vars` на control node и печатает по одной карточке на хост — что каждый узел разрешает во `input`. Это инструмент обзора всех firewall-правил инфраструктуры разом.

`homelab/ssh-hardening.yml` применяет роль `ssh-hardening` к `ssh_hardening_managed`. Раскатывать по одному хосту, держа наготове второй SSH-сеанс и доступ через `pct enter` / `qm terminal` на случай ошибки в конфиге.

## 7. Зависимости

- **Control node в MGMT** — nftables всех хостов разрешают SSH (`22`) из `MGMT_NET` и `VPN_NET`, поэтому Ansible достаёт любой узел; при сужении правил это учитывается.
- **`group_vars/all.yml → nft_defines`** — единственный источник адресов для рендера ruleset; правки адресации идут сюда.
- **`02-conventions.md`** — канонические baseline'ы (nftables-шаблон, SSH-hardening), которые роли воплощают в коде.
- **`04-firewall.md`** — двухслойная модель фильтрации, внутрисегментный слой которой (per-host nftables) и разворачивается ролью `nftables`.
