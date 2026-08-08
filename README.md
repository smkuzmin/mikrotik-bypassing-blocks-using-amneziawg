## MikroTik: Обход блокировок используя AmneziaWG

В связи с ужесточением блокировок приходится менять протокол VPN с **WireGuard** на **AmneziaWG**.

В качестве роутера использую **MikroTik**, который не хотелось бы менять, но очень хотелось бы использовать туннель **AmneziaWG** на нём.

Есть ещё один **MikroTik** (**hAP ac lite**), перепрошитый на **OpenWrt**. Туннель с **AmneziaWG** запустим на нём и используем в качестве VPN-шлюза. А роутер уже будет маршрутизировать нужный трафик в VPN-шлюз на **OpenWrt**.

### Схема подключения VPN-шлюза (OpenWrt) к роутеру (RouterOS)

```
[---------- RouterOS --------]        [-------- OpenWrt -------]
[ bridge-wan:                ]        [                        ]
[ bridge-lan:    10.x.y.1/24 ]  <==>  [ eth1  :    10.x.y.z/24 ]
[ bridge-awg: 192.168.1.2/24 ]  <==>  [ br-lan: 192.168.1.1/24 ]
```

***

## Настройка VPN-шлюза (OpenWrt)

1. **Подготавливаем дефолтную OpenWrt 24.10.4**

```bash
### Разрешаем подключения на WAN-интерфейсе:
### Network -> Firewall -> Zones -> на пересечении wan и Input выбираем accept и жмем Save & Apply.
uci set firewall.@zone[1].input='ACCEPT'

### Отключаем IPv6: Отключаем DHCPv6-клиент на WAN
uci delete dhcp.wan.dhcpv6 2>/dev/null
uci delete dhcp.wan.ra     2>/dev/null
uci set network.wan.ipv6='0'
uci set network.wan.delegate='0'

### Отключаем IPv6: Отключаем DHCPv6 и RA на LAN
uci set dhcp.lan.dhcpv6='disabled'
uci set dhcp.lan.ra='disabled'
uci set dhcp.lan.ndp='disabled'
uci set network.lan.delegate='0'

### Отключаем IPv6: Удаляем IPv6-адреса с WAN
uci delete network.wan.ip6addr   2>/dev/null
uci delete network.wan.ip6prefix 2>/dev/null

### Отключаем IPv6: Удаляем IPv6-адрес с LAN
uci delete network.lan.ip6addr   2>/dev/null
uci delete network.lan.ip6prefix 2>/dev/null

### Отключаем IPv6: Удаляем IPv6-туннели
uci delete network.wan6 2>/dev/null
uci delete network.6in4 2>/dev/null
uci delete network.6to4 2>/dev/null

### Отключаем IPv6: Удаляем IPv6-правила в Firewall
for rule in $(uci show firewall|grep "family='ipv6'"|cut -d'.' -f2|cut -d'=' -f1|sort -t'[' -k2 -rn); do uci delete firewall.$rule; done
for rule in $(uci show firewall|grep "ip6proto="|cut -d'.' -f2|cut -d'=' -f1); do uci delete firewall.$rule.ip6proto; done
for rule in $(uci show firewall|grep "@rule"|grep -v "family="|cut -d'[' -f2|cut -d']' -f1); do uci set firewall.@rule[$rule].family='ipv4'; done

### Отключаем IPv6: Добавляем параметры в sysctl
grep -q "net.ipv6.conf.all.disable_ipv6=1"     /etc/sysctl.conf || echo "net.ipv6.conf.all.disable_ipv6=1"     >>/etc/sysctl.conf
grep -q "net.ipv6.conf.default.disable_ipv6=1" /etc/sysctl.conf || echo "net.ipv6.conf.default.disable_ipv6=1" >>/etc/sysctl.conf
grep -q "net.ipv6.conf.lo.disable_ipv6=1"      /etc/sysctl.conf || echo "net.ipv6.conf.lo.disable_ipv6=1"      >>/etc/sysctl.conf
grep -q "net.ipv6.conf.br-lan.disable_ipv6=1"  /etc/sysctl.conf || echo "net.ipv6.conf.br-lan.disable_ipv6=1"  >>/etc/sysctl.conf

### Устанавливаем часовой пояс
# System -> System -> General Settings -> Timezone: Europa/Samara -> Save & Apply
uci set system.@system[0].timezone='<+04>-4'
uci set system.@system[0].zonename='Europe/Samara'

### Применяем изменения
uci commit
/etc/init.d/system   restart
/etc/init.d/network  restart
/etc/init.d/firewall restart
sysctl -p -q

### Удаляем IPv6-пакеты
opkg remove --force-removal-of-dependent-packages odhcp6c odhcpd-ipv6only kmod-ip6tables kmod-nf-nat6 kmod-ipt-nat6 ip6tables luci-proto-ipv6 2>/dev/null

### Отключаем ненужные сервисы в автозагрузке:
/etc/init.d/cron           stop 2>/dev/null && /etc/init.d/cron           disable
/etc/init.d/wpad           stop 2>/dev/null && /etc/init.d/wpad           disable
/etc/init.d/youtubeUnblock stop 2>/dev/null && /etc/init.d/youtubeUnblock disable
```

