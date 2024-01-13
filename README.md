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



