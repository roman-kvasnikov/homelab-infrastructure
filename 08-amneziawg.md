---
name: amneziawg
description: |
  AmneziaWG — обфусцированный VPN-сервер для удалённого доступа в домашнюю сеть. LXC в INFRA, вход по белому IP через порт-форвард на OPNsense, минуя VPS. Документ описывает конфигурацию сервера, обфускацию, клиентов, маршрутизацию VPN-подсети, политику доступа по пирам, nftables, автозапуск и связку с OPNsense. Используй для вопросов по удалённому доступу к дому, VPN, AmneziaWG.
---

# AmneziaWG

AmneziaWG обеспечивает удалённый доступ в домашнюю сеть извне. Протокол — обфусцированный WireGuard (маскировка трафика под случайный поток для обхода DPI). Подключившись, клиент получает доступ к внутренним сегментам в объёме, определённом правилами OPNsense для его адреса в туннеле.

AmneziaWG развёрнут непривилегированным LXC в сегменте INFRA, `192.168.20.11`. Слушает UDP `51820`. Вход снаружи — по белому IP дома через порт-форвард на OPNsense, напрямую, без участия VPS. VPN-подсеть клиентов — `10.8.0.0/24`, адрес сервера в туннеле — `10.8.0.1`.

Реализация — userspace (`amneziawg-go`): kernel-модуля AmneziaWG в LXC нет, поэтому интерфейс поднимается через userspace-бэкенд. Это штатно для контейнера и не требует модулей на хосте.

## 1. Поток трафика

```
[Клиент вне дома] --AWG / UDP 51820--> [белый IP дома]
      --> [OPNsense: порт-форвард 51820 -> 192.168.20.11]
            --> [AmneziaWG LXC awg0 10.8.0.1]
                  --> [OPNsense: маршрут 10.8.0.0/24 через AMNEZIA_GW]
                        --> [внутренние сегменты]
```

Клиент подключается на белый IP дома, порт 51820. OPNsense пробрасывает этот UDP-порт на AmneziaWG-LXC в INFRA. LXC принимает туннель и форвардит трафик клиента в сеть с сохранением оригинального адреса `10.8.0.<N>`. Обратный путь обеспечен статическим маршрутом на OPNsense: `10.8.0.0/24` через шлюз `192.168.20.11`.

## 2. Конфигурация сервера

Конфиг — `/etc/amnezia/amneziawg/awg0.conf`.

**Interface.** `Address = 10.8.0.1/24`, `ListenPort = 51820`, приватный ключ сервера. Параметры обфускации заголовков: `S1`, `S2`, `H1–H4` — задают структуру пакетов и должны совпадать с клиентскими. Junk-параметры (`Jc/Jmin/Jmax`) на сервере отсутствуют — это клиентские настройки (мусорные пакеты перед рукопожатием генерирует инициатор соединения), серверу они не нужны.

**Peers.** Каждый клиент — отдельная секция `[Peer]` с публичным ключом клиента, симметричным preshared-ключом и `AllowedIPs = 10.8.0.<N>/32` (уникальный адрес клиента в VPN-подсети).

Запуск требует userspace-переменной: интерфейс поднимается с `AWG_QUICK_USERSPACE_IMPLEMENTATION=amneziawg-go`, иначе `awg-quick` пытается создать kernel-интерфейс, которого в LXC нет.

## 3. Клиенты

Клиентский конфиг зеркалит обфускацию сервера и добавляет junk-параметры.

