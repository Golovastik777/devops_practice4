# Отчет по лабораторной работе №2


## Задание

Вам необходимо сделать сеть связи в трех геораспределенных офисах "RogaIKopita Games" изображенную на рисунке ниже в ContainerLab. Необходимо создать все устройства указанные на схеме и соединения между ними.

* Помимо этого вам необходимо настроить IP адреса на интерфейсах.
* Создать DHCP сервера на роутерах в сторону клиентских устройств.
* Настроить статическую маршрутизацию.
* Настроить имена устройств, сменить логины и пароли.

<img width="541" height="361" alt="image" src="https://github.com/user-attachments/assets/99929a67-daae-4ab6-abfb-fafe552f93ed" />

## Описание работы


### Топология 
Файл `lab.yml` описывает топологию сети, состоящую из трёх маршрутизаторов (R01, R02, R03), представляющих три разных офиса компании, и трёх ПК (PC1, PC2, PC3), которые подключены к своим локальным маршрутизаторам. Маршрутизаторы соединены между собой линками, и каждый ПК подключён к своему маршрутизатору через локальный интерфейс.
```yaml
name: rogakopita-games

topology:
    kinds:
        vr-ros:
            image: vrnetlab/mikrotik_routeros:6.47.9
        linux:
            image: alpine:latest
    nodes:
        R01_Moscow:
            kind: vr-ros
            mgmt-ipv4: 192.168.50.10
            startup-config: ./configs/r1.rsc
        R02_Berlin:
            kind: vr-ros
            mgmt-ipv4: 192.168.50.20
            startup-config: ./configs/r2.rsc
        R03_Frankfurt:
            kind: vr-ros
            mgmt-ipv4: 192.168.50.30
            startup-config: ./configs/r3.rsc
        PC1:
            kind: linux
            binds:
              - ./configs:/configs
        PC2:
            kind: linux
            binds:
              - ./configs:/configs
        PC3:
            kind: linux
            binds:
              - ./configs:/configs
    links:
        - endpoints: ["R01_Moscow:eth2", "R02_Berlin:eth2"]
        - endpoints: ["R01_Moscow:eth3", "R03_Frankfurt:eth3"]
        - endpoints: ["R01_Moscow:eth4", "PC1:eth2"]
        - endpoints: ["R02_Berlin:eth3", "R03_Frankfurt:eth2"]
        - endpoints: ["R02_Berlin:eth4", "PC2:eth2"]
        - endpoints: ["R03_Frankfurt:eth4", "PC3:eth2"]

mgmt:
  network: mgmt-net
  ipv4-subnet: 192.168.50.0/24
```

Ниже можно ознакомиться с графическим представлением этой схемы:


<img width="562" height="510" alt="lab2-topology drawio" src="https://github.com/user-attachments/assets/a420fec7-a47f-49ac-9f20-d539cead097d" />


### Настройка маршрутизаторов

Пример настройки `R01`:
```rsc
/ip pool
add name=dhcp_pool_Moscow ranges=192.168.1.3-192.168.1.200
/ip dhcp-server
add address-pool=dhcp_pool_Moscow disabled=no interface=ether5 name=dhcp_Moscow
/ip address
add address=10.10.10.1/30 interface=ether3
add address=30.30.30.2/30 interface=ether4
add address=192.168.1.2/24 interface=ether5
/ip dhcp-server network
add address=192.168.1.0/24 gateway=192.168.1.2
/ip route
add distance=1 dst-address=192.168.2.10/24 gateway=10.10.10.2
add distance=1 dst-address=192.168.3.10/24 gateway=30.30.30.1
/system identity
set name=R01_Moscow
```

### Настройка ПК

Пример настройки PC:
```bash
#!/bin/sh
udhcpc -i eth2
ip route del default via 192.168.50.1 dev eth0
```

### Пример работы

После настройки маршрутов и запуска контейнеров была проведена проверка. Результаты выводятся в командной строке маршрутизаторов и ПК.

Команда `ip route show` на маршрутизаторе `R01` и `R02` показала корректную таблицу маршрутизации:
![Image alt](https://github.com/Golovastik777/devops_practice2/raw/main/lab2_iproute.png)

Проверка командой ping с `PC1` до `PC2` и `PC3` показала успешное прохождение пакетов:

![Image alt](https://github.com/Golovastik777/devops_practice2/raw/main/lab2_ping.png)


## Заключение

В ходе выполнения лабораторной работы была создана и настроена сеть из трёх геораспределенных офисов с использованием маршрутизаторов и клиентских ПК. Были настроены cтатическая маршрутизация, и проверена корректность работы сети с помощью пингов и трассировки. Все поставленные задачи выполнены, и сеть работает корректно, обеспечивая связь между офисами компании и клиентскими устройствами.