2. **Устанавливаем клиента AmneziaWG на hAP ac lite, перепрошитый на OpenWrt 24.10.4**

```bash
### Обновляем списки пакетов
opkg update

### Устанавливаем необходимые зависимости для AmneziaWG
opkg install kmod-udptunnel4 kmod-udptunnel6 kmod-crypto-lib-chacha20poly1305 kmod-crypto-lib-curve25519 kmod-crypto-hash kmod-crypto-aead

### Скачиваем и устанавливаем пакеты AmneziaWG для OpenWrt 24.10.4:
# - kmod-amneziawg_v24.10.4_mips_24kc_ath79_mikrotik.ipk (модуль ядра)
# - amneziawg-tools_v24.10.4_mips_24kc_ath79_mikrotik.ipk (утилиты)
# - luci-proto-amneziawg_v24.10.4_mips_24kc_ath79_mikrotik.ipk (интеграция в веб-интерфейс LuCI)
# System -> Software -> Upload Package.. -> Browse.. -> Upload -> Install -> Dismiss
URL="https://github.com/Slava-Shchipunov/awg-openwrt/releases/download/v24.10.4"
wget -qO /tmp/kmod-amneziawg.ipk "$URL/kmod-amneziawg_v24.10.4_mips_24kc_ath79_mikrotik.ipk" && opkg install /tmp/kmod-amneziawg.ipk
wget -qO /tmp/amneziawg-tools.ipk "$URL/amneziawg-tools_v24.10.4_mips_24kc_ath79_mikrotik.ipk" && opkg install /tmp/amneziawg-tools.ipk
wget -qO /tmp/luci-proto-amneziawg.ipk "$URL/luci-proto-amneziawg_v24.10.4_mips_24kc_ath79_mikrotik.ipk" && opkg install /tmp/luci-proto-amneziawg.ipk
rm -f /tmp/*.ipk

### Перезагружаемся
# System -> Reboot -> Perform reboot
reboot
```

