---
name: vpn-amneziawg
description: |
  AmneziaWG — обфусцированный VPN-сервер для удалённого доступа в домашнюю сеть. LXC в INFRA, вход по белому IP через порт-форвард на OPNsense, минуя VPS. Документ описывает конфигурацию сервера, обфускацию, клиентов, маскарад, nftables, автозапуск и связку с OPNsense. Используй для вопросов по удалённому доступу к дому, VPN, AmneziaWG.
---

# AmneziaWG

AmneziaWG обеспечивает удалённый доступ в домашнюю сеть извне. Протокол — обфусцированный WireGuard (маскировка трафика под случайный поток для обхода DPI). Подключившись, клиент получает доступ к внутренним сегментам так, как будто находится дома.

AmneziaWG развёрнут непривилегированным LXC в сегменте INFRA, `192.168.20.11`. Слушает UDP `51820`. Вход снаружи — по белому IP дома через порт-форвард на OPNsense, напрямую, без участия VPS. VPN-подсеть клиентов — `10.8.0.0/24`, адрес сервера в туннеле — `10.8.0.1`.

Реализация — userspace (`amneziawg-go`): kernel-модуля AmneziaWG в LXC нет, поэтому интерфейс поднимается через userspace-бэкенд. Это штатно для контейнера и не требует модулей на хосте.

## 1. Поток трафика

```
[Клиент вне дома] --AWG / UDP 51820--> [белый IP дома]
      --> [OPNsense: порт-форвард 51820 -> 192.168.20.11]
            --> [AmneziaWG LXC awg0 10.8.0.1]
                  --masquerade--> [внутренние сегменты]
```

Клиент подключается на белый IP дома, порт 51820. OPNsense пробрасывает этот UDP-порт на AmneziaWG-LXC в INFRA. LXC принимает туннель, и трафик клиента маскарадится в адрес LXC (`192.168.20.11`), после чего расходится по внутренним сегментам через OPNsense.

## 2. Конфигурация сервера

Конфиг — `/etc/amnezia/amneziawg/awg0.conf`.

**Interface.** `Address = 10.8.0.1/24`, `ListenPort = 51820`, приватный ключ сервера. Параметры обфускации заголовков: `S1`, `S2`, `H1–H4` — задают структуру пакетов и должны совпадать с клиентскими. Junk-параметры (`Jc/Jmin/Jmax`) на сервере отсутствуют — это клиентские настройки (мусорные пакеты перед рукопожатием генерирует инициатор соединения), серверу они не нужны.

**Peers.** Каждый клиент — отдельная секция `[Peer]` с публичным ключом клиента, симметричным preshared-ключом и `AllowedIPs = 10.8.0.<N>/32` (уникальный адрес клиента в VPN-подсети).

Запуск требует userspace-переменной: интерфейс поднимается с `AWG_QUICK_USERSPACE_IMPLEMENTATION=amneziawg-go`, иначе `awg-quick` пытается создать kernel-интерфейс, которого в LXC нет.

## 3. Клиенты

Клиентский конфиг зеркалит обфускацию сервера и добавляет junk-параметры.

```ini
[Interface]
PrivateKey = <client private key>
Address = 10.8.0.<N>/32
DNS = 192.168.10.1

# Junk packets (client-side obfuscation)
Jc = 4
Jmin = 40
Jmax = 70
# Header obfuscation - must match server
S1 = 50
S2 = 100
H1 = <...>
H2 = <...>
H3 = <...>
H4 = <...>

[Peer]
PublicKey = <server public key>
PresharedKey = <preshared key>
Endpoint = <home white IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Ключевые поля: `S1/S2/H1–H4` — точно как на сервере (иначе рукопожатие не пройдёт); `Jc/Jmin/Jmax` — клиентские, на сервере их нет, задаются здесь; `PresharedKey` — симметричный, одинаковый на сервере и клиенте; `DNS = 192.168.10.1` — Unbound на OPNsense (split-horizon для `*.kvasok.xyz` работает и через туннель); `AllowedIPs = 0.0.0.0/0` — full tunnel (весь трафик через дом; для доступа только к домашним сетям используется split — `192.168.0.0/16, 10.8.0.0/24`).

Приложение на устройстве — именно **AmneziaWG**, не обычный WireGuard: обычный клиент не понимает параметры обфускации. Конфиг удобно передавать QR-кодом: `qrencode -t ansiutf8 < client.conf`.

### 3.1. Добавление клиента

Генерация ключей на сервере: `awg genkey | tee client.key | awg pubkey > client.pub`, PSK — `awg genpsk`. Новый `[Peer]` добавляется в `awg0.conf` с уникальным `AllowedIPs = 10.8.0.<N>/32` и применяется на живом интерфейсе без разрыва существующих клиентов: `awg syncconf awg0 <(awg-quick strip awg0)`.

## 4. Маскарад и модель адресации

VPN-трафик **маскарадится** на самом AmneziaWG-LXC: выходя из туннеля в остальную сеть, пакеты клиентов получают source-адрес LXC (`192.168.20.11`, INFRA). Это осознанное упрощение — VPN-клиентов немного (владелец с парой устройств), различать их по отдельным адресам не нужно.

Следствие для firewall всей инфраструктуры: VPN-трафик приходит к сервисам под адресом INFRA (`192.168.20.11`), а не под своим `10.8.0.<N>`. Поэтому в allow-листах сервисов и в middleware `trusted` на Traefik достаточно разрешить INFRA (`192.168.20.0/24`) — отдельное правило для `10.8.0.0/24` избыточно везде, **кроме самого AmneziaWG-LXC**: SSH к нему через VPN приходит с реальным `10.8.0.<N>` (пакет доставляется локально, не форвардится, потому не маскарадится), поэтому VPN-подсеть остаётся в его SSH-allow.

Компромисс маскарада: весь INFRA (включая Xray) для сервисов выглядит одинаково с VPN — firewall не различает «пришёл через VPN» и «пришёл хост INFRA». Для текущего масштаба это приемлемо; при необходимости разделить — маскарад снимается, добавляется статический маршрут `10.8.0.0/24` на OPNsense, и клиенты сохраняют оригинальный адрес.

## 5. nftables

Полный `/etc/nftables.conf`. Whitelist с `policy drop` на input, маскарад VPN-подсети в nat. Автозагрузка через `nftables.service` (enabled).

```nft
#!/usr/sbin/nft -f

