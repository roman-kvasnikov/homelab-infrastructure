---
name: proxy-xray
description: |
  Xray-прокси для гео-обхода: VM в INFRA. Два режима — forward-прокси (SOCKS/HTTP) для явных потребителей (arr-стек на DockerHost) и прозрачное проксирование телевизоров через REDIRECT + policy-routing на OPNsense. Документ описывает конфиг Xray, nftables, sysctl, автозапуск и связку с OPNsense. Используй для вопросов по гео-обходу, проксированию телевизоров, arr-стека.
---

# Xray-прокси

Xray обеспечивает выборочный гео-обход: заблокированные ресурсы идут через зарубежный VLESS-outbound, российские и локальные — напрямую. Работает в двух режимах: forward-прокси (SOCKS/HTTP) для сервисов, которые настроены ходить через него явно, и прозрачное проксирование конкретных устройств (телевизоров) через заворот трафика на OPNsense.

Xray развёрнут **виртуальной машиной** (не LXC) в сегменте INFRA, `192.168.20.12`. Выбор VM принципиален: прозрачное проксирование требует манипуляций с сетевым стеком ядра (форвардинг, nat-redirect, отключение rp_filter), которые в unprivileged-LXC недоступны без послаблений контейнера. ОС — Ubuntu 24.04, Xray-core под systemd.

## 1. Роль и потоки

Forward-прокси: сервисы, которым нужен гео-обход (медиа-стек — Jellyfin, Radarr, Sonarr, Jellyseerr и др. на DockerHost), настроены ходить в интернет через прокси-порты Xray (`192.168.20.12:10809` HTTP или `:10808` SOCKS). Xray сам решает по routing-правилам, что проксировать, а что пустить напрямую.

Прозрачное проксирование: телевизоры (сегмент IOT) не поддерживают ручную настройку прокси, поэтому их трафик заворачивается на Xray на уровне маршрутизации OPNsense, без настройки на самих устройствах. На Xray-VM трафик перехватывается через nat-REDIRECT и попадает в transparent-inbound.

```
Forward-прокси:
  [DockerHost сервис] --HTTP/SOCKS--> [Xray 192.168.20.12:10809/10808] --VLESS--> зарубежный сервер
                                                              \--direct--> RU / локалка

Прозрачно (ТВ):
  [ТВ 192.168.60.11] --> [OPNsense policy-route на XRAY_GW]
        --> [Xray-VM: nat REDIRECT tcp -> :12345] --> transparent inbound
              --VLESS--> зарубежный сервер (заблокированное)
              --direct--> RU / локалка
```

## 2. Конфигурация Xray

Конфиг — `/usr/local/etc/xray/config.json`. Ключевые блоки.

**Outbounds.** Основной — `proxy` (VLESS + REALITY на зарубежный сервер `ee.be-free.online:443`, транспорт xhttp, fingerprint firefox). Служебные — `direct` (freedom) и `reject` (blackhole).

**DNS.** DoH-серверы: `https://77.88.8.8/dns-query` (Yandex) для российских доменов (`geosite:category-ru`, `domain:ru/su/рф/xn--p1ai`), `https://8.8.8.8/dns-query` для остального. `queryStrategy: UseIPv4`, `disableFallbackIfMatch`. Это разрешает RU-домены через российский DNS (корректная гео-выдача), остальное — через Google.

**Routing.** Порядок правил определяет выборочность: приватные сети (`geoip:private`, `geosite:private`) → `direct`; DNS к `77.88.8.8` → `direct`; DNS-inbound → `proxy`; bittorrent → `direct`; telegram (geoip/geosite) → `proxy`; российские адреса и домены (`geoip:ru`, `geosite:category-ru`, `domain:ru/su/рф`, `kvasok.xyz`, `be-free.online`, `boosty.to`, `geosite:ozon/wildberries/adobe`, список конкретных IP) → `direct`; почтовые/служебные порты (25, 465, 587, 1080, 9050, 6881-6889) → `direct`; всё остальное → `proxy`. `domainStrategy: IPIfNonMatch`, `domainMatcher: hybrid`.

**Inbounds.** Три штуки:

