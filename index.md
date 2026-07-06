---
name: Homelab-Infrastructure
description: |
  Индекс документации домашней инфраструктуры. Homelab на двух хостах Proxmox с OPNsense в роли маршрутизатора и firewall, сегментацией на 8 VLAN, VPS как точкой публикации сервисов через WireGuard-туннель к Traefik и AmneziaWG для удалённого доступа. Этот файл — точка входа: высокоуровневая архитектура и карта остальных документов. За деталями по конкретной области переходи в соответствующий файл.
---

# Домашняя инфраструктура

Индекс и высокоуровневое описание homelab. Для конкретной области — сети, гипервизоров, reverse-proxy, аутентификации, бэкапов и т.д. — переходи в отдельный документ по карте в разделе 3. Общие шаблоны (базлайн LXC, nftables, systemd-sandbox, restic) вынесены в `conventions.md`, сервисные документы ссылаются на них вместо дублирования.

## 1. Общая архитектура

Ядро инфраструктуры — **OPNsense**, единственный маршрутизатор, firewall и DHCP/DNS-сервер. Он запущен виртуальной машиной на мини-ПК под Proxmox. Кабель провайдера заходит напрямую в мини-ПК на интерфейс WAN OPNsense; белый IP терминируется на нём. Весь трафик между сегментами и выход в интернет идут через OPNsense.

Сеть разделена на **восемь VLAN** (MGMT, INFRA, TRUSTED, DMZ, SERVICES, IOT, CCTV, GUEST). В каждом сегменте OPNsense — шлюз `.1`, раздаёт адреса через Kea DHCP и резолвит имена через Unbound. Все домашние устройства, проводные и беспроводные, находятся за OPNsense в своих VLAN.

Вычислительная нагрузка распределена на **два хоста Proxmox**: мини-ПK несёт сетевую плоскость (OPNsense, Omada, AmneziaWG, Xray), основной сервер — прикладные сервисы (DockerHost, Traefik, Vaultwarden, Authelia, Monitoring и др.). Управляемый коммутатор раздаёт VLAN тегированным транком; два роутера Keenetic работают беспроводными точками доступа (L2), привязывая SSID к VLAN.

Вход извне организован по двум независимым путям. **Публичные сервисы** доступны через VPS: домашний Traefik держит исходящий WireGuard-туннель к VPS, наружу порты не пробрасываются (интернет → VPS → WG-туннель → Traefik → бэкенд). **Удалённый доступ в сеть** — через AmneziaWG (INFRA) с обфускацией трафика, по белому IP дома напрямую, минуя VPS.

```
                         Интернет
                            │
              ┌─────────────┴──────────────┐
              ▼                            ▼
       ┌─────────────┐              белый IP дома
       │ VPS         │              (WAN OPNsense)
       │ WireGuard   │                     │
       └──────┬──────┘                     │
              │ WG-туннель                 ▼
              ▼                    ┌──────────────────┐
       ┌──────────────┐           │ OPNsense          │
       │ Traefik       │ ────────► │ маршрутизатор     │
       │ DMZ .40.11    │           │ + firewall + DNS  │
       └──────────────┘           │ (VM на PVE-Mini)  │
                                   └────────┬─────────┘
       ┌──────────────┐  порт-форвард       │
       │ AmneziaWG     │ ◄──────────────────┤
       │ INFRA .20.11  │                    ▼
       └──────────────┘        8 VLAN за одним роутером
                               MGMT · INFRA · TRUSTED · DMZ
                               SERVICES · IOT · CCTV · GUEST
```

## 2. Домен и публикация

Основной домен — **kvasok.xyz**. Wildcard `*.kvasok.xyz` публикует self-hosted сервисы. Сертификаты Let's Encrypt получает Traefik через DNS-01 challenge; VPS сертификаты не хранит и TLS не терминирует (работает на L4).

Резолвинг внутри сети — Unbound на OPNsense со split-horizon `*.kvasok.xyz → 192.168.40.11` (Traefik): устройства обращаются к сервисам по локальному адресу reverse-proxy, не выходя в интернет.

## 3. Карта документации

