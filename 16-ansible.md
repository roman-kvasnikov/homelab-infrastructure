---
name: ansible
description: |
  Ansible — декларативное управление конфигурацией homelab с одного control node (нативный LXC `ansible`, запуск плейбуков локально). Единый источник правды для двух областей: per-host nftables и SSH-hardening. Роли рендерят те же канонические конструкции, что описаны в `02-conventions.md`, из `group_vars` и `host_vars`, поэтому firewall-правила и sshd-конфиг каждого хоста задаются кодом, а не правятся руками. Документ описывает структуру репозитория, `ansible.cfg` и модель доступа, инвентарь homelab с группами, роли `nftables` и `ssh-hardening`, плейбуки обслуживания. Используй для вопросов по Ansible, автоматизации, управлению firewall-правилами и SSH через код, инвентарю, группам и плейбукам.
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
      amneziawg.yml  authelia.yml  entrypoint.yml  gotify.yml
      monitoring.yml  omada.yml  pbs.yml  pve.yml  pve-mini.yml
      traefik.yml  vaultwarden.yml
playbooks/
  update.yml               # apt update/upgrade + отчёт, без перезагрузки
  homelab/
    nftables.yml           # применить роль nftables
    nftables-audit.yml      # показать правила всех хостов без SSH
    ssh-hardening.yml       # применить роль ssh-hardening
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
| `traefik` | `192.168.40.11` | lxc |
| `vaultwarden` | `192.168.50.11` | lxc |
| `authelia` | `192.168.50.12` | lxc |
| `monitoring` | `192.168.50.21` | lxc |
| `gotify` | `192.168.50.22` | lxc |
| `xray` | `192.168.20.12` | vm |
| `dockerhost` | `192.168.50.30` | vm |
| `dev` | `192.168.50.40` | vm |
| `be-free-online` | `192.168.50.50` | vm |

### Группы

Базовые группы — по типу узла (`vps`, `proxmox`, `lxc`, `vm`); функциональные группы поверх них определяют, что и на каких хостах делает автоматизация.

| Группа | Состав | Назначение |
| :--- | :--- | :--- |
| `homelab` | `vps` + `proxmox` + `lxc` + `vm` | все управляемые хосты одним именем |
| `ssh_hardening_managed` | `proxmox` + `lxc` + `vm` | цель роли `ssh-hardening`; `vps` исключён |
| `nftables_managed` | `proxmox` + `lxc` + `vm` | цель роли `nftables`; `vps` исключён (держит свой firewall) |
| `tcp_forwarding` | `traefik`, `ansible`, `dockerhost`, `dev` | точечное разрешение TCP-forwarding через SSH-Match |
| `on_pve` / `on_pve_mini` | по хосту размещения | справочные группы физического расположения |

### group_vars

`all.yml` задаёт общие параметры подключения (`ansible_user: root`, `ansible_python_interpreter`) и — главное — словарь `nft_defines`: единый набор имя→адрес (VLAN-сети, служебные IP сервисов, `PVE_MINI_IP`, `PVE_IP`, `PBS_IP`, `VPS_WG`, `VPN_NET`, `TPLINK_SWITCH_IP`), из которого роль `nftables` генерирует блок `define` в ruleset каждого хоста. Это единственный источник адресов для всех firewall-правил, поэтому смена адреса правится в одном месте.

`vm.yml` переопределяет доступ для группы `vm`: подключение под `romank` с `become` и `ssh_extra_directives: [AllowUsers romank]`, которая добавляется в SSH-baseline этих хостов.

`tcp_forwarding.yml` задаёт `ssh_match_blocks`, разрешающий `AllowTcpForwarding yes` только для `root` с адреса админ-ноутбука (`192.168.10.50` и VPN `10.8.0.2`). Глобальный baseline forwarding запрещает (см. `02-conventions.md`, раздел 4), а эта группа точечно открывает его на хостах, через которые нужно строить SSH-туннели.

### host_vars

Файлы host_vars хранят специфику узла. Для большинства это `nft_service_rules` — порты сервиса и разрешённые источники сверх SSH; у маршрутизирующего `amneziawg` дополнительно `nft_forward_rules`; у гипервизоров `pve-mini` и `pve` — `nft_forward_policy: accept`, потому что через их мосты проходит bridged-трафик гостей (в том числе весь трафик OPNsense на PVE-Mini), который нельзя ронять host-политикой `forward`; у `entrypoint` — только параметры подключения (порт, пользователь, `become`, отключённый pipelining). Хосты без host_vars получают дефолтный ruleset: SSH из MGMT и VPN, всё остальное — drop.