3. **Настраиваем интерфейс AmneziaWG через веб-интерфейс**

 - **Network** -> **Interfaces** -> **Add new interface..** -> Name: `awg0`, Protocol: `AmneziaWG VPN` -> **Create interface**.
 - Import configuration: **Load configuration..**
 - Скачиваем конфиг AmneziaWG с сайта [WARP Генератор](https://warp-gen.github.io) или вставляем свой в пустое поле.
 - **Import settings** -> **OK**.
 - **Save** -> **Save & Apply**.

4. **Настраиваем Firewall и маршруты**

```bash
### Добавляем awg0 в зону WAN Firewall
# Network -> Interfaces -> awg0 -> Edit -> Create / Assign firewall-zone: wan -> Save -> Save & Apply
uci -q get firewall.@zone[1].network|grep -q 'awg0' || uci add_list firewall.@zone[1].network='awg0'

### Устанавливаем Persistent Keepalive для поддержания NAT-сессии у провайдера
# Network -> Interfaces -> awg0 -> Edit -> Peers -> Edit -> Persistent Keepalive: 25 -> Save -> Save -> Save & Apply
uci set network.awg0.persistent_keepalive='25'

### Добавляем маршрут по умолчанию через AmneziaWG:
# Network -> Routing -> Add -> Interface: awg0, Route type: unicast, Target: 0.0.0.0/0
uci set network.defroute_awg=route
uci set network.defroute_awg.interface='awg0'
uci set network.defroute_awg.target='0.0.0.0/0'

### Добавляем маршрут к LAN, которая находится за OpenWrt (если такая есть):
# Network -> Routing -> Add -> Interface: wan, Route type: unicast, Target: 10.0.0.0/8
uci set network.route_mikrotik=route
uci set network.route_mikrotik.interface='wan'
uci set network.route_mikrotik.target='10.0.0.0/8'

### Применяем изменения
uci commit network
uci commit firewall
/etc/init.d/network  restart
/etc/init.d/firewall restart
sleep 5

### Проверяем интерфейс AmneziaWG
awg show

### Проверяем маршруты
route -n
```

***

## Настройка роутера (RouterOS)

1. **Создаем интерфейс bridge-awg и добавляем в него любой свободный порт**

```bash
/interface bridge
add name=bridge-awg

/interface bridge port
add bridge=bridge-awg interface=ether4
```

2. **Назначаем IP-адрес интерфейсу bridge-awg**

```bash
/ip address
add address=192.168.1.2/24 interface=bridge-awg
```

3. **Создаем списки частных сетей и сетей заблокированных сервисов**

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

4. **Создаем правила для доменов заблокированных сервисов**

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

5. **Создаем таблицу маршрутизации для VPN**

```bash
/routing table
add fib name=bypass-vpn
```

6. **Создаем правила, маркирующие трафик к заблокированным сервисам**

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

7. **Создаем правила, которые согласуют MTU с VPN-интерфейсом**

```bash
/ip firewall mangle
add action=change-mss chain=forward comment="CLAMP MSS FOR TCP SYN TO BYPASS-VPN"   new-mss=clamp-to-pmtu out-interface=bridge-awg passthrough=yes protocol=tcp tcp-flags=syn
add action=change-mss chain=forward comment="CLAMP MSS FOR TCP SYN FROM BYPASS-VPN" new-mss=clamp-to-pmtu  in-interface=bridge-awg passthrough=yes protocol=tcp tcp-flags=syn
```

8. **Создаем правила маскарадинга для VPN-трафика**

```bash
/ip firewall nat
add action=accept     chain=srcnat comment="ACCEPT ALL FROM PRIVATE-LANS TO PRIVATE-LANS FOR VPN" dst-address-list=PRIVATE-LANS src-address-list=PRIVATE-LANS
add action=masquerade chain=srcnat comment="MASQ ALL FROM PRIVATE-LANS MARKED AS BYPASS-VPN --> BYPASS-VPN-IF" out-interface=bridge-awg routing-mark=bypass-vpn src-address-list=PRIVATE-LANS
```

9. **Настраиваем маршрутизацию VPN-трафика через интерфейс bridge-awg**

```bash
/ip route
add dst-address=0.0.0.0/0 gateway=192.168.1.1 routing-table=bypass-vpn
```

10. **Проверяем маршрутизацию с клиентского ПК**

  Трассировка ya.ru после нашего роутера НЕ ДОЛЖНА проходить через хоп из сетей 192.168.1.0/24 и 10.8.1.0/24:
```powershell
C:\>tracert ya.ru

Трассировка маршрута к ya.ru [77.88.55.242]
с максимальным числом прыжков 30:

  1     1 ms     1 ms     1 ms  10.255.253.1
  2     7 ms     6 ms     6 ms  178.45.160.1
```

  А трассировка googlevideo.com после нашего роутера ДОЛЖНА проходить через хопы из сетей 192.168.1.0/24 и 10.8.1.0/24:
```powershell
C:\>tracert googlevideo.com

Трассировка маршрута к googlevideo.com [172.217.19.228]
с максимальным числом прыжков 30:

  1     1 ms     1 ms     1 ms  10.255.253.1
  2     1 ms     1 ms     1 ms  192.168.1.1
  3    74 ms    74 ms    74 ms  10.8.1.0
```

  Если все так - поздравляю, вы сломали чебурнет!

**Ссылки по теме:**

 - [Смотрим рекламу на Youtube в 4K](https://telegra.ph/Smotrim-reklamu-na-Youtube-v-4K-08-12)
 - [Настройка клиента Wireguard на Mikrotik RouterOS для подключения к VPS, VDS серверу или готовой конфигурации](https://kiberlis.ru/mikrotik-wireguard-client)
 - [Wireguard в Mikrotik](https://www.youtube.com/live/eRcrZkwd5IM)
 - [Как определить оптимальный размер MTU?](https://help.keenetic.com/hc/ru/articles/214470885-%D0%9A%D0%B0%D0%BA-%D0%BE%D0%BF%D1%80%D0%B5%D0%B4%D0%B5%D0%BB%D0%B8%D1%82%D1%8C-%D0%BE%D0%BF%D1%82%D0%B8%D0%BC%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9-%D1%80%D0%B0%D0%B7%D0%BC%D0%B5%D1%80-MTU)
 - [YouTube Blocked hosts in Russia](https://gist.github.com/MashinaMashina/58eed9128ade2c8ff50a84d523ae97d1#file-youtube-blocked-hosts-in-russia-md)
 - [Диапазон IP-адресов Instagram, Netflix, ChatGPT, Youtube, Twitter](https://rockblack.su/vpn/dopolnitelno/diapazon-ip-adresov)

***

