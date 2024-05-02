# Задание: Сценарии iptables


Что нужно сделать?

реализовать knocking port
centralRouter может попасть на ssh inetrRouter через knock скрипт
пример в материалах.
добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост.
запустить nginx на centralServer.
пробросить 80й порт на inetRouter2 8080.
дефолт в инет оставить через inetRouter.

## Решение

С помощью Vagrant создаётся стенд, состоящий из следующих хостов:
1. router1
2. router2
3. router3

С помощью ansible выполняется конфигурация узлов:
```
- name: NetPlan
  hosts: all
  become: yes
  tasks:

# Отключим ufw везде
  - name: stop & disable ufw
    systemd_service:
      name: ufw
      state: stopped
      enabled: false

# Включим проброс пакетов на всех роутерах
  - name: 'allow forwarding'
    sysctl:
      name: net.ipv4.ip_forward
      value: 1
      sysctl_set: yes

# Установим telnet и quagga на всех хостах
  - name: install telnet & quagga
    apt:
      name: 
        - telnet
        - quagga
      state: present
      update_cache: yes

# Скопируем правила для quagga
  - name: Config quagga
    template: 
      src: "{{ item.src }}"
      dest: "{{ item.dest }}"
      owner: quagga
      group: quagga
      mode: "{{ item.mode }}"
    with_items:
      - { src: "templates/ospfd-{{ansible_hostname}}.conf", dest: "/etc/quagga/ospfd.conf", mode: "0640" }
      - { src: "templates/zebra-{{ansible_hostname}}.conf", dest: "/etc/quagga/zebra.conf", mode: "0640" }

# Включим ospfd
  - name: start and enable ospfd
    systemd:
      name: ospfd
      state: started
      enabled: yes

# Включим zebra
  - name: start and enable zebra
    systemd:
      name: zebra
      state: started
      enabled: yes
```




Проверим, как работает `Quagga` по протоколу OSPF.
Для этого на `router1` через оболочку `vtysh` посмотрим соседей по сети.
```bash
root@router1:~# vtysh

Hello, this is Quagga (version 1.2.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# sh ip ospf neighbor

Neighbor ID     Pri State           Dead Time Address         Interface            RXmtL RqstL DBsmL
127.0.0.2         1 Full/DR           35.438s 192.168.12.2    enp0s8:192.168.12.1      0     0     0
127.0.0.3         1 Full/DR           35.418s 192.168.31.1    enp0s9:192.168.31.2      0     0     0
router1# sh ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, P - PIM, A - Babel, N - NHRP,
       > - selected route, * - FIB route

K>* 0.0.0.0/0 via 10.0.2.2, enp0s3, src 10.0.2.15
O   10.0.2.0/24 [110/20] via 192.168.12.2, enp0s8, 00:02:17
C>* 10.0.2.0/24 is directly connected, enp0s3
K>* 10.0.2.2/32 is directly connected, enp0s3
C>* 127.0.0.0/8 is directly connected, lo
O   192.168.11.0/30 [110/20] via 192.168.12.2, enp0s8, 00:02:17
C>* 192.168.11.0/30 is directly connected, enp0s10
O   192.168.12.0/30 [110/10] is directly connected, enp0s8, 00:03:03
C>* 192.168.12.0/30 is directly connected, enp0s8
O>* 192.168.23.0/30 [110/1010] via 192.168.12.2, enp0s8, 00:02:18
O   192.168.31.0/30 [110/1000] is directly connected, enp0s9, 00:02:23
C>* 192.168.31.0/30 is directly connected, enp0s9
O>* 192.168.33.0/30 [110/20] via 192.168.31.1, enp0s9, 00:02:17
O   192.168.56.0/24 [110/20] via 192.168.12.2, enp0s8, 00:02:17
C>* 192.168.56.0/24 is directly connected, enp0s19
router1# 
```
Видим, ID, адреса и интерфайсы двух соседей.
Команда `sh ip route` показывает таблицу маршрутизации.


Проверим ассиметричный роутинг, посмотрим по какому маршруту пойдёт пакет с `router1` до 192.168.23.2 и по какому обратно:
```bash
root@router1:~# tracepath 192.168.23.2
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.12.2                                          0.394ms 
 1:  192.168.12.2                                          0.420ms 
 2:  192.168.23.2                                          0.570ms reached
     Resume: pmtu 1500 hops 2 back 1 
root@router1:~# 
```
Результат: по прямому маршруту пакет идёт через `router2`, а обратно - с `router3` сразу на `router1`.


Проверим симметричный роутинг, запустип пакет с `router3` до `router2`:
```bash
vagrant@router3:~$ tracepath 192.168.12.2
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.31.2                                          0.374ms 
 1:  192.168.31.2                                          0.358ms 
 2:  192.168.12.2                                          0.742ms reached
     Resume: pmtu 1500 hops 2 back 2 
vagrant@router3:~$ 
```
В оба направления пакет проходил через `router1`.