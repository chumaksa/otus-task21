# otus-task21

# OSPF

1. Развернуть 3 виртуальные машины
2. Объединить их разными vlan
	* настроить OSPF между машинами на базе Quagga;
	* изобразить ассиметричный роутинг;
	* сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.

## 1. Развернуть 3 виртуальные машины

### Решение

Виртуальные машины будем разворачивать с помощью vagrant. \
В качестве ОС для VM будем использовать Ubuntu 20.04.6 LTS. \
Итоговый Vagrantfile выглядит следущим образом:
```

# -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :router1 => {
        :box_name => "ubuntu/focal64",
        :vm_name => "router1",
        :net => [
                   {ip: '10.0.10.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"},
                   {ip: '10.0.12.1', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"},
                   {ip: '192.168.10.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net1"},
                   {ip: '192.168.56.10', adapter: 5},
                ]
  },

  :router2 => {
        :box_name => "ubuntu/focal64",
        :vm_name => "router2",
        :net => [
                   {ip: '10.0.10.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r1-r2"},
                   {ip: '10.0.11.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"},
                   {ip: '192.168.20.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net2"},
                   {ip: '192.168.56.11', adapter: 5},
                ]
  },

  :router3 => {
        :box_name => "ubuntu/focal64",
        :vm_name => "router3",
        :net => [
                   {ip: '10.0.11.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "r2-r3"},
                   {ip: '10.0.12.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "r1-r3"},
                   {ip: '192.168.30.1', adapter: 4, netmask: "255.255.255.0", virtualbox__intnet: "net3"},
                   {ip: '192.168.56.12', adapter: 5},
                ]
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|
    
    config.vm.define boxname do |box|
   
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]

      boxconfig[:net].each do |ipconf|
        box.vm.network :"private_network",
        		ip: ipconf[:ip],
        		adater: ipconf[:adapter],
        		netmask: ipconf[:netmask],
        		virtualbox__intnet: ipconf[:virtualbox__intnet]
      end

     end
  end
end
```

Запускаем наши VM с помощью vagrant up. \
Проверяем статус виртуальных машин.
```

chumaksa@debpc:~/otus/otus-task21$ vagrant status
Current machine states:

router1                   running (virtualbox)
router2                   running (virtualbox)
router3                   running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`
```
Видим, что наши виртуалки успешно запущены. 

## 2. Объединить их разными vlan

Настраиваются виртуальные машины по одному принципу. \
Опишем настройку одной VM, остальные сделаем аналогично. \

Начнём настройку первой виртуальной машины (router1). \
Выполним установку пакета frr.
```

vagrant@router1:~$ sudo apt install frr
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  frr-pythontools libc-ares2 libyang0.16
Suggested packages:
  frr-doc
The following NEW packages will be installed:
  frr frr-pythontools libc-ares2 libyang0.16
0 upgraded, 4 newly installed, 0 to remove and 22 not upgraded.
Need to get 2796 kB of archives.
After this operation, 10.4 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://archive.ubuntu.com/ubuntu focal-updates/main amd64 libc-ares2 amd64 1.15.0-1ubuntu0.4 [36.9 kB]
Get:2 http://archive.ubuntu.com/ubuntu focal/universe amd64 libyang0.16 amd64 0.16.105-3build1 [391 kB]
Get:3 http://archive.ubuntu.com/ubuntu focal-updates/universe amd64 frr amd64 7.2.1-1ubuntu0.2 [2350 kB]
Get:4 http://archive.ubuntu.com/ubuntu focal-updates/universe amd64 frr-pythontools all 7.2.1-1ubuntu0.2 [17.6 kB]
Fetched 2796 kB in 2s (1525 kB/s)     
Selecting previously unselected package libc-ares2:amd64.
(Reading database ... 63756 files and directories currently installed.)
Preparing to unpack .../libc-ares2_1.15.0-1ubuntu0.4_amd64.deb ...
Unpacking libc-ares2:amd64 (1.15.0-1ubuntu0.4) ...
Selecting previously unselected package libyang0.16.
Preparing to unpack .../libyang0.16_0.16.105-3build1_amd64.deb ...
Unpacking libyang0.16 (0.16.105-3build1) ...
Selecting previously unselected package frr.
Preparing to unpack .../frr_7.2.1-1ubuntu0.2_amd64.deb ...
Unpacking frr (7.2.1-1ubuntu0.2) ...
Selecting previously unselected package frr-pythontools.
Preparing to unpack .../frr-pythontools_7.2.1-1ubuntu0.2_all.deb ...
Unpacking frr-pythontools (7.2.1-1ubuntu0.2) ...
Setting up libc-ares2:amd64 (1.15.0-1ubuntu0.4) ...
Setting up libyang0.16 (0.16.105-3build1) ...
Setting up frr (7.2.1-1ubuntu0.2) ...
Adding group `frrvty' (GID 121) ...
Done.
Adding group `frr' (GID 122) ...
Done.
Warning: The home dir /nonexistent you specified can't be accessed: No such file or directory
Adding system user `frr' (UID 113) ...
Adding new user `frr' (UID 113) with group `frr' ...
Not creating home directory `/nonexistent'.
Created symlink /etc/systemd/system/multi-user.target.wants/frr.service → /lib/systemd/system/frr.service.
Setting up frr-pythontools (7.2.1-1ubuntu0.2) ...
Processing triggers for rsyslog (8.2001.0-1ubuntu1.3) ...
Processing triggers for systemd (245.4-4ubuntu3.22) ...
Processing triggers for man-db (2.9.1-1) ...
Processing triggers for libc-bin (2.31-0ubuntu9.14) ...
```