- `socks` — SOCKS5 на `0.0.0.0:10808`, `udp: true`, для потребителей forward-прокси.
- `http` — HTTP на `0.0.0.0:10809`, для потребителей forward-прокси.
- `transparent` — dokodemo-door на `12345`, `network: tcp`, `followRedirect: true`, sniffing (`http`, `tls`) с `routeOnly`. Принимает redirected-трафик телевизоров. **Без** tproxy-sockopt — используется REDIRECT-схема (см. раздел 4), а не TPROXY.

Все три inbound прогоняются через единый routing-блок — логика проксирования одна для forward-прокси и для прозрачного трафика.

## 3. Потребители forward-прокси

Сервисы на DockerHost (медиа-стек), которым нужен гео-обход, настроены ходить через HTTP-прокси Xray: `HTTP_PROXY=http://192.168.20.12:10809` (или SOCKS `192.168.20.12:10808`). Каждый такой сервис отправляет исходящий трафик в Xray, где routing решает proxy/direct. Это межсегментный доступ SERVICES → INFRA — при включении изоляции между VLAN его нужно явно разрешить на OPNsense (`192.168.50.30 → 192.168.20.12:10808,10809`).

## 4. Прозрачное проксирование телевизоров (REDIRECT)

Телевизоры не умеют настраивать прокси, поэтому их трафик заворачивается на Xml на уровне сети. Используется схема **REDIRECT** (nat-DNAT на локальный порт), а не TPROXY. REDIRECT проще и надёжнее: не требует policy-routing на loopback, отдельной таблицы маршрутизации, transparent-сокета и отключения rp_filter как обязательного условия. Ограничение REDIRECT — работает только с TCP; UDP/QUIC перехватить нельзя, поэтому QUIC (UDP 443) блокируется на OPNsense, и телевизоры откатываются на TCP, который проксируется.

### 4.1. На Xray-VM

**nat-REDIRECT** (nftables, таблица `ip xray_nat`): форвардящийся TCP-трафик телевизоров переписывается на локальный порт `12345`, где его принимает transparent-inbound. Трафик к приватным сетям не проксируется (`return`).

**Порт 12345 разрешён в input.** Критично: после REDIRECT пакет адресуется локально и проходит цепочку `input`. При `policy drop` без явного разрешения порта 12345 переписанный пакет молча отбрасывается — это тихо ломает всю схему. Порт 12345 открыт в input таблицы `inet filter`.

**sysctl** (`/etc/sysctl.d/99-xray-proxy.conf`): `net.ipv4.ip_forward=1` (VM форвардит транзитный трафик телевизоров), `net.ipv4.conf.all.rp_filter=0` и `default.rp_filter=0` (отключён reverse-path-filtering для перенаправляемого трафика).

### 4.2. На OPNsense

**Gateway** `XRAY_GW` = `192.168.20.12` (интерфейс INFRA), мониторинг отключён.

**Alias** `tv_hosts` — список IP телевизоров (`192.168.60.11`, `.12`, `.13`). Телевизорам зарезервированы фиксированные адреса в IOT по MAC, вне DHCP-пула (`.100–.200`).

**Правила на интерфейсе IOT** (порядок важен, first-match сверху вниз):

1. Anti-lockout (автоправило).
2. `tv_hosts → private_networks`, gateway default — локальный трафик телевизоров идёт обычной маршрутизацией, не в Xray.
3. `tv_hosts` UDP dport 443 → **block** — блок QUIC, телевизоры откатываются на TCP.
4. `tv_hosts → any`, gateway `XRAY_GW` — весь остальной (TCP) трафик телевизоров уходит на Xray.
5. Общие правила IOT.

Правило блока QUIC (3) обязательно стоит **до** заворота (4), а локальный трафик (2) — до блока QUIC и заворота, чтобы телевизоры сохраняли прямой доступ к DNS и локальным сервисам.

## 5. nftables на Xray-VM

Полный `/etc/nftables.conf`. Whitelist с `policy drop` на input, nat-таблица для REDIRECT. Автозагрузка через `nftables.service` (enabled).