| Хост | Разрешённые входящие (сверх SSH) |
| :--- | :--- |
| `pve-mini` | `8006` из MGMT, VPN, Traefik, Monitoring (pve-exporter); `9100` от Monitoring; `forward` — `accept` |
| `pve` | `8006` из MGMT, VPN, Traefik, Monitoring (pve-exporter); `3493` (NUT) от вторичных клиентов (PVE-Mini, DockerHost); `9100` от Monitoring; `forward` — `accept` |
| `pbs` | `8007` из MGMT, VPN, Traefik (публикация UI), Monitoring (pbs-exporter), DockerHost (Homepage); `8000` (rest-server) только от restic-клиентов (Vaultwarden, Authelia, DockerHost); `9100` (метрики) от Monitoring |
| `traefik` | `443` из внутренних VLAN (eth0); `443` из VPS через `wg0` (PROXY protocol); `8079` от DockerHost; `8081` и `6060` (метрики) от Monitoring |
| `vaultwarden` | `8000` только от Traefik |
| `authelia` | `9091` только от Traefik |
| `gotify` | `8060` от Traefik и DockerHost (Homepage) |
| `monitoring` | `3000` от Traefik и DockerHost; `9090` от Traefik |
| `omada` | `8043` от Traefik; UDP-discovery и TCP-adoption только от управляемого коммутатора |
| `amneziawg` | UDP `51820` из интернета; forward accept на интерфейсе `awg0` |

## 4. Роль nftables

Роль рендерит `/etc/nftables.conf` из шаблона `nftables.conf.j2` с заголовком «Managed by Ansible — do not edit by hand» и ссылкой на источник (`host_vars` + `group_vars/all.yml`). Ruleset собирается из блока `define` (весь `nft_defines`) и таблицы `inet filter`: цепочка `input` с `policy drop` содержит loopback, conntrack established/related, базовые ICMPv4/ICMPv6, SSH из `nft_ssh_saddr`, затем сгенерированные из `nft_service_rules` строки; цепочка `forward` с политикой `nft_forward_policy` и правилами `nft_forward_rules`; цепочка `output` — `accept`. Опциональная строка `nft_extra_tables` дописывается дословно после таблицы filter — для хостов, которым нужны дополнительные таблицы (например, nat REDIRECT у Xray).

Дефолты роли: `nft_ssh_saddr = { $MGMT_NET, $VPN_NET }`, пустые `nft_service_rules`, `nft_forward_policy: drop`, пустые `nft_forward_rules`. Каждое правило в `nft_service_rules` — словарь: `comment` (обязателен, идёт комментарием над правилом), `port` (одиночный или набор `{ ... }`), `proto` (`tcp` по умолчанию либо `udp`), `iif` (опционально — привязка к интерфейсу вроде `eth0`/`wg0`/`awg0`), `saddr` (опционально — источник; без него разрешено с любого адреса).

Задача деплоит шаблон с `validate: nft -c -f %s` — конфиг проверяется до записи, битый ruleset не применяется, — с `backup: true` и `mode 0644`, и по изменению дёргает handler, перечитывающий `nftables` через systemd; отдельным шагом сервис держится `enabled` и `started`. Это кодовое воплощение whitelist-шаблона из `02-conventions.md` (раздел 2) и двухслойной модели фильтрации из `04-firewall.md`.

## 5. Роль ssh-hardening

Роль управляет двумя drop-in'ами в `/etc/ssh/sshd_config.d/`, не трогая основной `sshd_config`, чтобы изменения переживали апгрейд openssh-server. `10-hardening.conf` — канонический baseline из дефолтов роли: `PermitRootLogin prohibit-password`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `X11Forwarding no`, `AllowAgentForwarding no`, `AllowTcpForwarding no`, `ClientAliveInterval 300`, `ClientAliveCountMax 2`, плюс любые `ssh_extra_directives`. `20-match.conf` — per-host блоки `Match` из `ssh_match_blocks`: каждый ослабляет одну директиву для конкретного пользователя/адреса и закрывается `Match all`, чтобы настройки не протекали дальше.

Задачи всегда деплоят `10-hardening.conf`; `20-match.conf` создаётся только при непустом `ssh_match_blocks`, иначе файл удаляется — состояние остаётся чистым. Перед перечитыванием выполняется `sshd -t` на полной эффективной конфигурации, и только при успехе handler делает reload sshd. Цель роли — группа `ssh_hardening_managed` (proxmox, lxc, vm); VPS-хост `entrypoint` под неё не подпадает и держит собственный доступ. `ssh_extra_directives` использует группа `vm` (`AllowUsers romank`), `ssh_match_blocks` — группа `tcp_forwarding`. Baseline и его обоснование — `02-conventions.md`, раздел 4.

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