| Документ | Область |
| :--- | :--- |
| **`Homelab-Infrastructure.md`** | Этот файл — индекс, высокоуровневая архитектура, глоссарий. |
| **`conventions.md`** | Канонические шаблоны: базлайн LXC, nftables сервисного контейнера, systemd-sandbox, hardening SSH, паттерн restic + rest-server, соглашения об именах. Остальные документы ссылаются сюда. |
| **`home-network.md`** | Сеть: VLAN-схема и адресация, OPNsense (WAN, интерфейсы, DHCP, Unbound, firewall), коммутатор и раскладка портов, беспроводная сеть (Keenetic AP), DNS, потоки трафика. |
| **`compute-proxmox.md`** | Гипервизоры: PVE-Mini и PVE-Main, storage и ZFS, лимиты ресурсов гостей, UPS/NUT, инвентарь VM и LXC. |
| **`edge-vps.md`** | Публичная точка входа: VPS, nginx L4 stream + PROXY protocol, служебный WireGuard-туннель к Traefik. |
| **`vpn-amneziawg.md`** | Удалённый доступ в сеть: AmneziaWG (INFRA), обфускация, клиенты, маскарад, порт-форвард на OPNsense. |
| **`proxy-xray.md`** | Xray-прокси (INFRA): гео-обход для сервисов (forward-прокси) и прозрачное проксирование телевизоров через REDIRECT. |
| **`ingress-traefik.md`** | Traefik и CrowdSec: домены, сертификаты, entrypoints, middleware и цепочки, catch-all, метрики. |
| **`identity-authelia.md`** | Authelia и общая модель аутентификации: IdP, forward-auth, OIDC, TOTP/WebAuthn, локальный Redis. |
| **`services-core.md`** | Одноцелевые нативные LXC: Vaultwarden, Gotify. Тонкие карточки со ссылками на `conventions.md` плюс специфика. |
| **`services-dockerhost.md`** | DockerHost VM и Docker-стек: ZFS-датасеты, networking, AppData, контейнеры, Samba. |
| **`monitoring.md`** | Monitoring LXC: Prometheus, Grafana, экспортеры, дашборды, алертинг в Gotify. |
| **`backup.md`** | Бэкап-сервер и PBS, rest-server, restic, трёхуровневая схема, retention, сценарий восстановления. |

## 4. Глоссарий

- **OPNsense** — маршрутизатор, firewall, DHCP и DNS всей сети; VM на PVE-Mini. Единственная точка маршрутизации между VLAN и выхода в интернет.
- **PVE-Mini / PVE-Main** — два хоста Proxmox: сетевая плоскость и прикладные сервисы соответственно.
- **PBS** — Proxmox Backup Server, приёмник снапшотов VM/LXC и host-бэкапов; на нём же rest-server для restic.
- **Traefik** — reverse-proxy в DMZ, единственная точка входа HTTP-трафика в сеть; держит WG-туннель к VPS.
- **VPS** — внешний сервер, точка публикации сервисов; проксирует HTTPS в WG-туннель к Traefik на L4, сертификаты не хранит.
- **AmneziaWG** — обфусцированный VPN-сервер (INFRA) для удалённого доступа в сеть по белому IP дома.
- **Xray** — прокси гео-обхода (INFRA): forward-прокси для сервисов и прозрачное проксирование телевизоров; через него же ходит исходящий трафик CrowdSec.
- **Unbound** — рекурсивный DNS-резолвер на OPNsense со split-horizon для `*.kvasok.xyz`.
- **Keenetic (Giga / Speedster)** — два роутера в режиме точки доступа (L2), вещают SSID с привязкой к VLAN.
- **Omada** — контроллер управляемого коммутатора TP-Link.
- **DockerHost** — VM с основным стеком self-hosted сервисов в Docker и Samba.
- **Authelia** — IdP и forward-auth для Traefik: TOTP, WebAuthn, OIDC.
- **CrowdSec** — движок реактивной защиты рядом с Traefik, bouncer как плагин Traefik.
- **rest-server** — HTTP-приёмник restic-бэкапов на PBS в режиме append-only + private-repos.
- **VLAN 10–80** — сегменты сети: MGMT (управление), INFRA (инфраструктурные сервисы), TRUSTED (доверенные устройства), DMZ (Traefik), SERVICES (прикладные сервисы), IOT, CCTV, GUEST.
