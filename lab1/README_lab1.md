# Отчет по лабораторной работе №1 Яковлев И.С.

## Задание

Вам необходимо сделать трехуровневую сеть связи классического предприятия, изображенную на рисунке 1, в ContainerLab. Необходимо создать все устройства, указанные на схеме ниже, и соединения между ними, правила работы с СontainerLab можно изучить по ссылке.

* Помимо этого вам необходимо настроить IP адреса на интерфейсах и 2 VLAN-a для `PC1` и `PC2`, номера VLAN-ов вы вольны выбрать самостоятельно.
* Также вам необходимо создать 2 DHCP сервера на центральном роутере в ранее созданных VLAN-ах для раздачи IP адресов в них. `PC1` и `PC2` должны получить по 1 IP адресу из своих подсетей.
* Настроить имена устройств, сменить логины и пароли.

![Схема из задания](https://itmo-ict-faculty.github.io/introduction-in-routing/education/labs2023_2024/lab1/3tiernetwork.png)

## Описание работы


### Топология 
В файле `lab.yaml` описана топология сети. Она включает маршрутизатор `R1`, три коммутатора (`SW1`, `SW2`, `SW3`), а также два конечных устройства (`PC1` и `PC2`). Каждый узел имеет свой файл конфигурации (в папке `configs`, который загружается при старте контейнера.

```
name: enterprise-lab

mgmt:
  network: static
  ipv4-subnet: 192.168.10.0/24

topology:
  kinds:
    vr-ros:
      image: vrnetlab/vr-routeros:6.47.9
    linux:
      image: alpine:latest

  nodes:
    R1:
      kind: vr-ros
      mgmt-ipv4: 192.168.10.11
      startup-config: ./configs/r1.rsc
      
    SW1:
      kind: vr-ros
      mgmt-ipv4: 192.168.10.12
      startup-config: ./configs/sw1.rsc
      
    SW2:
      kind: vr-mikrotik_ros
      mgmt-ipv4: 192.168.10.13
      startup-config: ./configs/sw2.rsc
      
    SW3:
      kind: vr-ros
      mgmt-ipv4: 192.168.10.14
      startup-config: ./configs/sw3.rsc
      
    PC1:
      kind: linux
      startup-config: /configs/pc1.sh
      binds:
        - ./configs:/configs
        
    PC2:
      kind: linux
      startup-config: /configs/pc2.sh
      binds:
        - ./configs:/configs

  links:
    - endpoints: ["R1:eth2", "SW1:eth2"]
    - endpoints: ["SW1:eth3", "SW2:eth2"]
    - endpoints: ["SW1:eth4", "SW3:eth2"]
    - endpoints: ["SW2:eth3", "PC1:eth2"]
    - endpoints: ["SW3:eth3", "PC2:eth2"]

```


### Настройка маршрутизатора R1

На маршрутизаторе `R1` созданы VLAN 10 и VLAN 20, каждый для своего сегмента (PC1 и PC2). Для обоих VLAN настроены DHCP-сервера, которые выдают IP-адреса из заданных диапазонов. Также добавлен новый пользователь и изменено имя устройства.


Команды для настройки `R1`:
```

/interface vlan
add name=vlan10 vlan-id=10 interface=ether2
add name=vlan20 vlan-id=20 interface=ether2

/ip address
add address=10.10.10.1/24 interface=vlan10
add address=10.10.20.1/24 interface=vlan20

/ip pool
add name=dhcp_vlan10 ranges=10.10.10.100-10.10.10.200
add name=dhcp_vlan20 ranges=10.10.20.100-10.10.20.200

/ip dhcp-server
add name=dhcp10 interface=vlan10 address-pool=dhcp_vlan10 disabled=no
add name=dhcp20 interface=vlan20 address-pool=dhcp_vlan20 disabled=no

/ip dhcp-server network
add address=10.10.10.0/24 gateway=10.10.10.1 dns-server=8.8.8.8
add address=10.10.20.0/24 gateway=10.10.20.1 dns-server=8.8.8.8

/ip firewall filter
add chain=forward action=accept

```


### Настройка коммутатора SW1

На `SW1` добавлены VLAN-интерфейсы для разных портов и созданы мосты для VLAN 10 и VLAN 20, обеспечивающие передачу трафика между соответствующими интерфейсами. Для каждого VLAN настроен клиент DHCP.



Команды для настройки `SW1`:
```

# Настройка trunk порта к R1
/interface ethernet
set ether2 name=trunk-to-R1

/interface vlan
add name=vlan10-trunk vlan-id=10 interface=trunk-to-R1
add name=vlan20-trunk vlan-id=20 interface=trunk-to-R1

/interface vlan
add name=vlan10-access vlan-id=10 interface=ether3
add name=vlan20-access vlan-id=20 interface=ether4

/interface bridge
add name=bridge-vlan10
/interface bridge port
add interface=vlan10-trunk bridge=bridge-vlan10
add interface=vlan10-access bridge=bridge-vlan10

/interface bridge
add name=bridge-vlan20
/interface bridge port
add interface=vlan20-trunk bridge=bridge-vlan20
add interface=vlan20-access bridge=bridge-vlan20

```

### Настройка SW2 и SW3

Конфигурация аналогична.





### Настройка ПК

`PC1` и `PC2` настраиваются для работы в своих VLAN, создаются подинтерфейсы и задаются IP-адреса.

Пример для `PC1`:


Пример настройки `PC1`:

```
#!/bin/sh
sleep 15

ip link add link eth2 name vlan10 type vlan id 10
ip link set vlan10 up

udhcpc -i vlan10 -t 5 -n -q

echo "PC1 (VLAN 10) настроен"
echo "IP адрес:"
ip addr show vlan10 | grep "inet "

while true; do sleep 3600; done

```

Пример настройки `PC2`:
```
#!/bin/sh

ip link add link eth2 name vlan20 type vlan id 20
ip link set vlan20 up

udhcpc -i vlan20 -t 5 -n -q

while true; do sleep 3600; done
```

### Пример работы

После настройки всех устройств, были выполнены тесты на подключение и маршрутизацию, а также тест с помощью команды `ping`.  В частности, `PC1` и `PC2` успешно получили IP-адреса через DHCP, что подтверждается скриншотом ниже:


![Image alt](https://github.com/Golovastik777/devops_practice1/raw/main/test.png)

## Заключение
В результате выполнения лабораторной работы была успешно настроена трехуровневая сеть с VLAN-ами и DHCP-серверами. Все устройства функционируют корректно, и конечные устройства получают IP-адреса согласно настройкам DHCP.