По умолчанию маршрутизация транзитных пакетов в ядре отключена. Проверим её состояние и включим. 
```

vagrant@router1:~$ sysctl net.ipv4.conf.all.forwarding 
net.ipv4.conf.all.forwarding = 0
vagrant@router1:~$ sudo sysctl net.ipv4.conf.all.forwarding=1 
net.ipv4.conf.all.forwarding = 1
vagrant@router1:~$ sysctl net.ipv4.conf.all.forwarding 
net.ipv4.conf.all.forwarding = 1
```

После установки необходимо отредактировать конфигурационный файл /etc/frr/daemons. \
В этом файле нам необходимо включить протокол маршрутизации ospf. Итоговый файл:
```

vagrant@router1:~$ sudo cat /etc/frr/daemons
# This file tells the frr package which daemons to start.
#
# Sample configurations for these daemons can be found in
# /usr/share/doc/frr/examples/.
#
# ATTENTION:
#
# When activating a daemon for the first time, a config file, even if it is
# empty, has to be present *and* be owned by the user and group "frr", else
# the daemon will not be started by /etc/init.d/frr. The permissions should
# be u=rw,g=r,o=.
# When using "vtysh" such a config file is also needed. It should be owned by
# group "frrvty" and set to ug=rw,o= though. Check /etc/pam.d/frr, too.
#
# The watchfrr and zebra daemons are always started.
#
bgpd=no
ospfd=yes
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no

#
# If this option is set the /etc/init.d/frr script automatically loads
# the config via "vtysh -b" when the servers are started.
# Check /etc/pam.d/frr if you intend to use "vtysh"!
#
vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
ospfd_options="  -A 127.0.0.1"
ospf6d_options=" -A ::1"
ripd_options="   -A 127.0.0.1"
ripngd_options=" -A ::1"
isisd_options="  -A 127.0.0.1"
pimd_options="   -A 127.0.0.1"
ldpd_options="   -A 127.0.0.1"
nhrpd_options="  -A 127.0.0.1"
eigrpd_options=" -A 127.0.0.1"
babeld_options=" -A 127.0.0.1"
sharpd_options=" -A 127.0.0.1"
pbrd_options="   -A 127.0.0.1"
staticd_options="-A 127.0.0.1"
bfdd_options="   -A 127.0.0.1"
fabricd_options="-A 127.0.0.1"
vrrpd_options="  -A 127.0.0.1"

#
# This is the maximum number of FD's that will be available.
# Upon startup this is read by the control files and ulimit
# is called.  Uncomment and use a reasonable value for your
# setup if you are expecting a large number of peers in
# say BGP.
#MAX_FDS=1024

# The list of daemons to watch is automatically generated by the init script.
#watchfrr_options=""

# for debugging purposes, you can specify a "wrap" command to start instead
# of starting the daemon directly, e.g. to use valgrind on ospfd:
#   ospfd_wrap="/usr/bin/valgrind"
# or you can use "all_wrap" for all daemons, e.g. to use perf record:
#   all_wrap="/usr/bin/perf record --call-graph -"
# the normal daemon command is added to this at the end.
```