flush ruleset

define MGMT_NET = 192.168.10.0/24
define VPN_NET  = 10.8.0.0/24

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

        # SSH - only from MGMT, VPN
        tcp dport 22 ip saddr { $MGMT_NET, $VPN_NET } accept

        # AmneziaWG - inbound VPN connections from the internet (any source)
        udp dport 51820 accept

        # Everything else falls into policy drop
    }

    chain forward {
        type filter hook forward priority filter; policy drop;

        # VPN traffic in both directions
        iifname "awg0" accept
        oifname "awg0" accept
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        # Masquerade VPN subnet traffic going out to the LAN
        ip saddr $VPN_NET oifname "eth0" masquerade
    }
}
```

`udp dport 51820 accept` без ограничения источника — VPN-подключения приходят с меняющихся внешних адресов. SSH — из MGMT и VPN по общему канону (см. `conventions.md`); VPN здесь реальный (`10.8.0.<N>`), потому что SSH к самому LXC доставляется локально и не маскарадится. Forward пропускает VPN-клиентов из `awg0` в `eth0` (наружу), обратное направление — только по conntrack established. IP forwarding включён (`net.ipv4.ip_forward=1`), без него LXC не маршрутизирует трафик клиентов.

## 6. OPNsense: порт-форвард

Вход снаружи обеспечивается правилом на OPNsense: Firewall → NAT → Destination NAT (Port Forward). Interface WAN, protocol UDP, destination — WAN address port `51820`, redirect target `192.168.20.11:51820`, associated filter rule. Это направляет входящий VPN-трафик с белого IP на AmneziaWG-LXC в INFRA. VPS в этом пути не участвует — доступ прямой по IP дома.

## 7. Автозапуск

Интерфейс `awg0` поднимается при старте LXC через шаблонный юнит `awg-quick@awg0.service` с drop-in, задающим userspace-бэкенд:

```ini
# /etc/systemd/system/awg-quick@awg0.service.d/override.conf
[Service]
Environment=AWG_QUICK_USERSPACE_IMPLEMENTATION=amneziawg-go
```

Юнит включён в автозапуск (`systemctl enable awg-quick@awg0`). nftables (`nftables.service`) и sysctl (`ip_forward`) тоже персистентны, поэтому после перезагрузки LXC весь VPN-стек поднимается сам, без ручных действий.

## 8. SSH

SSH ужесточён общим drop-in `/etc/ssh/sshd_config.d/10-hardening.conf` (см. `conventions.md`). Доступ к порту 22 ограничен на nftables источниками MGMT и VPN. Аутентификация — по ключам.

## 9. Зависимости

- **OPNsense** — порт-форвард UDP 51820 на LXC (вход снаружи), маршрутизация VPN-трафика между сегментами, Unbound как DNS для клиентов.
- **Белый IP дома** — endpoint для клиентов; VPN завязан на прямую доступность WAN-адреса дома.

## 10. Ключевые особенности

- **Прямой вход, минуя VPS.** AmneziaWG доступен по белому IP дома через порт-форвард на OPNsense. VPS используется только для публикации сервисов (HTTP через WG-туннель к Traefik), к VPN-доступу отношения не имеет.
- **Userspace-реализация в LXC.** Kernel-модуля AmneziaWG в контейнере нет — интерфейс работает через `amneziawg-go`. Требует userspace-переменной при подъёме (учтено в systemd drop-in).
- **Маскарад вместо per-client маршрутизации.** VPN-трафик выходит под адресом LXC (INFRA), что упрощает firewall (разрешить INFRA вместо VPN-подсети), ценой неразличимости VPN и остальных хостов INFRA. Приемлемо при малом числе клиентов.
- **Обфускация — клиент и сервер должны совпадать по `S1/S2/H1–H4`.** Junk-параметры (`Jc/Jmin/Jmax`) — только на клиенте.
