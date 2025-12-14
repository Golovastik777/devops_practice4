University: [ITMO University](https://itmo.ru/ru/)<br />
Faculty: [FICT](https://fict.itmo.ru)<br />
Course: [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)<br />
Year: 2025/2026<br />
Group: K3320<br />
Author: Yakovlev Igor Sergeevich<br />
Lab: Lab4<br />
Date of create: 14.12.25<br />
Date of finished: 14.12.25<br />


# Задание

Вам необходимо сделать IP/MPLS сеть связи для "RogaIKopita Games" изображенную на рисунке 1 в ContainerLab. Необходимо создать все устройства указанные на схеме и соединения между ними.

<img src="task.png" width=600px>

- Помимо этого вам необходимо настроить IP адреса на интерфейсах.
- Настроить OSPF и MPLS.
- Настроить iBGP с route reflector кластером

И вот тут лабораторная работа работа разделяется на 2 части, в первой части вам надо настроить L3VPN, во второй настроить VPLS, но при этом менять топологию не требуется. Вы можете просто разобрать VRF и на их месте собрать VPLS.

Первая часть:
- Настроить iBGP RR Cluster.
- Настроить VRF на 3 роутерах.
- Настроить RD и RT на 3 роутерах.
- Настроить IP адреса в VRF.
- Проверить связность между VRF
- Настроить имена устройств, сменить логины и пароли.

Вторая часть:
- Разобрать VRF на 3 роутерах (или отвязать их от интерфейсов).
- Настроить VPLS на 3 роутерах.
- Настроить IP адресацию на PC1,2,3 в одной сети.
- Проверить связность.

# Схема

Схема, построенная в draw.io:

<img src="images/graph-1.png" width=600px>

Схема, построенная ContainerLab:

<img src="images/graph-2.png" width=600px>


# Конфиг yaml

```
name: lab4_1
mgmt:
  network: custom_mgmt
  ipv4-subnet: 172.16.16.0/24

topology:
  nodes:
    R01.SPB:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.16.101
      startup-config: config/part1/R01.SPB.rsc
    R01.HKI:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.16.102
      startup-config: config/part1/R01.HKI.rsc
    R01.SVL:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.16.103
      startup-config: config/part1/R01.SVL.rsc
    R01.LND:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.16.104
      startup-config: config/part1/R01.LND.rsc
    R01.LBN:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.16.105
      startup-config: config/part1/R01.LBN.rsc
    R01.NY:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.16.106
      startup-config: config/part1/R01.NY.rsc
    PC1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.16.2
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh
    PC2:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.16.3
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh
    PC3:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.16.4
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh


  links:
    - endpoints: ["R01.SPB:eth1","R01.HKI:eth1"]
    - endpoints: ["R01.NY:eth1","R01.LND:eth1"]
    - endpoints: ["R01.SVL:eth1","R01.LBN:eth1"]
    - endpoints: ["R01.HKI:eth2","R01.LND:eth3"]
    - endpoints: ["R01.HKI:eth3","R01.LBN:eth2"]
    - endpoints: ["R01.LND:eth2","R01.LBN:eth3"]
    - endpoints: ["R01.SPB:eth2","PC1:eth1"]
    - endpoints: ["R01.NY:eth2","PC2:eth1"]
    - endpoints: ["R01.SVL:eth2","PC3:eth1"]
```
# Конфиги

## Роутеры

### OSPF + MPLS

Настройка OSPF + MPLS берётся из предыдущей работы с небольшими изменениями портов и подключённых сетей в соответствии с изменённой схемой сети.

### iBGP c Route Reflector кластером
 
1. Выбираем ASN

AS — система сетей под управлением единственной административной зоны. У нас только 1 такая зона, поэтому выбираем 1 приватное (64512-65534) число, например 65000.

2. Настраиваем iBGP

Мы работаем в рамках 1 зоны, поэтому мы используем протокол iBGP. Т.е. в конфигах мы будем указывать номер зоны конфигурируемого роутера как 65000, и в пирах/соседях тоже будет номер зоны 65000.

- `/routing bgp instance` (set default, номер AS и айди роутера. Здесь снова пользуемся айди таким же, что и у loopback)
- `/routing bgp peer` (remote-address, remote-as, route-reflect между RR роутерами, address-families=l2vpn,vpnv4, привязываем к loopback)
- `/routing bgp network` (сеть loopback)

3. Настраиваем vrf на внешних роутерах

- /interface bridge (привяжем рут vrf и свой адрес к мосту); 
- /ip address (адрес для моста, с 32-й маской)
- /ip route vrf (export-route-targets, import-route-targets, route-distinguisher с тем же ASN, routing-mark)
- /routing bgp instance vrf (redistribute-connected=yes и тот же routing-mark)

Вот здесь, преисполнившись чтением документации, важно не дать cluster-id кластеру роутеров :) В данной работе роутеры подключены только к одному RR-рефлектору, поэтому им надо пройти через 2 RR, чтобы друг к другу прийти. А что будет делать RR, получивший запрос со своим же cluster-id? Правильно, его дропать. 

4. Часть 2: Настраиваем VPLS

В 1-й части это было неактуально, сейчас важно отметить. В /mpls ldp interface нужно указать интерфейс, направленный на компьютер.

На всех 3-ёх внешних роутерах настраиваем:

- /interface bridge (имя)
- /interface bridge port (привязываем к порту, направленный на компьютер)
- /interface vpls bgp-vpls (тоже руты как у /ip route vrf, вместо routing-mark site-id с уникальным айди для каждого роутера)
- /ip address (привязываем к мосту)

Чтобы задать айпи всем компьютерам в одной этой сети впн-а, надо выбрать, на каком роутере поставить dhcp-сервер. Допустим, это будет Санкт-Петербургский.

На всех роутерах убираем/выключаем раздачу dhcp-адресов из 1-й части, на SPB создаём новый пул из сети впн и подключаем его к нему.


## Компьютеры

Аналогично предыдущим работам, в компьютеры запрашивают айпи у соответствующего dhcp-сервера на eth1 (в 2-й части этот сервер на SPB роутере).
```
#!/bin/sh
ip route del default via 172.16.18.1 dev eth0
udhcpc -i eth1
```
#### Тест

Пинг с PC1 на PC2

```
/ # ping -c 3 172.16.16.3
PING 172.16.16.3 (172.16.16.3): 56 data bytes
64 bytes from 172.16.16.3: seq=0 ttl=64 time=0.244 ms
64 bytes from 172.16.16.3: seq=1 ttl=64 time=0.135 ms
64 bytes from 172.16.16.3: seq=2 ttl=64 time=0.152 ms

--- 172.16.16.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.135/0.177/0.244 ms
```

Пинг с PC1 на PC3

```
/ # ping -c 3 172.16.16.4
PING 172.16.16.4 (172.16.16.4): 56 data bytes
64 bytes from 172.16.16.4: seq=0 ttl=64 time=0.276 ms
64 bytes from 172.16.16.4: seq=1 ttl=64 time=0.171 ms
64 bytes from 172.16.16.4: seq=2 ttl=64 time=0.141 ms

--- 172.16.16.4 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.141/0.196/0.276 ms
```

Работа OSPF:

```
Flags: X - disabled, A - active, D - dynamic, C - connect, S - static, r - rip, b - bgp, o - ospf, m - mme, B - blackhole, U - unreachable, P - prohibit 
 #      DST-ADDRESS        PREF-SRC        GATEWAY            DISTANCE
 0 ADC  10.100.1.1/32      10.100.1.1      br100                     0
 1 ADb  10.100.1.2/32                      10.255.255.6            200
 2 ADb  10.100.1.3/32                      10.255.255.3            200
 3 ADC  10.20.1.0/30       10.20.1.1       ether2                    0
 4 ADo  10.20.2.0/30                       10.20.1.2               110
 5 ADo  10.20.3.0/30                       10.20.1.2               110
 6 ADo  10.20.11.0/30                      10.20.1.2               110
 7 ADo  10.20.12.0/30                      10.20.1.2               110
 8 ADo  10.20.13.0/30                      10.20.1.2               110
 9 ADC  10.255.255.1/32    10.255.255.1    loopback                  0
10 ADo  10.255.255.2/32                    10.20.1.2               110
11 ADo  10.255.255.3/32                    10.20.1.2               110
12 ADo  10.255.255.4/32                    10.20.1.2               110
13 ADo  10.255.255.5/32                    10.20.1.2               110
14 ADo  10.255.255.6/32                    10.20.1.2               110
15 ADC  172.31.255.28/30   172.31.255.30   ether1                    0
16 ADC  192.168.10.0/24    192.168.10.1    ether3                    0
17 ADo  192.168.11.0/24                    10.20.1.2               110
18 ADo  192.168.12.0/24                    10.20.1.2               110
```

Работы MPLS:

```
Flags: H - hw-offload, L - ldp, V - vpls, T - traffic-eng 
 #    IN-LABEL                                      OUT-LABELS                                   DESTINATION                    INTERFACE                                   NEXTHOP        
 0    expl-null                                    
 1    16                                                                                         10.100.1.1/32@VRF_DEVOPS      
 2  L 17                                            16                                           10.255.255.5/32                ether2                                      10.20.1.2      
 3  L 18                                            18                                           192.168.11.0/24                ether2                                      10.20.1.2      
 4  L 19                                                                                         10.20.12.0/30                  ether2                                      10.20.1.2      
 5  L 20                                            21                                           10.20.13.0/30                  ether2                                      10.20.1.2      
 6  L 21                                            17                                           10.255.255.6/32                ether2                                      10.20.1.2      
 7  L 22                                            20                                           10.255.255.4/32                ether2                                      10.20.1.2      
 8  L 23                                                                                         10.255.255.2/32                ether2                                      10.20.1.2      
 9  L 24                                                                                         10.20.11.0/30                  ether2                                      10.20.1.2      
10  L 25                                            22                                           10.20.2.0/30                   ether2                                      10.20.1.2      
11  L 26                                            26                                           192.168.12.0/24                ether2                                      10.20.1.2      
12  L 27                                            24                                           10.255.255.3/32                ether2                                      10.20.1.2      
13  L 28                                            19                                           10.20.3.0/30                   ether2                                      10.20.1.2
```

Работа iBGP:

```
Flags: X - disabled, A - active, D - dynamic, C - connect, S - static, r - rip, b - bgp, o - ospf, m - mme, B - blackhole, U - unreachable, P - prohibit 
 #      DST-ADDRESS        PREF-SRC        GATEWAY            DISTANCE
 0 ADb  10.100.1.2/32                      10.255.255.6            200
 1 ADb  10.100.1.3/32                      10.255.255.3            200
```

VRF:

```
Flags: X - disabled, A - active, D - dynamic, C - connect, S - static, r - rip, b - bgp, o - ospf, m - mme, B - blackhole, U - unreachable, P - prohibit 
 #      DST-ADDRESS        PREF-SRC        GATEWAY            DISTANCE
 0 ADC  10.100.1.1/32      10.100.1.1      br100                     0
 1 ADb  10.100.1.2/32                      10.255.255.6            200
 2 ADb  10.100.1.3/32                      10.255.255.3            200
```

Связность между VRF:

```
  SEQ HOST                                     SIZE TTL TIME  STATUS                                                                                                                       
    0 10.100.1.2                                 56  62 1ms  
    1 10.100.1.2                                 56  62 1ms  
    2 10.100.1.2                                 56  62 1ms  
    sent=3 received=3 packet-loss=0% min-rtt=1ms avg-rtt=1ms max-rtt=1ms 
```


### Часть 2

#### Топология

```
name: lab4_2
mgmt:
  network: custom_mgmt-part2
  ipv4-subnet: 172.16.18.0/24

topology:
  nodes:
    R01.SPB:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.18.101
      startup-config: config/part2/R01.SPB.rsc
    R01.HKI:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.18.102
      startup-config: config/part2/R01.HKI.rsc
    R01.SVL:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.18.103
      startup-config: config/part2/R01.SVL.rsc
    R01.LND:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.18.104
      startup-config: config/part2/R01.LND.rsc
    R01.LBN:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.18.105
      startup-config: config/part2/R01.LBN.rsc
    R01.NY:
      kind: vr_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.16.18.106
      startup-config: config/part2/R01.NY.rsc
    PC1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.18.2
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh
    PC2:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.18.3
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh
    PC3:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.18.4
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh


  links:
    - endpoints: ["R01.SPB:eth1","R01.HKI:eth1"]
    - endpoints: ["R01.NY:eth1","R01.LND:eth1"]
    - endpoints: ["R01.SVL:eth1","R01.LBN:eth1"]
    - endpoints: ["R01.HKI:eth2","R01.LND:eth3"]
    - endpoints: ["R01.HKI:eth3","R01.LBN:eth2"]
    - endpoints: ["R01.LND:eth2","R01.LBN:eth3"]
    - endpoints: ["R01.SPB:eth2","PC1:eth1"]
    - endpoints: ["R01.NY:eth2","PC2:eth1"]
    - endpoints: ["R01.SVL:eth2","PC3:eth1"]
```


#### Тест

Выдача ip:

```

Flags: X - disabled, R - radius, D - dynamic, B - blocked 
 #   ADDRESS                             MAC-ADDRESS       HOST-NAME                   SERVER                   RATE-LIMIT                   STATUS  LAST-SEEN                             
 0 D 10.100.1.254                        AA:C1:AB:BF:EA:D6                             dhcp-vpls                                             bound   1m23s                                 
 1 D 10.100.1.253                        AA:C1:AB:01:7D:D0                             dhcp-vpls                                             bound   40s                                   
 2 D 10.100.1.252                        AA:C1:AB:3C:4A:11                             dhcp-vpls                                             bound   35s      
```

Пинг с PC1 на PC2:

```
/ # ping -c 3 10.100.1.252
PING 10.100.1.252 (10.100.1.252): 56 data bytes
64 bytes from 10.100.1.252: seq=0 ttl=64 time=3.825 ms
64 bytes from 10.100.1.252: seq=1 ttl=64 time=2.552 ms
64 bytes from 10.100.1.252: seq=2 ttl=64 time=2.626 ms

--- 10.100.1.252 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 2.552/3.001/3.825 ms
```

Пинг с PC1 на PC3:

```
/ # ping -c 3 10.100.1.253
PING 10.100.1.253 (10.100.1.253): 56 data bytes
64 bytes from 10.100.1.253: seq=0 ttl=64 time=5.977 ms
64 bytes from 10.100.1.253: seq=1 ttl=64 time=3.922 ms
64 bytes from 10.100.1.253: seq=2 ttl=64 time=3.689 ms

--- 10.100.1.253 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 3.689/4.529/5.977 ms
```

# Заключение

В ходе работы была создана IP/MPLS сеть связи. На нём были настроены протоколы OSPF, MPLS и iBGP с Route Reflector-кластером.  

В 1-й части был настроен L3VPN, используя VRF, в 2-й части был настроен VPLS.

Все устройства были успешно соединены, задачи работы выполнены.

Цель работы была выполнена.