После внесения изменений необходимо выполнить перезапуск frr. 
```

vagrant@router1:~$ sudo systemctl restart frr.service 
vagrant@router1:~$ sudo systemctl status frr.service 
● frr.service - FRRouting
     Loaded: loaded (/lib/systemd/system/frr.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2024-01-13 16:09:53 UTC; 6s ago
       Docs: https://frrouting.readthedocs.io/en/latest/setup.html
    Process: 8059 ExecStart=/usr/lib/frr/frrinit.sh start (code=exited, status=0/SUCCESS)
      Tasks: 8 (limit: 1117)
     Memory: 9.8M
     CGroup: /system.slice/frr.service
             ├─8080 /usr/lib/frr/watchfrr -d zebra ospfd staticd
             ├─8096 /usr/lib/frr/zebra -d -A 127.0.0.1 -s 90000000
             ├─8101 /usr/lib/frr/ospfd -d -A 127.0.0.1
             └─8105 /usr/lib/frr/staticd -d -A 127.0.0.1
```

Настройку будем выполнять с помощью vtysh.
```

vagrant@router1:~$ sudo vtysh 

Hello, this is FRRouting (version 7.2.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# configure terminal 
router1(config)# router ospf
router1(config-router)# network 10.0.10.0/24 area 0
router1(config-router)# network 10.0.12.0/24 area 0
router1(config-router)# network 192.168.10.0/24 area 0
router1(config-router)# neighbor 10.0.10.2
router1(config-router)# neighbor 10.0.12.2
```

Повторяем все настройки на остальных вирталках, учитывая актуальные ip. \
Выполним проверку доступности всех интерфейсов router3 и router2 с router1.
```

vagrant@router1:~$ ping 10.0.12.2 -c 2
PING 10.0.12.2 (10.0.12.2) 56(84) bytes of data.
64 bytes from 10.0.12.2: icmp_seq=1 ttl=64 time=0.756 ms
64 bytes from 10.0.12.2: icmp_seq=2 ttl=64 time=1.12 ms

--- 10.0.12.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1004ms
rtt min/avg/max/mdev = 0.756/0.937/1.119/0.181 ms
vagrant@router1:~$ ping 10.0.11.1 -c 2
PING 10.0.11.1 (10.0.11.1) 56(84) bytes of data.
64 bytes from 10.0.11.1: icmp_seq=1 ttl=64 time=0.534 ms
64 bytes from 10.0.11.1: icmp_seq=2 ttl=64 time=1.07 ms

--- 10.0.11.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1023ms
rtt min/avg/max/mdev = 0.534/0.800/1.067/0.266 ms
vagrant@router1:~$ ping 192.168.30.1 -c 2
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=1.09 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=1.01 ms

--- 192.168.30.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.007/1.047/1.087/0.040 ms
vagrant@router1:~$ ping 10.0.11.2 -c 2
PING 10.0.11.2 (10.0.11.2) 56(84) bytes of data.
64 bytes from 10.0.11.2: icmp_seq=1 ttl=64 time=0.439 ms
64 bytes from 10.0.11.2: icmp_seq=2 ttl=64 time=1.11 ms

--- 10.0.11.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1058ms
rtt min/avg/max/mdev = 0.439/0.774/1.110/0.335 ms
vagrant@router1:~$ ping 10.0.10.2 -c 2
PING 10.0.10.2 (10.0.10.2) 56(84) bytes of data.
64 bytes from 10.0.10.2: icmp_seq=1 ttl=64 time=0.363 ms
64 bytes from 10.0.10.2: icmp_seq=2 ttl=64 time=1.04 ms

--- 10.0.10.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1012ms
rtt min/avg/max/mdev = 0.363/0.702/1.041/0.339 ms
vagrant@router1:~$ ping 192.168.20.1 -c 2
PING 192.168.20.1 (192.168.20.1) 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.430 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=1.03 ms

--- 192.168.20.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1005ms
rtt min/avg/max/mdev = 0.430/0.730/1.031/0.300 ms
```

Видим, что все интерфейсы доступны. Маршрутизация настроена и работает правильно. \
Проверим, что будет если отключить один из интерфейсов. \
Для начала посмотрим маршрут до 192.168.30.1 с router1
```

vagrant@router1:~$ traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  192.168.30.1 (192.168.30.1)  0.971 ms  1.237 ms  1.433 ms
```

Теперь выключим интерфейс и проверим ещё раз.
```

vagrant@router1:~$ sudo ip link set enp0s9 down
vagrant@router1:~$ traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  0.775 ms  0.747 ms  0.740 ms
 2  192.168.30.1 (192.168.30.1)  1.031 ms  0.936 ms  0.704 ms
```

