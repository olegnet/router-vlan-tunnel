# Настраиваем роутер и WiFi с VLAN в тоннель

<!-- TOC -->
* [Настраиваем роутер и WiFi с VLAN в тоннель](#настраиваем-роутер-и-wifi-с-vlan-в-тоннель)
    * [Роутер: pfSense](#роутер-pfsense)
      * [Добавляем VLAN](#добавляем-vlan)
      * [Добавляем gateway](#добавляем-gateway)
      * [Добавляем правила в firewall](#добавляем-правила-в-firewall)
      * [Включаем DHCP сервер](#включаем-dhcp-сервер)
    * [Сервер: Debian на VM в Proxmox](#сервер-debian-на-vm-в-proxmox)
      * [Конфигурация VM на Proxmox](#конфигурация-vm-на-proxmox)
      * [Настройка сервера](#настройка-сервера)
    * [WiFi: OpenWRT](#wifi-openwrt)
      * [Добавляем VLAN](#добавляем-vlan-1)
      * [Добавляем bridge](#добавляем-bridge)
      * [Добавляем интерфейс](#добавляем-интерфейс)
      * [Добавляем SSID](#добавляем-ssid)
    * [Бонус: доступ к сети снаружи (remote access)](#бонус-доступ-к-сети-снаружи-remote-access)
  * [Disclaimer](#disclaimer)
<!-- TOC -->

На некоторые устройства не очень удобно или вообще невозможно поставить приложения, которые позволяют
создавать тоннели или изменять роутинг.

Если у вас есть точка доступа с WiFi, которую вы сами настраиваете, обычно можно создать в сети несколько VLAN,
на каждую создать свой SSID (имя точки) и иметь несколько "виртуальных" WiFi точек с разными маршрутами,
авторизацией и прочими настройками.

Здесь мы рассмотрим настройку такой схемы с роутером на pfSense, точкой доступа на OpenWRT и отдельным линуксом
для тоннеля в VM на Proxmox. Эти идеи можно использовать и в других комбинациях железа и софта.
Например, сервер может быть не VM, а отдельной коробкой и т.д.

Подразумевается, что вы знаете, как настраивать ваше железо, но вам некогда самому придумывать эту схему и искать
все пункты многочисленных меню в нужном порядке. 


### Роутер: pfSense

Все примеры валидны для pfSense 2.8.0, но кажется уже несколько версий в этих меню ничего не менялось.

Рассмотрим добавление дополнительного VLAN с роутингом этого VLAN через сервер, который мы настроим в следующем разделе.

#### Добавляем VLAN

Заходим в `Interfaces -> VLANs`, нажимаем `Add` и добавляем новый тег. Например, `4`. 
`Parent Interface` должен быть `LAN`.

Теперь в `Interfaces -> Interface Assignments` в `Available network ports` выбираем `VLAN4` и нажимаем кнопку `Add`.
Новый интерфейс появляется в списке. Удобно исправить его имя `OPTn` на тот же `VLAN4` и настроить подсеть.

 - IPv4 Configuration Type: Static IPv4
 - IPv4 Address: 192.168.4.1 / 24
 - IPv4 Upstream gateway: none

Ниже здесь же мелким шрифтом подписано, почему последний пункт важен и где нужно добавлять gateway 
(куда пойдёт весь трафик нашего VLAN вместо маршрута по-умолчанию).

#### Добавляем gateway

Переходим в `System -> Routing -> Gateways`. Здесь у вас скорее всего есть `Default gateway` для IPv4 и, возможно,
для IPv6. Нажимаем `Add` и добавляем ещё один.

 - Interface: LAN – или другой, если ваш сервер в отдельном VLAN
 - Address Family: IPv4
 - Name: GW_1
 - Gateway: 192.168.10.12
 - Gateway Action: Disable Gateway Monitoring Action

Осталось добавить правила в файрвол.

#### Добавляем правила в firewall

Переходим в `Firewall -> Rules -> VLAN4`.

Первое правило, пропускаем весь трафик из `VLAN4` в `LAN`:
 - Action: Pass
 - Interface: VLAN4
 - Address Family: IPv4
 - Protocol: Any
 - Source: VLAN4 Subnets
 - Destination: LAN Subnets

Для этого правила в `Advanced Options` ничего не трогаем. 

Второе правило, весь остальной трафик отправляем в созданный выше `GW_1`.
 - Action: Pass
 - Interface: VLAN4
 - Address Family: IPv4
 - Protocol: Any
 - Source: VLAN4 Subnets
 - Destination: Any
 - Gateway: GW_1

Последний пункт находится в `Advanced Options`.

Правила должны идти именно в таком порядке.

#### Включаем DHCP сервер

Переходим в `Services -> DHCP Server -> VLAN4`:
 - Enable: ✅
 - Address Pool Range: 192.168.4.128 – 192.168.4.253

Здесь обычно можно ничего не трогать, но посмотрите внимательно на настройки DNS с точки зрения доступности
серверов с тоннелем и без.


### Сервер: Debian на VM в Proxmox

Сразу отвечу на вопрос, если Proxmox и Linux, то почему VM, а не CT. 

Мой тоннель не захотел запускаться в контейнере. Он коммерческий и возможно это часть его защиты.
Если ваш тоннель работает в CT, ничего не мешает запускать его там.

#### Конфигурация VM на Proxmox

Мы не будем рассматривать настройку какого-то конкретного тоннеля. Для целей этого текста подойдёт любой, 
который может автоматически запускаться на старте сервера и поднимать `tun0`.

Параметры VM с запасом: 2 ядра CPU, 2 GB RAM, остальные настройки по-умолчанию.
Обычный Debian 12 подойдёт. Выбираем чекбоксы без UI, но с доступом по ssh.
Для сети указываем статический адрес, который мы описали выше, как `GW_1`.

Не забудьте добавить нужные сети в файрвол вашей VM. 

В одном из следующих разделов сюда возможно добавятся правила 
[для других сетей.](#бонус-доступ-к-сети-снаружи-remote-access)
 - Direction: in
 - Action: ACCEPT
 - Source: 192.168.4.0/24
 - Enable: ✅
 - Log level: nolog – но можно поставить info на время настройки

Естественно, нужно разрешить и доступ по ssh с того места, откуда вы будете настраивать VM.

#### Настройка сервера

Теперь нужно в /etc/sysctl.conf раскомментировать или добавить строку `net.ipv4.ip_forward=1`

```shell
# Обновляем систему после установки и ставим нужные пакеты
apt update
apt dist-upgrade
apt install iptables iptables-persistent

# Удаляем правила
iptables -F
iptables -t nat -F

# Принимаем всё, что прошло файрвол Proxmox
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# Добавляем NAT для VLAN4 через tun0
iptables -t nat -A POSTROUTING -s 192.168.4.0/24 -o tun0 -j MASQUERADE
# Повторяем эту строку для каждой сети из конфигурации раздела настройки remote access ниже

# Проверяем, что получилось
iptables -t nat -L -v

# Сохраняем
netfilter-persistent save
```

Можно перезагрузиться, чтобы сработало обновление sysctl.conf и проверить, как сохранились правила файрвола.

Должно быть примерно так:
```text
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether bc:24:11:37:09:9e brd ff:ff:ff:ff:ff:ff
    altname enp0s18
    inet 192.168.10.12/24 brd 192.168.10.255 scope global ens18
       valid_lft forever preferred_lft forever
31: tun0: ...
    link/none
    inet x.x.x.x peer x.x.x.x/32 scope global tun0
       valid_lft forever preferred_lft forever

$ sudo iptables -t nat -L -v
Chain PREROUTING (policy ACCEPT 4388 packets, 1089K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 58 packets, 6016 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 36935 packets, 2448K bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 36507 packets, 2420K bytes)
 pkts bytes target     prot opt in     out     source               destination
 3300  919K MASQUERADE  all  --  any    tun0    192.168.4.0/24       anywhere
```


### WiFi: OpenWRT

Производитель давно перестал обновлять прошивку для моей точки доступа, хотя железо ещё достаточно хорошо.
Тем более, что сама прошивка скорее [напоминает анекдот](https://openwrt.org/toh/tp-link/eap245_v1#recipe) (см. п.2).

При этом нельзя сказать, что такую настройку невозможно сделать в оригинальной прошивке.
Так что если у вас не OpenWRT, скорее всего ваша точка доступа на это тоже способна. 
Общие принципы будут те же, менюшки немного другие.

Полностью настройку OpenWRT описывать нет смысла, это тема для другого текста, и не одного. 
Опишем процесс добавления отдельного SSID для выбранного VLAN. Пути в меню верны для версии OpenWrt 24.10.

#### Добавляем VLAN

Переходим в `Network -> Interfaces` и выбираем таб `Devices`. Нажимаем `Add device configuration`, там выбираем:
 - Device type: VLAN (802.1q)
 - Base device: eth0
 - VLAN ID: 4
 - Device name: eth0.4 – подставится само
 - Enable IPv6: disabled

Дальше `Save` и `Save&Apply`.

#### Добавляем bridge

Ещё раз нажимаем `Add device configuration`:
 - Device type: bridge device
 - Device name: br-lan4
 - Ports: eth0.4

Опять `Save` и `Save&Apply`.

#### Добавляем интерфейс

Теперь выбираем таб `Interfaces`, там `Add new interface...`
 - Name: vlan4
 - Protocol: Static address
 - Device: br-lan4

и нажимаем `Create interface`. 

Здесь указываем адрес интерфейса из нашего VLAN:
 - IPv4 address: 192.168.4.3 – любой уникальный, который не будет выдаваться роутером по DHCP 
 - IPv4 netmask: 255.255.255.0
 - IPv4 gateway: 192.168.4.1

В табе `DHCP Server` выбираем чекбокс `Ignore interface`. Ешё раз `Save` и `Save&Apply`.

#### Добавляем SSID

Конфигурация почти готова, осталось в `Network -> Wireless` добавить собственно SSID.
Выберите `Add` в `radio0` или `radio1`. Поскольку у вас уже есть минимум один SSID, 
считаем, что в секции `Device Configuration` всё настроено и переходим сразу в `Interface Configuration`.
 - ESSID: имя вашей точки. можно просто добавить `-4` к основному имени
 - Network: vlan4

Остальные настройки по вкусу, не забудте заглянуть в `Wireless Security`.


### Бонус: доступ к сети снаружи (remote access)

Если у вас есть "белый" IPv4 (без NAT провайдера, обычно за отдельную регулярную плату) или IPv6,
можно легко сделать доступ не только к вашей сети, но и к добавленному выше тоннелю.

В pfSense его можно настроить, например, через WireGuard.

Как настраивать сам WireGuard, лучше посмотреть в документации pfSense, дублировать здесь нет смысла.

Основные моменты, которые важны для нашей конфигурации. 

Нужно взять из `WireGuard -> Tunnels -> Interface Addresses` все сети, которые вы хотите маршрутизировать через
этот тоннель, и добавить их в правила всех файрволов, настроенных выше.

В `Firewall -> Rules -> WireGuard` нужно добавить два правила.
Первое правило:
- Action: Pass
- Interface: WireGuard
- Address Family: IPv4
- Protocol: Any
- Source: Network xxx.xxx.xxx.0/24 – подсеть для WireGuard
- Destination: LAN Subnets
- Gateway: GW_1

Второе правило:
- Action: Pass
- Interface: WireGuard
- Address Family: IPv4
- Protocol: Any
- Source: Network xxx.xxx.xxx.0/24 – подсеть для WireGuard
- Destination: Any
- Gateway: GW_1

Можно не добавлять первое правило и тогда весь трафик будет уходить сразу в тоннель, без доступа к внутренним сетям.

Также нужно добавить правила в файрвол VM и в NAT на сервере с тоннелем, как было описано
[выше](#сервер-debian-на-vm-в-proxmox).


## Disclaimer

Автор делится личным опытом, полученным на собственном оборудовании, и не призывает ни повторять,
ни избегать описанных действий.
Все возможные последствия использования этой информации остаются на ответственности читателя.


**Copyright © 2025 Oleg Okhotnikov. All rights reserved.**
