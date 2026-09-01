## MikroTik: Обход блокировок используя AmneziaWG


В связи с ужесточением блокировок приходится менять протокол VPN с **WireGuard** на **AmneziaWG**.

В качестве *роутера* использую **MikroTik**, который не хотелось бы менять, но очень хотелось бы использовать с ним VPN-туннель **AmneziaWG**.

Я буду использовать в качестве *VPN-шлюза* с **AmneziaWG** еще один **MikroTik** прошитый в **OpenWrt**. Эта инструкция подойдет для моделей:
- [RB951G-2HnD](https://mikrotik.wiki/wiki/MikroTik_RB951G-2HnD)
- [RB951Ui-2HnD](https://mikrotik.wiki/wiki/MikroTik_RB951Ui-2HnD)
- [RB952Ui-5ac2nD (hAP ac lite)](https://mikrotik.wiki/wiki/MikroTik_hAP_ac_lite_(RB952Ui-5ac2nD))
- [RB2011UiAS-2HnD-IN](https://mikrotik.wiki/wiki/MikroTik_RB2011UiAS-2HnD-IN)
- [RB2011UiAS-IN](https://mikrotik.wiki/wiki/MikroTik_RB2011UiAS-IN)
- [RB2011iL-IN](https://mikrotik.wiki/wiki/MikroTik_RB2011iL-IN)
- [RB2011iL-RM](https://mikrotik.wiki/wiki/MikroTik_RB2011iL-RM)
- [RBwAPG-5HacT2HnD (wAP ac)](https://mikrotik.wiki/wiki/WAP_ac_BE_(RBwAPG-5HacT2HnD-BE))
- [RBwAPG-5HacT2HnD-BE (wAP ac BE)](https://mikrotik.wiki/wiki/WAP_ac_BE_(RBwAPG-5HacT2HnD-BE))
- [RB912UAG-2HPnD-OUT (BaseBox 2)](https://mikrotik.wiki/wiki/MikroTik_BaseBox_2_(RB912UAG-2HPnD-OUT))
- [RB912UAG-5HPnD-OUT (BaseBox 5)](https://mikrotik.wiki/wiki/MikroTik_BaseBox_5_(RB912UAG-5HPnD-OUT))
- [RB911G-5HPacD-NB (NetBox 5)](https://mikrotik.wiki/wiki/MikroTik_NetBox_5_(RB911G-5HPacD-NB))

Итак, мой *роутер* (**RouterOS**) будет маршрутизировать нужный трафик (**Telegram**, **YouTube**, и т.д.) в *VPN-шлюз* (**OpenWrt**).

## Схема подключения к роутеру (RouterOS) VPN-шлюза (OpenWrt)

```
[---------- RouterOS --------]        [-------- OpenWrt -------]
[ bridge-wan:                ]        [                        ]
[ bridge-lan:    10.x.y.1/24 ]  <==>  [ eth1  :    10.x.y.z/24 ]
[ bridge-awg: 192.168.1.2/24 ]  <==>  [ br-lan: 192.168.1.1/24 ]
```

***

## Настройка VPN-шлюза (OpenWrt)


### 1. Подключаем устройство

1. Втыкаем **LAN**-кабель (на котором по DHCP раздается Интернет), в **WAN**-порт нашего устройства.
2. Подключаем **LAN**-порт нашего устройства к ПК. Дефолтные настройки подключения **OpenWrt**:
```ini
WAN IP: Dynamic
LAN IP: 192.168.1.1
  USER: root
  PASS:
```
3. Подключаемся к устройству по протоколу **SSH** через терминал [PuTTY](https://the.earth.li/~sgtatham/putty/latest/w32/putty.exe):
```powershell
   putty.exe 192.168.1.1 -l root
```
4. Следующие этапы подразумевают подключение к устройству через терминал и выполнение в нем указанных блоков кода.

### 2. Сбрасываем OpenWrt к дефолтным настройкам и перезагружаемся

```bash
### Сбрасываем OpenWrt к дефолтным настройкам и перезагружаемся
# System -> Backup / Flash Firmware -> Restore -> Reset to defaults -> Perform reset -> OK
firstboot -y && reboot
```

### 3. Подготавливаем дефолтную OpenWrt и перезагружаемся

```bash
### Устанавливаем английский язык интерфейса
# System -> System -> Language and Style -> Language: English
uci set luci.main.lang='en'

### Устанавливаем часовой пояс
# System -> System -> General Settings -> Timezone: Europe/Samara
uci set system.@system[0].timezone='<+04>-4'
uci set system.@system[0].zonename='Europe/Samara'

### Отключаем PoE-Out (чтобы порт не горел красным) - добавляем команды (перед exit 0) в скрипт автозапуска
# System -> Startup -> Local Startup:
# sleep 2; for f in /sys/class/gpio/*poe*/value; do echo 0 >$f; done
# exit 0
# -> Save -> Dismiss
grep -q 'gpio.*poe' /etc/rc.local || sed -i '/exit 0/i sleep 2; for f in /sys/class/gpio/*poe*/value; do echo 0 >$f; done' /etc/rc.local

### Разрешаем подключения на WAN-интерфейсе
# Network -> Firewall -> Zones -> at the intersection of wan and Input, select accept -> Save & Apply
uci set firewall.@zone[1].input='ACCEPT'

### Отключаем IPv6
# Удаляем IPv6-туннели и интерфейсы
# Network -> Interfaces -> удаляем WAN6, 6in4, 6to4
uci -q delete network.wan6
uci -q delete network.6in4
uci -q delete network.6to4
# Отключаем IPv6 на LAN-интерфейсе
# Network -> Interfaces -> LAN -> Advanced Settings
uci    set    network.lan.ipv6='off'
uci    set    network.lan.delegate='0'
uci -q delete network.lan.ip6assign
uci -q delete network.lan.ip6addr
uci -q delete network.lan.ip6prefix
# Отключаем IPv6 на WAN-интерфейсе
# Network -> Interfaces -> WAN -> Advanced Settings
uci    set    network.wan.ipv6='0'
uci    set    network.wan.delegate='0'
uci -q delete network.wan.ip6addr
uci -q delete network.wan.ip6prefix
# Отключаем IPv6 в DHCP
uci    set    dhcp.lan.dhcpv6='disabled'
uci    set    dhcp.lan.ndp='disabled'
uci    set    dhcp.lan.ra='disabled'
uci -q delete dhcp.wan.dhcpv6
uci -q delete dhcp.wan.ra
# Удаляем IPv6-правила в Firewall
for rule in $(uci show firewall|grep "family='ipv6'"|cut -d'.' -f2|cut -d'=' -f1|sort -t'[' -k2 -rn); do uci delete firewall.$rule; done
for rule in $(uci show firewall|grep 'ip6proto='|cut -d'.' -f2|cut -d'=' -f1); do uci delete firewall.$rule.ip6proto; done
for rule in $(uci show firewall|grep '@rule'|grep -v 'family='|cut -d'[' -f2|cut -d']' -f1); do uci set firewall.@rule[$rule].family='ipv4'; done
# Отключаем IPv6 в sysctl
grep -q 'net.ipv6.conf.all.disable_ipv6=1'     /etc/sysctl.conf || echo >>/etc/sysctl.conf 'net.ipv6.conf.all.disable_ipv6=1'
grep -q 'net.ipv6.conf.default.disable_ipv6=1' /etc/sysctl.conf || echo >>/etc/sysctl.conf 'net.ipv6.conf.default.disable_ipv6=1'
grep -q 'net.ipv6.conf.lo.disable_ipv6=1'      /etc/sysctl.conf || echo >>/etc/sysctl.conf 'net.ipv6.conf.lo.disable_ipv6=1'
grep -q 'net.ipv6.conf.br-lan.disable_ipv6=1'  /etc/sysctl.conf || echo >>/etc/sysctl.conf 'net.ipv6.conf.br-lan.disable_ipv6=1'
# Удаляем IPv6-пакеты (удаляем пакеты, и все зависящие от них)
opkg remove --force-removal-of-dependent-packages odhcp6c
opkg remove --force-removal-of-dependent-packages odhcpd-ipv6only
opkg remove --force-removal-of-dependent-packages kmod-ip6tables
opkg remove --force-removal-of-dependent-packages kmod-nf-nat6
opkg remove --force-removal-of-dependent-packages kmod-ipt-nat6
opkg remove --force-removal-of-dependent-packages ip6tables
opkg remove --force-removal-of-dependent-packages luci-proto-ipv6

### Удаляем ненужные пакеты (не обращая внимания на зависимости)
# Оставляем в Services только:
#  - Kernel Manager
#  - QoS over Nftables
#  - youtubeUnblock
#  - Bandwith Monitor
#  - Watchcat
#  - Network Shares
#  - Terminal
#  - UPnP IGD & PCP
opkg remove --force-depends adblock  luci-app-adblock
opkg remove --force-depends aria2    luci-app-aria2
opkg remove --force-depends ddns     luci-app-ddns
opkg remove --force-depends hd-idle  luci-app-hd-idle
opkg remove --force-depends minidlna luci-app-minidlna
opkg remove --force-depends netdata  luci-app-netdata
opkg remove --force-depends qos      luci-app-qos
opkg remove --force-depends smartdns luci-app-smartdns

### Отключаем ненужные сервисы в автозагрузке
# System -> Startup:
#
# Pr  Имя              Описание                                   Можно ли отключать
# --  ---------------  -----------------------------------------  -------------------------------------------------------------------------
# 19  wpad             WPA-Enterprise / 802.1X                    Да, если используется только WPA2-Personal
# 21  fa-fancontrol    Управление вентилятором                    Да, если роутер с пассивным охлаждением
# 30  radius           RADIUS-клиент                              Да, если не корпоративная сеть 802.1X
# 35  odhcpd           DHCPv6 и RA для IPv6                       Да, если IPv6 отключён (рекомендуется для youtubeUnblock)
# 50  cron             Планировщик задач                          Да, если не используете расписания
# 50  vsftpd           FTP-сервер                                 Да, если не используется FTP
# 61  avahi-daemon     mDNS/Bonjour (локальное обнаружение)       Да, если не используете AirPlay/Chromecast discovery
# 80  blockd           Авто-монтирование USB-накопителей          Да, если нет USB-дисков
# 80  collectd         Сбор метрик для мониторинга                Да, если не используете внешние системы мониторинга
# 94  miniupnpd        Автоматический проброс портов (UPnP)       Да, если не используете торренты или устройства, которые сами открывают порты (IP-камеры, NAS и т.п.)
# 97  watchcat         Автоматическая перезагрузка при зависании  Да, если не используется Watchcat
# 98  samba4           SMB-сервер для общего доступа к файлам     Да, если не расшариваете папки по сети
# 99  wsdd2            Web Service Discovery для SMB              Да, если не нужен в Windows-сети
# --  ---------------  -----------------------------------------  -------------------------------------------------------------------------
# 50  sqm              Smart Queue Management (QoS)               Да, если не используете торренты/видеозвонки на перегруженном канале
# 60  nlbwmon          Мониторинг трафика по хостам               Да, если не нужен Bandwith Monitor
# 79  luci_statistics  Статистика в веб-интерфейсе                Да, если не смотрите графики в LuCI
/etc/init.d/avahi-daemon   stop 2>/dev/null && /etc/init.d/avahi-daemon   disable
/etc/init.d/blockd         stop 2>/dev/null && /etc/init.d/blockd         disable
/etc/init.d/collectd       stop 2>/dev/null && /etc/init.d/collectd       disable
/etc/init.d/cron           stop 2>/dev/null && /etc/init.d/cron           disable
/etc/init.d/fa-fancontrol  stop 2>/dev/null && /etc/init.d/fa-fancontrol  disable
/etc/init.d/miniupnpd      stop 2>/dev/null && /etc/init.d/miniupnpd      disable
/etc/init.d/odhcpd         stop 2>/dev/null && /etc/init.d/odhcpd         disable
/etc/init.d/radius         stop 2>/dev/null && /etc/init.d/radius         disable
/etc/init.d/samba4         stop 2>/dev/null && /etc/init.d/samba4         disable
/etc/init.d/vsftpd         stop 2>/dev/null && /etc/init.d/vsftpd         disable
/etc/init.d/watchcat       stop 2>/dev/null && /etc/init.d/watchcat       disable
/etc/init.d/wpad           stop 2>/dev/null && /etc/init.d/wpad           disable
/etc/init.d/wsdd2          stop 2>/dev/null && /etc/init.d/wsdd2          disable
/etc/init.d/youtubeUnblock stop 2>/dev/null && /etc/init.d/youtubeUnblock disable

### Применяем все изменения
uci commit
sysctl -p -q

### Перезагружаемся
# System -> Reboot -> Perform reboot
reboot
```

### 4. Устанавливаем клиент AmneziaWG и перезагружаемся

```bash
(
  ### Обновляем списки пакетов
  # System -> Software -> Update lists..
  opkg update || { echo 'OPKG UPDATE ERROR'; exit 1; }

  ### Устанавливаем необходимые зависимости для AmneziaWG
  # System -> Software -> Download and install package: kmod-udptunnel4                  -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-udptunnel6                  -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-crypto-lib-chacha20poly1305 -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-crypto-lib-curve25519       -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-crypto-hash                 -> OK -> Install -> Dismiss
  # System -> Software -> Download and install package: kmod-crypto-aead                 -> OK -> Install -> Dismiss
  opkg install kmod-udptunnel4 kmod-udptunnel6 kmod-crypto-lib-chacha20poly1305 kmod-crypto-lib-curve25519 kmod-crypto-hash kmod-crypto-aead

  ### Скачиваем и устанавливаем модуль ядра, утилиты и LuCI-интерфейс AmneziaWG для OpenWrt 24.10
  # System -> Software -> Upload Package.. -> Browse.. -> kmod-amneziawg_v24.10.8_mips_24kc_ath79_mikrotik.ipk       -> Upload -> Install -> Dismiss
  # System -> Software -> Upload Package.. -> Browse.. -> amneziawg-tools_v24.10.8_mips_24kc_ath79_mikrotik.ipk      -> Upload -> Install -> Dismiss
  # System -> Software -> Upload Package.. -> Browse.. -> luci-proto-amneziawg_v24.10.8_mips_24kc_ath79_mikrotik.ipk -> Upload -> Install -> Dismiss
   VERSION='24.10.8'
  BASE_URL="https://github.com/Slava-Shchipunov/awg-openwrt/releases/download/v${VERSION}"
      ARCH='mips_24kc_ath79_mikrotik'
  pkg='kmod-amneziawg'
  url="${BASE_URL}/${pkg}_v${VERSION}_${ARCH}.ipk"
  wget -qO                     "/tmp/${pkg}.ipk" "${url}" || { echo "DOWNLOAD ERROR: ${url}"; exit 1; }
  opkg install --force-depends "/tmp/${pkg}.ipk"          || { echo "INSTALL ERROR: ${pkg}" ; exit 1; }
  rm -f                        "/tmp/${pkg}.ipk"
  pkg='amneziawg-tools'
  url="${BASE_URL}/${pkg}_v${VERSION}_${ARCH}.ipk"
  wget -qO                     "/tmp/${pkg}.ipk" "${url}" || { echo "DOWNLOAD ERROR: ${url}"; exit 1; }
  opkg install --force-depends "/tmp/${pkg}.ipk"          || { echo "INSTALL ERROR: ${pkg}" ; exit 1; }
  rm -f                        "/tmp/${pkg}.ipk"
  pkg='luci-proto-amneziawg'
  url="${BASE_URL}/${pkg}_v${VERSION}_${ARCH}.ipk"
  wget -qO                     "/tmp/${pkg}.ipk" "${url}" || { echo "DOWNLOAD ERROR: ${url}"; exit 1; }
  opkg install --force-depends "/tmp/${pkg}.ipk"          || { echo "INSTALL ERROR: ${pkg}" ; exit 1; }
  rm -f                        "/tmp/${pkg}.ipk"

  ### Перезагружаемся
  # System -> Reboot -> Perform reboot
  reboot
)
```

### 5. Настраиваем клиент AmneziaWG (через веб-интерфейс)

 - **Network** -> **Interfaces** -> **Add new interface..** -> Name: `awg0`, Protocol: `AmneziaWG VPN` -> **Create interface**
 - Import configuration: **Load configuration..**
 - Скачиваем конфиг AmneziaWG с сайта [WARP Генератор](https://warp-gen.github.io/) и вставляем содержимое в пустое поле
 - **Import settings** -> **OK**
 - **Firewall Settings** -> Create / Assign firewall-zone: `wan`
 - **Peers** -> **Edit** -> Persistent Keepalive: `25`
 - **Save** -> **Save** -> **Save & Apply**

### 6. Донастраиваем клиент AmneziaWG и маршрутизацию для него

```bash
(
  ### Добавляем интерфейс awg0 в зону WAN Firewall
  # Network -> Interfaces -> awg0 -> Edit -> Create / Assign firewall-zone: wan -> Save -> Save & Apply
  uci -q get firewall.@zone[1].network|grep -q 'awg0' || uci add_list firewall.@zone[1].network='awg0'

  ### Устанавливаем Persistent Keep Alive для поддержания NAT-сессии у провайдера, иначе handshake может не быть
  # Network -> Interfaces -> awg0 -> Edit -> Peers -> Edit -> Persistent Keep Alive: 25 -> Save -> Save -> Save & Apply
  uci set network.awg0.persistent_keepalive='25'
  uci -q get network.@amneziawg_awg0[0] >/dev/null && uci set network.@amneziawg_awg0[0].persistent_keepalive='25'

  ### Применяем изменения
  uci commit
  ifdown awg0; sleep 3; ifup awg0

  ### Проверяем, что AWG-туннель работает (ждем до 30 секунд)
  awg_is_alive() { awg show awg0 2>/dev/null|grep handshake|grep -qvi never; }
  for i in $(seq 1 30); do sleep 1; awg_is_alive && break; done
  awg_is_alive || { echo 'ERROR: awg0 - no handshake! Check VPN connection'; exit 1; }

  ### Настраиваем маршрутизацию
  # Трафик от клиентов (in='lan') в приватные сети отправляем в таблицу main (LAN/WAN-интерфейсы). За это отвечают:
  #   - правило с приоритетом 10: in='lan' -> 10.0.0.0/8     => таблица main
  #   - правило с приоритетом 10: in='lan' -> 172.16.0.0/12  => таблица main
  #   - правило с приоритетом 10: in='lan' -> 192.168.0.0/16 => таблица main
  #   - маршруты, динамически создаваемые LAN/WAN-интерфейсами в таблице main
  # Остальной трафик от клиентов (in='lan') - в Интернет, его отправляем в таблицу 100 (AWG-туннель). За это отвечают:
  #   - правило c приоритетом 20: in='lan' -> ANY => таблица 100
  #   - маршрут по умолчанию через интерфейс awg0 в таблице 100
  # Трафик от самого роутера (in='lo') идет через LAN/WAN-интерфейсы (нужно для обновления времени, DNS, и т.д.). За это отвечают:
  #   - маршруты, динамически создаваемые LAN/WAN-интерфейсами в таблице main
  # Примечания:
  #   in='' - входящий интерфейс       (UCI), iif='' - то же в ядре Linux (его показывает ip rule show)
  #   lan   - логическое имя LAN-моста (UCI), br-lan - то же в ядре Linux (его показывает ip rule show)

  ### Добавляем маршрут по умолчанию через интерфейс awg0 в таблице 100
  # Network -> Routing -> Add -> Interface: awg0, Route type: unicast, Target: 0.0.0.0/0 -> Advanced Settings -> Table: 100 -> Save -> Save & Apply
  # Перед добавлением маршрута: Удаление всех старых маршрутов через интерфейс awg0
  uci show network|grep 'route.*awg0'|cut -d. -f2|sort -Vr|while read r; do uci -q delete network.$r; done
  uci add network route >/dev/null
  uci set network.@route[-1].interface='awg0'
  uci set network.@route[-1].target='0.0.0.0/0'
  uci set network.@route[-1].table='100'

  ### Добавляем правила с приоритетом 10: Трафик от клиентов (in='lan') в приватные сети отправляем в таблицу main (LAN/WAN-интерфейсы)
  # Network -> Routing -> IPv4 Rules -> Add -> Priority: 10, Incoming interface: lan, Destination: 10.0.0.0/8     -> Save -> Save & Apply
  # Network -> Routing -> IPv4 Rules -> Add -> Priority: 10, Incoming interface: lan, Destination: 172.16.0.0/12  -> Save -> Save & Apply
  # Network -> Routing -> IPv4 Rules -> Add -> Priority: 10, Incoming interface: lan, Destination: 192.168.0.0/16 -> Save -> Save & Apply
  # Перед добавлением правил: Удаление всех старых правил для входящего интерфейса lan
  uci show network|grep "rule.*\.in='lan'"|cut -d. -f2|sort -Vr|while read r; do uci -q delete network.$r; done
  uci add network rule >/dev/null
  uci set network.@rule[-1].in='lan'
  uci set network.@rule[-1].dest='10.0.0.0/8'
  uci set network.@rule[-1].lookup='main'
  uci set network.@rule[-1].priority='10'
  uci add network rule >/dev/null
  uci set network.@rule[-1].in='lan'
  uci set network.@rule[-1].dest='172.16.0.0/12'
  uci set network.@rule[-1].lookup='main'
  uci set network.@rule[-1].priority='10'
  uci add network rule >/dev/null
  uci set network.@rule[-1].in='lan'
  uci set network.@rule[-1].dest='192.168.0.0/16'
  uci set network.@rule[-1].lookup='main'
  uci set network.@rule[-1].priority='10'

  ### Добавляем правило с приоритетом 20: Остальной трафик (в Интернет) от клиентов (in='lan') отправляем в таблицу 100 (AWG-туннель)
  # Network -> Routing -> IPv4 Rules -> Add -> Priority: 20, Incoming interface: lan -> Advanced Settings -> Table: 100 -> Save -> Save & Apply
  uci add network rule >/dev/null
  uci set network.@rule[-1].in='lan'
  uci set network.@rule[-1].lookup='100'
  uci set network.@rule[-1].priority='20'

  ### Применяем изменения
  uci commit
  /etc/init.d/network restart
  /etc/init.d/firewall restart
  sleep 15
)
```

### 7. Проверяем ключевые настройки

```bash
(
  check() { r="31m[-]"; eval "$2" &>/dev/null && r="32m[+]"; printf "\033[1;%s\033[0m %s\n" "$r" "$1"; }
  wan() { uci get network.wan.device || uci get network.wan.ifname || echo none; }
  check "  INTERNET: Интернет доступен (ping 8.8.8.8)"                              "ping -c 1 -W 5 8.8.8.8"
  check "   ROUTING: Пересылка между интерфейсами включена"                         "sysctl -n net.ipv4.ip_forward|grep 1"
  check " VPN / AWG: Интерфейс AWG добавлен в зону WAN"                             "uci get firewall.@zone[1].network|grep awg0"
  check " VPN / AWG: Параметр 'Persistent Keep Alive' включен"                      "uci get network.awg0.persistent_keepalive|grep '[1-9]'"
  check " VPN / AWG: Конфигурация импортирована (есть peer)"                        "uci show network|grep amneziawg_awg0"
  check " VPN / AWG: Соединение установлено (есть handshake)"                       "awg show awg0|grep handshake|grep -vi never"
  check " DEF ROUTE: Есть маршрут по умолчанию через WAN, и он только в main"       "ip route show table all|grep 'default .* dev `wan`\>'|grep -v table"
  check " DEF ROUTE: Есть маршрут по умолчанию через AWG, и он только в 100"        "ip route show table all|grep 'default dev awg0 table 100\>'"
  check "ROUTE RULE: Есть правило: трафик от клиентов в 10.0.0.0/8     => main"     "ip rule show|grep br-lan|grep '10.0.0.0/8.*main'"
  check "ROUTE RULE: Есть правило: трафик от клиентов в 172.16.0.0/12  => main"     "ip rule show|grep br-lan|grep '172.16.0.0/12.*main'"
  check "ROUTE RULE: Есть правило: трафик от клиентов в 192.168.0.0/16 => main"     "ip rule show|grep br-lan|grep '192.168.0.0/16.*main'"
  check "ROUTE RULE: Есть правило: трафик от клиентов в Интернет       => 100"      "ip rule show|grep br-lan|grep 'lookup 100\>'"
  check " ROUTE GET: Проверка маршрута: трафик от клиентов в 10.0.0.0/8     => LAN" "ip route get 10.0.0.1    from 192.168.1.50 iif br-lan|grep br-lan"
  check " ROUTE GET: Проверка маршрута: трафик от клиентов в 172.16.0.0/12  => LAN" "ip route get 172.16.0.1  from 192.168.1.50 iif br-lan|grep br-lan"
  check " ROUTE GET: Проверка маршрута: трафик от клиентов в 192.168.0.0/16 => LAN" "ip route get 192.168.0.1 from 192.168.1.50 iif br-lan|grep br-lan"
  check " ROUTE GET: Проверка маршрута: трафик от клиентов в Интернет       => AWG" "ip route get 8.8.8.8     from 192.168.1.50 iif br-lan|grep awg0"
  check "  NTP SYNC: Время синхронизировано с pool.ntp.org"                         "ntpd -n -q -p pool.ntp.org"
)
```

***

## Настройка роутера (RouterOS)


### 1. Создаем интерфейс bridge-awg и добавляем в него любой свободный порт

```bash
/interface bridge
add name=bridge-awg

/interface bridge port
add bridge=bridge-awg interface=ether4
```

### 2. Назначаем IP-адрес интерфейсу bridge-awg

```bash
/ip address
add address=192.168.1.2/24 interface=bridge-awg
```

### 3. Создаем списки частных сетей и сетей заблокированных сервисов

```bash
/ip firewall address-list
remove [find dynamic=no list=PRIVATE-LANS]
add address=10.0.0.0/8     list=PRIVATE-LANS
add address=172.16.0.0/12  list=PRIVATE-LANS
add address=192.168.0.0/16 list=PRIVATE-LANS

/ip firewall address-list
remove [find dynamic=no list=CLOUDFLARE]
:delay 5s
add address=1.0.0.0/24         list=CLOUDFLARE
add address=1.1.1.0/24         list=CLOUDFLARE
add address=3.164.206.0/24     list=CLOUDFLARE
add address=8.6.112.0/24       list=CLOUDFLARE
add address=8.47.69.0/24       list=CLOUDFLARE
add address=13.33.235.0/24     list=CLOUDFLARE
add address=18.165.140.0/24    list=CLOUDFLARE
add address=34.36.205.0/24     list=CLOUDFLARE
add address=35.190.80.0/24     list=CLOUDFLARE
add address=35.201.124.0/24    list=CLOUDFLARE
add address=63.140.62.0/24     list=CLOUDFLARE
add address=103.21.244.0/22    list=CLOUDFLARE
add address=103.22.200.0/22    list=CLOUDFLARE
add address=103.31.4.0/22      list=CLOUDFLARE
add address=104.16.0.0/13      list=CLOUDFLARE
add address=104.24.0.0/14      list=CLOUDFLARE
add address=108.162.192.0/18   list=CLOUDFLARE
add address=131.0.72.0/22      list=CLOUDFLARE
add address=141.101.64.0/18    list=CLOUDFLARE
add address=162.158.0.0/15     list=CLOUDFLARE
add address=172.64.0.0/13      list=CLOUDFLARE
add address=173.245.48.0/20    list=CLOUDFLARE
add address=188.114.96.0/20    list=CLOUDFLARE
add address=190.93.240.0/20    list=CLOUDFLARE
add address=197.234.240.0/22   list=CLOUDFLARE
add address=198.41.128.0/17    list=CLOUDFLARE

/ip firewall address-list
remove [find dynamic=no list=GOOGLEAI]
:delay 5s
add address=34.54.74.0/24      list=GOOGLEAI
add address=35.166.25.0/24     list=GOOGLEAI
add address=44.245.75.0/24     list=GOOGLEAI
add address=44.254.195.0/24    list=GOOGLEAI
add address=64.233.161.0/24    list=GOOGLEAI
add address=64.233.162.0/23    list=GOOGLEAI
add address=64.233.164.0/23    list=GOOGLEAI
add address=74.125.131.0/24    list=GOOGLEAI
add address=74.125.205.0/24    list=GOOGLEAI
add address=108.177.14.0/24    list=GOOGLEAI
add address=142.250.74.0/24    list=GOOGLEAI
add address=142.250.150.0/24   list=GOOGLEAI
add address=142.251.1.0/24     list=GOOGLEAI
add address=142.251.38.0/24    list=GOOGLEAI
add address=142.251.142.0/23   list=GOOGLEAI
add address=172.217.19.0/24    list=GOOGLEAI
add address=172.217.20.0/24    list=GOOGLEAI
add address=172.253.130.0/24   list=GOOGLEAI
add address=172.253.152.0/24   list=GOOGLEAI
add address=173.194.73.0/24    list=GOOGLEAI
add address=173.194.220.0/23   list=GOOGLEAI
add address=173.194.222.0/24   list=GOOGLEAI
add address=192.178.25.0/24    list=GOOGLEAI
add address=209.85.233.0/24    list=GOOGLEAI
add address=216.58.201.0/24    list=GOOGLEAI
add address=216.239.32.0/24    list=GOOGLEAI
add address=216.239.34.0/24    list=GOOGLEAI
add address=216.239.36.0/24    list=GOOGLEAI
add address=216.239.38.0/24    list=GOOGLEAI

/ip firewall address-list
remove [find dynamic=no list=GOOGLEPLAY]
:delay 5s
add address=34.104.35.0/24     list=GOOGLEPLAY
add address=46.61.154.0/24     list=GOOGLEPLAY
add address=64.233.161.0/24    list=GOOGLEPLAY
add address=64.233.162.0/23    list=GOOGLEPLAY
add address=64.233.164.0/23    list=GOOGLEPLAY
add address=74.125.4.0/24      list=GOOGLEPLAY
add address=74.125.8.0/24      list=GOOGLEPLAY
add address=74.125.11.0/24     list=GOOGLEPLAY
add address=74.125.13.0/24     list=GOOGLEPLAY
add address=74.125.97.0/24     list=GOOGLEPLAY
add address=74.125.99.0/24     list=GOOGLEPLAY
add address=74.125.100.0/24    list=GOOGLEPLAY
add address=74.125.104.0/23    list=GOOGLEPLAY
add address=74.125.110.0/23    list=GOOGLEPLAY
add address=74.125.131.0/24    list=GOOGLEPLAY
add address=74.125.153.0/24    list=GOOGLEPLAY
add address=74.125.154.0/24    list=GOOGLEPLAY
add address=74.125.160.0/24    list=GOOGLEPLAY
add address=74.125.162.0/23    list=GOOGLEPLAY
add address=74.125.168.0/24    list=GOOGLEPLAY
add address=74.125.173.0/24    list=GOOGLEPLAY
add address=74.125.175.0/24    list=GOOGLEPLAY
add address=74.125.205.0/24    list=GOOGLEPLAY
add address=87.245.216.0/24    list=GOOGLEPLAY
add address=87.245.220.0/24    list=GOOGLEPLAY
add address=87.245.222.0/24    list=GOOGLEPLAY
add address=108.177.14.0/24    list=GOOGLEPLAY
add address=142.250.74.0/24    list=GOOGLEPLAY
add address=142.250.150.0/24   list=GOOGLEPLAY
add address=142.251.1.0/24     list=GOOGLEPLAY
add address=142.251.38.0/24    list=GOOGLEPLAY
add address=142.251.142.0/23   list=GOOGLEPLAY
add address=172.217.19.0/24    list=GOOGLEPLAY
add address=172.217.20.0/24    list=GOOGLEPLAY
add address=172.217.132.0/23   list=GOOGLEPLAY
add address=172.253.130.0/24   list=GOOGLEPLAY
add address=172.253.152.0/24   list=GOOGLEPLAY
add address=173.194.1.0/24     list=GOOGLEPLAY
add address=173.194.3.0/24     list=GOOGLEPLAY
add address=173.194.5.0/24     list=GOOGLEPLAY
add address=173.194.6.0/24     list=GOOGLEPLAY
add address=173.194.10.0/24    list=GOOGLEPLAY
add address=173.194.16.0/24    list=GOOGLEPLAY
add address=173.194.18.0/23    list=GOOGLEPLAY
add address=173.194.73.0/24    list=GOOGLEPLAY
add address=173.194.153.0/24   list=GOOGLEPLAY
add address=173.194.182.0/23   list=GOOGLEPLAY
add address=173.194.187.0/24   list=GOOGLEPLAY
add address=173.194.188.0/24   list=GOOGLEPLAY
add address=173.194.220.0/23   list=GOOGLEPLAY
add address=173.194.222.0/24   list=GOOGLEPLAY
add address=192.178.25.0/24    list=GOOGLEPLAY
add address=209.85.226.0/24    list=GOOGLEPLAY
add address=209.85.233.0/24    list=GOOGLEPLAY
add address=213.59.237.0/24    list=GOOGLEPLAY
add address=216.58.201.0/24    list=GOOGLEPLAY
add address=216.239.32.0/24    list=GOOGLEPLAY
add address=216.239.34.0/24    list=GOOGLEPLAY
add address=216.239.36.0/24    list=GOOGLEPLAY
add address=216.239.38.0/24    list=GOOGLEPLAY

/ip firewall address-list
remove [find dynamic=no list=META]
:delay 5s
add address=3.33.221.0/24      list=META
add address=3.33.252.0/24      list=META
add address=15.197.206.0/24    list=META
add address=15.197.210.0/24    list=META
add address=31.13.24.0/21      list=META
add address=31.13.64.0/18      list=META
add address=34.192.181.0/24    list=META
add address=34.193.38.0/24     list=META
add address=34.194.71.0/24     list=META
add address=34.194.255.0/24    list=META
add address=45.64.40.0/22      list=META
add address=57.141.0.0/20      list=META
add address=57.144.0.0/14      list=META
add address=66.220.144.0/20    list=META
add address=74.119.76.0/22     list=META
add address=102.132.96.0/20    list=META
add address=103.4.96.0/22      list=META
add address=129.134.0.0/17     list=META
add address=157.240.0.0/16     list=META
add address=173.252.64.0/18    list=META
add address=179.60.192.0/22    list=META
add address=185.60.216.0/22    list=META
add address=185.89.216.0/22    list=META
add address=204.15.20.0/22     list=META

/ip firewall address-list
remove [find dynamic=no list=TELEGRAM]
:delay 5s
add address=91.105.192.0/23    list=TELEGRAM
add address=91.108.4.0/22      list=TELEGRAM
add address=91.108.8.0/21      list=TELEGRAM
add address=91.108.12.0/22     list=TELEGRAM
add address=91.108.16.0/21     list=TELEGRAM
add address=91.108.56.0/22     list=TELEGRAM
add address=149.154.160.0/20   list=TELEGRAM
add address=185.76.151.0/24    list=TELEGRAM

/ip firewall address-list
remove [find dynamic=no list=TORRENTS]
:delay 5s
add address=3.135.72.151       list=TORRENTS
add address=3.140.119.203      list=TORRENTS
add address=5.45.74.7          list=TORRENTS
add address=18.219.255.217     list=TORRENTS
add address=37.1.219.253       list=TORRENTS
add address=37.221.67.160      list=TORRENTS
add address=45.137.66.127      list=TORRENTS
add address=104.21.7.164       list=TORRENTS
add address=104.21.12.243      list=TORRENTS
add address=104.21.32.39       list=TORRENTS
add address=104.21.95.93       list=TORRENTS
add address=168.119.95.238     list=TORRENTS
add address=172.67.136.246     list=TORRENTS
add address=172.67.144.20      list=TORRENTS
add address=172.67.153.242     list=TORRENTS
add address=172.67.182.196     list=TORRENTS
add address=185.81.128.108     list=TORRENTS
add address=188.114.96.1       list=TORRENTS
add address=188.114.97.1       list=TORRENTS
add address=188.137.178.236    list=TORRENTS
add address=193.46.255.26      list=TORRENTS
add address=193.46.255.28/31   list=TORRENTS

/ip firewall address-list
remove [find dynamic=no list=TWITTER]
:delay 5s
add address=69.195.0.0/16      list=TWITTER
add address=104.16.0.0/12      list=TWITTER
add address=104.244.0.0/15     list=TWITTER
add address=107.167.27.0/24    list=TWITTER
add address=138.201.219.0/24   list=TWITTER
add address=146.75.0.0/16      list=TWITTER
add address=162.158.0.0/15     list=TWITTER
add address=172.64.0.0/13      list=TWITTER
add address=185.45.5.0/24      list=TWITTER
add address=185.45.6.0/23      list=TWITTER
add address=185.199.0.0/16     list=TWITTER
add address=188.114.88.0/21    list=TWITTER
add address=188.186.0.0/16     list=TWITTER
add address=192.133.0.0/16     list=TWITTER
add address=199.16.0.0/13      list=TWITTER
add address=199.59.0.0/16      list=TWITTER
add address=199.96.56.0/23     list=TWITTER
add address=199.232.0.0/16     list=TWITTER
add address=209.237.0.0/16     list=TWITTER

/ip firewall address-list
remove [find dynamic=no list=VIBER]
:delay 5s
add address=3.65.141.0/24      list=VIBER
add address=3.67.81.0/24       list=VIBER
add address=3.83.208.0/24      list=VIBER
add address=3.86.37.0/24       list=VIBER
add address=3.101.175.0/24     list=VIBER
add address=3.105.5.0/24       list=VIBER
add address=3.112.85.0/24      list=VIBER
add address=3.164.230.0/24     list=VIBER
add address=3.164.240.0/24     list=VIBER
add address=3.167.227.0/24     list=VIBER
add address=3.173.161.0/24     list=VIBER
add address=3.174.230.0/24     list=VIBER
add address=3.209.202.0/24     list=VIBER
add address=3.212.255.0/24     list=VIBER
add address=3.224.85.0/24      list=VIBER
add address=3.233.54.0/24      list=VIBER
add address=3.234.168.0/24     list=VIBER
add address=13.33.235.0/24     list=VIBER
add address=13.219.117.0/24    list=VIBER
add address=13.226.244.0/24    list=VIBER
add address=13.227.173.0/24    list=VIBER
add address=18.66.112.0/24     list=VIBER
add address=18.165.122.0/24    list=VIBER
add address=18.195.4.0/23      list=VIBER
add address=18.201.0.0/16      list=VIBER
add address=18.232.24.0/24     list=VIBER
add address=18.239.83.0/24     list=VIBER
add address=18.245.31.0/24     list=VIBER
add address=23.21.13.0/24      list=VIBER
add address=23.21.92.0/24      list=VIBER
add address=23.23.113.0/24     list=VIBER
add address=23.211.65.0/24     list=VIBER
add address=32.193.113.0/24    list=VIBER
add address=34.192.170.0/24    list=VIBER
add address=34.197.129.0/24    list=VIBER
add address=35.168.57.0/24     list=VIBER
add address=35.169.174.0/24    list=VIBER
add address=35.173.81.0/24     list=VIBER
add address=44.192.201.0/24    list=VIBER
add address=44.214.47.0/24     list=VIBER
add address=44.218.5.0/24      list=VIBER
add address=44.223.245.0/24    list=VIBER
add address=52.0.252.0/22      list=VIBER
add address=52.3.4.0/24        list=VIBER
add address=52.28.13.0/24      list=VIBER
add address=52.54.103.0/24     list=VIBER
add address=52.87.17.0/24      list=VIBER
add address=52.201.57.0/24     list=VIBER
add address=52.206.190.0/24    list=VIBER
add address=52.222.136.0/24    list=VIBER
add address=52.222.214.0/24    list=VIBER
add address=54.81.171.0/24     list=VIBER
add address=54.145.76.0/24     list=VIBER
add address=54.156.174.0/24    list=VIBER
add address=54.172.20.0/24     list=VIBER
add address=54.174.190.0/24    list=VIBER
add address=54.197.57.0/24     list=VIBER
add address=54.198.114.0/24    list=VIBER
add address=54.236.198.0/24    list=VIBER
add address=63.182.87.0/24     list=VIBER
add address=65.8.131.0/24      list=VIBER
add address=65.9.46.0/24       list=VIBER
add address=65.9.175.0/24      list=VIBER
add address=98.85.147.0/24     list=VIBER
add address=99.86.171.0/24     list=VIBER
add address=100.51.234.0/24    list=VIBER
add address=104.101.246.0/24   list=VIBER
add address=108.138.26.0/24    list=VIBER
add address=108.156.22.0/24    list=VIBER
add address=108.156.60.0/24    list=VIBER
add address=108.157.214.0/24   list=VIBER
add address=108.157.229.0/24   list=VIBER
add address=177.71.100.0/22    list=VIBER
add address=185.117.96.0/24    list=VIBER

/ip firewall address-list
remove [find dynamic=no list=YOUTUBE]
:delay 5s
add address=5.143.239.0/24     list=YOUTUBE
add address=8.8.4.0/24         list=YOUTUBE
add address=8.8.8.0/24         list=YOUTUBE
add address=8.34.208.0/20      list=YOUTUBE
add address=8.35.192.0/20      list=YOUTUBE
add address=23.236.48.0/20     list=YOUTUBE
add address=23.251.128.0/19    list=YOUTUBE
add address=34.0.0.0/9         list=YOUTUBE
add address=34.128.0.0/10      list=YOUTUBE
add address=35.184.0.0/13      list=YOUTUBE
add address=35.192.0.0/14      list=YOUTUBE
add address=35.196.0.0/15      list=YOUTUBE
add address=35.198.0.0/16      list=YOUTUBE
add address=35.199.0.0/17      list=YOUTUBE
add address=35.199.128.0/18    list=YOUTUBE
add address=35.200.0.0/13      list=YOUTUBE
add address=35.208.0.0/12      list=YOUTUBE
add address=46.61.136.0/24     list=YOUTUBE
add address=46.61.154.0/24     list=YOUTUBE
add address=46.61.170.0/24     list=YOUTUBE
add address=46.61.216.0/24     list=YOUTUBE
add address=64.18.0.0/15       list=YOUTUBE
add address=64.233.128.0/18    list=YOUTUBE
add address=66.102.0.0/20      list=YOUTUBE
add address=66.249.64.0/18     list=YOUTUBE
add address=70.32.128.0/19     list=YOUTUBE
add address=72.14.192.0/18     list=YOUTUBE
add address=74.114.24.0/21     list=YOUTUBE
add address=74.125.0.0/16      list=YOUTUBE
add address=80.252.155.0/24    list=YOUTUBE
add address=83.174.196.0/24    list=YOUTUBE
add address=83.174.199.0/24    list=YOUTUBE
add address=85.234.4.0/24      list=YOUTUBE
add address=87.245.216.0/24    list=YOUTUBE
add address=87.245.220.0/23    list=YOUTUBE
add address=87.245.222.0/24    list=YOUTUBE
add address=95.167.73.0/24     list=YOUTUBE
add address=104.21.47.0/24     list=YOUTUBE
add address=104.132.0.0/14     list=YOUTUBE
add address=104.152.0.0/14     list=YOUTUBE
add address=104.156.64.0/18    list=YOUTUBE
add address=104.196.0.0/14     list=YOUTUBE
add address=104.237.160.0/19   list=YOUTUBE
add address=107.167.160.0/19   list=YOUTUBE
add address=108.59.80.0/20     list=YOUTUBE
add address=108.170.192.0/18   list=YOUTUBE
add address=108.177.14.0/24    list=YOUTUBE
add address=109.195.25.0/24    list=YOUTUBE
add address=130.211.0.0/16     list=YOUTUBE
add address=136.112.0.0/12     list=YOUTUBE
add address=142.248.0.0/14     list=YOUTUBE
add address=146.148.0.0/14     list=YOUTUBE
add address=162.216.144.0/21   list=YOUTUBE
add address=162.222.176.0/21   list=YOUTUBE
add address=172.67.146.0/24    list=YOUTUBE
add address=172.110.32.0/21    list=YOUTUBE
add address=172.217.0.0/16     list=YOUTUBE
add address=172.253.0.0/16     list=YOUTUBE
add address=173.194.0.0/16     list=YOUTUBE
add address=173.255.112.0/20   list=YOUTUBE
add address=178.66.83.0/24     list=YOUTUBE
add address=185.38.0.0/24      list=YOUTUBE
add address=188.43.61.0/24     list=YOUTUBE
add address=188.43.87.0/24     list=YOUTUBE
add address=188.114.96.0/23    list=YOUTUBE
add address=192.158.28.0/22    list=YOUTUBE
add address=192.178.0.0/15     list=YOUTUBE
add address=193.186.4.0/24     list=YOUTUBE
add address=195.68.132.0/24    list=YOUTUBE
add address=199.36.154.0/23    list=YOUTUBE
add address=199.36.156.0/24    list=YOUTUBE
add address=199.192.112.0/21   list=YOUTUBE
add address=199.223.232.0/21   list=YOUTUBE
add address=207.126.144.0/20   list=YOUTUBE
add address=207.223.160.0/20   list=YOUTUBE
add address=208.65.152.0/22    list=YOUTUBE
add address=208.68.108.0/22    list=YOUTUBE
add address=208.81.188.0/22    list=YOUTUBE
add address=208.117.224.0/19   list=YOUTUBE
add address=209.85.128.0/17    list=YOUTUBE
add address=212.188.32.0/21    list=YOUTUBE
add address=213.59.210.0/24    list=YOUTUBE
add address=213.59.237.0/24    list=YOUTUBE
add address=216.58.192.0/19    list=YOUTUBE
add address=216.239.32.0/19    list=YOUTUBE
add address=217.118.183.0/24   list=YOUTUBE
```

### 4. Создаем правила для доменов заблокированных сервисов

  Эти правила будут направлять нужные нам запросы вместе с поддоменами на DNS **77.88.8.88**, а полученные ответы автоматически заносить в соответствующие адрес-листы, где они будут жить до протухания кэша DNS.

  Вместо DNS от Яндекса (**77.88.8.88**) можно использовать любой симпатичный вам DNS, например от Cloudflare: **1.1.1.1**.

```bash
/ip dns static
remove [find address-list=CLOUDFLARE]
add address-list=CLOUDFLARE forward-to=77.88.8.88 match-subdomain=yes name=cdnjs.com            type=FWD
add address-list=CLOUDFLARE forward-to=77.88.8.88 match-subdomain=yes name=cloudflare.com       type=FWD
add address-list=CLOUDFLARE forward-to=77.88.8.88 match-subdomain=yes name=cloudflarestatus.com type=FWD
add address-list=CLOUDFLARE forward-to=77.88.8.88 match-subdomain=yes name=one.one.one.one      type=FWD

/ip dns static
remove [find address-list=GOOGLEAI]
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=ai.google.dev                      type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=aisandbox-pa.googleapis.com        type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=aistudio.google.com                type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=bard.google.com                    type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=clients6.google.com                type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=deepmind.com                       type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=deepmind.google                    type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=geller-pa.googleapis.com           type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=gemini.google                      type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=gemini.google.com                  type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=generativeai.google                type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=generativelanguage.googleapis.com  type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=jules.google                       type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=jules.google.com                   type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=labs.google                        type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=makersuite.google.com              type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=notebooklm.google                  type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=notebooklm.google.com              type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=proactivebackend-pa.googleapis.com type=FWD
add address-list=GOOGLEAI forward-to=77.88.8.88 match-subdomain=yes name=stitch.withgoogle.com              type=FWD

/ip dns static
remove [find address-list=GOOGLEPLAY]
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=android.clients.google.com                        type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=beacons.gvt2.com                                  type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=connectivitycheck.gstatic.com                     type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=googleplay.com                                    type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=gvt1.com                                          type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=lh3.googleusercontent.com                         type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=play-fe.googleapis.com                            type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=play-games.googleusercontent.com                  type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=play.google.com                                   type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=play.googleapis.com                               type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=play-lh.googleusercontent.com                     type=FWD
add address-list=GOOGLEPLAY forward-to=77.88.8.88 match-subdomain=yes name=prod-lt-playstoregatewayadapter-pa.googleapis.com type=FWD

/ip dns static
remove [find address-list=META]
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=cdninstagram.com type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=facebook.com     type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=facebook.net     type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=fb.com           type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=fbcdn.net        type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=fbsbx.com        type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=ig.me            type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=instagram.com    type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=internalfb.com   type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=meta.com         type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=oculus.com       type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=threads.net      type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=wa.me            type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=whatsapp.biz     type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=whatsapp.com     type=FWD
add address-list=META forward-to=77.88.8.88 match-subdomain=yes name=whatsapp.net     type=FWD

/ip dns static
remove [find address-list=TELEGRAM]
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=cdn-telegram.org type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=comments.app     type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=contest.com      type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=fragment.com     type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=graph.org        type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=quiz.directory   type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=t.me             type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=tdesktop.com     type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=telega.one       type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=telegra.ph       type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=telegram-cdn.org type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=telegram.dog     type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=telegram.me      type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=telegram.org     type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=telegram.space   type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=telesco.pe       type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=tg.dev           type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=tx.me            type=FWD
add address-list=TELEGRAM forward-to=77.88.8.88 match-subdomain=yes name=usercontent.dev  type=FWD

/ip dns static
remove [find address-list=TORRENTS]
add address-list=TORRENTS forward-to=77.88.8.88 match-subdomain=yes name=anybt.eth.limo  type=FWD
add address-list=TORRENTS forward-to=77.88.8.88 match-subdomain=yes name=cdnbase.com     type=FWD
add address-list=TORRENTS forward-to=77.88.8.88 match-subdomain=yes name=fast-torrent.ru type=FWD
add address-list=TORRENTS forward-to=77.88.8.88 match-subdomain=yes name=nnmclub.to      type=FWD
add address-list=TORRENTS forward-to=77.88.8.88 match-subdomain=yes name=rutor.info      type=FWD

/ip dns static
remove [find address-list=TWITTER]
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=ads-twitter.com         type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=cms-twdigitalassets.com type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=periscope.tv            type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=pscp.tv                 type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=t.co                    type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=tellapart.com           type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=tweetdeck.com           type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twimg.com               type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twitpic.com             type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twitter.biz             type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twitter.com             type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twitter.jp              type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twittercommunity.com    type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twitterflightschool.com type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twitterinc.com          type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twitteroauth.com        type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twitterstat.us          type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twtrdns.net             type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twttr.com               type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twttr.net               type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=twvid.com               type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=vine.co                 type=FWD
add address-list=TWITTER forward-to=77.88.8.88 match-subdomain=yes name=x.com                   type=FWD

/ip dns static
remove [find address-list=VIBER]
add address-list=VIBER forward-to=77.88.8.88 match-subdomain=yes name=vb.me         type=FWD
add address-list=VIBER forward-to=77.88.8.88 match-subdomain=yes name=viber-bot.com type=FWD
add address-list=VIBER forward-to=77.88.8.88 match-subdomain=yes name=viber-dev.com type=FWD
add address-list=VIBER forward-to=77.88.8.88 match-subdomain=yes name=viber.com     type=FWD
add address-list=VIBER forward-to=77.88.8.88 match-subdomain=yes name=viber.net     type=FWD
add address-list=VIBER forward-to=77.88.8.88 match-subdomain=yes name=viberapi.com  type=FWD
add address-list=VIBER forward-to=77.88.8.88 match-subdomain=yes name=viberdns.com  type=FWD

/ip dns static
remove [find address-list=YOUTUBE]
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=ggpht.com                            type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=googlevideo.com                      type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=jnn-pa.googleapis.com                type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=nhacmp3youtube.com                   type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=returnyoutubedislikeapi.com          type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=wide-youtube.l.google.com            type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=youtu.be                             type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=youtube.com                          type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=youtube-nocookie.com                 type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=youtube-ui.l.google.com              type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=youtube.googleapis.com               type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=youtubeembeddedplayer.googleapis.com type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=youtubei.googleapis.com              type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=youtubekids.com                      type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=yt-video-upload.l.google.com         type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=yt.be                                type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=yt3.googleusercontent.com            type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=ytimg.com                            type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=ytimg.l.google.com                   type=FWD
add address-list=YOUTUBE forward-to=77.88.8.88 match-subdomain=yes name=yting.com                            type=FWD
```

### 5. Создаем таблицу маршрутизации для VPN

```bash
/routing table
add fib name=bypass-vpn
```

### 6. Создаем правила, маркирующие трафик к заблокированным сервисам

  Первое правило не позволит локальным адресам улететь в туннель.

```bash
/ip firewall mangle
add action=accept          chain=prerouting comment="ACCEPT ALL FROM PRIVATE-LANS TO PRIVATE-LANS"                            dst-address-list=PRIVATE-LANS src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO CLOUDFLARE AS BYPASS-VPN-CONN" connection-mark=no-mark connection-state=new dst-address-list=CLOUDFLARE new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO GOOGLEAI AS BYPASS-VPN-CONN"   connection-mark=no-mark connection-state=new dst-address-list=GOOGLEAI   new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO GOOGLEPLAY AS BYPASS-VPN-CONN" connection-mark=no-mark connection-state=new dst-address-list=GOOGLEPLAY new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO META AS BYPASS-VPN-CONN"       connection-mark=no-mark connection-state=new dst-address-list=META       new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO TELEGRAM AS BYPASS-VPN-CONN"   connection-mark=no-mark connection-state=new dst-address-list=TELEGRAM   new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO TORRENTS AS BYPASS-VPN-CONN"   connection-mark=no-mark connection-state=new dst-address-list=TORRENTS   new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO TWITTER AS BYPASS-VPN-CONN"    connection-mark=no-mark connection-state=new dst-address-list=TWITTER    new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO VIBER AS BYPASS-VPN-CONN"      connection-mark=no-mark connection-state=new dst-address-list=VIBER      new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-connection chain=prerouting comment="MARK CONNECTIONS ALL FROM PRIVATE-LANS TO YOUTUBE AS BYPASS-VPN-CONN"    connection-mark=no-mark connection-state=new dst-address-list=YOUTUBE    new-connection-mark=bypass-vpn-conn passthrough=yes src-address-list=PRIVATE-LANS
add action=mark-routing    chain=prerouting comment="MARK ROUTING ALL FROM PRIVATE-LANS MARKED VPN-CONN AS BYPASS-VPN"        connection-mark=bypass-vpn-conn                                             new-routing-mark=bypass-vpn      passthrough=no  src-address-list=PRIVATE-LANS
```

### 7. Создаем правила, которые согласуют MTU с VPN-интерфейсом

```bash
/ip firewall mangle
add action=change-mss chain=forward comment="CLAMP MSS FOR TCP SYN TO BYPASS-VPN"   new-mss=clamp-to-pmtu out-interface=bridge-awg passthrough=yes protocol=tcp tcp-flags=syn
add action=change-mss chain=forward comment="CLAMP MSS FOR TCP SYN FROM BYPASS-VPN" new-mss=clamp-to-pmtu  in-interface=bridge-awg passthrough=yes protocol=tcp tcp-flags=syn
```

### 8. Создаем правила маскарадинга для VPN-трафика

```bash
/ip firewall nat
add action=accept     chain=srcnat comment="ACCEPT ALL FROM PRIVATE-LANS TO PRIVATE-LANS FOR VPN" dst-address-list=PRIVATE-LANS src-address-list=PRIVATE-LANS
add action=masquerade chain=srcnat comment="MASQ ALL FROM PRIVATE-LANS MARKED AS BYPASS-VPN --> BYPASS-VPN-IF" out-interface=bridge-awg routing-mark=bypass-vpn src-address-list=PRIVATE-LANS
```

### 9. Настраиваем маршрутизацию VPN-трафика через интерфейс bridge-awg

```bash
/ip route
add dst-address=0.0.0.0/0 gateway=192.168.1.1 routing-table=bypass-vpn
```

### 10. Проверяем маршрутизацию с клиентского ПК

  Трассировка `ya.ru` после нашего роутера **НЕ ДОЛЖНА** проходить через хоп из сетей `192.168.1.0/24` и `10.8.1.0/24`:
```powershell
C:\>tracert ya.ru

Трассировка маршрута к ya.ru [77.88.55.242]
с максимальным числом прыжков 30:

  1     1 ms     1 ms     1 ms  10.x.y.1
  2     7 ms     6 ms     6 ms  178.45.160.1
```

  А трассировка `googlevideo.com` после нашего роутера **ДОЛЖНА** проходить через хопы из сетей `192.168.1.0/24` и `10.8.1.0/24`:
```powershell
C:\>tracert googlevideo.com

Трассировка маршрута к googlevideo.com [172.217.19.228]
с максимальным числом прыжков 30:

  1     1 ms     1 ms     1 ms  10.x.y.1
  2     1 ms     1 ms     1 ms  192.168.1.1
  3    74 ms    74 ms    74 ms  10.8.1.0
```

  Если все так - поздравляю, вы сломали чебурнет!

***

## Ссылки

 - [Смотрим рекламу на Youtube в 4K](https://telegra.ph/Smotrim-reklamu-na-Youtube-v-4K-08-12)
 - [Настройка клиента Wireguard на Mikrotik RouterOS для подключения к VPS, VDS серверу или готовой конфигурации](https://kiberlis.ru/mikrotik-wireguard-client)
 - [Wireguard в Mikrotik](https://www.youtube.com/live/eRcrZkwd5IM)
 - [Как определить оптимальный размер MTU?](https://help.keenetic.com/hc/ru/articles/214470885-%D0%9A%D0%B0%D0%BA-%D0%BE%D0%BF%D1%80%D0%B5%D0%B4%D0%B5%D0%BB%D0%B8%D1%82%D1%8C-%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9-%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80-MTU)
 - [YouTube Blocked hosts in Russia](https://gist.github.com/MashinaMashina/58eed9128ade2c8ff50a84d523ae97d1#file-youtube-blocked-hosts-in-russia-md)
 - [Диапазон IP-адресов Instagram, Netflix, ChatGPT, Youtube, Twitter](https://rockblack.su/vpn/dopolnitelno/diapazon-ip-adresov)