Видим, что маршрут поменялся. Динамическая маршрутизация работает.

### 2.2 Настройка ассиметричного роутинга

По умолчанию в ядре включена блокировка ассиметричной маршрутизации. \
Для выполнения задания нам нужно выключить эту блокировку.
```

vagrant@router1:~$ sysctl net.ipv4.conf.all.rp_filter
net.ipv4.conf.all.rp_filter = 2
vagrant@router1:~$ sudo sysctl net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.all.rp_filter = 0
```

Далее, выбираем один из роутеров, на котором изменим «стоимость интерфейса». \
Например, поменяем стоимость интерфейса enp0s8 на router1:
```

vagrant@router1:~$ sudo vtysh

Hello, this is FRRouting (version 7.2.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router1# show ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, 00:37:29
O>* 10.0.11.0/30 [110/200] via 10.0.10.2, enp0s8, 00:01:56
  *                        via 10.0.12.2, enp0s9, 00:01:56
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, 00:01:56
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, 00:37:39
O>* 192.168.20.0/24 [110/200] via 10.0.10.2, enp0s8, 00:37:15
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, 00:01:56
router1# configure terminal 
router1(config)# interface enp0s8
router1(config-if)# ip ospf cost 1000
router1(config-if)# exit
router1(config)# exit
router1# show ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O   10.0.10.0/30 [110/300] via 10.0.12.2, enp0s9, 00:00:30
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, 00:00:30
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, 00:03:36
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, 00:39:19
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, 00:00:30
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, 00:03:36
```

Видим, что после изменения стоимости маршрут к сети 192.168.20.0/24 изменился и теперь идёт через enp0s9. \
Т.е. через router3. Если зайти на router2 и посмотреть маршрут к сети 192.168.10.0/24, то увидим прямой маршрут к router1. 
```

vagrant@router2:~$ sudo vtysh 

Hello, this is FRRouting (version 7.2.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# show ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, 00:47:16
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, 00:46:25
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, enp0s8, 00:11:43
  *                        via 10.0.11.1, enp0s9, 00:11:43
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, 00:47:06
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, 00:47:15
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, 00:46:15
router2# exit
vagrant@router2:~$ ip -c a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:7f:55:95:07:8e brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 74752sec preferred_lft 74752sec
    inet6 fe80::7f:55ff:fe95:78e/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:71:67:6e brd ff:ff:ff:ff:ff:ff
    inet 10.0.10.2/30 brd 10.0.10.3 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe71:676e/64 scope link 
       valid_lft forever preferred_lft forever
4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:81:1f:23 brd ff:ff:ff:ff:ff:ff
    inet 10.0.11.2/30 brd 10.0.11.3 scope global enp0s9
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe81:1f23/64 scope link 
       valid_lft forever preferred_lft forever
5: enp0s10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:76:f2:0f brd ff:ff:ff:ff:ff:ff
    inet 192.168.20.1/24 brd 192.168.20.255 scope global enp0s10
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe76:f20f/64 scope link 
       valid_lft forever preferred_lft forever
6: enp0s16: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:fc:25:d6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.11/24 brd 192.168.56.255 scope global enp0s16
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fefc:25d6/64 scope link 
       valid_lft forever preferred_lft forever
```

Проверим это с помощью tcpdump. \
На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1:
```

vagrant@router1:~$ ping -I 192.168.10.1 192.168.20.1
PING 192.168.20.1 (192.168.20.1) from 192.168.10.1 : 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=1.54 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=1.69 ms
```

На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s9:
```

vagrant@router2:~$ sudo tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
07:51:19.246114 IP 192.168.10.1 > router2: ICMP echo request, id 43, seq 1, length 64
07:51:20.327404 IP 192.168.10.1 > router2: ICMP echo request, id 43, seq 2, length 64
07:51:20.811486 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
07:51:21.331215 IP 192.168.10.1 > router2: ICMP echo request, id 43, seq 3, length 64
07:51:22.334438 IP 192.168.10.1 > router2: ICMP echo request, id 43, seq 4, length 64
07:51:23.337834 IP 192.168.10.1 > router2: ICMP echo request, id 43, seq 5, length 64
07:51:24.467896 IP 192.168.10.1 > router2: ICMP echo request, id 43, seq 6, length 64
07:51:25.096468 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
```