```nft
#!/usr/sbin/nft -f
flush ruleset

define MGMT_NET    = 192.168.10.0/24
define VPN_NET     = 10.8.0.0/24

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        iifname "lo" accept
        ct state established,related accept
        ct state invalid drop

        ip protocol icmp icmp type {
            destination-unreachable, time-exceeded, parameter-problem, echo-request
        } accept
        ip6 nexthdr icmpv6 icmpv6 type {
            destination-unreachable, packet-too-big, time-exceeded, parameter-problem,
            echo-request, echo-reply,
            nd-router-advert, nd-router-solicit, nd-neighbor-advert, nd-neighbor-solicit
        } accept

        # SSH - only from MGMT and VPN
        tcp dport 22 ip saddr { $MGMT_NET, $VPN_NET } accept

        # Xray forward-proxy ports (socks/http) for LAN consumers
        tcp dport { 10808, 10809 } accept

        # Xray transparent redirect port (TVs via REDIRECT)
        tcp dport 12345 accept
    }

    chain forward {
        type filter hook forward priority filter; policy accept;
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}

table ip xray_nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;

        # Local / private destinations are not proxied
        ip daddr { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 127.0.0.0/8, 224.0.0.0/4, 255.255.255.255 } return

        # Forwarded TCP traffic (from TVs) -> local Xray redirect port
        ip protocol tcp redirect to :12345
    }
}
```

Цепочка `forward` пока `policy accept` (VM транзитно форвардит трафик телевизоров) — ужесточается при общем hardening. Прокси-порты `10808/10809` открыты без ограничения источника — сужаются до SERVICES при hardening. SSH — из MGMT и VPN по общему канону (см. `conventions.md`). SSH ужесточён drop-in'ом `10-hardening.conf` (см. `conventions.md`).

## 6. sysctl

`/etc/sysctl.d/99-xray-proxy.conf`:

```ini
# IP forwarding for transit traffic (TVs -> Xray -> internet)
net.ipv4.ip_forward=1

# Disable reverse path filtering for redirected proxy traffic
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```

## 7. Автозапуск

Всё поднимается при старте VM без ручных действий: Xray (`systemctl enable xray`), nftables (`nftables.service` enabled, грузит `/etc/nftables.conf` с nat-таблицей), sysctl (из `/etc/sysctl.d/`). Policy-routing и таблицы маршрутизации не используются (это было только для TPROXY), поэтому после перезагрузки ничего добавлять руками не нужно.

## 8. Зависимости

- **Зарубежный VLESS-сервер** (`ee.be-free.online`) — outbound-узел, через него идёт проксируемый трафик. Недоступность сервера ломает гео-обход (но `direct`-трафик работает).
- **OPNsense** — gateway `XRAY_GW`, policy-route и блок QUIC для телевизоров; маршрутизация INFRA.
- **DockerHost (`192.168.50.30`)** — потребитель forward-прокси (медиа-стек через `10809`).

## 9. Ключевые особенности

- **VM, не LXC.** Прозрачное проксирование требует форвардинга, nat-redirect и правки rp_filter — в unprivileged-LXC это упирается в ограничения контейнера. VM снимает их.
- **REDIRECT вместо TPROXY.** TPROXY даёт нативный перехват TCP и UDP, но требует хрупкой связки (метка → отдельная таблица маршрутизации → loopback → transparent-сокет, отключение rp_filter, поддержка модулей ядра). REDIRECT проще: обычный nat-DNAT на локальный порт, без policy-routing и transparent-сокета. Цена — только TCP; QUIC (UDP 443) блокируется на OPNsense, YouTube откатывается на TCP.
- **`policy drop` в input и redirect-порт.** После REDIRECT пакет адресуется локально и проходит `input`. Если порт (12345) не разрешён явно при `policy drop` — пакет молча отбрасывается, и проксирование тихо не работает без единой ошибки в логах. Порт redirect обязателен в input-allowlist.
- **Разделение RU/зарубеж по routing.** Российские ресурсы (по geoip/geosite/списку доменов) идут `direct` — не через зарубежный прокси. Это и быстрее, и не ломает сервисы с гео-привязкой к РФ.