```
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

Генерация ключей на сервере: `awg genkey | tee client.key | awg pubkey > client.pub`, PSK — `awg genpsk`. Новый `[Peer]` добавляется в `awg0.conf` с уникальным `AllowedIPs = 10.8.0.<N>/32` и применяется на живом интерфейсе без разрыва существующих клиентов: `awg syncconf awg0 <(awg-quick strip awg0)`. Политика доступа для нового пира задаётся правилами интерфейса INFRA на OPNsense — nftables сервисов править не требуется, они оперируют VPN-подсетью целиком.

## 4. Маршрутизация и модель адресации

VPN-подсеть **маршрутизируется**, а не маскарадится: выходя из туннеля, пакеты сохраняют оригинальный source-адрес `10.8.0.<N>`. Обратный путь обеспечен статическим маршрутом на OPNsense (`10.8.0.0/24` через `AMNEZIA_GW`), поэтому подмена адреса не нужна.

Это даёт различимость пиров на уровне firewall. Адрес в туннеле — криптографически подтверждённая идентичность: `AllowedIPs` в секции пира означает, что пакет с этим source-адресом принимается только от владельца соответствующего приватного ключа, подделка невозможна. Внешний IP клиента при этом меняется произвольно и в правилах не участвует.

Текущие пиры:

| Адрес | Устройство | Доступ |
| --- | --- | --- |
| `10.8.0.2` | Ноутбук | Полный — все сегменты, включая This Firewall (удалённое администрирование в объёме MGMT) |
| `10.8.0.3` | Телефон | Только Traefik `192.168.40.11:443` |

Разделение осмысленно: ноутбук — рабочий инструмент администрирования, телефон нужен лишь для доступа к опубликованным сервисам. Компрометация телефона (устройство на чужих сетях, больше стороннего софта) не даёт доступа к инфраструктуре.

**Следствие для allow-листов сервисов.** VPN-трафик приходит к сервисам с адресами `10.8.0.<N>`, поэтому в nftables LXC и VM разрешается VPN-подсеть целиком (`10.8.0.0/24`), наравне с MGMT. То же в middleware `trusted` на Traefik и в whitelist CrowdSec. Адресная политика по пирам живёт только на OPNsense: nftables отвечает на вопрос «из каких зон меня можно достичь», OPNsense — «кто куда имеет право ходить». Такое разделение сохраняет nftables-шаблон единым для всех хостов и локализует изменения при добавлении пира.

## 5. nftables

Полный `/etc/nftables.conf`. Whitelist с `policy drop` на input, транзит VPN — в forward. NAT-таблицы нет: VPN-подсеть маршрутизируется. Автозагрузка через `nftables.service` (enabled).

```
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
```

`udp dport 51820 accept` без ограничения источника — VPN-подключения приходят с меняющихся внешних адресов. SSH — из MGMT и VPN по общему канону (см. `02-conventions.md`). Forward пропускает транзит клиентов в обе стороны: `iifname "awg0"` — из туннеля в сеть, `oifname "awg0"` — обратно; при маршрутизируемой подсети ответный трафик приходит на LXC как транзитный и без второго правила был бы съеден `policy drop`. IP forwarding включён (`net.ipv4.ip_forward=1`), без него LXC не маршрутизирует трафик клиентов.

## 6. OPNsense

**Порт-форвард.** Firewall → NAT → Destination NAT (Port Forward). Interface WAN, protocol UDP, destination — WAN address port `51820`, redirect target `192.168.20.11:51820`, associated filter rule. Это направляет входящий VPN-трафик с белого IP на AmneziaWG-LXC в INFRA. VPS в этом пути не участвует — доступ прямой по IP дома.

**Шлюз и маршрут.** System → Gateways → Configuration: `AMNEZIA_GW`, interface INFRA, IPv4, адрес `192.168.20.11`, мониторинг отключён (шлюз — контейнер, лишняя зависимость от ICMP-проверки не нужна: при неответе OPNsense пометил бы шлюз как down и снял маршрут). System → Routes → Configuration: сеть `10.8.0.0/24`, шлюз `AMNEZIA_GW`. Без маршрута обратный трафик к клиентам не находит пути.

**Правила интерфейса INFRA.** Политика по пирам задаётся здесь. Anti-lockout в INFRA отсутствует осознанно: сегмент одной ногой торчит в интернет (открытый 51820, исходящие соединения Xray), доступ к веб-морде фаервола хостам INFRA не нужен. Удалённое администрирование OPNsense обеспечивается правилом для ноутбука, где `any` включает This Firewall.

| # | Правило | Source | Destination | Порт |
| --- | --- | --- | --- | --- |
| 1 | Allow VPN notebook full access | `vpn_notebook` (`10.8.0.2`) | any | any |
| 2 | Allow VPN phone to Traefik | `vpn_phone` (`10.8.0.3`) | `traefik_lxc` (`192.168.40.11`) | 443 |
| 3 | Allow INFRA to NTP on firewall | INFRA net | This Firewall | 123/udp |
| 4 | Block access to private networks | INFRA net | `private_networks` | any |
| 5 | Allow INFRA to internet | INFRA net | any | any |

Правила 1–2 адресуют VPN-пиров, которые не входят в `INFRA net` и потому не попадают под блок. Правила 3–5 описывают собственный исходящий трафик хостов INFRA (`amnezia`, `xray`): им нужен интернет и время, в локальную сеть они не ходят. Входящие подключения к Xray на `10809` рулятся правилами интерфейсов-источников (SERVICES, DMZ, IOT), а не INFRA.

## 7. Автозапуск

Интерфейс `awg0` поднимается при старте LXC через шаблонный юнит `awg-quick@awg0.service` с drop-in, задающим userspace-бэкенд:

```
# /etc/systemd/system/awg-quick@awg0.service.d/override.conf
[Service]
Environment=AWG_QUICK_USERSPACE_IMPLEMENTATION=amneziawg-go
```

Юнит включён в автозапуск (`systemctl enable awg-quick@awg0`). nftables (`nftables.service`) и sysctl (`ip_forward`) тоже персистентны, поэтому после перезагрузки LXC весь VPN-стек поднимается сам, без ручных действий.

## 8. SSH

SSH ужесточён общим drop-in `/etc/ssh/sshd_config.d/10-hardening.conf` (см. `02-conventions.md`). Доступ к порту 22 ограничен на nftables источниками MGMT и VPN. Аутентификация — по ключам.

## 9. Зависимости

- **OPNsense** — порт-форвард UDP 51820 на LXC (вход снаружи), шлюз и статический маршрут `10.8.0.0/24` (обратный путь), правила интерфейса INFRA (политика доступа по пирам), маршрутизация VPN-трафика между сегментами, Unbound как DNS для клиентов.
- **Белый IP дома** — endpoint для клиентов; VPN завязан на прямую доступность WAN-адреса дома.

## 10. Ключевые особенности

- **Прямой вход, минуя VPS.** AmneziaWG доступен по белому IP дома через порт-форвард на OPNsense. VPS используется только для публикации сервисов (HTTP через WG-туннель к Traefik), к VPN-доступу отношения не имеет.
- **Userspace-реализация в LXC.** Kernel-модуля AmneziaWG в контейнере нет — интерфейс работает через `amneziawg-go`. Требует userspace-переменной при подъёме (учтено в systemd drop-in).
- **Маршрутизация вместо маскарада.** Клиенты сохраняют адрес `10.8.0.<N>`, что делает пиров различимыми на firewall и даёт Traefik реальные адреса вместо адреса LXC. Цена — статический маршрут на OPNsense и VPN-подсеть в allow-листах сервисов.
- **Адрес в туннеле — криптографическая идентичность.** `AllowedIPs` привязывает source-адрес к приватному ключу пира; подделка невозможна, внешний IP клиента роли не играет. Это и позволяет строить политику доступа по адресам VPN-подсети.
- **Разделение слоёв фильтрации.** nftables сервисов разрешают VPN-подсеть целиком (зона), адресная политика по пирам — только на OPNsense (маршрутизация). Добавление пира не требует правки хостов.
- **Обфускация — клиент и сервер должны совпадать по `S1/S2/H1–H4`.** Junk-параметры (`Jc/Jmin/Jmax`) — только на клиенте.