Видим что данный порт только получает ICMP-трафик с адреса 192.168.10.1. \

На router2 запускаем tcpdump, который будет смотреть трафик только на порту enp0s8:
```

vagrant@router2:~$ sudo tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
07:52:27.042440 IP 10.0.10.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
07:52:30.905326 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
07:52:31.267251 IP router2 > 192.168.10.1: ICMP echo reply, id 44, seq 1, length 64
07:52:32.269344 IP router2 > 192.168.10.1: ICMP echo reply, id 44, seq 2, length 64
07:52:33.271475 IP router2 > 192.168.10.1: ICMP echo reply, id 44, seq 3, length 64
07:52:34.276003 IP router2 > 192.168.10.1: ICMP echo reply, id 44, seq 4, length 64
07:52:35.279629 IP router2 > 192.168.10.1: ICMP echo reply, id 44, seq 5, length 64
07:52:36.281445 IP router2 > 192.168.10.1: ICMP echo reply, id 44, seq 6, length 64
07:52:36.410704 ARP, Request who-has 10.0.10.1 tell router2, length 28
07:52:36.411791 ARP, Reply 10.0.10.1 is-at 08:00:27:62:50:2a (oui Unknown), length 46
```

Видим, что порт enp0s8 на router2 только отправляет трафик. \
Таким образом трафик с router1 на router2 поступает по одному маршруту, а уходит по другому. \
Ассиметричный трафик работает.

### 2.3 Настройка симметичного роутинга

В предыдущем задании мы уже добавили один "дорогой" маршрут. \
Чтобы выполнить это задание нам нужно сделать ещё один маршрут "дорогим" и \
исправить ассиметричную маршрутизацию в симметричную. \
Для этого сделаем стоимость маршрута на router2 через enp0s8 "дорогим" и поставим "стоимость" 1000:
```

vagrant@router2:~$ sudo vtysh 

Hello, this is FRRouting (version 7.2.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

router2# show ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, 01:17:14
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, 01:16:23
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, enp0s8, 00:41:41
  *                        via 10.0.11.1, enp0s9, 00:41:41
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, 01:17:04
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, 01:17:13
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, 01:16:13
router2# configure terminal 
router2(config)# interface enp0s8
router2(config-if)# ip ospf cost 1000
router2(config-if)# exit
router2(config)# exit
router2# show ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O   10.0.10.0/30 [110/1000] is directly connected, enp0s8, 00:00:16
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, 01:17:20
O>* 10.0.12.0/30 [110/200] via 10.0.11.1, enp0s9, 00:00:16
O>* 192.168.10.0/24 [110/300] via 10.0.11.1, enp0s9, 00:00:16
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, 01:18:10
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, 01:17:10
```

Видим, что изначально маршрут к сети 192.168.10.0 был через enp0s8. \
После увеличения его стоимости, маршрут изменился на интерфейс enp0s9. \
Теперь трафик от router1 (сеть 192.168.10.0) к router2 (сеть 192.168.20.0) \
будет идти симметрично в обе стороны через router3. \
Проверим это с помощью tcpdump, способом описанным выше (в предыдущей задаче):
```

vagrant@router1:~$ ping -I 192.168.10.1 192.168.20.1
PING 192.168.20.1 (192.168.20.1) from 192.168.10.1 : 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=63 time=1.81 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=63 time=1.92 ms


vagrant@router2:~$ sudo tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
08:20:16.823023 IP 192.168.10.1 > router2: ICMP echo request, id 45, seq 23, length 64
08:20:16.823077 IP router2 > 192.168.10.1: ICMP echo reply, id 45, seq 23, length 64
08:20:17.825947 IP 192.168.10.1 > router2: ICMP echo request, id 45, seq 24, length 64
08:20:17.826004 IP router2 > 192.168.10.1: ICMP echo reply, id 45, seq 24, length 64
08:20:18.827203 IP 192.168.10.1 > router2: ICMP echo request, id 45, seq 25, length 64
08:20:18.827260 IP router2 > 192.168.10.1: ICMP echo reply, id 45, seq 25, length 64
08:20:19.829151 IP 192.168.10.1 > router2: ICMP echo request, id 45, seq 26, length 64
08:20:19.829208 IP router2 > 192.168.10.1: ICMP echo reply, id 45, seq 26, length 64
```

Видим, что трафик ходит в обе стороны по одному маршруту. \
Путём изменения стоимости мы получили симметричный роутинг.



